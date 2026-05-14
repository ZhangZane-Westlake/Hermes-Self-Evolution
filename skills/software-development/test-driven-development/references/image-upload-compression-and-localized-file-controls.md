# Testing image upload compression and localized file controls

Use this reference when adding or fixing image-upload behavior in Flask/browser apps.

## Backend compression test pattern

For multipart upload endpoints that forward images to a vision model, mock the downstream model call and assert the bytes handed to it, not the HTTP response alone.

```python
def configure_vision(database):
    database.set_config("vision_api_key", "sk-vision")
    database.set_config("vision_base_url", "https://vision.example/v1")
    database.set_config("vision_model", "vision-test")


def test_upload_compresses_large_images(client, app_modules, monkeypatch):
    app_module, database = app_modules
    configure_vision(database)

    from PIL import Image
    source = io.BytesIO()
    # Random/noise image avoids PNG compressing too well; keep under server max upload limit.
    Image.effect_noise((1800, 1800), 100).convert("RGB").save(source, format="PNG")
    uploaded = source.getvalue()
    assert len(uploaded) > app_module.TARGET_VISION_IMAGE_BYTES
    assert len(uploaded) < app_module.MAX_IMAGE_BYTES

    captured = {}
    monkeypatch.setattr(
        app_module,
        "describe_images",
        lambda **kwargs: captured.update(kwargs) or {"descriptions": [], "combined": "ok"},
    )

    resp = client.post(
        "/api/vision/describe",
        data={"images": (io.BytesIO(uploaded), "huge.png", "image/png")},
        content_type="multipart/form-data",
    )

    assert resp.status_code == 200
    image = captured["images"][0]
    assert image["mime_type"] == "image/jpeg"
    assert len(image["bytes"]) <= app_module.TARGET_VISION_IMAGE_BYTES
    assert len(image["bytes"]) < len(uploaded)
```

Also add a small-image regression asserting small uploads keep their original bytes and MIME type.

## Frontend localized file chooser pattern

Browsers localize native `<input type="file">` inconsistently (often showing English "Choose File"). To guarantee Chinese copy:

```html
<div class="file-picker">
  <input type="file" id="image-files" accept="image/*" multiple>
  <label for="image-files" class="file-picker-button">选择图片</label>
  <span id="image-files-label" class="file-picker-label">未选择图片</span>
</div>
```

Hide the native input accessibly, not with `display:none`, so the label still opens the picker:

```css
.file-picker input[type="file"] {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

Add JS to update the label to `未选择图片`, filename for one file, or `已选择 N 张图片` for multiple files. Include a static test that reads template/JS text and asserts `choose file` is absent and Chinese strings are present.
