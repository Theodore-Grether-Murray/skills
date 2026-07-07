# X scrape payloads (for BrowserClaw `evaluate`)

Paste these into `mcp__BrowserClaw__evaluate` (the `code` field) against the X `page` id. They're written to be resilient to X's virtualized DOM and to X's frequent markup churn. Each `return`s a value; if the value is large, BrowserClaw writes it to a file and returns the path — read that file.

## Contents
- [Login / state check](#login--state-check)
- [For You feed scrape](#for-you-feed-scrape)
- [Reload + Retry recovery](#reload--retry-recovery)
- [Explore: News cards](#explore-news-cards)
- [Explore: Trending topics](#explore-trending-topics)
- [Engagement aria-label reference](#engagement-aria-label-reference)

---

## Login / state check
After opening `https://x.com/home`:
```js
await new Promise(r=>setTimeout(r,4000));
return {
  url: location.href,
  loggedIn: !!document.querySelector('[data-testid="SideNav_AccountSwitcher"], [data-testid="SideNav_AccountSwitcher_Button"]'),
  onAuthwall: /login|i\/flow|logout/.test(location.href),
  tabs: [...document.querySelectorAll('[role="tab"]')].map(t=>t.innerText.trim()).filter(Boolean),
  timelineError: /Something went wrong/.test(document.body.innerText)
};
```
`loggedIn:false` with a "Home / X" title and your name in `document.body.innerText` usually just means the selector is stale, not that you're logged out — check `onAuthwall` (a real logout redirects to `/login` or `/i/flow`).

## For You feed scrape
Ensures the **For you** tab is active, scrolls to accumulate, and extracts structured tweets sorted by reach. Tune the loop count for a bigger/smaller sample.
```js
// Activate "For you"
const tab = [...document.querySelectorAll('[role="tab"]')].find(t=>/for you/i.test(t.innerText));
if (tab && tab.getAttribute('aria-selected') !== 'true') tab.click();
await new Promise(r=>setTimeout(r,2500));

const now = Date.now();
const seen = new Set();
const tweets = [];
function parseCounts(label){
  if(!label) return {};
  const m = re => { const x = label.match(re); return x ? +x[1].replace(/,/g,'') : null; };
  return {
    replies:  m(/([\d,]+)\s+repl/i),
    reposts:  m(/([\d,]+)\s+repost/i),
    likes:    m(/([\d,]+)\s+like/i),
    views:    m(/([\d,]+)\s+view/i),
    bookmarks:m(/([\d,]+)\s+bookmark/i),
  };
}
function collect(){
  document.querySelectorAll('article').forEach(a=>{
    const textEl = a.querySelector('[data-testid="tweetText"]');
    const text = textEl ? textEl.innerText.trim() : '';
    const timeEl = a.querySelector('time');
    const dt = timeEl ? timeEl.getAttribute('datetime') : null;   // null age ≈ promoted ad
    const link = timeEl && timeEl.closest('a') ? timeEl.closest('a').href : null;
    const userEl = a.querySelector('[data-testid="User-Name"]');
    const user = userEl ? userEl.innerText.replace(/\n·.*/,'').replace(/\n/g,' ').trim() : '';
    const grp = a.querySelector('[role="group"][aria-label]');
    const c = parseCounts(grp ? grp.getAttribute('aria-label') : null);
    const key = link || text.slice(0,60);
    if(!key || seen.has(key) || !text) return;
    seen.add(key);
    tweets.push({user, ageH: dt ? +((now-new Date(dt))/3.6e6).toFixed(1) : null, ...c, link, text: text.slice(0,300)});
  });
}
for(let i=0;i<8;i++){ collect(); window.scrollBy(0,2800); await new Promise(r=>setTimeout(r,1700)); }
collect();
tweets.sort((a,b)=>((b.views||b.likes*30||0)-(a.views||a.likes*30||0)));
return {count: tweets.length, tweets};
```
**Filtering after the scrape:** drop items with `ageH === null` and a brand handle — those are ads/promoted. Keep `ageH ≤ 12` as primary; flag anything older you choose to include.

## Reload + Retry recovery
If the login check shows `timelineError:true`, or the For You scrape returns `count:0`, recover before scraping:
1. `navigate(page, action:"reload")`
2. Then run:
```js
await new Promise(r=>setTimeout(r,6000));
const retry = [...document.querySelectorAll('span,button')].find(e=>/^Retry$/.test(e.innerText?.trim()));
if (retry) retry.click();
await new Promise(r=>setTimeout(r,4000));
window.scrollBy(0,1500);
await new Promise(r=>setTimeout(r,3000));
return { articles: document.querySelectorAll('article').length, err: /Something went wrong/.test(document.body.innerText) };
```
Only proceed to the For You scrape once `articles > 0` and `err:false`.

## Explore: News cards
`navigate` to `https://x.com/explore/tabs/news`, then:
```js
await new Promise(r=>setTimeout(r,5500));
const seen=new Set(); const items=[];
function collect(){
  document.querySelectorAll('[data-testid="cellInnerDiv"]').forEach(c=>{
    const t=c.innerText.trim(); if(!t) return;
    const key=t.slice(0,80); if(seen.has(key)) return; seen.add(key);
    items.push(t.split('\n').map(s=>s.trim()).filter(Boolean).slice(0,6).join(' | '));
  });
}
for(let i=0;i<5;i++){ collect(); window.scrollBy(0,2200); await new Promise(r=>setTimeout(r,1400)); }
collect();
return {tab:[...document.querySelectorAll('[role="tab"]')].map(t=>t.innerText.trim()), count:items.length, items};
```
News cards come back like `"Headline | 18 hours ago · News · 14K posts"` — parse the age and post count from that line for freshness + volume.

## Explore: Trending topics
`navigate` to `https://x.com/explore/tabs/trending`, then run the same loop as News (it also uses `[data-testid="cellInnerDiv"]`). Rows look like `"9 | · | Technology · Trending | LovescapeAI | Trending with Grok-user"` — rank number, category, topic. Trending gives topics without volumes; use it to spot clusters (e.g. many rows about one live event) and category (Technology/Politics/Sports).

## Engagement aria-label reference
The `[role="group"]` on each tweet carries a single aria-label with all counts, e.g.:
`"1086 replies, 5022 reposts, 24428 likes, 16777 bookmarks, 7710103 views"`
Some counts may be absent (views hidden on older tweets; bookmarks/reposts zero). The `parseCounts` helper above handles missing fields by returning `null` for them.
