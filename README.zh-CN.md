# 钉钉本地机器人执行器 Skill

这是一个 Codex Skill，用于搭建“钉钉群消息触发本机脚本，并把生成文件发回钉钉群”的自动化流程。

典型链路：

```text
钉钉群消息
-> 钉钉 Stream 机器人或 dws 轮询
-> 本机运行 Python/脚本
-> 用 dws 上传生成文件
-> 用当前 dws 登录账号发送钉钉原生群文件消息
```

可复用 Skill 位于：

```text
dingtalk-local-bot-runner/
```

## 安装 Skill

将 Skill 文件夹复制到 Codex 的 skills 目录：

```powershell
Copy-Item -Recurse .\dingtalk-local-bot-runner "$env:USERPROFILE\.codex\skills\"
```

然后重启 Codex，或开启一个新对话。可以这样调用：

```text
使用 $dingtalk-local-bot-runner 创建一个钉钉 Stream 机器人，用于运行本机脚本并通过 dws 回传生成文件。
```

## 前置条件

### 1. 安装 dws

安装钉钉 Workspace CLI，也就是 `dws`。安装方式以你所在组织提供的方法为准。

验证：

```powershell
dws --help
dws auth status --format json
```

如果还没有登录：

```powershell
dws auth login
dws auth status --format json
dws profile list --format json
```

如果你的账号登录了多个组织，执行命令时要确认使用正确组织：

```powershell
dws profile list --format json
dws <command> --profile <corpId-or-profile-name> --format json
```

### 2. 钉钉应用权限

操作账号需要具备以下能力：

- 创建或管理钉钉开放平台应用；
- 配置应用的机器人能力；
- 发布应用版本，或在需要审批时选择审批人；
- 将机器人加入目标群；
- 向目标群会话空间上传文件；
- 用当前 dws 登录账号向群里发送消息。

### 3. 本机运行条件

运行本地脚本的电脑需要：

- 保持开机；
- 保持 `dws dev connect` 运行；
- 保持 dws 登录态有效；
- 安装脚本所需运行环境，例如 Python、依赖包、ODBC 驱动等；
- 能访问脚本所需的本地路径、数据库、网络盘、Excel COM 等资源。

如果脚本使用 Excel COM，建议在已登录的 Windows 桌面会话里运行，不建议直接作为无桌面服务运行。

## 钉钉配置流程概览

### 1. 查找目标群

```powershell
dws chat search --query "<群名称>" --format json
```

保存返回中的：

- `openConversationId`
- 群会话空间 ID，通常在 `extension.newCSpaceIdIM`

### 2. 创建 Stream 机器人

```powershell
dws dev app robot submit --name "<应用名称>" --robot-name "<机器人名称>" --desc "<机器人描述>" --dry-run --format json
dws dev app robot submit --name "<应用名称>" --robot-name "<机器人名称>" --desc "<机器人描述>" --yes --format json
dws dev app robot result --task-id <taskId> --format json
```

### 3. 确认 unifiedAppId

```powershell
dws dev app list --format json
dws dev app robot get --unified-app-id <unifiedAppId> --format json
```

如果创建结果提示缺少 `unifiedAppId`，不要用 AppKey 反查猜测，应从 `dws dev app list` 中选择明确匹配的新应用。

### 4. 发布应用版本

```powershell
dws dev app version create --unified-app-id <unifiedAppId> --desc "发布 Stream 机器人" --dry-run --format json
dws dev app version create --unified-app-id <unifiedAppId> --desc "发布 Stream 机器人" --yes --format json
dws dev app version check-approval --unified-app-id <unifiedAppId> --version-id <versionId> --format json
dws dev app version publish --unified-app-id <unifiedAppId> --version-id <versionId> --dry-run --format json
dws dev app version publish --unified-app-id <unifiedAppId> --version-id <versionId> --yes --format json
```

如果需要审批，根据 `check-approval` 返回的候选项选择审批人。

### 5. 将机器人加入群

```powershell
dws chat group members add-bot --id <openConversationId> --robot-code <robotCode> --format json
dws chat group bots --group <openConversationId> --format json
```

### 6. 启动 Stream

```powershell
dws dev connect --unified-app-id <unifiedAppId> --channel custom --agent-cmd <wrapper-command> --agent-workdir <project-root> --allowed-groups <openConversationId>
```

Windows 上建议使用 `.cmd` 包装脚本，避免路径空格、中文路径或 `py -m` 参数被拆错。

## 文件回传模式

Skill 文档中包含两种模式：

- `user_file`：把生成文件上传到群会话空间，然后用当前 dws 登录账号发送钉钉原生群文件消息。
- `bot_link`：上传文件后，让机器人发送 Markdown 链接。适合更在意“消息由机器人发出”的场景，但用户收到的是链接，不是原生文件卡片。

当前 `dws chat message send-by-bot` 没有原生文件附件参数，因此机器人可以发文件链接，但不能发和 `dws chat message send --msg-type file` 一样的原生文件卡片。

## 隐私与公开仓库注意事项

本仓库使用占位符，不包含真实业务信息。你在复用或二次发布时也应注意：

- 不要提交 `.env`；
- 不要提交 dws 日志、任务日志、生成报表、截图或凭证；
- 将真实群会话 ID、空间 ID、应用 ID、机器人编码、智能体 ID、用户 ID、应用密钥、Webhook 地址、访问令牌全部替换为占位符；
- 将本地个人路径替换为类似 `C:\path\to\...` 的通用路径。

## 校验 Skill

```powershell
py "$env:USERPROFILE\.codex\skills\.system\skill-creator\scripts\quick_validate.py" ".\dingtalk-local-bot-runner"
```

期望输出：

```text
Skill is valid!
```
