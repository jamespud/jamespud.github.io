---
title: "VSCode 远程 SSH 下 Copilot 调用 Claude 模型"
date: 2026-01-30 19:50:00 +0800
categories: [tool]
tags: [vscode, IDE]
---

# VSCode 远程 SSH 下 Copilot 调用 Claude 模型问题

## 现象
1.  本地环境开启代理后，VSCode Copilot 可正常显示并使用 Claude 模型。
2.  切换至**远程 SSH 环境**后，Claude 模型消失，或出现 `copilot Allow edits to sensitive files?The model wants to edit files outside of your workspace` 报错。
3.  直接在本地 `settings.json` 配置 `remote.extensionKind` 强制 Copilot 本地运行，会导致远程工作区路径无法识别，引发工作区异常。

## 思路
1.  将本地代理端口穿透到远程服务器，让远程环境也能使用代理。
2.  不在本地配置远程扩展相关参数，转而在**远程的 `settings.json`** 中完成代理和扩展配置。
3.  补充配置项确保远程优先使用本地代理配置，解决模型加载问题。

## 具体操作步骤
### 步骤1：注释本地 `settings.json(user)` 中的相关配置
打开本地的 `settings.json`，将原有代理和远程扩展相关配置注释，示例如下：
```json
// 注释以下内容
// "http.proxy":"http://127.0.0.1:1082",
// "remote.extensionKind": {
//     "GitHub.copilot": [ "ui" ],
//     "GitHub.copilot-chat": [ "ui" ]
// }
```

### 步骤2：修改 SSH 配置文件，实现端口转发

以本地代理端口为`1082`为例:

1.  打开 SSH 配置文件（通常路径为 `~/.ssh/config`）。
2.  在对应远程主机的配置中，添加端口转发配置，示例如下：
    ```
    Host 221
        HostName 远程服务器IP
        User root
        RemoteForward 1082 127.0.0.1:1082
    ```
3.  说明：`1082` 为本地代理端口，需根据自己的实际代理端口修改。

### 步骤3：配置远程的 `settings.json(remote)`
1.  在 VSCode 远程 SSH 连接成功后，按下 `Ctrl+,` 打开远程的设置界面。
2.  点击右上角 **打开设置(JSON)**，进入远程的 `settings.json` 文件。
3.  添加以下配置内容：
    ```json
    {
        "http.proxy": "http://127.0.0.1:1082",
        "http.proxyStrictSSL": false,
        "http.useLocalProxyConfiguration": true,
        "remote.extensionKind": {
            "pub.name": [
                "ui"
            ]
        }
    }
    ```
4.  关键补充：`http.useLocalProxyConfiguration": true` 用于确保远程环境优先使用本地代理配置。

### 步骤4：重启 VSCode 验证效果
重启 VSCode 并重新连接远程 SSH，此时 Copilot 中可正常显示 Claude 系列模型，且 Agent 模式能正常编辑，无工作区异常报错。

