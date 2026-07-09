# DingTalk Local Bot Runner Skill

Codex Skill for building DingTalk Stream robot workflows that:

```text
DingTalk group message
-> DingTalk Stream robot or dws polling
-> local Python/script execution
-> generated file upload with dws
-> native DingTalk group file message from the current dws account
```

The reusable Skill lives in:

```text
dingtalk-local-bot-runner/
```

## Install The Skill

Copy the Skill folder into your Codex skills directory:

```powershell
Copy-Item -Recurse .\dingtalk-local-bot-runner "$env:USERPROFILE\.codex\skills\"
```

Then restart Codex or start a new thread. You can invoke it with:

```text
Use $dingtalk-local-bot-runner to create a DingTalk Stream bot that runs a local script and returns generated files with dws.
```

## Prerequisites

### 1. Install dws

Install the DingTalk Workspace CLI (`dws`) using the method provided by your organization.

Verify:

```powershell
dws --help
dws auth status --format json
```

If needed, authenticate:

```powershell
dws auth login
dws auth status --format json
dws profile list --format json
```

If your account belongs to multiple organizations, use the intended profile:

```powershell
dws profile list --format json
dws <command> --profile <corpId-or-profile-name> --format json
```

### 2. DingTalk app permissions

The operating account should be able to:

- create or manage DingTalk developer apps;
- configure robot capability;
- publish app versions or choose approvers;
- add bots to the target group;
- upload files to the target group conversation space;
- send group messages through the current dws-authenticated account.

### 3. Local machine requirements

The local machine must:

- stay powered on while automation is expected to run;
- keep `dws dev connect` running;
- keep dws authentication valid;
- have the script runtime and dependencies installed;
- have access to local files, network paths, ODBC drivers, Excel COM, or other resources required by the script.

For Excel COM automation, run in an interactive Windows user session rather than a headless service unless you have explicitly hardened that environment.

## DingTalk Setup Overview

1. Find the target group:

```powershell
dws chat search --query "<group-name>" --format json
```

Save:

- `openConversationId`
- group conversation space ID, usually `extension.newCSpaceIdIM`

2. Create a Stream robot:

```powershell
dws dev app robot submit --name "<app-name>" --robot-name "<robot-name>" --desc "<description>" --dry-run --format json
dws dev app robot submit --name "<app-name>" --robot-name "<robot-name>" --desc "<description>" --yes --format json
dws dev app robot result --task-id <taskId> --format json
```

3. Find or confirm `unifiedAppId`:

```powershell
dws dev app list --format json
dws dev app robot get --unified-app-id <unifiedAppId> --format json
```

4. Publish the app version:

```powershell
dws dev app version create --unified-app-id <unifiedAppId> --desc "Publish Stream robot" --dry-run --format json
dws dev app version create --unified-app-id <unifiedAppId> --desc "Publish Stream robot" --yes --format json
dws dev app version check-approval --unified-app-id <unifiedAppId> --version-id <versionId> --format json
dws dev app version publish --unified-app-id <unifiedAppId> --version-id <versionId> --dry-run --format json
dws dev app version publish --unified-app-id <unifiedAppId> --version-id <versionId> --yes --format json
```

5. Add the robot to the group:

```powershell
dws chat group members add-bot --id <openConversationId> --robot-code <robotCode> --format json
dws chat group bots --group <openConversationId> --format json
```

6. Start Stream:

```powershell
dws dev connect --unified-app-id <unifiedAppId> --channel custom --agent-cmd <wrapper-command> --agent-workdir <project-root> --allowed-groups <openConversationId>
```

On Windows, use `.cmd` wrappers for commands containing spaces, Chinese characters, or `py -m`.

## File Return Modes

The Skill documents two modes:

- `user_file`: upload the generated file to the group conversation space and send a native DingTalk file message from the current dws-authenticated account.
- `bot_link`: upload the file and let the robot send a Markdown link. Use only when robot sender identity matters more than native file attachment UX.

Current `dws chat message send-by-bot` does not expose native file attachment flags, so a robot can send a link but not the same native file card that `dws chat message send --msg-type file` sends.

## Privacy And Public Repo Hygiene

This repository intentionally uses placeholders. Before publishing or forking derived work:

- Do not commit `.env` files.
- Do not commit dws logs, task logs, generated reports, screenshots, or credentials.
- Replace real `openConversationId`, `spaceId`, `unifiedAppId`, `robotCode`, `agentId`, user IDs, app keys, app secrets, webhook URLs, and access tokens with placeholders.
- Replace local user paths with generic paths such as `C:\path\to\...`.

## Validation

Validate the Skill:

```powershell
py "$env:USERPROFILE\.codex\skills\.system\skill-creator\scripts\quick_validate.py" ".\dingtalk-local-bot-runner"
```

Expected:

```text
Skill is valid!
```
