---
title: "看不到 Chrome 内置 Gemini？地区判断机制与可逆开启方法"
description: "从 Google 官方范围和 Chromium 源码出发，解释 Chrome 内置 Gemini 的地区、语言与账号资格判断，并给出可逆的开启和排错方法。"
date: 2026-09-02 00:00:00 +0800
categories: [AI]
tags: [Chrome, Gemini, Glic, Chromium]
---

Chrome 更新之后，窗口右上角本应出现一个 Gemini 按钮。点击它，可以在不离开当前网页的情况下总结页面、比较多个标签页，或者直接询问当前内容。但在中国大陆，即使 Chrome 已经更新到最新版、`gemini.google.com` 也能打开，这个按钮仍可能完全不出现。

这并不只是一个前端入口被隐藏的问题。Chrome 会综合判断设备地区、当前网络位置、浏览器语言、登录账号与灰度资格。网上常见的“打开一个 flag 就行”或者“把 `is_glic_eligible` 改成 `true`”只覆盖了其中一部分，因此经常出现按钮短暂出现、重启后消失，或者面板打开后仍提示地区不可用的情况。

本文记录的是我根据 Google 官方说明、Chromium 当前源码和社区案例整理出的排查过程。它的目标不是承诺某个组合一定成功，而是把每一层限制分别找出来，并尽量用 Chrome 自带、可撤销的设置处理。

> Chrome 内置 Gemini 在 Chromium 源码中的项目名是 **Glic**。后文出现的 `glic` flag、源码目录和诊断字段，指的都是这项功能。
{: .prompt-tip }

## 先确认：网页 Gemini 与 Chrome 内置 Gemini 不是同一层功能

Google 对“Chrome 中的 Gemini”列出了独立的使用条件：用户需要位于受支持地区，使用 Chromebook Plus、Mac 或 Windows，安装最新版 Chrome，登录浏览器，并使用受支持的设备语言。无痕模式不可用；工作或学校账号还可能需要管理员放行。[^1]

截至 2026 年 9 月 2 日，Google 公布的成年用户支持列表包含香港、澳门、台湾、新加坡、日本、美国等地区，但没有中国大陆。与此同时，中文已经位于支持语言列表中。[^2] 因此，在当前版本下，单纯把 Chrome 界面从中文改成英文并不能解决大陆地区限制；真正影响资格的是地区信号，以及与账号相关的其他条件。

这也解释了一个常见现象：`gemini.google.com` 能打开，不等于 Chrome 一定会显示内置 Gemini。前者是网页应用的访问结果，后者还要经过 Chrome 自己的功能启用和资料资格判断。

## Chrome 到底检查了哪些条件

Chromium 的 `GlicEnabling` 实现把启用条件拆成了多项检查，其中包括：

- Glic 主功能是否被启用；
- 永久地区与当前会话地区是否落在允许列表；
- 浏览器 locale 是否受支持；
- 当前资料是否为正常用户资料，而不是无痕、访客或系统资料；
- Google 账号是否完整登录、是否具备相应账号能力；
- 是否被 Chrome 企业策略或远端管理策略禁止；
- 是否已进入服务端灰度，以及是否完成首次授权。[^3]

地区判断尤其容易被忽略。源码中的 `EvaluateCountryEnablement()` 同时接收 `permanent_country_code` 和 `session_country_code`。当 `GlicUseSessionCountryForFiltering` 生效时，Chrome 不仅看保存在本地的长期地区，还会检查当前会话地区。[^3]

可以把整个过程简化成三层：

| 层次 | Chrome 看到的信号 | 常见现象 |
| --- | --- | --- |
| 网络层 | 当前出口 IP 推导出的会话地区 | 面板提示当前位置不支持，断开线路后功能消失 |
| 浏览器层 | Variations 永久地区、locale、Glic flags | 设置中没有“AI 创新功能”，工具栏没有 Gemini 按钮 |
| 账号层 | 年龄、账号类型、服务端能力与灰度资格 | 地区和界面都正常，但仍显示账号不符合条件 |

本地设置只能处理前两层。账号能力与服务端灰度由 Google 返回，无法靠修改一个 JSON 字段可靠绕过。

## 一套侵入性较低的开启方法

下面的顺序有意把网络检查放在最前面。否则 Chrome 可能在启动时先取得中国大陆的会话地区，后面再修改本地地区也没有效果。

### 1. 先让网络出口落在受支持地区

