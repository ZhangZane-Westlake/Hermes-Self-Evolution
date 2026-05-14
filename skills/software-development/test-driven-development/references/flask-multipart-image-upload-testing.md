# Flask multipart image upload regression tests

Use this when adding or fixing Flask endpoints that receive uploaded images and transform them before calling an external service (vision model, OCR, storage, etc.).

## Fixture shape

- Set `HOME` to a temp directory before importing/reloading modules if the app initializes SQLite or user config at import time.
- Use `app.test_client()` and `content_type="multipart/form-data"`.
- For Werkzeug test uploads, include an explicit MIME type when endpoint validation relies on `file.mimetype`:

```python
resp = client.post(
    "/api/vision/describe",
    data={"images": (io.BytesIO(uploaded), "huge.png", "image/png")},
    content_type="multipart/form-data",
)
```

Without the third tuple item, `file.mimetype` may be empty or inferred differently, causing MIME validation failures instead of testing the intended behavior.

## Generating a realistically large compressible/non-trivial image

Solid-color PNGs can be tiny even at very large dimensions. For tests that must exceed a byte threshold, generate noise:

```python
from PIL import Image

source = io.BytesIO()
image = Image.effect_noise((1800, 1800), 100).convert("RGB")
image.save(source, format="PNG")
uploaded = source.getvalue()
assert len(uploaded) > TARGET_BYTES
```

Tune dimensions so the generated image is above the compression target but below the endpoint's hard upload limit.

## Testing transform-before-external-call behavior

Monkeypatch the external service call and inspect the bytes handed to it:

```python
captured = {}

def fake_external_call(**kwargs):
    captured.update(kwargs)
    return {"combined": "ok", "descriptions": []}

monkeypatch.setattr(app_module, "describe_images", fake_external_call)

resp = client.post(...)
assert resp.status_code == 200
image = captured["images"][0]
assert image["mime_type"] == "image/jpeg"
assert len(image["bytes"]) <= TARGET_BYTES
assert len(image["bytes"]) < len(uploaded)
```

Also add the inverse regression test for small images to ensure they remain unchanged.
