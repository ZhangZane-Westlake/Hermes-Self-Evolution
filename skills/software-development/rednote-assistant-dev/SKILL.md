---
name: rednote-assistant-dev
description: "Develop features for the Rednote-Assistant Flask + pywebview macOS app."
version: 1.0.0
metadata:
  hermes:
    tags: [rednote, xiaohongshu, flask, sqlite, pywebview]
---

# Rednote-Assistant Development

Rednote-Assistant is a Flask + pywebview macOS desktop app for 小红书笔记助手 (Xiaohongshu content assistant). The project lives at `/Users/zane/Documents/Personal/Rednote-Assistant`.

## Project Map

| File | Role |
|------|------|
| `app.py` | Flask backend — all API routes, config helpers |
| `database.py` | SQLite multi-account layer (`~/.xhs-assistant/`) |
| `deepseek_client.py` | LLM prompts + OpenAI-compatible client calls |
| `vision_client.py` | Image recognition via vision models |
| `templates/index.html` | Single-page app HTML |
| `static/app.js` | All frontend logic (vanilla JS) |
| `tests/` | pytest suite |

## Architecture: Per-Account SQLite

Each account gets its own SQLite DB at `~/.xhs-assistant/accounts/<uuid>/notes.db` with two tables:
- `notes` — note CRUD
- `config` — key-value store for all settings + persisted content (profile, analysis, histories)

All persisted data (profile, analysis, content history, suggestion history) lives in the `config` table as JSON strings.

## Pattern: Adding History Support to a Feature

This is the canonical recipe for adding history (list/view/delete) to any LLM-generated content tab. Followed for content creation and replicated for suggestions.

### Backend (app.py)

**1. Add history helpers:**
```python
def _get_<name>_history():
    raw = db.get_config("<name>_history", "[]")
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        return []

def _save_<name>_history(history):
    db.set_config("<name>_history", json.dumps(history, ensure_ascii=False))
```

**2. Add GET history endpoint:**
```python
@app.route("/api/<name>/history", methods=["GET"])
def api_<name>_history():
    return jsonify({"items": _get_<name>_history()})
```

**3. Add DELETE history endpoint:**
```python
@app.route("/api/<name>/history/<int:index>", methods=["DELETE"])
def api_<name>_history_delete(index):
    history = _get_<name>_history()
    if 0 <= index < len(history):
        history.pop(index)
        _save_<name>_history(history)
        return jsonify({"ok": True})
    return jsonify({"error": "记录不存在"}), 404
```

**4. Modify the generation POST to save to history:**
```python
# Inside the try block after successful generation:
history = _get_<name>_history()
import datetime
history.insert(0, {
    "id": datetime.datetime.now().strftime("%Y%m%d%H%M%S"),
    "timestamp": datetime.datetime.now().strftime("%Y-%m-%d %H:%M"),
    "extra_prompt": extra[:200],
    "content": result,         # the full generated output
})
_save_<name>_history(history[:50])  # cap at 50 entries
```

**5. Add to clear-module keys in `api_clear_module()`:**
```python
"<name>": ["<name>_history"],
# Also add to "all" list
```

### Frontend (templates/index.html)

**1. Add history section in the tab:**
```html
<div id="<name>-history-section" style="margin-top:24px">
  <h3 style="font-size:15px;font-weight:600;margin-bottom:12px">📋 历史记录</h3>
  <div id="<name>-history-list" class="history-list"></div>
</div>
```

**2. Add clear button in Settings:**
```html
<button class="btn-danger btn-clear" data-module="<name>">清除<label>历史</button>
```

### Frontend (static/app.js)

**1. Add `load<Name>History()`:**
```javascript
async function load<Name>History() {
  const resp = await fetch('/api/<name>/history');
  const data = await resp.json();
  const list = document.getElementById('<name>-history-list');
  if (!data.items || !data.items.length) {
    list.innerHTML = '<div class="card" style="text-align:center;color:var(--text-light);padding:20px">暂无历史记录</div>';
    return;
  }
  list.innerHTML = data.items.map((item, idx) => `
    <div class="history-item">
      <div class="history-info" onclick="view<Name>History(${idx})" style="cursor:pointer">
        <div class="history-desc">${esc(item.extra_prompt || '(无额外提示)')}</div>
        <div class="history-time">${item.timestamp}</div>
      </div>
      <button class="btn-ghost" style="font-size:12px;padding:4px 12px" onclick="event.stopPropagation();delete<Name>History(${idx})">🗑</button>
    </div>
  `).join('');
}
```

