# Building a Google Doc through BrowserClaw

Google Docs is a **canvas-based editor**: it renders text to a canvas and captures keystrokes through a hidden input. That means two things:
1. You **cannot** insert text with `evaluate` / `innerText` / `execCommand` / synthetic paste events — Docs ignores them.
2. You **can** type into it with **real CDP keystrokes** via `act` (`kind:"type"` / `kind:"press"`), which is how a human types. That's the whole trick.

This recipe is the proven path. Follow it in order.

## Prerequisite
The user must be signed into Google in the BrowserClaw browser (a new doc opens straight into the editor; if it redirects to `accounts.google.com`, stop and tell them to sign in — don't try to authenticate for them).

## 1. Create the doc
```
tabs(action:"new", url:"https://docs.new", background:false)
```
Then wait and confirm it's the editor (not a sign-in page):
```js
await new Promise(r=>setTimeout(r,6000));
return { url:location.href, isDoc:/document\/d\//.test(location.href),
         canvas:!!document.querySelector('.kix-appview-editor'),
         signin:/accounts\.google|sign in/i.test(location.href+document.title) };
```
`isDoc:true` + `canvas:true` → good. Note the doc URL (`/document/d/<id>/edit`) to report later.

## 2. Click into the body
Get a click point inside the editing canvas and click it for real (programmatic focus won't stick):
```js
const p=document.querySelector('.kix-appview-editor'); const r=p.getBoundingClientRect();
return { x:Math.round(r.left+r.width/2), y:Math.round(r.top+80) };
```
Then `act(kind:"click_at", x, y)`.

## 3. Build the body with real heading hierarchy (this is what stops it feeling cluttered)

A flat wall of same-size text with a URL stacked under every line reads as cluttered — that's the #1 complaint. Fix it with Google Docs' built-in **heading styles**, applied via keyboard shortcuts *as you type*. This also populates the Doc's **outline panel** (left sidebar), which makes the whole thing skimmable at a glance.

The pattern: apply a paragraph style with a shortcut → type that paragraph → Enter (Docs auto-returns to normal body text after a heading). Build section by section rather than in one giant `type()`:

- **Title → Heading 1:** `act(kind:"press", key:"Meta+Alt+1")` (Mac; `Control+Alt+1` elsewhere), then `act(kind:"type", ...)` the title line + a one-line dek (the dek lands as normal text after the title's newline).
- **Each section header → Heading 2:** `act(kind:"press", key:"Meta+Alt+2")`, then `type` the header line followed by that section's item lines. Headings carry their own spacing, so you **don't** need blank lines between sections.
- Repeat the `Meta+Alt+2` + `type` pair for each section. A confirmation that the style took: the `act` result's `afterUrl` gains a `#heading=…` anchor.

**Item formatting — keep it lean:**
- **One concise line per story, with its single source link right after it.** URLs auto-link when followed by a newline. Keep it to **one primary link per item** — stacking multiple URLs per story is exactly what makes these Docs feel heavy.
- Prefer one sentence per item; push detail into the "Why it matters" section.
- **Don't hand-number** ("1) 2)") — Docs' autolist turns `1)` into a list and can double it ("2) 2)"). Use prose, or a real bulleted list via `Meta+Shift+8`.
- Skip emoji in the Doc (they look busy next to auto-links); the chat brief can keep them.

**On hiding URLs behind labels:** it's cleaner but each label needs the text selected + `Meta+k` (link dialog) + paste URL — roughly 4 actions *per link*, which doesn't scale across ~10 items. So **inline auto-linked URLs (one per line) are the reliable default.** If the user specifically wants clean labels, do the `Meta+k` flow for just the handful of most important links, not all of them.

## 4. Set the title (careful — this is where it goes wrong)
Docs **auto-fills the title from the first typed line** after a moment, so often you don't need to. If you want a custom title, do NOT click near the top-left and type — that click frequently misses the title field and your text lands in the body. Instead focus the real title input and type into it:
```js
const ti=document.querySelector('input.docs-title-input');
ti.focus(); ti.select();
return { focused: document.activeElement===ti, value: ti.value };
```
Only if `focused:true` → `act(kind:"type", text:"World News Brief — <date>")` then `act(kind:"press", key:"Enter")`. If it's not focused, leave the auto-filled title alone.

## 5. Verify, and fix stray text if needed
Screenshot the doc (`screenshot`) and read it. If earlier mis-clicks dropped stray text into the body (e.g. a title line at the end, or a stray horizontal rule from `---`/`—`):
- Put the cursor at the stray line (it's usually last), then `act(kind:"press", key:"Shift+Home")` to select the line, `act(kind:"press", key:"Backspace")` to delete it, and a couple more `Backspace` presses to remove the empty line / horizontal rule above it.
- Re-screenshot to confirm it ends where it should.

Docs autosaves ("Saved to Drive" appears top-left). The doc is **private to the user's account** by default; mention that, and offer to set a shareable link if they want to send it.

## Why real keystrokes (the one thing to remember)
Every reliable step here is a real mouse/keyboard event through `act`. Every failure mode (composer-style empty inserts, ignored `innerText`, uncommitted titles) comes from trying to shortcut that with `evaluate`. When in doubt: click it, then type it, like a person would.