使用符合所在地法规及所在单位规定的跨境网络连接，并选择 Google 官方列表内的地区，例如香港、新加坡、日本或美国。地区不必固定选美国；更重要的是网络出口与后面设置的 Chrome 地区保持一致。

连接后先确认以下页面能够正常访问：

```text
https://gemini.google.com/
```

首次开启内置 Gemini 时保持该连接，不要在 Chrome 启动完成后才切换。社区案例中，仅修改本地文件但保留不受支持的出口 IP，按钮可能出现，服务端仍会返回地区不匹配；关闭网络线路后，Chrome 也可能重新记录实际地区。[^4]

### 2. 更新 Chrome，并确认资料满足基础条件

打开：

```text
chrome://settings/help
```

更新到当前稳定版并重新启动。随后确认：

- Chrome 已登录个人 Google 账号；
- 当前不是无痕或访客窗口；
- Google 账号年龄信息完整；
- 如果是工作或学校账号，管理员已经启用相应服务。[^1]

中文已经属于支持语言，不需要为了当前版本强制改成英语。若后面的诊断结果明确显示 locale 不通过，再临时将 Chrome 显示语言改为 `English (United States)` 进行对照测试。

### 3. 使用 Chrome 自带页面覆盖 Variations 地区

打开：

```text
chrome://translate-internals/
```

在页面右侧找到 **Override Variations Country**，输入与当前网络出口一致的两位 ISO 国家或地区代码：

| 地区 | 代码 |
| --- | --- |
| 香港 | `hk` |
| 新加坡 | `sg` |
| 日本 | `jp` |
| 美国 | `us` |

点击 **Update** 后，彻底退出 Chrome，再重新打开。

虽然这个入口位于 `translate-internals`，它修改的不是翻译目标语言，而是 Chrome Variations 使用的永久地区覆盖值。Chromium 将该字段保存为 `variations_permanent_overridden_country`；源码说明它不会随 Chrome 更新自动变化，并明确将其定义为测试和开发用途。[^5]

这种方式比手动编辑 `%LOCALAPPDATA%\Google\Chrome\User Data\Local State` 安全：不用处理压缩的 Variations seed，也不容易因为 JSON 格式错误破坏整个浏览器资料。

### 4. 只打开必要的 Glic flag

如果 Chrome 设置中仍然没有“AI 创新功能”，打开：

```text
chrome://flags/#glic
```

将 **Glic** 设为 `Enabled`。如果当前 Chrome 版本还提供单独的侧栏实验项，并且 Gemini 按钮依然没有出现，再打开：

```text
chrome://flags/#glic-side-panel
```

将它设为 `Enabled`，然后重新启动 Chrome。`glic` 是整个功能的主开关；`glic-side-panel` 对应多实例侧栏实现。两者都能在 Chromium 的 flags 注册代码中找到。[^6]

不建议看到 `glic` 就把所有实验项全部打开。诸如 Actor、自动填充、预热、入口样式等 flag 控制的是后续实验能力，并不是基础资格条件；其中还有专门用于测试、会关闭安全检查的选项，与开启对话面板无关。

### 5. 在设置中完成首次授权

重新启动后，进入：

```text
Chrome 设置 → AI 创新功能 → Chrome 中的 Gemini
```

如果入口已经出现，按页面提示完成首次授权。工具栏右上角随后会出现 **Ask Gemini** 按钮。Google 提醒该功能仍在分批推出，因此满足公开条件也不代表账号一定立即获得资格。[^1]

## 按诊断字段排错

新版 Chromium 提供了 Glic 内部状态页面，用来展示参与资格计算的各项状态。先尝试：

```text
chrome://glic/internals
```

部分版本使用或暴露了另一个入口：

```text
chrome://glic-internals
```

Chromium 在 2026 年对该页面进行了扩充，加入了 rollout、locale、country 和资料启用状态等调试信息。[^7] 页面能打开时，可以按下面的字段定位：