**2. Add `view<Name>History(idx)` — restores content + extra prompt input:**
```javascript
async function view<Name>History(idx) {
  const resp = await fetch('/api/<name>/history');
  const data = await resp.json();
  const item = data.items[idx];
  if (!item) return;
  document.getElementById('<name>-display').innerHTML = md2html(item.content);
  document.getElementById('<name>-extra-prompt').value = item.extra_prompt || '';
  document.getElementById('<name>-display').scrollIntoView({ behavior: 'smooth' });
}
```

**3. Add `delete<Name>History(idx)`:**

**4. Wire into tab-switch loader:**
```javascript
if (btn.dataset.tab === '<name>') load<Name>History();
```

**5. Wire into account switch + deleteAccount handlers:**

**6. Add label to clear-module handler:**
```javascript
const labels = { ..., '<name>': '<label>历史', ... };
// and refresh line:
if (module === '<name>' || module === 'all') load<Name>History();
```

**7. Call `load<Name>History()` after successful generation in the generate button handler.**

## Testing

```bash
cd /Users/zane/Documents/Personal/Rednote-Assistant
.venv/bin/python -m pytest tests -q
```

Tests use an in-memory/per-test SQLite — they don't touch real `~/.xhs-assistant/`.

## Pitfalls

- The `config` table is per-account. When switching accounts, all history/config switches with it. The frontend MUST reload data on account switch.
- History records use `json.dumps(..., ensure_ascii=False)` for Chinese text support.
- History entries are capped at 50. Adjust the slice if needed.
- The history DELETE route uses `<int:index>` — sequential index, not an ID. Deleting entries shifts indices of remaining records, which is OK since the UI re-fetches after each delete.
- The `import datetime` is placed inside the function rather than top-level to keep startup imports minimal (existing convention in the codebase).

## Markdown Rendering: Tables & Code Blocks

The frontend `md2html()` in `app.js` is a minimal regex converter — it handles headings, bold, lists, blockquotes, but NOT tables or fenced code blocks. LLM-generated content (especially deep analysis) often contains markdown tables (e.g., `| 排名 | 标题 | 阅读 | 总互动 |`). When `md2html()` encounters a table, it outputs garbled inline text. **Always use server-side rendering for LLM output.**

### Install the dependency

```bash
cd /Users/zane/Documents/Personal/Rednote-Assistant
.venv/bin/pip install markdown
```

### Server-side helper (app.py)

Add after imports:
```python
import markdown

_md = markdown.Markdown(extensions=["tables", "fenced_code", "codehilite"])

def _render_md(text: str) -> str:
    """Render markdown text to HTML, with table support."""
    if not text:
        return ""
    return _md.convert(text)
```

### API endpoints — add `html` field

Every endpoint that returns markdown content should also return a server-rendered `html` field:

| Endpoint | Field added |
|----------|-------------|
| `POST /api/suggestions` | `html` |
| `GET /api/analytics/deep` | `html` |
| `POST /api/analytics/deep` | `html` |
| `PUT /api/analytics/deep` | `html` |
| `GET /api/analytics/stats` | `analysis_html` |

Also add a general-purpose render endpoint for history viewers:
```python
@app.route("/api/markdown", methods=["POST"])
def api_render_markdown():
    data = request.get_json(force=True)
    text = data.get("text", "")
    return jsonify({"html": _render_md(text)})
```

### Frontend: prefer `data.html` over `md2html()`

Every place that currently does `md2html(data.content)` should become:
```javascript
data.html || md2html(data.content)
```

For history items that lack pre-rendered HTML, call `POST /api/markdown`:
```javascript
const renderResp = await fetch('/api/markdown', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({ text: item.content }),
});
const renderData = await renderResp.json();
element.innerHTML = renderData.html;
```

### Pitfall: `api_save_deep_analysis()` (PUT)

The manual-edit save endpoint (`PUT /api/analytics/deep`) must return `html` so the frontend can display the saved content with proper rendering. Without this, the edit→save flow falls back to `md2html()` and tables break again.
