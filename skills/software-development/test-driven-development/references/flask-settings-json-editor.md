# Flask settings JSON editor pattern

Use this when a Flask app already has structured settings stored in a database/config table, but the UI should expose an "edit setting file" affordance.

## Backend pattern

1. Keep runtime config reads unchanged. Add a file-like JSON view over the same config store rather than creating a second source of truth.
2. Add two endpoints:
   - `GET /api/settings/<name>-file` returns `{ filename, content, settings }`
   - `PUT /api/settings/<name>-file` accepts `{ content }`, parses JSON, validates shape, persists supported keys, and returns the normalized JSON.
3. Split helpers:
   - `_settings_file_data() -> dict`: builds a JSON-serializable object from existing config values and defaults.
   - `_apply_settings_file_data(settings: dict)`: maps JSON fields back to config keys.
4. Validate enough to protect users:
   - invalid JSON -> 400 with a localized error
   - non-object root -> 400
   - non-object sections -> 400
   - ignore unknown keys unless strict schema is required
5. Return normalized JSON after saving so the UI can replace the editor content with the exact persisted shape.

## Frontend pattern

1. Add a settings card with:
   - localized "edit setting file" button
   - hidden editor container
   - textarea with monospace/JSON-friendly editing
   - save and cancel buttons
2. On edit click: `GET` the endpoint, fill textarea, show editor, focus textarea.
3. On save: `PUT` `{ content: textarea.value }`, show backend errors verbatim/localized, reload the normal settings form after success.
4. Keep existing field-based settings UI in sync by calling the normal `loadSettings()` after saving the JSON editor.

## TDD checklist

Backend tests:
- GET returns filename, raw content string, parsed settings object, and default model values.
- PUT valid JSON persists values to the existing runtime config store.
- PUT invalid JSON returns 400 with an error mentioning JSON.
- PUT invalid shape returns 400.

Frontend/static tests:
- Template contains button, editor container, textarea, save/cancel controls.
- JS contains named load/save functions and the correct endpoint path.

## Pitfall

Do not write a separate physical settings file unless the app truly loads runtime config from that file. If runtime calls read SQLite/env/config-table values, a separate file creates drift. A file-like JSON editor over the existing store gives users the desired editing UX without a second source of truth.
