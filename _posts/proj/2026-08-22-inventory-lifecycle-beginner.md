---
title: 库存是怎么流转的？—— tiny-store 下单到付款的完整底层故事
date: 2026-08-22 12:00:00 +0800
categories: [tiny-store]
tags: [tiny-store, 库存, 电商, redis]
---

> 这篇文章**假设你完全不了解 tiny-store**，也不需要懂 Redis / Kafka / PostgreSQL / Java。
> 我们从"买家点下单"开始，一步一步讲清库存到底发生了什么，每一步为什么这么设计，
> 以及每个时间阈值、每个配置开关、每种"服务挂了"的情况。
>
> 基于 commit `d800f26`

---

## 0. 一分钟看懂这套系统

tiny-store 是一个电商卖货系统。卖货最怕两件事：

1. **超卖**：库存只有 100 件，却卖出去 101 件。钱收了，货发不出。
2. **少卖**：明明还有 90 件，顾客却买不到。白白损失生意。

为了对付这两个问题，系统用了"**两道门**"：

- **Redis 门（闸机）**：反应极快，专门负责回答"现在能不能买？"——高并发下先在这儿放行/拦住。
- **PostgreSQL 门（账本）**：慢但可靠，记录"到底卖出了多少、每个人占了多少"——是最终裁判。

整个库存流转，就是**先让顾客快速通过 Redis 闸机，再把这笔账慢慢记进 PostgreSQL 账本**的过程。

一句话概括下单到付款的库存旅程：

```
下单：Redis 秒扣（占用名额）
  ↓ 异步
落库：PostgreSQL 记一条"预留中"（PRE_DEDUCTED）
  ↓ 等待付款（最长 15 分钟）
付款成功：PostgreSQL 把"预留中"变成"已确认"（CONFIRMED），真正扣减库存
付款超时/取消：PostgreSQL 把"预留中"变成"已释放"（RELEASED）或"已过期"（EXPIRED），Redis 归还名额
```

---

## 1. 先认识几个"黑话"（零基础必备）

在看代码之前，先搞懂这些词。后面会反复用到。

| 术语 | 大白话解释 | 在本系统里的作用 |
|---|---|---|
| **SKU** | 一件具体可卖的商品（如"红色 M 码 T 恤"） | 库存按 SKU 计数 |
| **shopId** | 哪个店铺 | 同一 SKU 在不同店铺算不同库存 |
| **Redis** | 一种超快的内存数据库 | 秒级扣减，挡住高并发 |
| **PostgreSQL (DB)** | 一种慢但可靠的磁盘数据库 | 最终记账，权威来源 |
| **幂等（Idempotency）** | 同一个操作重复做，结果只算一次 | 防止重复下单/重复扣款 |
| **预扣（preDeduct）** | 先占住名额，但还没真扣 | 下单时只占坑，等到付款才真扣 |
| **确认（confirm）** | 付款成功后把占的坑真正扣掉 | 付款时的核心动作 |
| **释放（release）** | 订单取消了，把坑还回去 | 取消/超时的核心动作 |
| **过期（expire）** | 一直没付款，坑自动作废 | 定时任务兜底 |
| **Outbox 模式** | 先在自己数据库里写"待发送事件"，再异步发给别人 | 保证"订单数据"和"通知库存"不会丢掉 |
| **Kafka** | 一个消息管道，A 服务写消息，B 服务异步读 | 订单和库存之间的"邮递员" |
| **DLT（死信队列）** | 消息反复失败后的"垃圾收容所" | 失败消息进 DLT，等人来修 |
| **CAS（Compare-And-Swap）** | 先看版本号对不对，才允许改 | 防止两个人同时改同一份数据互相覆盖 |
| **TTL（过期时间）** | 数据自动消失的时间 | 临时数据自动清理 |
| **悲观锁（SELECT FOR UPDATE）** | 改之前先把行锁住，别人不能碰 | 防止 confirm 和 expire 互相打架 |
| **ZSet（有序集合）** | Redis 里按分数排序的集合 | 记录每个"未确认的预扣"，用于定位和超时清理 |

