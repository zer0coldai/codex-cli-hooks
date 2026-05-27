# WeChat Work Webhook Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-event WeChat Work webhook notifications to the Codex CLI hooks pack.

**Architecture:** Add a `HOOK_WEBHOOK_MAP` and `send_webhook()` function to the existing `hooks.py`, parallel to the sound system. Webhook URL and per-event toggles live in the existing config layer. Uses `urllib.request` (stdlib) for HTTP POST — no new dependencies.

**Tech Stack:** Python 3 stdlib (`urllib.request`, `json`), `unittest.mock` for tests

---

### Task 1: Add webhook config keys to hooks-config.json

**Files:**
- Modify: `.codex/hooks/config/hooks-config.json`

- [ ] **Step 1: Add 9 new keys to hooks-config.json**

Append `webhookUrl` and 8 per-event toggle keys to the existing JSON object:

```json
{
  "disableSessionStartHook": false,
  "disablePreToolUseHook": false,
  "disablePermissionRequestHook": false,
  "disablePostToolUseHook": false,
  "disableStopHook": false,
  "disableUserPromptSubmitHook": false,
  "disablePreCompactHook": false,
  "disablePostCompactHook": false,
  "disableLogging": true,
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

- [ ] **Step 2: Verify JSON is valid**

Run: `python3 -c "import json; json.load(open('.codex/hooks/config/hooks-config.json')); print('valid')"`
Expected: `valid`

- [ ] **Step 3: Commit**

```bash
git add .codex/hooks/config/hooks-config.json
git commit -m "add webhook config keys to hooks-config.json"
```

---

### Task 2: Add HOOK_WEBHOOK_MAP to hooks.py

**Files:**
- Modify: `.codex/hooks/scripts/hooks.py:44` (after `HOOK_SOUND_MAP` closing brace)

- [ ] **Step 1: Add HOOK_WEBHOOK_MAP dict**

Insert after the closing `}` of `HOOK_SOUND_MAP` (line 44) and before `HOOK_CONFIG_MAP`:

```python
# ===== HOOK EVENT TO WEBHOOK TOGGLE MAPPING =====
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

- [ ] **Step 2: Verify it loads without error**

Run: `python3 -c "import sys; sys.path.insert(0,'.codex/hooks/scripts'); import hooks; print(len(hooks.HOOK_WEBHOOK_MAP))"`
Expected: `8`

- [ ] **Step 3: Commit**

```bash
git add .codex/hooks/scripts/hooks.py
git commit -m "add HOOK_WEBHOOK_MAP to hooks.py"
```

---

### Task 3: Write failing tests for send_webhook

**Files:**
- Modify: `tests/test_hooks.py` (append before `if __name__` block at line 330)

- [ ] **Step 1: Write 5 failing webhook tests**

Add a new test class `TestSendWebhook` before the `if __name__` block:

