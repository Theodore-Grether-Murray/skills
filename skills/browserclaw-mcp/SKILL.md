---
name: browserclaw-mcp
description: Use whenever you need to drive a real web browser through the BrowserClaw MCP (tools named mcp__BrowserClaw__*) — navigating to URLs, reading or scraping page content, filling forms, clicking through flows, logging in, extracting data, capturing screenshots/PDFs, downloading files, or automating any multi-step web interaction. Reach for this whenever a task involves a live website and BrowserClaw tools are available, even if the user just says "go to this page", "scrape this", "fill out this form", "click through", or "check this site" without naming BrowserClaw. It teaches the snapshot→act→diff loop, which tool to pick, and how to avoid the common failure modes.
---

# Working with the BrowserClaw MCP

BrowserClaw drives a **real Chromium browser over CDP** (Chrome DevTools Protocol). It is not an HTTP fetcher — it runs a live page with JavaScript, cookies, and session state. That makes it right for anything a plain fetch can't do: JS-rendered content, logged-in pages, multi-step flows, downloads, and visual verification.

Every tool operates on a specific tab, identified by a numeric **`page` id**. Your first move in any session is always to get a page id (see below). If you skip that, tool calls fail.

## First, split the task: reading vs. interacting

Before you reach for any tool, decide which kind of task you have — they take different paths, and picking the wrong one is the most common way to waste time:

- **Just reading / extracting** (scrape an article, pull a list, grab a table, read page state)? Get a page id, then go straight to **`read`** (for prose/markdown) or **`evaluate`** (to query the DOM and return structured data). One `evaluate` that returns exactly the fields you want usually beats a whole snapshot→act loop. Don't snapshot a page you only need to *read*.
- **Interacting** (click, fill, log in, multi-step flow)? Use the snapshot→act→diff loop below.

This split matters: the accessibility snapshot is for *finding things to act on*. If your goal is data, not clicks, `read`/`evaluate` are the direct route.

## The core loop (for interaction)

Driving a page with BrowserClaw is a perception→action→confirmation rhythm:

```
snapshot  →  act  →  (read the diff)  →  act  →  ...
```

1. **`snapshot(page)`** returns an indented accessibility tree. Every actionable element carries a stable handle like `[ref=e12]`. This is how you "see" the page's *structure* — cheaper and more precise than a screenshot for deciding what to click or type into.
2. **`act(page, kind, ref, ...)`** performs an interaction (click, fill, press, etc.) using a `ref` from the latest snapshot. It automatically **reads back a diff** of what changed — so you usually don't need to re-snapshot to see the effect.
3. If the diff isn't enough, **`diff(page)`** replays "what changed since the last snapshot/diff" without re-dumping the whole tree.

**Refs are ephemeral.** A `[ref=eN]` is only valid against the snapshot it came from. Navigation (`navigate`) and large DOM changes invalidate every ref — after those, take a fresh `snapshot` before acting again. If an `act` fails with a stale/missing ref, re-snapshot; don't guess ref numbers.

**If refs don't come back, don't fight the loop.** Depending on the page and environment, a `snapshot` may not surface usable `[ref=eN]` handles (content can render server-side, iframes can be opaque, etc.). When that happens, ref-based `act` has nothing to target and you'll waste calls retrying. Switch strategies immediately: use **`evaluate`** to read state and drive the page directly (set input `value`s, click by selector, submit a form), and `wait for:"selector"`/`"text"` to confirm. `evaluate` runs real JS in the page, so it works whether or not the snapshot exposed refs. Reserve minutes-long ref debugging for never.

**Prefer the diff over re-snapshotting.** Re-dumping a big page on every step is slow and floods your context. Act, read the diff it returns, and only take a full snapshot when the page changed structurally or a ref went stale.

## Step 0: get a page id

Do exactly one of these before anything else:

- **`tabs(action: "list")`** — see all open pages and their ids. Use `action: "active"` for the focused one. If a relevant tab is already open, reuse its id.
- **`tabs(action: "new", url: "...")`** — open a fresh tab and get its id. `background: true` (default) opens without stealing focus; `hidden: true` uses a hidden window.
- **`navigate(page, action: "url", url: "...")`** — if you already have a page id, load a URL into it. `navigate` returns a fresh snapshot, so you don't need a separate `snapshot` call right after.

