# Stop Words Filter — Design Spec

## Summary

Add configurable stop words filtering to hermes-agent. When enabled, the agent scans all messages in the conversation history before sending them to the LLM API and replaces any matched substrings with a placeholder. Matching uses case-insensitive substring search. Detections are logged.

## Motivation

Users may want to prevent certain sensitive terms (API keys, internal codenames, personal data) from ever reaching the LLM provider. The filter acts as a safety net on the outgoing request pipeline.

## Configuration

New section in `DEFAULT_CONFIG` (`hermes_cli/config.py`):

```yaml
stop_words:
  enabled: false
  words: []
  placeholder: "[FILTERED]"
```

- `enabled` (bool) — master switch. When `false`, no filtering occurs.
- `words` (list[str]) — list of stop word strings. Matched as case-insensitive substrings anywhere in message content.
- `placeholder` (str) — replacement text for matched substrings. Default: `"[FILTERED]"`.

No default words are provided. The list is entirely user-defined.

`_config_version` bump required (currently 14 → 15) to trigger migration for existing users.

## Architecture

### Filter Function

A new method `_filter_stop_words(self, api_messages: list) -> list` on the `AIAgent` class in `run_agent.py`:

1. Skip entirely if `self._stop_words` is empty or `self._stop_words_enabled` is `False`.
2. Iterate over `api_messages` (these are already shallow copies of the original `messages`).
3. For each message, scan all text-bearing fields:
   - `content` — can be `str` or `list[dict]` (multimodal format with `{"type": "text", "text": "..."}` blocks)
   - `tool_calls[].function.name` — tool name strings
   - `tool_calls[].function.arguments` — JSON string of arguments
4. For each text field, perform case-insensitive substring replacement of all stop words with the placeholder.
5. Log each detection via `logging.info()` with: stop word found, message role, message index.
6. Return the modified list (mutation in-place on the copies is acceptable).

### Initialization

In `AIAgent.__init__()`:
- Read `stop_words` config section from the agent config.
- Store as `self._stop_words_enabled` (bool) and `self._stop_words` (list[str]).
- Lowercase all words at init time for efficient case-insensitive matching.

### Call Sites

The filter is called on `api_messages` at **3 points**, always after message construction/sanitization and before the API call:

1. **Main agent loop** (`run_conversation()`, after `_sanitize_api_messages()`, before `_build_api_kwargs()`):
   ```
   api_messages = self._sanitize_api_messages(api_messages)
   api_messages = self._filter_stop_words(api_messages)  # <-- HERE
   api_kwargs = self._build_api_kwargs(api_messages)
   ```

2. **Memory flush** (`flush_memories()`, after the message copy loop, before the API call):
   ```
   api_messages = self._filter_stop_words(api_messages)  # <-- HERE
   ```

3. **Iteration-limit summary** (`_handle_max_iterations()`, after the message copy loop, before the API call):
   ```
   api_messages = self._filter_stop_words(api_messages)  # <-- HERE
   ```

### What Is NOT Filtered

- **System prompt** — passed as a separate string, not inside `api_messages`. Could be added later if needed.
- **Tool schemas** — static JSON schemas sent alongside messages. Not part of the conversation history.
- **Original `messages` list** — the filter only touches `api_messages` (copies), preserving prompt caching integrity.

## Prompt Caching Safety

The filter operates on `api_messages`, which are shallow copies of the original `messages` list created fresh on every API call. The original `messages` (used for prompt cache matching across turns) are never modified.

Since filtering is deterministic (same stop words → same output), the filtered `api_messages` remain stable across requests, preserving Anthropic prompt cache hits.

The only cache-breaking scenario is if the user changes the stop words list mid-conversation, which causes a single cache miss before the new filtered content stabilizes.

## Content Format Handling

Messages can have `content` in two formats:

**String:**
```python
msg["content"] = "some text with secret_key in it"
# → "some text with [FILTERED] in it"
```

**Multimodal (list of dicts):**
```python
msg["content"] = [
    {"type": "text", "text": "some text with secret_key"},
    {"type": "image_url", "image_url": {"url": "..."}}
]
# → [{"type": "text", "text": "some text with [FILTERED]"}, ...]
```

Only `{"type": "text"}` blocks are scanned. Image URLs, file references, etc. are left untouched.

## Logging

Each detection produces a `logging.info()` entry:
```
Stop word 'secret_key' found in message [3] (role=user), replaced with [FILTERED]
```

This keeps the user informed without disrupting the conversation flow.

## Testing

Tests should cover:
- Basic substring matching (case-insensitive)
- Multiple stop words in a single message
- Stop words across multiple messages
- Multimodal content format (list-of-dicts)
- Tool call arguments containing stop words
- Empty words list → no filtering
- `enabled: false` → no filtering
- Custom placeholder
- No modification of original `messages` list
- Tool call function names
- Non-text content blocks (images) are not touched

## Files Modified

| File | Change |
|------|--------|
| `hermes_cli/config.py` | Add `stop_words` section to `DEFAULT_CONFIG`, bump `_config_version` |
| `run_agent.py` | Add `_filter_stop_words()` method, read config in `__init__()`, call at 3 API call sites |

No new files needed. No changes to tools, gateway, or CLI.
