# WeChat Work Webhook Integration Design

## Summary

Add enterprise WeChat (企业微信) webhook notifications to the Codex CLI hooks pack. Each of the 8 existing hook events can individually trigger a text message to a WeChat Work group bot, toggleable per-event via the existing config layer.

## Requirements

- Send webhook notifications to WeChat Work on any of the 8 hook events
- Per-event enable/disable toggle (mirrors sound config pattern)
- Webhook URL stored in `hooks-config.local.json` (gitignored)
- Message format: text with emoji markers, including event name, timestamp, tool name, and last assistant message where available
- Non-blocking: failures and timeouts must not block the main hook flow
- Zero new dependencies — use `urllib.request` from stdlib

## Configuration

### hooks-config.json additions

```json
{
  "webhookUrl": "",
  "webhookSessionStart": true,
  "webhookPreToolUse": true,
  "webhookPermissionRequest": true,
  "webhookPostToolUse": true,
  "webhookStop": true,
  "webhookUserPromptSubmit": true,
  "webhookPreCompact": true,
  "webhookPostCompact": true
}
```

- `webhookUrl` — defaults to empty string; empty means all webhook notifications are skipped. Users set the real URL in `hooks-config.local.json`.
- `webhook<EventName>` — per-event toggle, defaults to `true`. When `webhookUrl` is set, all events notify unless individually disabled.
- Uses existing `get_config_value()` fallback: `hooks-config.local.json` → `hooks-config.json` → default.

## Code Changes

### hooks.py

**New map — `HOOK_WEBHOOK_MAP`**

Sits alongside `HOOK_SOUND_MAP`, maps event name to its config toggle key:

```python
HOOK_WEBHOOK_MAP = {
    "SessionStart": "webhookSessionStart",
    "PreToolUse": "webhookPreToolUse",
    "PermissionRequest": "webhookPermissionRequest",
    "PostToolUse": "webhookPostToolUse",
    "Stop": "webhookStop",
    "UserPromptSubmit": "webhookUserPromptSubmit",
    "PreCompact": "webhookPreCompact",
    "PostCompact": "webhookPostCompact",
}
```

**New function — `send_webhook(event_name, input_data)`**

1. Read `webhookUrl` via `get_config_value("webhookUrl", default="")`. If empty, return.
2. Look up the event's config key in `HOOK_WEBHOOK_MAP`. Read its value via `get_config_value()`. If false, return.
3. Build message text with emoji markers:

```
🔔 Codex CLI Hook
📌 事件：SessionStart
⏰ 时间：2026-05-27 14:30:00
🔧 工具：Bash
💬 消息：last assistant message...
```

Fields are extracted from `input_data`. Missing fields are omitted from the message.

4. POST to `{webhookUrl}` with JSON body per WeChat Work API format:

```json
{
  "msgtype": "text",
  "text": {
    "content": "<formatted message>"
  }
}
```

5. Uses `urllib.request.urlopen` with a 5-second timeout. All exceptions are caught and logged to stderr — never blocks the main hook flow.

**main() change**

Single line added after `play_sound()`:

```python
send_webhook(event_type, input_data)
```

### Files NOT changed

- `.codex/hooks.json` — same script, same invocation, no change needed
- `install/` templates — hooks.json is unchanged
- `parse_args()`, `log_hook_data()`, `get_session_context()` — untouched
- Sound logic — completely independent

## Message Format

Text messages with emoji markers. The content varies by what's available in `input_data`:

| Field | Source | Always present |
|-------|--------|---------------|
| 事件 (event) | `event_type` from CLI arg | Yes |
| 时间 (time) | `datetime.now()` | Yes |
| 工具 (tool) | `input_data.get("tool_name")` | PreToolUse, PostToolUse only |
| 消息 (message) | `input_data.get("last_assistant_message")` | When available |

## Testing

5 new test cases in `tests/test_hooks.py`, using `unittest.mock.patch` to mock `urllib.request.urlopen`:

1. `test_webhook_url_empty_skips` — no POST when webhookUrl is empty
2. `test_webhook_event_disabled_skips` — no POST when per-event toggle is false
3. `test_webhook_payload_format` — POST body matches WeChat Work API JSON structure
4. `test_webhook_builds_correct_message` — text content includes emoji, event name, timestamp
5. `test_webhook_timeout_fails_silently` — timeout/exception does not raise

## Files Changed

| File | Change |
|------|--------|
| `.codex/hooks/scripts/hooks.py` | Add `HOOK_WEBHOOK_MAP`, `send_webhook()`, call in `main()` |
| `.codex/hooks/config/hooks-config.json` | Add 9 new keys |
| `tests/test_hooks.py` | Add 5 webhook test cases |