```python
class TestSendWebhook(unittest.TestCase):
    """Test WeChat Work webhook notification logic."""

    @patch("hooks.get_config_value")
    def test_webhook_url_empty_skips(self, mock_config):
        """No POST when webhookUrl is empty."""
        mock_config.return_value = ""
        with patch("hooks.urllib.request.urlopen") as mock_urlopen:
            hooks.send_webhook("SessionStart", {"type": "SessionStart"})
            mock_urlopen.assert_not_called()

    @patch("hooks.get_config_value")
    def test_webhook_event_disabled_skips(self, mock_config):
        """No POST when per-event toggle is false."""
        def config_side_effect(key, default=""):
            if key == "webhookUrl":
                return "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=test-key"
            if key == "webhookSessionStart":
                return False
            return default
        mock_config.side_effect = config_side_effect
        with patch("hooks.urllib.request.urlopen") as mock_urlopen:
            hooks.send_webhook("SessionStart", {"type": "SessionStart"})
            mock_urlopen.assert_not_called()

    @patch("hooks.get_config_value")
    def test_webhook_payload_format(self, mock_config):
        """POST body matches WeChat Work API JSON structure."""
        def config_side_effect(key, default=""):
            if key == "webhookUrl":
                return "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=test-key"
            if key == "webhookStop":
                return True
            return default
        mock_config.side_effect = config_side_effect
        with patch("hooks.urllib.request.urlopen") as mock_urlopen:
            hooks.send_webhook("Stop", {"type": "Stop"})
            mock_urlopen.assert_called_once()
            call_args = mock_urlopen.call_args
            request_obj = call_args[0][0]
            self.assertEqual(request_obj.get_full_url(), "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=test-key")
            body = json.loads(request_obj.data.decode("utf-8"))
            self.assertEqual(body["msgtype"], "text")
            self.assertIn("content", body["text"])

    @patch("hooks.get_config_value")
    def test_webhook_builds_correct_message(self, mock_config):
        """Text content includes emoji, event name, and timestamp."""
        def config_side_effect(key, default=""):
            if key == "webhookUrl":
                return "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=test-key"
            if key == "webhookPreToolUse":
                return True
            return default
        mock_config.side_effect = config_side_effect
        with patch("hooks.urllib.request.urlopen") as mock_urlopen:
            hooks.send_webhook("PreToolUse", {
                "type": "PreToolUse",
                "tool_name": "Bash",
                "last_assistant_message": "running ls"
            })
            mock_urlopen.assert_called_once()
            call_args = mock_urlopen.call_args
            request_obj = call_args[0][0]
            body = json.loads(request_obj.data.decode("utf-8"))
            content = body["text"]["content"]
            self.assertIn("PreToolUse", content)
            self.assertIn("Bash", content)
            self.assertIn("running ls", content)

    @patch("hooks.get_config_value")
    def test_webhook_timeout_fails_silently(self, mock_config):
        """Timeout or exception does not raise."""
        def config_side_effect(key, default=""):
            if key == "webhookUrl":
                return "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=test-key"
            if key == "webhookStop":
                return True
            return default
        mock_config.side_effect = config_side_effect
        with patch("hooks.urllib.request.urlopen", side_effect=Exception("timeout")):
            # Should not raise
            hooks.send_webhook("Stop", {"type": "Stop"})
```

- [ ] **Step 2: Run tests to verify they fail (function not yet defined)**

Run: `python3 -m unittest tests.test_hooks -v 2>&1 | tail -20`
Expected: FAIL — `AttributeError: module 'hooks' has no attribute 'send_webhook'`

- [ ] **Step 3: Commit**

```bash
git add tests/test_hooks.py
git commit -m "add failing webhook tests"
```

---

### Task 4: Implement send_webhook function

**Files:**
- Modify: `.codex/hooks/scripts/hooks.py` (add after `is_logging_disabled()` function, around line 242)

- [ ] **Step 1: Add import and send_webhook function**

First, add `urllib.request` to the imports at the top of `hooks.py` (after `import json`, around line 22):

```python
import urllib.request
```

Then add the function after `is_logging_disabled()` (after line 242):

```python
def send_webhook(event_name, input_data):
    """
    Send a WeChat Work webhook notification for the given event.

    Reads webhookUrl and per-event toggle from config. Skips silently
    if URL is empty or event is disabled. Never raises — failures are
    logged to stderr only.

    Args:
        event_name: The hook event name (e.g., "SessionStart")
        input_data: The parsed input dict from stdin
    """
    try:
        webhook_url = get_config_value("webhookUrl", default="")
        if not webhook_url:
            return

        config_key = HOOK_WEBHOOK_MAP.get(event_name)
        if not config_key:
            return

        if not get_config_value(config_key, default=True):
            return

        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        lines = [
            "\U0001f514 Codex CLI Hook",
            f"\U0001f4cc 事件：{event_name}",
            f"⏰ 时间：{timestamp}",
        ]

        tool_name = input_data.get("tool_name")
        if tool_name:
            lines.append(f"\U0001f527 工具：{tool_name}")

        last_msg = input_data.get("last_assistant_message")
        if last_msg:
            lines.append(f"\U0001f4ac 消息：{last_msg}")

        content = "\n".join(lines)
        payload = json.dumps({
            "msgtype": "text",
            "text": {"content": content}
        }, ensure_ascii=False).encode("utf-8")

        req = urllib.request.Request(
            webhook_url,
            data=payload,
            headers={"Content-Type": "application/json"}
        )
        urllib.request.urlopen(req, timeout=5)
    except Exception as e:
        print(f"Webhook failed: {e}", file=sys.stderr)
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `python3 -m unittest tests.test_hooks -v 2>&1 | tail -20`
Expected: All tests PASS

- [ ] **Step 3: Commit**

```bash
git add .codex/hooks/scripts/hooks.py
git commit -m "implement send_webhook function"
```

---

### Task 5: Wire send_webhook into main()

**Files:**
- Modify: `.codex/hooks/scripts/hooks.py` (in `main()`, after `play_sound` call around line 358)

- [ ] **Step 1: Add send_webhook call in main()**

After the `play_sound` block (around line 358), add:

```python
        # Send webhook notification
        send_webhook(event_type, input_data)
