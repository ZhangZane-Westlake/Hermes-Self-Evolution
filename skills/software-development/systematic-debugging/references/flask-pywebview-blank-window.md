# Flask + pywebview blank-window debugging notes

Use this reference when a Python GUI wrapper opens a native/WebView window but the content is blank while the Flask app works in a browser.

## High-value checks

1. Reproduce both entry points separately:
   - `python app.py`, then `curl -D - http://127.0.0.1:<port>/`
   - `python app_gui.py`, then `curl -D - http://127.0.0.1:<port>/`
   - If curl returns complete HTML/CSS/JS but WebView is blank, isolate WebView timing/rendering rather than Flask routing.
2. Check import-time failures in packaged apps:
   - Any new top-level import in `app.py` must be copied into the `.app/Contents/Resources/` bundle or included in the packager spec/build script.
   - A missing module can make a double-clicked macOS app appear blank or silently quit because stdout/stderr are hidden.
3. Check server readiness race:
   - If Flask starts on a background thread and WebView loads immediately, some WebKit/pywebview environments may load before the server is ready and remain blank.
   - Add a small readiness loop that polls the exact app URL before `webview.create_window(..., url=APP_URL)`.
4. Centralize host/port constants:
   - Define `APP_HOST`, `APP_PORT`, `APP_URL` once in the Flask app module and import them in the GUI entry point.
   - Avoid hard-coded mismatches between `app.py`, `app_gui.py`, tests, and packaging scripts.
5. Verify static assets separately:
   - `curl -D - /static/app.js` and `/static/style.css` to confirm the template can resolve packaged static files.

## Known-good readiness pattern

```python
import time
import urllib.request

from app import APP_HOST, APP_PORT, APP_URL, app as flask_app


def run_flask():
    flask_app.run(
        host=APP_HOST,
        port=APP_PORT,
        debug=False,
        use_reloader=False,
        threaded=True,
    )


def wait_for_flask(timeout: float = 10.0) -> bool:
    deadline = time.time() + timeout
    last_error = None
    while time.time() < deadline:
        try:
            with urllib.request.urlopen(APP_URL, timeout=0.5) as response:
                return 200 <= response.status < 500
        except Exception as exc:
            last_error = exc
            time.sleep(0.1)
    print(f"Flask server did not become ready: {last_error}", file=sys.stderr)
    return False
```

Then start the thread, call `wait_for_flask()`, and only then create the WebView window.

## Packaging regression tests worth adding

- Assert GUI uses the same configured URL/port as Flask.
- Assert the build script copies every top-level module imported by the app, especially newly added client modules.
- Add endpoint tests for multipart image upload if the GUI includes image/file selection.