记不住没关系，读到哪就回来看哪。开始正文。

---

## 2. 数据放在哪儿？—— 两张"账本"

库存信息同时住在两套系统里。我们来回对比。

### 2.1 Redis 侧（快，只记"名额数字"）

对**每一个** (shopId, skuId)，Redis 里存了 4 个 key（可以理解成 4 个小格子）：

| key | 类型 | 代表什么 | 什么时候变 |
|---|---|---|---|
| `inventory:total:{shop}:{sku}` | 数字 | **库存上限**（最多允许卖出多少） | 商品进货/退货时 +，对账修复时 -。**永远不会直接 SET 覆盖** |
| `inventory:deducted:{shop}:{sku}` | 数字 | **已经放行出去的名额**（正在占坑的 + 已付款的） | 每次下单占坑 +1，取消/过期 -1。**付款确认时不减**（见下） |
| `inventory:uncommit:{shop}:{sku}` | 有序集合 | **还没确认付款的占坑明细**（一条一条的） | 占坑时加入，取消/过期/超时清理时移除 |
| `inventory:version:{shop}:{sku}` | 数字 | **版本号**（每次改都 +1） | 用于对账修复时做 CAS，防止误改 |

> **为什么付款确认时不减 `deducted`？**
> 这是本系统最重要的设计之一。想象一下：
> - `total = 100`（上限）
> - 预扣了 5 件（`deducted = 5`），这 5 件里有人付款、有人还没付
> - 付款确认时，如果 `deducted` 减成 0，那"已付款的 5 件"就不在占用里了
> - 但 `total` 也没减，于是系统会以为还能继续放行 100 件 → **超卖**
>
> 所以规则是：**`deducted` 永远记着"放出去的名额总数"（包括已付款的），`total` 是"能放行的上限"。**
> 只有真正扣库存时（确认），才去动 **PostgreSQL** 里的 `total_quantity`。

### 2.2 PostgreSQL 侧（慢，但最可信）

其实就 3 张表（都在 `tinystore_inventory` 这个 schema 下）：

#### `inventory_stock`（库存主表——每个 SKU 一行）

| 列 | 意思 |
|---|---|
| `shop_id` + `sku_id` | 唯一索引，表示"哪个店哪个商品" |
| `total_quantity` | **剩余可售库存**。确认时把它 -qty，进货/退货时 +delta。**有 CHECK >= 0 约束，永远不能为负** |
| `reserved_quantity` | 预留量（这个字段架构里基本不用，冻结由 Redis 管） |
| `version` | JPA 乐观锁版本号 |

#### `inventory_reservation`（预留记录表——一次预扣一行）

| 列 | 意思 |
|---|---|
| `reservation_id` | 主键，其实就是一个"占用凭证" |
| `shop_id` / `sku_id` / `quantity` | 占了多少 |
| `status` | 预留状态（见下） |
| `expire_at` | 过期时间（到期没付款就得释放） |
| `trade_id` / `operation_id` / `release_reason` / `confirmed_at` | 关联订单、幂等键、释放原因、确认时间 |
| `version` | JPA 乐观锁版本号 |

#### `inventory_reconcile_log`（对账审计日志——只增不改）

每次对账发现异常/修复，就写一行，把修复前后数字都记下来，方便排查。

### 2.3 预留状态机（reservation 的一生）

```
    ┌────────────────────────────────────────────┐
    │                PRE_DEDUCTED                │   ← 下单占坑成功（还没付款）
    └──────┬─────────────────┬─────────────────┬─┘
           │                 │                 │
           ▼                 ▼                 ▼
  ┌────────────┐   ┌────────────┐   ┌────────────┐
  │ CONFIRMED  │   │  RELEASED  │   │  EXPIRED   │
  │  (已付款)  │   │  (已取消)  │   │ (超时作废) │
  └────────────┘   └────────────┘   └────────────┘
```