```

The surrounding context should look like:

```python
        # Play the sound
        if sound_name:
            play_sound(sound_name)

        # Send webhook notification
        send_webhook(event_type, input_data)

        sys.exit(0)
```

- [ ] **Step 2: Update existing integration tests to include send_webhook mock**

The existing `TestMainIntegration` tests mock `play_sound` but `main()` now also calls `send_webhook`. Add `@patch("hooks.send_webhook")` to each test method in `TestMainIntegration`, and add `mock_webhook` to the parameter list. For example, `test_session_start_outputs_context_and_plays_sound` becomes:

```python
    @patch("hooks.send_webhook")
    @patch("hooks.play_sound", return_value=True)
    @patch("hooks.is_hook_disabled", return_value=False)
    @patch("hooks.log_hook_data")
    @patch("hooks.get_session_context", return_value="Date: 2026-03-17\nGit branch: main")
    def test_session_start_outputs_context_and_plays_sound(
        self, mock_context, mock_log, mock_disabled, mock_play, mock_webhook
    ):
        with patch("sys.argv", ["hooks.py", "--hook", "SessionStart"]):
            with self.assertRaises(SystemExit) as ctx:
                hooks.main()
            self.assertEqual(ctx.exception.code, 0)
            mock_context.assert_called_once()
            mock_play.assert_called_once_with("SessionStart")
            mock_webhook.assert_called_once()
```

Apply the same pattern (`@patch("hooks.send_webhook")` + `mock_webhook` param) to all other test methods in `TestMainIntegration`:
- `test_session_stop_plays_sound`
- `test_pre_tool_use_plays_sound`
- `test_post_tool_use_plays_sound`
- `test_UserPromptSubmit_plays_sound`
- `test_disabled_hook_skips_sound`
- `test_no_args_exits_cleanly`

For `test_disabled_hook_skips_sound` and `test_no_args_exits_cleanly`, also add `mock_webhook.assert_not_called()` to verify webhook is not sent when hook is disabled or no args.

- [ ] **Step 3: Run all tests**

Run: `python3 -m unittest tests.test_hooks -v`
Expected: All tests PASS (0 failures)

- [ ] **Step 4: Commit**

```bash
git add .codex/hooks/scripts/hooks.py
git commit -m "wire send_webhook into main()"
```

```bash
git add tests/test_hooks.py
git commit -m "update integration tests for webhook calls"
```

---

### Task 6: Final verification

- [ ] **Step 1: Run full test suite**

Run: `python3 -m unittest tests.test_hooks -v`
Expected: All tests PASS

- [ ] **Step 2: Manual smoke test — verify webhook skips with empty URL**

Run: `python3 .codex/hooks/scripts/hooks.py --hook Stop`
Expected: exits cleanly, no errors, no HTTP request

- [ ] **Step 3: Verify existing hooks still work (sound, config, logging)**

Run: `python3 -m unittest tests.test_hooks -v 2>&1 | grep -E "^test_|^FAIL|^ERROR|OK"`
Expected: All `test_` lines pass, ends with `OK`
