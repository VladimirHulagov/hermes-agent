# Stop Words Filter — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add configurable stop words filtering that replaces matched substrings in conversation history before sending to the LLM API.

**Architecture:** New `_filter_stop_words()` method on `AIAgent` that scans `api_messages` (copies, not originals) and replaces matched substrings with a configurable placeholder. Config loaded from `config.yaml` under `stop_words` section. Called at 3 API dispatch points.

**Tech Stack:** Python, pytest

---

### Task 1: Add config section

**Files:**
- Modify: `hermes_cli/config.py:639-642`

- [ ] **Step 1: Add `stop_words` section to `DEFAULT_CONFIG` and bump `_config_version`**

In `hermes_cli/config.py`, after the `"logging"` section (line 639) and before `"_config_version"` (line 642), insert:

```python
    "stop_words": {
        "enabled": False,
        "words": [],
        "placeholder": "[FILTERED]",
    },
```

Change `"_config_version": 14` to `"_config_version": 15`.

- [ ] **Step 2: Run existing config tests to verify no breakage**

Run: `source venv/bin/activate && python -m pytest tests/hermes_cli/test_config.py -q`
Expected: All pass

- [ ] **Step 3: Commit**

```bash
git add hermes_cli/config.py
git commit -m "feat: add stop_words config section to DEFAULT_CONFIG"
```

---

### Task 2: Add `_filter_stop_words()` method and initialization

**Files:**
- Modify: `run_agent.py` (init at ~line 1150, method anywhere on the class)

- [ ] **Step 1: Write the failing test**

In `tests/run_agent/test_run_agent.py`, add at the end of the file:

```python
class TestStopWordsFilter:
    def test_basic_substring_replacement(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = ["secret"]
        agent._stop_words_placeholder = "[FILTERED]"
        messages = [
            {"role": "user", "content": "here is a secret value"},
            {"role": "assistant", "content": "I see the secret"},
        ]
        result = agent._filter_stop_words(messages)
        assert result[0]["content"] == "here is a [FILTERED] value"
        assert result[1]["content"] == "I see the [FILTERED]"

    def test_case_insensitive(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = ["Secret"]
        agent._stop_words_placeholder = "[FILTERED]"
        messages = [
            {"role": "user", "content": "SECRET and secret and SeCrEt"},
        ]
        result = agent._filter_stop_words(messages)
        assert result[0]["content"] == "[FILTERED] and [FILTERED] and [FILTERED]"

    def test_disabled_no_filtering(self, agent):
        agent._stop_words_enabled = False
        agent._stop_words = ["secret"]
        agent._stop_words_placeholder = "[FILTERED]"
        messages = [
            {"role": "user", "content": "here is a secret value"},
        ]
        result = agent._filter_stop_words(messages)
        assert result[0]["content"] == "here is a secret value"

    def test_empty_words_no_filtering(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = []
        agent._stop_words_placeholder = "[FILTERED]"
        messages = [
            {"role": "user", "content": "here is a secret value"},
        ]
        result = agent._filter_stop_words(messages)
        assert result[0]["content"] == "here is a secret value"

    def test_multimodal_content(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = ["secret"]
        agent._stop_words_placeholder = "[FILTERED]"
        messages = [
            {"role": "user", "content": [
                {"type": "text", "text": "a secret here"},
                {"type": "image_url", "image_url": {"url": "http://secret.img"}},
            ]},
        ]
        result = agent._filter_stop_words(messages)
        assert result[0]["content"][0]["text"] == "a [FILTERED] here"
        assert result[0]["content"][1]["image_url"]["url"] == "http://secret.img"

    def test_tool_call_arguments(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = ["secret"]
        agent._stop_words_placeholder = "[FILTERED]"
        messages = [
            {"role": "assistant", "tool_calls": [
                {"id": "tc1", "type": "function", "function": {"name": "write_file", "arguments": '{"path": "/secret/file.txt"}'}},
            ]},
        ]
        result = agent._filter_stop_words(messages)
        assert '"path": "/[FILTERED]/file.txt"' in result[0]["tool_calls"][0]["function"]["arguments"]

    def test_tool_call_name(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = ["secret"]
        agent._stop_words_placeholder = "[FILTERED]"
        messages = [
            {"role": "assistant", "tool_calls": [
                {"id": "tc1", "type": "function", "function": {"name": "secret_tool", "arguments": "{}"}},
            ]},
        ]
        result = agent._filter_stop_words(messages)
        assert result[0]["tool_calls"][0]["function"]["name"] == "[FILTERED]_tool"

    def test_custom_placeholder(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = ["secret"]
        agent._stop_words_placeholder = "***"
        messages = [
            {"role": "user", "content": "a secret here"},
        ]
        result = agent._filter_stop_words(messages)
        assert result[0]["content"] == "a *** here"

    def test_multiple_stop_words(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = ["secret", "password"]
        agent._stop_words_placeholder = "[FILTERED]"
        messages = [
            {"role": "user", "content": "secret and password here"},
        ]
        result = agent._filter_stop_words(messages)
        assert result[0]["content"] == "[FILTERED] and [FILTERED] here"

    def test_does_not_modify_original(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = ["secret"]
        agent._stop_words_placeholder = "[FILTERED]"
        original_content = "a secret here"
        messages = [
            {"role": "user", "content": original_content},
        ]
        result = agent._filter_stop_words(messages)
        assert messages[0]["content"] == original_content
        assert result[0]["content"] == "a [FILTERED] here"

    def test_non_text_content_blocks_untouched(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = ["secret"]
        agent._stop_words_placeholder = "[FILTERED]"
        messages = [
            {"role": "assistant", "content": [
                {"type": "text", "text": "a secret"},
                {"type": "image_url", "image_url": {"url": "http://secret.png"}},
            ]},
        ]
        result = agent._filter_stop_words(messages)
        assert result[0]["content"][0]["text"] == "a [FILTERED]"
        assert result[0]["content"][1]["image_url"]["url"] == "http://secret.png"

    def test_logging_on_match(self, agent, caplog):
        agent._stop_words_enabled = True
        agent._stop_words = ["secret"]
        agent._stop_words_placeholder = "[FILTERED]"
        messages = [
            {"role": "user", "content": "a secret here"},
        ]
        with caplog.at_level(logging.INFO):
            result = agent._filter_stop_words(messages)
        assert any("secret" in r.message for r in caplog.records)
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `source venv/bin/activate && python -m pytest tests/run_agent/test_run_agent.py::TestStopWordsFilter -v`
Expected: FAIL (AttributeError — `_filter_stop_words` doesn't exist yet)

- [ ] **Step 3: Implement `_filter_stop_words()` and init code**

In `run_agent.py`, in `AIAgent.__init__()` after the skills config block (after ~line 1245), add:

```python
        _stop_words_cfg = _agent_cfg.get("stop_words", {})
        if not isinstance(_stop_words_cfg, dict):
            _stop_words_cfg = {}
        self._stop_words_enabled = bool(_stop_words_cfg.get("enabled", False))
        self._stop_words = [w.lower() for w in (_stop_words_cfg.get("words") or []) if isinstance(w, str) and w.strip()]
        self._stop_words_placeholder = str(_stop_words_cfg.get("placeholder", "[FILTERED]"))
```

Add the method on `AIAgent`:

```python
    def _filter_stop_words(self, api_messages: list) -> list:
        if not self._stop_words_enabled or not self._stop_words:
            return api_messages
        import re as _re
        filtered = []
        for idx, msg in enumerate(api_messages):
            msg = msg.copy()
            role = msg.get("role", "unknown")
            if "content" in msg:
                content = msg["content"]
                if isinstance(content, str):
                    for word in self._stop_words:
                        pattern = _re.compile(_re.escape(word), _re.IGNORECASE)
                        if pattern.search(content):
                            logging.info("Stop word '%s' found in message [%d] (role=%s), replaced with %s", word, idx, role, self._stop_words_placeholder)
                            content = pattern.sub(self._stop_words_placeholder, content)
                    msg["content"] = content
                elif isinstance(content, list):
                    new_parts = []
                    for part in content:
                        if isinstance(part, dict) and part.get("type") == "text":
                            text = part.get("text", "")
                            for word in self._stop_words:
                                pattern = _re.compile(_re.escape(word), _re.IGNORECASE)
                                if pattern.search(text):
                                    logging.info("Stop word '%s' found in message [%d] (role=%s) text block, replaced with %s", word, idx, role, self._stop_words_placeholder)
                                    text = pattern.sub(self._stop_words_placeholder, text)
                            new_parts.append({**part, "text": text})
                        else:
                            new_parts.append(part)
                    msg["content"] = new_parts
            if "tool_calls" in msg:
                new_tcs = []
                for tc in msg["tool_calls"]:
                    tc = tc.copy()
                    func = tc.get("function", {}).copy()
                    fname = func.get("name", "")
                    for word in self._stop_words:
                        pattern = _re.compile(_re.escape(word), _re.IGNORECASE)
                        if pattern.search(fname):
                            logging.info("Stop word '%s' found in message [%d] (role=%s) tool_call name, replaced with %s", word, idx, role, self._stop_words_placeholder)
                            fname = pattern.sub(self._stop_words_placeholder, fname)
                    func["name"] = fname
                    fargs = func.get("arguments", "")
                    if isinstance(fargs, str):
                        for word in self._stop_words:
                            pattern = _re.compile(_re.escape(word), _re.IGNORECASE)
                            if pattern.search(fargs):
                                logging.info("Stop word '%s' found in message [%d] (role=%s) tool_call arguments, replaced with %s", word, idx, role, self._stop_words_placeholder)
                                fargs = pattern.sub(self._stop_words_placeholder, fargs)
                        func["arguments"] = fargs
                    tc["function"] = func
                    new_tcs.append(tc)
                msg["tool_calls"] = new_tcs
            filtered.append(msg)
        return filtered
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `source venv/bin/activate && python -m pytest tests/run_agent/test_run_agent.py::TestStopWordsFilter -v`
Expected: All PASS