- **PRE_DEDUCTED**：唯一"可继续"的状态。占坑成功，等付款。
- **CONFIRMED / RELEASED / EXPIRED**：都是**终态**，到了就不能再改。
  - CONFIRMED：付款成功，真扣库存
  - RELEASED：取消，归还名额
  - EXPIRED：过期，自动作废

> 只有系统的定时任务能写 EXPIRED，业务代码不能直接写 EXPIRED。

---

## 3. 下单 → 付款 的完整流程（一步一步）

我们把一个具体例子从头走到尾。假设库存 `total = 100`，顾客想买 1 件 SKU。

### 阶段 A：下单（这一步很快，只占坑）

#### 步骤 ① 买家点下单 → order 服务生成订单

order 服务（`tinystore-domain-order`）做几件事：

1. **幂等检查**：这个"下单请求"之前处理过吗？处理过就直接返回上次结果，不重做。幂等键在 Redis 里存 **600 秒（10 分钟）**。
2. 生成一个 `tradeId`（整个交易）和若干个 `orderId`（每个店铺一个子单）。
3. 拆出要买的商品，按**店铺**分组。

#### 步骤 ② 去 Redis 占坑（预扣）

order 服务通过 HTTP 调 inventory 服务的 `pre-deduct` 接口。

inventory 服务执行一段 **Lua 脚本**（Redis 里运行的原子小程序），一个动作做完 5 件事：

```lua
-- 1. 读当前名额
total = GET("inventory:total:shop:sku")      -- 100
deducted = GET("inventory:deducted:shop:sku") -- 0

-- 2. 判重 / 判库存：放行后不能超过上限
if (deducted + 1) > total then return "库存不足" end

-- 3. 记录放行 + 记一笔占坑明细
SET("inventory:deducted:shop:sku", deducted + 1)          -- deducted 变成 1
ZADD("inventory:uncommit:shop:sku", 时间戳, "订单号_时间_数量")  -- 记一条明细

-- 4. 版本号 +1
INCR("inventory:version:shop:sku")

-- 5. 返回一个"占坑凭证"（reservationId）
return "订单号_时间_1"
```

**为什么用 Lua？** 因为这 5 步必须是"一气呵成"的，不能有人在中间插队。如果分开做，两个顾客可能同时读到"还有 1 件"，都以为能买到，结果都放行 → 超卖。Lua 在 Redis 里是**单线程原子执行**的，天然避免这个问题。

> ⚠️ 注意：这一步**只改 Redis，不写 PostgreSQL**。数据库里此时还没有这笔预扣记录。

#### 步骤 ③ 写数据库 + 发出"通知"

这步在 order 服务的**一个事务**里完成（要么全成功，要么全失败）：

1. 写一条 **Outbox 事件**叫 `INVENTORY_RESERVE_DB`（内容是"某订单占了某 SKU 几件"）
2. 写 **Trade**（状态：未支付 `UNPAID`）
3. 写 **ShopOrder**（状态：待付款，inventoryStatus=预扣中）
4. 写 **PaymentIntent**（支付单，默认 15 分钟后过期）
5. 再写几条 Outbox 事件（交易创建、订单创建、支付单创建）

> **Outbox 是个什么设计？** 想象你在微信里发了条消息，如果先发消息（Kafka）再写数据库，消息可能发出去了但数据库写失败了 → 消息内容是假的。Outbox 的做法是：**先把自己数据库里要通知的事记下来，同一事务提交，然后再由一个定时器把"待通知"的记录发给 Kafka**。这样保证"数据库有订单"和"通知了库存"永远一致。

#### 步骤 ④ Outbox → Kafka → inventory 落库

一个**每 1 秒**跑一次的定时器（`OutboxEventPublisherScheduler`），从 Outbox 表捞"还没发出去"的事件，每次最多捞 **5000 条**，发给 Kafka。

Kafka 消息到达 inventory 服务后，inventory 再写数据库：

- 收到 `INVENTORY_RESERVE_DB` → 在 `inventory_reservation` 写一行 `PRE_DEDUCTED`
- 收到 `INVENTORY_CONFIRM` → 把 `PRE_DEDUCTED` 变 `CONFIRMED`
- 收到 `INVENTORY_RELEASE` → 把 `PRE_DEDUCTED` 变 `RELEASED`

