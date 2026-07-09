---
name: dingtalk-local-bot-runner
description: Create, configure, debug, or document DingTalk Stream robot workflows where a DingTalk group message triggers local Python or script execution on the user's PC, then sends generated files/messages back to the group with the current dws-authenticated account. Use for tasks involving DingTalk app creation, robot creation/publishing, Stream connect, group message listening, local script runners, dws file upload/send, group bot automations, or converting such workflows into reusable project structure.
---

# DingTalk Local Bot Runner

Use this skill to build or maintain a DingTalk automation where:

```text
DingTalk group message
-> Stream robot or dws polling receives it
-> local machine runs a whitelisted script
-> generated output file is uploaded with dws
-> current dws account sends the native file message back to the group
```

For the complete playbook, read `references/dingtalk-stream-local-runner.md`.

## Required Companion Skill

When executing DingTalk operations, also use the `dws` skill and follow its command-safety rules:

- Confirm command paths and flags with product references or `--help`.
- Use `--format json` for machine-readable dws commands.
- Use `--dry-run` before write/publish/send operations when available.
- Use `--yes` only after the user has asked to proceed or the operation is clearly part of the approved workflow.
- Do not invent IDs; read them from dws output.

## Standard Workflow

1. Identify the target group.
   - Search with `dws chat search --query "<group>" --format json`.
   - Save `openConversationId`.
   - Save the group conversation space ID, usually `extension.newCSpaceIdIM`.

2. Create or choose the DingTalk app and robot.
   - New robot: `dws dev app robot submit ... --dry-run --format json`, then submit with `--yes`.
   - Poll with `dws dev app robot result --task-id ... --format json`.
   - If the result lacks `unifiedAppId`, search `dws dev app list --format json` for the new app name and use the returned `unifiedAppId`.
   - Confirm robot status with `dws dev app robot get --unified-app-id ... --format json`.

3. Publish the app version.
   - Create version with dry-run, then `--yes`.
   - Run `check-approval`.
   - If no approval is required, publish with dry-run, then `--yes`.
   - Confirm `versionStatus` is `RELEASE` or an accepted review state.

4. Add the robot to the target group.
   - `dws chat group members add-bot --id <openConversationId> --robot-code <robotCode> --format json`.
   - Verify with `dws chat group bots --group <openConversationId> --format json`.

5. Build the local project.
   - Keep trigger rules and failure text in YAML.
   - Use a whitelist mapping from trigger text to exact local script path.
   - Never execute arbitrary user text as a shell command.
   - Capture task logs as JSONL with incoming text, match result, script return code, output path, send result, and stderr tail.

6. Connect Stream.
   - Prefer `dws dev connect --unified-app-id <id> --channel custom --agent-cmd <wrapper>` for local Stream.
   - On Windows, use `.cmd` wrappers for commands containing spaces or `py -m`.
   - Use `--allowed-groups <openConversationId>` to restrict who can trigger the local runner.
   - Check live status from process/log files; `dws dev connect status` may not track externally launched foreground processes.

7. Send files.
   - For native group file messages, upload to the group conversation space with `dws drive upload --space-id <newCSpaceIdIM>`.
   - Get metadata with `dws drive info`.
   - Send with `dws chat message send --msg-type file ...`.
   - This native file message is sent by the current dws-authenticated account.
   - If the user insists the robot must be the sender, use `send-by-bot` Markdown links only; current `send-by-bot` does not support native file attachment flags.

8. Harden behavior.
   - Add upload retries for transient OSS failures.
   - Force UTF-8 for Python/CMD wrappers to avoid mojibake.
   - Ignore encrypted/control payloads and the bot's own failure replies to avoid loops.
   - Keep Stream running only on a machine that can run the local script; Excel COM scripts usually require an interactive Windows user session.

## Validation Checklist

- `py -m compileall src` passes.
- Dry-run trigger finds the expected output file without running the real script.
- `dws dev app robot get` shows robot `ONLINE` and mode `STREAM`.
- App version is `RELEASE` or otherwise usable.
- Robot is present in the target group.
- Stream log shows `connect success`.
- A controlled group test triggers the script once and logs one JSONL record.
- File upload/send succeeds or logs the exact dws stderr for retry/debugging.
