# BrowserClaw tool reference

Exact parameters for every `mcp__BrowserClaw__*` tool. All tools except `tabs`/`tab_groups`/`windows` require a `page` id (get one from `tabs` or `navigate`).

## Contents
- [Navigation & tabs](#navigation--tabs)
- [Perceiving the page](#perceiving-the-page)
- [Acting on the page](#acting-on-the-page)
- [Scripting](#scripting)
- [Output & files](#output--files)
- [Windows & groups](#windows--groups)

---

## Navigation & tabs

### `tabs`
Manage tabs. `action`: `list` (default — all pages + ids), `active` (focused page), `new`, `close`.
- `url` — for `new` (defaults `about:blank`).
- `background` (default `true`) — open `new` without stealing focus.
- `hidden` (default `false`) — create `new` in a hidden window.
- `page` — required for `close`.

### `navigate`
`action`: `url` (default), `back`, `forward`, `reload`. Requires `page`.
- `url` — required when `action:"url"`.
- Returns a **fresh snapshot** of the resulting page. Navigation invalidates all prior `[ref=eN]` handles.

---

## Perceiving the page

### `snapshot`
`page` only. Returns the accessibility tree, indented, with `[ref=eN]` on actionable elements. Iframe content is stitched inline. Re-snapshot after navigation or large DOM changes. This is the start of the loop.

### `grep`
Search the page without dumping it.
- `pattern` (required) — case-insensitive regex.
- `over` — `ax` (default: snapshot lines; **matches keep their `[ref=eN]`**) or `content` (visible text).
- `limit` — max matching lines (default 50).

### `read`
Extract content. For reading/scraping, not acting.
- `format` — `markdown` (default), `text`, or `links`.
- `selector` — restrict to a CSS subtree.
- `includeImages` / `includeLinks` — for markdown, include image refs / render links.
- `viewportOnly` — markdown of only the visible viewport.

### `screenshot`
- `format` — `jpeg` (default), `png`, `webp`.
- `quality` — 0–100 (jpeg/webp).
- `fullPage` — capture beyond the viewport.
- `size` — `{width, height}`, default 1024×768 (max 4096).
- `annotate` — overlay numbered refs from a fresh snapshot.
- Prefer `snapshot` for structure/actions; use screenshots for visual questions.

### `diff`
`page` only. Shows what changed since the last snapshot/diff. Cheap way to confirm an action's effect.

---

## Acting on the page

### `act`
Requires `page` + `kind`. Reads back a diff of what changed.

`kind` values:
- `click` — click `ref`. Optional `button` (`left`/`middle`/`right`), `clickCount`.
- `type` — type `text` into the currently focused element.
- `fill` — set a field. Either `ref` + `value` for one field, or `fields: [{ref, value}, ...]` to fill many in order. `clear` empties first.
- `press` — a `key`/combo, e.g. `"Enter"`, `"Control+a"`.
- `hover` / `focus` — on `ref`.
- `check` / `uncheck` — a checkbox `ref`.
- `select` — choose option `value` on a `<select>` `ref`.
- `scroll` — `direction` (`up`/`down`/`left`/`right`), `amount` (wheel notches, default 3), optional `ref` to scroll within an element.
- `drag` — from `ref` to `targetRef`.
- Coordinate variants (`click_at`, `type_at`, `hover_at`, `drag_at`) — use viewport `x`/`y` (and `startX/startY`/`endX/endY` for drag) when an element has no usable ref. Prefer ref-based kinds.

### `wait`
- `for` — `time` (default), `text`, or `selector`.
- `value` — for `time`, ms to pause (default 2000); for `text`/`selector`, the substring/CSS selector.
- `timeout` — max wait in ms (default 2000). Prefer acting + reading the diff over waiting.

### `upload`
Set file(s) on an `<input type="file">`.
- `ref` (required) — the file input element.
- `file` — single path, or `files` — array of paths. Files must exist on the **server** filesystem.

### `download`
Click a `ref` that triggers a download; saves it and returns `{path, filename}`.

---

## Scripting

### `evaluate`
Run JS **inside the page** via CDP `Runtime.evaluate`.
- `code` — async-capable body; `return` a value to read it back.
- `timeout` — default 30000ms.
- Use for DOM reads / small page scripts awkward with read/grep.

### `run`
Run JS in the **server runtime** against the `browser` SDK for multi-step flows and extraction. See `scripting.md` for the SDK surface.
- `code` — async-capable body; top-level `await` and `return` supported. `console.log` is captured. Exceptions come back as a *result*, not a thrown error.
- `timeout` — default 30000ms.

---

## Output & files

### `pdf`
Print the page to a PDF file; returns the path.
- `landscape`, `printBackground` (alias `background`), `preferCSSPageSize`.
- For extracting text, prefer `read`.

---

## Windows & groups

### `tab_groups`
`action`: `list` (default), `create`, `update`, `ungroup`, `close`.
- `pages` — page ids for `create`/`ungroup`.
- `groupId` — for `update`/`close` (optional on `create` to add to an existing group).
- `title`, `color` (`grey`/`blue`/`red`/`yellow`/`green`/`pink`/`purple`/`cyan`/`orange`), `collapsed` — for `create`/`update`.

### `windows`
`action`: `list` (default), `create`, `close`, `activate`, `set_visibility`.
- `hidden` — create a hidden window.
- `windowId` — for `close`/`activate`/`set_visibility`.
- `visible` — target visibility for `set_visibility`; `activate` to focus after showing.
