# DingTalk Stream Local Runner Playbook

## Architecture

Use this architecture when a DingTalk group command should run code on a specific PC:

```text
DingTalk group
  -> @Stream robot or group message
  -> dws dev connect Stream bridge
  -> local wrapper command
  -> whitelisted script runner
  -> output locator
  -> dws drive upload
  -> dws chat message send
```

Separate the concerns:

- DingTalk app/robot lifecycle: `dws dev app ...`
- Stream receive path: `dws dev connect`
- Local execution path: project code, config, logs
- File return path: `dws drive` then `dws chat message`

## Installation And Prerequisites

### 1. Install dws

Install the DingTalk Workspace CLI (`dws`) using the distribution method approved by your organization. This skill assumes `dws` is available on `PATH`.

Verify:

```powershell
dws --help
dws auth status --format json
```

If not authenticated, log in before doing any app, robot, chat, or drive operation:

```powershell
dws auth login
dws auth status --format json
dws profile list --format json
```

For multi-organization accounts, use the correct profile:

```powershell
dws profile list --format json
dws <command> --profile <corpId-or-profile-name> --format json
```

Do not hard-code app secrets, access tokens, or personal webhook URLs in public repositories.

### 2. Prepare DingTalk permissions

Before creating or publishing a bot, make sure the operator account can:

- Create or manage DingTalk developer applications.
- Configure robot capabilities for the app.
- Publish app versions, or select an approver if the organization requires approval.
- Add a bot to the target group, or ask a group owner/admin to do it.
- Upload files to the target group conversation space.
- Send group messages with the current dws-authenticated account.

### 3. Prepare the local machine

The local machine must:

- Stay powered on while Stream is expected to handle messages.
- Keep `dws dev connect` running.
- Keep dws authentication valid.
- Have the script runtime installed, for example Python plus required packages.
- Have access to any local files, databases, network drives, ODBC drivers, or Excel COM dependencies used by the script.

For Excel COM automation, run inside an interactive Windows user session. Avoid a headless service unless Excel automation has been explicitly hardened for that environment.

### 4. Public repository hygiene

Before publishing a reusable Skill or template:

- Replace real `openConversationId`, `spaceId`, `unifiedAppId`, `robotCode`, `agentId`, user IDs, webhook URLs, app keys, and secrets with placeholders.
- Replace local user paths with generic paths such as `C:\path\to\...`.
- Do not commit `.env`, logs, generated reports, screenshots containing credentials, or task history.
- Include examples that use placeholders only.

## Project Structure

Recommended Windows project layout:

```text
project-root/
  config/
    commands.yaml
  logs/
    tasks.jsonl
    stream.out.log
    stream.err.log
    stream.pid
  scripts/
    agent_entry.cmd
    start_stream.cmd
  src/
    agent_entry.py
    app.py
    command_registry.py
    config_loader.py
    dws_sender.py
    runner.py
```

`commands.yaml` should store:

```yaml
bot:
  test_group_name: <TARGET_GROUP_NAME>
  group_open_conversation_id: <OPEN_CONVERSATION_ID>
  group_space_id: "<GROUP_SPACE_ID>"
  unified_app_id: <UNIFIED_APP_ID>
  robot_code: <ROBOT_CODE>
  agent_id: "<AGENT_ID>"
  fail_reply: 指令识别失败，请尝试“@机器人 <TRIGGER_TEXT>”或联系管理员

delivery:
  mode: user_file
  send_file_title: <MESSAGE_TITLE>
  send_file_text: 文件已生成，请查收附件。

commands:
  - name: booking_apply
    triggers:
      - <TRIGGER_TEXT>
      - "@机器人 <TRIGGER_TEXT>"
    python_executable: C:\path\to\python.exe
    script: C:\path\to\your_script.py
    workdir: C:\path\to\script_workdir
    timeout_seconds: 900
    outputs:
      - C:\path\to\output\Report - {today_yyyyMMdd} *.xlsx
```

## App And Robot Lifecycle

New robot:

```powershell
dws dev app robot submit --name "舱位申请助手" --robot-name "舱位申请助手" --desc "接收群内舱位申请指令并运行本机脚本回传文件" --dry-run --format json
dws dev app robot submit --name "舱位申请助手" --robot-name "舱位申请助手" --desc "接收群内舱位申请指令并运行本机脚本回传文件" --yes --format json
dws dev app robot result --task-id <taskId> --format json
```

If `robot result` returns `BLOCKED_BY_MISSING_UNIFIED_APP_ID`, do not infer from `clientId`. Search the app list and select the exact app:

