# Scripting BrowserClaw: `run` vs `evaluate`

Two escape hatches let you collapse many tool round-trips into one call. Choosing correctly matters.

## `evaluate` — JavaScript inside the page

Runs in the page's own context (CDP `Runtime.evaluate`), with access to `document`, `window`, and the page's globals. Use it for DOM reads and small in-page scripts that are awkward to express with `read`/`grep`.

```js
// evaluate(page, code)
return [...document.querySelectorAll('h2')].map(h => h.textContent.trim());
```

```js
// Pull structured data straight out of the DOM
return [...document.querySelectorAll('.product')].map(el => ({
  name: el.querySelector('.name')?.textContent?.trim(),
  price: el.querySelector('.price')?.textContent?.trim(),
}));
```

- `return` a value to read it back; it's serialized for you.
- Async-capable — you can `await fetch(...)` from the page's origin (inherits its cookies/session).
- Runs in *one* page. It cannot open tabs, navigate, or drive the browser.

## `run` — JavaScript in the server runtime with the `browser` SDK

Runs on the **server side**, orchestrating the browser across pages. Use it for multi-step flows and extraction that would otherwise be a dozen snapshot/act calls: paginate-and-collect, fill-many-across-steps, crawl several pages, etc.

`console.log` is captured; `return` a value to read it back. **Exceptions come back as a result, not a thrown error** — so inspect the returned value to see what happened.

### The `browser` SDK surface

```
browser.pages.list()                     // open pages
browser.pages.newPage(url)               // open a page, returns its id
browser.pages.close(pageId)
browser.pages.getInfo(pageId)

browser.observe(pageId).snapshot()       // -> { text, refs }
browser.observe(pageId).diff()           // -> { text, added, removed, changed }
browser.observe(pageId).resolveRef(ref)

browser.input(pageId).click(ref)
browser.input(pageId).fill(ref, value)
browser.input(pageId).type(text)
browser.input(pageId).press(key)
browser.input(pageId).hover(ref)
browser.input(pageId).selectOption(ref, value)
browser.input(pageId).scroll(dir, amount, ref?)

browser.nav(pageId).goto(url)
browser.nav(pageId).back()
browser.nav(pageId).forward()
browser.nav(pageId).reload()

browser.cdp(method, params?, sessionId?)             // raw CDP escape hatch
browser.cdpJsonForPage(pageId, method, paramsJson)   // page-scoped raw CDP
```

Refs (`eN`) come from a snapshot's `text`/`refs`. Snapshotting inside `run` gives you refs to feed straight into `browser.input(...)`.

### Example: paginate and collect

```js
const pageId = (await browser.pages.list())[0].id;
const rows = [];
for (let i = 0; i < 5; i++) {
  const snap = await browser.observe(pageId).snapshot();
  // ... parse snap.text / use the page's DOM via a nested evaluate if needed ...
  const next = snap.refs.find(r => /next/i.test(r.label));
  if (!next) break;
  await browser.input(pageId).click(next.ref);
}
return rows;
```

### Example: open a page, read it, return data

```js
const { id } = await browser.pages.newPage('https://example.com/list');
const snap = await browser.observe(id).snapshot();
return snap.text.slice(0, 2000);
```

## Which one?

- Need the page's **DOM/JS in a single page** (query selectors, read state, call the site's own fetch)? → `evaluate`.
- Need to **drive the browser** — open/close tabs, navigate, click across multiple pages, loop a flow? → `run`.
- Simple one-shot interaction? → skip both; use `snapshot` + `act`.

## Raw CDP

When neither the SDK nor page JS is enough, `browser.cdp(method, params)` (or `browser.cdpJsonForPage`) exposes the raw Chrome DevTools Protocol — network interception, emulation, tracing, etc. Reach for it only when a specific CDP domain is genuinely required; it's the lowest-level, least-portable option.