**为什么下单时不直接写 DB？** 为了快。直接写 DB 会拖慢下单接口；先让顾客赶紧通过 Redis 扣减，DB 慢慢记账即可。

### 阶段 B：付款（这一步真正扣库存）

#### 步骤 ⑤ 等付款（最长 15 分钟）

从下单那一刻起，订单有 **15 分钟** 付款窗口。相关阈值：

| 阈值 | 默认值 | 含义 |
|---|---|---|
| `PaymentIntent.expireAt` | +15 分钟 | 支付单过期时间 |
| `inventory_reservation.expireAt` | +15 分钟 | 库存预扣过期时间 |
| `order.payment-timeout-seconds` | +15 分钟 | order 侧判断超时的阈值 |
| `inventory.reservation.expiry.default-minutes` | 15 | inventory 侧默认占坑时长 |

#### 步骤 ⑥ 支付成功回调

支付成功，支付服务回调 order 服务。order 服务：

1. **幂等检查**：用 `paymentId` 查，如果这单已处理过 `PAID`，直接返回，不重复扣。
2. **金额校验**：支付金额必须等于订单应付款。
3. **异步 promotion 门控**（仅当开启异步促销时）：确认优惠已锁定，否则不让付款。
4. 写 `INVENTORY_CONFIRM` Outbox 事件。
5. 更新 trade（已支付）、shopOrder（待发货）、paymentIntent（已支付）。

#### 步骤 ⑦ inventory 执行确认

Kafka 把 `INVENTORY_CONFIRM` 送到 inventory，inventory 做"确认"：

1. **Pass 1（先看）**：逐个看这条预留什么状态。
   - 已是 `RELEASED`/`EXPIRED`（没来得及付就取消/过期了）→ 冲突，返回冲突
   - 是 `PRE_DEDUCTED` 但已超过 `expireAt`（逻辑过期）→ 冲突
   - 已是 `CONFIRMED` → 幂等，跳过
2. **Pass 2a（锁住）**：`SELECT FOR UPDATE` 把这几行**锁住**，防止别人同时改。
3. **Pass 2b（改）**：
   - 把 `PRE_DEDUCTED` 改成 `CONFIRMED`
   - 把 `inventory_stock.total_quantity` **减掉数量**
   - **Redis 不动！**

> **为什么 confirm 用悲观锁（锁行）而不是乐观锁（版本号）？**
> 因为"付款确认"和"定时过期"是两拨人在抢同一行数据。如果乐观锁，抢失败的一方要重试、要补发事件、可能要人工处理，代价很高。直接锁行，让它们排队一个个来，简单又稳。

### 阶段 C：付不了款 / 取消（归还名额）

#### 情况 1：15 分钟没付 → 订单取消

一个**每 1 分钟**跑一次的定时器，扫出超时未付的订单，把支付单关掉，然后：

1. order 写 `INVENTORY_RELEASE` Outbox 事件
2. inventory 收到后：锁行 → `PRE_DEDUCTED` → `RELEASED`
3. 调 Redis 回滚：`deducted -= 数量`，删掉 uncommit 里的明细

#### 情况 2：库存过期（定时兜底）

inventory 有个**每 1 分钟**的定时器，专门扫"超过 `expire_at` 但还是 PRE_DEDUCTED"的记录：

1. 锁行 → `PRE_DEDUCTED` → `EXPIRED`
2. 调 Redis 回滚（同样归还名额）

#### 情况 3：用户主动取消

和情况 1 类似，只是触发原因是用户取消而不是超时。

---

## 4. "服务挂了"怎么办？（每类故障的兜底）

这是最重要的部分。系统里任何一环出问题，都要保证**不超卖**（宁可少卖）。

### 故障 1：下单时 Redis 挂了

- **发生了什么**：Lua 预扣执行不了。
- **结果**：下单直接失败，返回报错。订单不会生成，也不会占任何坑。
- **兜底**：幂等键防重复下单（同一个下单请求重试多次只算一次）。用户重试即可。
- **时延**：即时。
- **风险**：无。宁可拒售，不可超卖。