Almost every other tool requires the `page` argument, so keep the id handy.

## Choosing the right tool

Pick the cheapest tool that answers your question. Group them by intent:

**Orient / navigate**
- `tabs` — list/open/close/switch tabs, get page ids.
- `navigate` — load a URL, or go `back` / `forward` / `reload`. Returns a fresh snapshot.

**Perceive (read the page)**
- `snapshot` — accessibility tree with `[ref=eN]` handles. Use this to decide what to act on.
- `grep` — search the page without dumping it. `over: "ax"` searches snapshot lines (matches keep their refs — great for *finding the ref* of a button/field by label); `over: "content"` searches visible text. Use this instead of a full snapshot when the page is large and you know what you're looking for.
- `read` — extract page content as `markdown` (default), `text`, or `links`. This is the tool for **scraping/reading**, not acting. Use `selector` to restrict to a CSS subtree; `format: "links"` to harvest every link for crawling.
- `screenshot` — a visual capture (JPEG by default). Use for *visual* verification, layout/design questions, or when the user wants to see the page. For deciding what to click, prefer `snapshot` — it's cheaper and gives you refs. `annotate: true` overlays numbered refs onto the image.
- `diff` — what changed since the last snapshot/diff. The cheap way to confirm an action's effect.

**Act (change the page)**
- `act` — the workhorse. One tool, many `kind`s: `click`, `type` (into the focused element), `fill` (one field via `ref`+`value`, or many at once via `fields[]`), `press` (a key/combo like `Enter` or `Control+a`), `hover`, `focus`, `check`/`uncheck`, `select` (a dropdown option value), `scroll`, `drag`. There are also coordinate-based `*_at` kinds (`click_at`, `type_at`, `hover_at`, `drag_at`) for the rare case where an element has no usable ref — pass `x`/`y` viewport coordinates. Prefer ref-based kinds; fall back to `_at` only when necessary.
- `wait` — pause before continuing. Prefer acting and reading the diff; only wait when there's no reliable UI signal yet. `for: "text"` waits for a substring to appear, `for: "selector"` for a CSS match, `for: "time"` (default) for a fixed pause (~2s default). Waiting on `text`/`selector` beats a blind time pause.
- `upload` — set file path(s) on an `<input type="file">` via its ref. Files must exist on the **server** filesystem.
- `download` — click an element (by ref) that triggers a download; saves the file and returns its path.

**Script (often the fastest path, not just an escape hatch)**
- `evaluate` — run JS *inside the page* (CDP `Runtime.evaluate`). This is a **primary tool**, not a last resort: for extracting structured data it's usually the single cheapest call (`return document.querySelectorAll(...)` mapped to the fields you want), and it's the reliable way to read state or drive inputs when a snapshot doesn't give you refs. `return` a value to read it back.
- `run` — run JS in the **server runtime** against a `browser` SDK, for multi-step flows and extraction that would otherwise take many round-trips (paginate-and-collect, crawl several pages). See `references/scripting.md`.

**Capture / output**
- `screenshot` — image of the page.
- `pdf` — print the page to a PDF file (archiving, or reading a page as a document). For plain text, prefer `read`.
- `download` — save a file the page offers.

**Manage the workspace**
- `tab_groups` — group/label/collapse tabs.
- `windows` — list/create/close/activate windows; create hidden ones.

For exact parameters of every tool, see `references/tool-reference.md`.

## Principles that keep you fast and correct

