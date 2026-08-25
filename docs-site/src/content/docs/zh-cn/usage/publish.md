---
title: 发布浏览器只读对话链接
description: 用 Sivtr 把本地 Agent 对话安全地发布成浏览器可打开的只读链接。
---

`sivtr publish` 可以把本地的一段 Agent 对话变成一个浏览器链接。查看者不需要安装 Sivtr，也不需要登录；你的电脑关机后，链接仍然可以打开。

它发布的是一次性的“快照”，不是实时共享。对话之后再发生变化，旧链接不会自动更新，需要重新生成一个新链接。

## 先记住三件事

1. PowerShell 里的 `@share_ready` 要加引号：`'@share_ready'`。
2. `publish` 只能发布同一个本地 Agent session 中连续的对话轮次。
3. 链接本身就是查看凭据。拿到完整链接的人都可以查看，不要把链接放到公开 issue 或不可信群聊中。

## 第一步：确认 CLI 版本

先检查当前终端实际使用的是不是包含 `publish` 的版本：

```powershell
sivtr --version
sivtr publish --help
```

如果看到：

```text
error: unrecognized subcommand 'publish'
```

说明当前 `sivtr.exe` 太旧。升级后，`publish --help` 应该能看到 `preview`、`create`、`list`、`link` 和 `revoke`。

## 第二步：准备一段要分享的对话

`publish` 的输入是 WorkSet。最常见的流程是先从一个 Codex session 搜索出需要的连续轮次，并保存为名字：

```powershell
sivtr search codex/<session-id> --save share_ready --refs
```

把 `<session-id>` 换成实际的 session ID。例如：

```powershell
sivtr search codex/abc123 --save share_ready --refs
```

保存成功后，WorkSet 名字就是 `share_ready`。在 PowerShell 中引用它时必须写成：

```powershell
'@share_ready'
```

如果已经有合适的 WorkSet，也可以使用它的名字；例如 `@review` 要写成 `'@review'`。

### 选择范围时的建议

尽量只选择真正需要分享的连续对话轮次。不要直接使用混合了多个 session 的 `@last`，否则发布时会被拒绝。也不要把终端日志、远程 workspace 或多个 Agent session 混在一起。

## 第三步：本地预览

`preview` 完全在本地运行，不会上传内容：

```powershell
sivtr publish preview '@share_ready' --format human
```

预览会告诉你：

- 标题和来源 provider；
- 将发布多少轮对话；
- 快照大小和有效期；
- 自动脱敏了多少项；
- 是否发现路径、邮箱、内网地址等风险提示。

预览内容时重点检查：

- 是否包含不想公开的 User 消息或 Assistant 回复；
- 是否包含文件路径、邮箱、内网地址；
- 是否有不应该出现的密钥或账号信息；
- 对话起止范围是否正确。

识别出的 token、私钥、Bearer 和 secret assignment 会自动替换成 `[REDACTED]`。路径、邮箱和内网地址默认只警告，不会擅自改写正常对话。

## 第四步：创建链接

确认预览没有问题后，创建一个 7 天有效的链接：

```powershell
sivtr publish create '@share_ready' --expires 7d --yes
```

有效期可以选择：

```text
1d    1 天
7d    7 天，默认值
30d   30 天
90d   90 天
```

没有永久链接选项。

如果风险报告里还有未自动处理的警告，非交互创建需要明确允许：

```powershell
sivtr publish create '@share_ready' --expires 7d --yes --allow-warnings
```

`create` 成功后，完整链接会输出到 stdout，方便复制。链接通常类似：

```text
https://share.hnnulwh.cn/s/7d_xxxxxxxxxxxxxxxxxxxxxx#k=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

其中 `#k=...` 是解密密钥。它位于 URL fragment，不会发送给服务器；但是浏览器地址栏中的完整链接持有者可以查看内容。

### 在 PowerShell 中保存链接

如果想把链接保存到变量而不立即打印：

```powershell
$link = sivtr publish create '@share_ready' --expires 7d --yes --allow-warnings
$link
```

不要把 `$link` 写入公开日志、issue 或聊天记录。

## 查看、查找和撤销链接

查看本机发布记录，但不显示完整解密链接：

```powershell
sivtr publish list
```

使用 JSON 输出：

```powershell
sivtr publish list --json
```

需要重新打印某个完整链接时，使用 publication ID：

```powershell
sivtr publish link 7d_xxxxxxxxxxxxxxxxxxxxxx
```

提前撤销：

```powershell
sivtr publish revoke 7d_xxxxxxxxxxxxxxxxxxxxxx --yes
```

撤销会立即让链接不可访问。重复撤销同一条本机已撤销记录可以安全执行。管理凭据只保存在本机的 `publication-state.db` 中；如果这个数据库丢失，v1 没有账号恢复或远程找回管理权的功能。

## `publish` 和 `share` 的区别

| 功能 | `publish` | `share` |
| --- | --- | --- |
| 结果 | 不可变的浏览器只读快照 | 实时 workspace mount |
| 查看者 | 不需要 Sivtr，不需要登录 | 通常需要 Sivtr/daemon 和授权 |
| 分享者是否需要在线 | 不需要 | 通常需要 |
| 内容变化 | 不会自动更新，需创建新链接 | 读取共享 workspace 的当前内容 |
| 服务端看到的内容 | 只有加密密文 | 按远程共享协议提供数据 |

## 当前 v1 不会发布什么

v1 只支持同一 provider、同一 session 中连续的本地 Agent 对话轮次，并只保留 User 和 Assistant 文本。以下内容会被拒绝、排除或不进入公开快照：

- Terminal 记录；
- remote/group 内容；
- 跨 session 或跨 provider 内容；
- ToolCall、ToolResult、Thinking、Skill；
- WorkSet、WorkRef、`cwd`、session path；
- provider 原始事件、附件和图片。

## 常见错误

### `unrecognized subcommand 'publish'`

当前终端使用的是旧版 `sivtr.exe`。执行：

```powershell
sivtr --version
Get-Command sivtr -All
```

确认 PATH 中实际使用的二进制已经升级，并重新打开一个 PowerShell 窗口。

### `failed to resolve publication source '@share_ready'`

本机没有名为 `share_ready` 的 WorkSet。先执行搜索并保存：

```powershell
sivtr search codex/<session-id> --save share_ready --refs
```

### `publication cannot mix agent sessions`

你选择的 WorkSet 包含多个 Agent session。缩小搜索范围，只选择一个 session 中连续的对话轮次，再重新保存 WorkSet。

### `non-interactive publish requires --yes`

脚本或重定向环境不是交互终端，创建时加上：

```powershell
--yes
```

### `non-interactive publish with warnings requires --allow-warnings`

预览仍有未自动处理的风险提示。请先看完预览；确认可以公开后，再显式加上：

```powershell
--allow-warnings
```

## 隐私和数据位置

本机的 `publication-state.db` 保存标题、来源摘要、期限、查看密钥和撤销凭据，但不保存公开快照明文。

服务器只保存加密后的 envelope 和撤销/过期所需的元数据。服务器不能从 URL 请求中得到 `#k=...` fragment，也不会生成标题预览或搜索引擎内容。

如果对话包含高敏感内容，建议使用更短的 `1d` 有效期，并在确认查看者完成阅读后主动撤销。

更多精确参数见 [CLI 参考](/zh-cn/reference/cli/)，配置说明见 [配置](/zh-cn/usage/configuration/)。