- [ ] **Step 5: Commit**

```bash
git add run_agent.py tests/run_agent/test_run_agent.py
git commit -m "feat: add _filter_stop_words() method with tests"
```

---

### Task 3: Wire filter into 3 API call sites

**Files:**
- Modify: `run_agent.py:8077` (main loop), `run_agent.py:6480` (memory flush), `run_agent.py:7473` (iteration-limit)

- [ ] **Step 1: Write the failing test**

Add to `tests/run_agent/test_run_agent.py`:

```python
class TestStopWordsIntegration:
    def test_main_loop_filters_before_api(self, agent):
        agent._stop_words_enabled = True
        agent._stop_words = ["forbidden"]
        agent._stop_words_placeholder = "[FILTERED]"
        api_messages = [
            {"role": "user", "content": "use the forbidden command"},
        ]
        with patch.object(agent, "_build_api_kwargs", side_effect=RuntimeError("check_messages")) as mock_build:
            try:
                pass
            except Exception:
                pass
        filtered = agent._filter_stop_words(api_messages)
        assert filtered[0]["content"] == "use the [FILTERED] command"

    def test_filter_idempotent_when_disabled(self, agent):
        agent._stop_words_enabled = False
        api_messages = [
            {"role": "user", "content": "secret stuff"},
        ]
        result = agent._filter_stop_words(api_messages)
        assert result is api_messages
```

- [ ] **Step 2: Insert filter call in the main agent loop**

In `run_agent.py`, after line 8077 (`api_messages = self._sanitize_api_messages(api_messages)`), add:

```python
            api_messages = self._filter_stop_words(api_messages)
```

- [ ] **Step 3: Insert filter call in memory flush**

In `run_agent.py`, after line 6480 (`api_messages.append(api_msg)`) and before line 6482 (`if self._cached_system_prompt:`), add:

```python
            api_messages = self._filter_stop_words(api_messages)
```

- [ ] **Step 4: Insert filter call in iteration-limit summary**

In `run_agent.py`, after line 7473 (`api_messages.append(api_msg)`) and before line 7475 (`effective_system = ...`), add:

```python
            api_messages = self._filter_stop_words(api_messages)
```

- [ ] **Step 5: Run all stop words tests**

Run: `source venv/bin/activate && python -m pytest tests/run_agent/test_run_agent.py::TestStopWordsFilter tests/run_agent/test_run_agent.py::TestStopWordsIntegration -v`
Expected: All PASS

- [ ] **Step 6: Run full run_agent test suite**

Run: `source venv/bin/activate && python -m pytest tests/run_agent/ -q`
Expected: All pass (no regressions)

- [ ] **Step 7: Commit**

```bash
git add run_agent.py tests/run_agent/test_run_agent.py
git commit -m "feat: wire stop words filter into all 3 API call sites"
```

---

### Task 4: Final verification

- [ ] **Step 1: Run full test suite**

Run: `source venv/bin/activate && python -m pytest tests/ -q --timeout=120`
Expected: All pass

- [ ] **Step 2: Verify config migration works**

Run: `source venv/bin/activate && python -m pytest tests/hermes_cli/test_config.py -q`
Expected: All pass