```powershell
dws dev app list --format json
```

Check robot config:

```powershell
dws dev app robot get --unified-app-id <unifiedAppId> --format json
```

Publish:

```powershell
dws dev app version create --unified-app-id <unifiedAppId> --desc "发布 Stream 机器人" --dry-run --format json
dws dev app version create --unified-app-id <unifiedAppId> --desc "发布 Stream 机器人" --yes --format json
dws dev app version check-approval --unified-app-id <unifiedAppId> --version-id <versionId> --format json
dws dev app version publish --unified-app-id <unifiedAppId> --version-id <versionId> --dry-run --format json
dws dev app version publish --unified-app-id <unifiedAppId> --version-id <versionId> --yes --format json
```

Add to group:

```powershell
dws chat group members add-bot --id <openConversationId> --robot-code <robotCode> --format json
dws chat group bots --group <openConversationId> --format json
```

## Stream Connect On Windows

Use wrappers to avoid command-line splitting bugs.

`scripts/agent_entry.cmd`:

```bat
@echo off
set PYTHONUTF8=1
py -X utf8 -m src.agent_entry %*
```

`scripts/start_stream.cmd`:

```bat
@echo off
cd /d "%~dp0.."
dws dev connect --unified-app-id "<unifiedAppId>" --channel custom --agent-cmd scripts\agent_entry.cmd --agent-workdir "%cd%" --allowed-groups "<openConversationId>"
```

Foreground:

```powershell
scripts\start_stream.cmd
```

Hidden background process:

```powershell
Start-Process -FilePath "cmd.exe" -ArgumentList "/c", "scripts\start_stream.cmd" -WorkingDirectory "<project-root>" -WindowStyle Hidden
```

Watch logs:

```powershell
Get-Content "<project-root>\logs\stream.out.log" -Tail 100 -Wait
Get-Content "<project-root>\logs\tasks.jsonl" -Tail 20 -Wait
```

## Local Script Runner Rules

Use exact whitelist matching. Normalize common mention prefixes such as `@机器人 舱位申请` and `@Name 舱位申请` to `舱位申请`.

Do not run arbitrary text. Only run configured commands.

Record each task:

- timestamp
- sender
- incoming text
- matched command
- local script return code
- output file path
- stdout/stderr tail
- dws upload/send result
- final reply

For scripts that use Excel COM:

- Run in an interactive Windows session.
- Avoid running as a headless Windows service.
- Close workbooks and quit Excel in `finally`.
- Treat generated files as ready only after Excel closes and the path exists.

## File Return Modes

`user_file` mode sends a native group file message:

1. Upload:
   ```powershell
   dws drive upload --file "<file>" --file-name "<name>" --space-id <groupSpaceId> --format json
   ```
2. Inspect:
   ```powershell
   dws drive info --node <fileId> --space-id <groupSpaceId> --format json
   ```
3. Send:
   ```powershell
   dws chat message send --group <openConversationId> --title "<title>" --text "<text>" --msg-type file --dentry-id <dentryId> --space-id <groupSpaceId> --file-name "<name>" --file-type "<ext>" --file-path "/<name>" --file-size <bytes> --uuid <uuid> --format json
   ```

This is sent by the current dws-authenticated account.

`bot_link` mode sends a Markdown link from the robot:

```powershell
dws chat message send-by-bot --robot-code <robotCode> --group <openConversationId> --title "<title>" --text "[file.xlsm](https://alidocs...)" --format json
```

Use this only when the user wants the robot as sender and accepts that users see a link, not a native file attachment.

## Common Failures

`OSS 上传失败 ... connection forcibly closed`

- Treat as transient network failure.
- Retry `dws drive upload` up to three times with short delays.
- Log full stderr.

Mojibake in robot replies:

- Set `PYTHONUTF8=1`.
- Use `py -X utf8`.
- Decode dws subprocess output as UTF-8 with replacement.

Bot loops on its own replies:

- Ignore messages containing known fail text such as `指令识别失败`.
- Ignore encrypted/control payloads containing separators like `||`.
- Restrict with `--allowed-groups`.

`dws dev connect status` says `not_running` while logs show connected:

- This can happen when Stream is launched as an external foreground/cmd process rather than dws daemon.
- Trust the process and `stream.out.log`, especially `connect success`.

## Final User-Facing Summary Shape

When done, report:

- App/robot name.
- `unifiedAppId`, `robotCode`, and `agentId` if useful.
- Version status.
- Target group and whether the robot was added.
- Stream process/log location.
- Exact test phrase.
- Whether files are sent by robot link or current dws account native file.