### 故障 2：Redis 占坑成功，但 order 事务挂了

- **发生了什么**：Redis 已经记了 `deducted+1`，可订单数据库没建成（比如 DB 挂了）。
- **结果**：Redis 里有个"幽灵占坑"，订单却不存在。
- **兜底**：
  1. order 立刻反过来调用"回滚"，把 Redis 占的坑还回去（尽力而为）。
  2. 如果回滚也失败，有个**每 1 分钟**的清理任务，扫描 Redis 里所有"未确认占坑明细"，超过 **30 分钟**没确认的自动清理，把 `deducted` 减回来。
- **时延**：回滚是即时；兜底清理最多延迟 30 分钟。
- **风险**：这 30 分钟里该 SKU 可售量偏低（少卖），但绝不超卖。

### 故障 3：Outbox / Kafka 消息丢了

- **发生了什么**：`INVENTORY_RESERVE_DB` 这个"通知"没送到 inventory，数据库没写预留行。
- **结果**：Redis 占了坑，PostgreSQL 没记录。以后付款时找不到这条预留 → 报"预留不存在"。
- **兜底**：
  1. Outbox 定时器每 1 秒重发"未发送"的事件。
  2. Kafka 消费失败最多尝试 **5 次**（失败后重试 4 次，退避 500ms→1s→2s→4s，上限 10s）。
  3. 还是失败 → 进 DLT（死信队列），需要人工补发。
  4. Redis 侧的幽灵占坑由 30 分钟清理兜底。
- **时延**：几秒到几分钟。
- **风险**：这是**最危险的口子**——如果消息彻底丢了，会出现"已付款但库存没确认"，必须靠人工 + DLT 监控。这也是为什么审计日志和监控很重要。

### 故障 4：付款确认遇到"预留已被取消/过期"

- **发生了什么**：顾客付了钱，但这条预留已经被取消或过期了（比如计时刚过）。
- **结果**：confirm 返回"冲突"，不会发"交易已支付"事件。
- **兜底**：悲观锁让确认和过期排队；逻辑过期检查提前拦截；paymentId 幂等防止重复扣。冲突时发"确认冲突"事件，人工处理。
- **时延**：即时。
- **风险**：已付款但库存已释放的订单，必须人工介入（退款或补库存）。

### 故障 5：付款时异步优惠"还没锁定"

- **发生了什么**：开启异步促销时，付款必须等优惠确认，否则拒付。
- **结果**：付款被拒。
- **兜底**：有个**每 10 秒**的定时器，对"待提交优惠超过 30 秒"的订单自动取消；回执幂等保证不会重复处理。
- **时延**：30 秒。
- **风险**：极窄窗口（刚锁定就被取消），由幂等逻辑兜底。

### 故障 6：付款后"确认"事件丢了

- **发生了什么**：付款已记录（订单已付），但 `INVENTORY_CONFIRM` 没送到 inventory。
- **结果**：数据库没确认、库存没真扣。
- **兜底**：同故障 3（Outbox 重发 + Kafka 重试 + DLT + 幂等）。
- **时延**：几秒到几分钟。
- **风险**：依赖 DLT 补发。

### 故障 7：取消/过期后，Redis 回滚失败

- **发生了什么**：数据库已经把预留改成了"已释放/已过期"，但 Redis 名额没还回去。
- **结果**：Redis 的 `deducted` 偏高（多占了），顾客买不到。
- **兜底**：
  1. **数据库终态优先，绝不回滚数据库**（这是铁律：数据库说了算）。
  2. 写一条"回滚失败"执行日志 + 告警。
  3. 对账任务每 10 分钟巡检：发现 `deducted` 偏高 → **只告警，不自动修复**（因为修复可能带来超卖风险）。
  4. 如果明细还在 Redis 的 uncommit 里，30 分钟清理任务会回收。
- **时延**：对账 10 分钟 / 清理 30 分钟。
- **风险**：**这是系统的软肋**——"多占"方向（漏卖）只能靠告警 + 人工，不会自动修。属于"宁可错杀，不可放过"的取舍。