- **Match the tool to the goal.** If you need *data*, `read`/`evaluate` are the direct route — don't snapshot a page you only mean to read. If you need to *act*, snapshot for refs and use `act`. If refs don't appear, fall to `evaluate`. Reaching for the interaction loop on a pure-extraction task is the most common source of wasted calls.
- **Structure before pixels.** Default to text tools (`snapshot`/`grep`/`read`/`evaluate`) over `screenshot` (an image). Reserve screenshots for genuinely visual questions or final proof for the user. Text is cheaper, more precise, and (for snapshots) gives you refs to act on.
- **Search, don't dump.** On a big page, `grep` for the label or text you need instead of snapshotting the whole tree. `grep over:"ax"` returns matching lines *with their refs*, so you often go straight from "find the Submit button" to acting on it.
- **Let `act` report back.** `act` returns a diff automatically. Read it instead of immediately re-snapshotting. Re-snapshot only after navigation, a stale ref, or a big structural change.
- **Re-snapshot after navigation.** `navigate` (and any full page load) invalidates all refs. `navigate` conveniently returns a fresh snapshot; if you loaded the page another way, snapshot before acting.
- **Wait on signals, not clocks.** Prefer `wait for:"text"`/`"selector"` (or just act and check the diff) over `for:"time"`. A fixed sleep is a last resort when nothing observable tells you the page is ready.
- **Batch multi-step work with `run`.** If a task would take a dozen snapshot/act round-trips (paginate + collect, fill many fields across steps, extract a table), one `run` script against the `browser` SDK is far faster and cleaner. See `references/scripting.md`.
- **`evaluate` runs in the page; `run` runs on the server.** `evaluate` is for reading/poking the page's own DOM/JS. `run` orchestrates the browser (navigate, snapshot, click across pages) from the server side.
- **Reuse tabs.** Check `tabs(action:"list")` before opening new ones — the page you need may already be open, and piling up tabs wastes resources.

## Common recipes

**Read/scrape an article or page**
1. `tabs(action:"new", url)` → get page id. 2. `read(page, format:"markdown")` for prose (restrict noise with `selector` like `"main"`/`"article"`; `format:"links"` to pull every URL). For *structured* data (titles + URLs, rows of fields), one `evaluate` that queries the DOM and returns exactly those fields is usually the cheapest, cleanest option. No snapshot needed for reading.

**Fill and submit a form**
1. `navigate`/`tabs` to the page. 2. `snapshot` (or `grep over:"ax"` for the field labels) to get refs. 3. `act kind:"fill", fields:[{ref, value}, ...]` to fill several fields in one call. 4. `act kind:"click", ref:<submit>`. 5. Read the diff (or `wait for:"text"` on a success message) to confirm. **If the snapshot yields no usable refs**, do the same flow with `evaluate` — set the inputs' `value`s and submit — then `wait for:"selector"` on the result.

**Multi-step flow / login**
1. Fill username/password and submit (via refs, or via `evaluate` if refs aren't surfaced). 2. `wait for:"text"`/`"selector"` on something only the logged-in page shows (or read the diff). 3. After the navigation, refs are invalid — re-snapshot (or keep using `evaluate`) before the next step. Never hardcode ref numbers across a navigation.

**Extract a table or repeated list efficiently**
For a single page, one `evaluate` that maps the rows to structured objects is typically enough. For data spread across pages, use one `run` script that loops in the server runtime (read/collect → paginate → repeat) instead of many manual round-trips. Fallback: `read` the page and parse the markdown.

**Verify something visually / show the user**
`screenshot(page)` (add `fullPage:true` for the whole page). Use `pdf` to hand back a document-style capture.

**Handle a file upload/download**
Upload: snapshot to find the `<input type="file">` ref → `upload(page, ref, file)`. Download: find the download trigger's ref → `download(page, ref)` → use the returned path.

## When things go wrong

- **"ref not found" / action had no effect** → the ref is stale. Take a fresh `snapshot` and use the new ref. This is the single most common failure; when in doubt, re-snapshot.
- **Content missing from snapshot/read** → it may be lazy-loaded. `scroll` toward it (`act kind:"scroll"`) or `wait for:"text"`/`"selector"`, then re-perceive.
- **Element is in an iframe** → snapshots stitch iframe content inline, so refs still work. If you're scripting, note the frame boundary.
- **Page seems stuck loading** → `wait for:"selector"` on a known element, or `navigate action:"reload"`.
- **A flow needs many fiddly steps** → stop hand-driving it; write a `run` script (see `references/scripting.md`).

Keep it tight: first decide read-vs-interact, use the cheapest tool that answers the question (`read`/`evaluate` for data, snapshot→act→diff for clicks), confirm with the diff, and don't grind on a ref loop that isn't producing refs — `evaluate` is right there.
