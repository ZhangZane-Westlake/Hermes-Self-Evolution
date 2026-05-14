# Testing Flask + SQLite apps with module-level initialization

Use this when a Flask app initializes a SQLite data layer at import time, especially when tests need isolated per-test data directories.

## Pattern

1. Add the project root to `sys.path` in tests if the app is a flat-file project rather than an installed package.
2. Use `tmp_path` plus `monkeypatch.setenv("HOME", str(tmp_path))` before importing/reloading the modules that compute paths from `Path.home()`.
3. Reload both the database module and app module so module-level constants and `db.init_db()` run against the temporary home.
4. Enable Flask testing mode and yield a `test_client()`.

Example:

```python
import importlib
import sys
from pathlib import Path

import pytest

PROJECT_ROOT = Path(__file__).resolve().parents[1]
if str(PROJECT_ROOT) not in sys.path:
    sys.path.insert(0, str(PROJECT_ROOT))

@pytest.fixture()
def client(tmp_path, monkeypatch):
    monkeypatch.setenv("HOME", str(tmp_path))
    import database
    import app as app_module

    database = importlib.reload(database)
    app_module = importlib.reload(app_module)
    app_module.app.config.update(TESTING=True)
    with app_module.app.test_client() as c:
        yield c
```

## Why this matters

If `database.py` sets `DATA_DIR = Path.home() / ...` at import time and `app.py` calls `db.init_db()` at import time, changing `HOME` after imports is too late. Tests may accidentally read/write the real user database or share state across tests.

## Vision/file upload endpoint test pattern

For Flask multipart tests:

```python
import io

resp = client.post(
    "/api/vision/describe",
    data={"images": (io.BytesIO(b"fake-jpeg"), "test.jpg")},
    content_type="multipart/form-data",
)
```

For external model calls, monkeypatch the function imported by the app module, not necessarily the original module function:

```python
monkeypatch.setattr(app_module, "describe_images", fake_describe_images)
```

Patch at the lookup site used by the route under test.