### 故障 8：数据库宕机

- **发生了什么**：落库或确认时写不了 PostgreSQL。
- **结果**：落库失败 → 事件重试；确认失败 → 订单停在"已支付待确认"。
- **兜底**：Kafka 重试 + DLT；Redis 预扣独立于 DB（下单仍可用）；确认幂等重试；DB 恢复后自动重试。
- **时延**：取决于 DB 恢复速度。
- **风险**：DB 长时间宕机 → 事件堆积，Redis 幽灵占坑由 30 分钟清理兜底。

---

## 5. 所有定时任务一览（后台在跑什么）

系统依赖一堆"后台定时小工"来兜底。汇总如下：

| 定时任务 | 哪个模块 | 多久跑一次 | 干什么 | 关键阈值 |
|---|---|---|---|---|
| **InventoryReconcileJob** | inventory | 10 分钟 | 拿 Redis 和 DB 的数字对账，发现超卖风险就修 | 版本号 CAS 保证不会重复修 |
| **ReservationExpiryTask** | inventory | 1 分钟 | 找过期未付款的 `PRE_DEDUCTED`，改成 EXPIRED + 归还名额 | 过期 15 分钟 |
| **InventoryUncommitV2CleanupTask** | inventory | 1 分钟 | 扫 Redis 里超过 30 分钟还"未确认"的占坑，清理掉 | 超时 30 分钟 |
| **OrderTimeoutScheduler** | order | 1 分钟 | 找超时（15 分钟）未付订单，取消 | 超时 15 分钟 |
| **PendingCommitTimeoutScheduler** | order | 10 秒 | 异步优惠超时（30 秒）未锁定 → 自动取消 | 超时 30 秒 |
| **OutboxEventPublisherScheduler** | order | 1 秒 | 把"待发送"事件发给 Kafka | 每批 5000 |
| **OutboxCleanupScheduler** | order | 每天 03:00 | 清理 7 天前的老 Outbox 记录 | 保留 7 天 |

---

## 6. 配置参数速查表（改哪个影响什么）

### 6.1 Inventory 模块（`tinystore-domain-inventory/src/main/resources/application.yml`）

| 参数 | 默认值 | 影响什么 |
|---|---|---|
| `inventory.reconciliation.enabled` | true | 是否开对账 |
| `inventory.reconciliation.fixed-delay` | PT10M | 对账间隔 |
| `inventory.reservation.expiry.enabled` | true | 是否开过期清理 |
| `inventory.reservation.expiry.fixed-delay` | PT1M | 过期清理间隔 |
| `inventory.reservation.expiry.default-minutes` | 15 | 占坑默认时长（分钟） |
| `inventory.kafka.consumer.retry.max-attempts` | 5 | 消费失败重试几次 |
| `inventory.kafka.consumer.retry.backoff.initial-ms` | 500 | 首次退避（毫秒） |
| `inventory.kafka.consumer.retry.backoff.multiplier` | 2.0 | 退避倍率 |
| `inventory.kafka.consumer.retry.backoff.max-ms` | 10000 | 最大退避（毫秒） |
| `inventory.kafka.consumer.dlt.enabled` | true | 是否开死信队列 |
| Redis uncommit 清理超时 | 30 分钟（硬编码 `DEFAULT_TIMEOUT_MS`） | 幽灵占坑多久后清理 |

### 6.2 Order 模块（`tinystore-domain-order/src/main/resources/application.yml`）