| 字段或状态 | 含义 | 下一步 |
| --- | --- | --- |
| `disallowed_by_country_filter` | 永久地区或会话地区不在允许范围 | 检查出口 IP；确认 Variations 覆盖代码与出口一致；完全退出 Chrome 后重试 |
| `disallowed_by_locale_filter` | 当前浏览器 locale 未通过 | 将 Chrome 显示语言临时改为英语或其他官方支持语言，重启对照 |
| `primary_account_not_fully_signed_in` | 账号没有完整登录到 Chrome | 重新登录浏览器，而不只是登录某个 Google 网页 |
| `primary_account_not_capable` / `ineligible account` | 账号年龄、类型或服务端能力不符合 | 完成年龄验证；先用新的 Chrome 资料测试；单位账号联系管理员 |
| `not_rolled_out` | 账号尚未进入服务端灰度 | 本地没有稳定的强制办法，只能等待灰度或换符合条件的账号验证 |
| `disallowed_by_chrome_policy` | 本机 Chrome 策略禁止 | 在 `chrome://policy` 检查组织策略 |
| `disallowed_by_remote_admin` | Google Workspace 管理设置禁止 | 由单位或学校管理员调整 |
| `not_consented` | 尚未完成首次授权 | 从“AI 创新功能”页面重新进入授权流程 |

如果诊断页本身打不开，先确认 `chrome://flags/#glic` 已经启用，并检查 Chrome 版本。内部页面并不是面向普通用户的稳定接口，名称和展示字段可能随版本变化。

## 为什么不优先修改 `is_glic_eligible`

不少早期教程要求关闭 Chrome，然后在 `Local State` 中查找：

```json
"is_glic_eligible": false
```

再把它改成 `true`。这个办法的问题不只是字段可能不存在。当前 Chromium 的启用流程已经明确拆分为地区、locale、账号能力、远端策略、首次授权和 rollout 等状态；即便旧字段让按钮显示出来，服务端地区或账号检查仍然可以让面板不可用。[^3]

手动修改 `Local State` 还存在两个额外问题：Chrome 没有完全退出时，修改会被正在运行的进程覆盖；Chrome Sync 或新的 Variations seed 也可能更新其中的相关值。社区中“重启后失效”的案例大多发生在这一层。[^4]

因此更合理的顺序是：

1. 让当前会话地区符合要求；
2. 用 Chrome 自带的 Variations Country override 处理永久地区；
3. 仅在入口没有被创建时启用 Glic 主 flag；
4. 最后通过内部状态区分地区问题与账号问题。

这套顺序不能绕过 Google 的账号资格，但可以避免把所有失败都归因于“线路不行”或“Chrome 版本不对”。

## 如何恢复原状

所有修改都可以撤销：

1. 回到 `chrome://translate-internals/`，清除 **Override Variations Country**；
2. 在 `chrome://flags` 中把 `Glic` 与 `Glic Side Panel` 恢复为 `Default`；
3. 如果临时修改过 Chrome 或 Windows 的语言、地区，再恢复原设置；
4. 完全退出并重新启动 Chrome。

如果只是想使用 Gemini 对话，而不需要读取当前标签页、跨标签页比较或浏览器内操作，直接使用 Gemini 网页通常比维护这些内部状态简单。Chrome 内置 Gemini 的价值主要来自浏览器上下文集成，而这也意味着它的资格判断比普通网页应用更复杂。

## 参考资料

[^1]: Google Gemini Apps Help, *Use Gemini in Chrome*. <https://support.google.com/gemini/answer/16283624>
[^2]: Google Gemini Apps Help, *Chrome 中的 Gemini 的支持范围*. <https://support.google.com/gemini/answer/17140089?hl=zh-Hans>
[^3]: Chromium Source, `chrome/browser/glic/public/glic_enabling.cc`. <https://chromium.googlesource.com/chromium/src/+/main/chrome/browser/glic/public/glic_enabling.cc>
[^4]: r/chrome, *Gemini Side Panel / AI Innovations Missing in Chrome — Geofence/VPN Fix*, community reports, 2026. <https://www.reddit.com/r/chrome/comments/1rzjp25/fix_gemini_side_panel_ai_innovations_missing_in/>
[^5]: Chromium Source, `components/variations/pref_names.h` and `variations_service.cc`. <https://chromium.googlesource.com/chromium/src/+/main/components/variations/pref_names.h>；<https://chromium.googlesource.com/chromium/src/+/main/components/variations/service/variations_service.cc>
[^6]: Chromium Source, `chrome/browser/about_flags.cc`. <https://chromium.googlesource.com/chromium/src/+/main/chrome/browser/about_flags.cc>
[^7]: Chromium Commit `682f06c`, *Refactor glic internals and share webclient state creation*, 2026-06-18. <https://chromium.googlesource.com/chromium/src/+/682f06c50e6a62e16a8d0c2e4891e08204de1e09>