| 参数 | 默认值 | 影响什么 |
|---|---|---|
| `order.idempotency.ttl-seconds` | 600 | 下单幂等键在 Redis 存多久（10 分钟） |
| `order.idempotency.fail-on-redis-error` | true | Redis 挂了时：true=严格拒绝（返回 503），false=放行（有重复风险） |
| `order.payment-timeout-seconds` | 900 | 支付超时（15 分钟） |
| `order.auto-receive-timeout-seconds` | 604800 | 自动确认收货（7 天） |
| `order.outbox.poll-interval` | 1000 | Outbox 轮询间隔（毫秒，1 秒） |
| `order.outbox.batch-size` | 5000 | Outbox 单批最大条数 |
| `order.scheduler.payment-timeout-interval` | 60000 | 支付超时检查间隔（毫秒，1 分钟） |
| `order.promotion.commit-async-enabled` | false | 是否用异步优惠提交 |
| `order.promotion.pending-timeout-seconds` | 30 | 异步优惠待提交超时 |
| `order.promotion.pending-timeout-batch-size` | 200 | 单批处理上限 |
| `order.promotion.pending-timeout-check-ms` | 10000 | 检查间隔（毫秒，10 秒） |
| Kafka topic | `tinystore.order.general` | 订单事件发到哪个 topic |

> 注意：`application-local.yml` 里的 `order.payment.timeout-seconds: 120` 和 `order.auto-receive.timeout-seconds: 60` 是**本地开发**用的短超时，别和生产搞混。

---

## 7. 一段完整的时间线（从 0 到 30 分钟）

```
t = 0s      买家下单，Redis 秒占坑（deducted+1，写 uncommit 明细）
t = 0~1s    Outbox 定时器把"落库事件"发给 Kafka
t = 1~5s    inventory 收到，写 PRE_DEDUCTED 到数据库
t = 0~900s  付款窗口（15 分钟）
              ├── 付款成功 → CONFIRMED（真正扣库存，Redis 不动）
              └── 一直没付
t = 900s     超时未付：OrderTimeoutScheduler 取消 → RELEASED + Redis 回滚
             或 ReservationExpiryTask 先到 → EXPIRED + Redis 回滚
             （两个兜底都卡在 15 分钟，谁先锁到行谁生效，另一个幂等跳过）
t = 600s     对账巡检（每 10 分钟一次）
t = 1800s    Redis 未确认占坑超过 30 分钟 → 自动清理兜底
```

看到关键点了吗：**正常情况下，占坑的生命很短**（要么付款确认，要么 15 分钟过期）。所有 30 分钟、10 分钟的定时任务，都是在**极端情况下兜底**的保险丝。

---

## 8. 一句话总结设计哲学

这套系统最重要的三条原则：

1. **先快后慢**：先用 Redis 快速放行/拦截，再异步把账记进 PostgreSQL。
2. **数据库是权威**：所有冲突都听数据库的，一旦数据库写成终态，Redis 失败也不能回滚数据库。
3. **宁少卖，不超卖**：任何不确定的情况，都选择"拒绝"而不是"放行"。超卖是不可接受的，少卖最多损失一点生意。

---

## 附：关键代码文件在哪

| 文件 | 干什么 |
|---|---|
| `tinystore-domain-inventory/.../scheduler/InventoryReconcileJob.java` | 对账定时任务 |
| `tinystore-domain-inventory/.../scheduler/ReservationExpiryTask.java` | 过期清理 |
| `tinystore-domain-inventory/.../scheduler/InventoryUncommitV2CleanupTask.java` | Redis 幽灵占坑清理 |
| `tinystore-domain-inventory/.../domain/service/InventoryReservationDomainService.java` | 预扣/确认/释放/过期的核心逻辑 |
| `tinystore-domain-inventory/.../util/InventoryRedisManager.java` | Redis 各自的 Lua 原子脚本 |
| `tinystore-domain-inventory/.../domain/value/ReconciliationSnapshot.java` | 对账公式 |
| `tinystore-domain-inventory/.../kafka/OrderEventConsumer.java` | Kafka 消息消费者 |
| `tinystore-domain-order/.../service/TradeApplicationService.java` | 下单 / 支付回写 |
| `tinystore-domain-order/.../scheduler/OrderTimeoutScheduler.java` | 支付超时取消 |
| `tinystore-domain-order/.../scheduler/PendingCommitTimeoutScheduler.java` | 异步优惠兜底 |
| `tinystore-domain-order/.../event/outbox/OutboxEventPublisherScheduler.java` | Outbox 轮询发布 |
