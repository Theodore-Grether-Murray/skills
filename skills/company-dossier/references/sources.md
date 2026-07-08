# Dossier source extraction (for BrowserClaw `evaluate`)

Paste these into `mcp__BrowserClaw__evaluate` against each source's `page` id. Large returns get written to a file — read the returned path.

## Contents
- [Resolve name → LinkedIn slug](#resolve-name--linkedin-slug)
- [Company website (what they do)](#company-website-what-they-do)
- [LinkedIn company — About](#linkedin-company--about)
- [LinkedIn company — People](#linkedin-company--people)
- [Funding & news search](#funding--news-search)

## Resolve name → LinkedIn slug
On a LinkedIn company search page (`linkedin.com/search/results/companies/?keywords=<name>`):
```js
await new Promise(r=>setTimeout(r,4000));
const c=[...document.querySelectorAll('a[href*="/company/"]')].map(a=>({name:(a.innerText||'').split('\n')[0].trim(), slug:(a.href.match(/\/company\/([^\/?]+)/)||[])[1]})).filter(x=>x.slug);
return [...new Map(c.map(x=>[x.slug,x])).values()].slice(0,6);
```
Pick by name/industry match, then navigate to `linkedin.com/company/<slug>/about/` and `/people/`.

## Company website (what they do)
```js
await new Promise(r=>setTimeout(r,4500));
const heads=[...document.querySelectorAll('h1,h2')].map(h=>h.innerText.trim()).filter(Boolean).slice(0,14);
const nav=[...document.querySelectorAll('nav a, a[href*="product"]')].map(a=>a.innerText.trim()).filter(t=>t&&t.length<28);
return {title:document.title, h1:document.querySelector('h1')?.innerText?.trim(), headings:heads, products:[...new Set(nav)].slice(0,12), body:document.body.innerText.replace(/\s+/g,' ').slice(0,900)};
```
On an `/about` page, also scan for funding language:
```js
const t=document.body.innerText.replace(/\s+/g,' ');
return {funding:[...t.matchAll(/(raised|series [a-e]|\$[\d.]+ ?[mb]|valuation|led by|investors?|backed by)[^.]{0,120}/gi)].map(m=>m[0]).slice(0,10), story:t.slice(0,900)};
```

## LinkedIn company — About
`linkedin.com/company/<slug>/about/`:
```js
await new Promise(r=>setTimeout(r,5000));
const t=document.body.innerText.replace(/\r/g,'');
const g=re=>{const m=t.match(re);return m?m[1].trim():null;};
return {
  overview: t.split('\n').filter(l=>l.trim().length>40).slice(0,3),
  website: g(/Website\s*\n?([^\n]+)/i), industry: g(/Industry\s*\n?([^\n]+)/i),
  size: g(/Company size\s*\n?([^\n]+)/i), hq: g(/Headquarters\s*\n?([^\n]+)/i),
  founded: g(/Founded\s*\n?([^\n]+)/i), specialties: g(/Specialties\s*\n?([^\n]+)/i),
  followers: (t.match(/([\d,]+)\s+followers/)||[])[1]
};
```
Note: the "Company size" band (e.g. 51–200) is coarse; the People page gives a truer count.

## LinkedIn company — People
`linkedin.com/company/<slug>/people/`:
```js
await new Promise(r=>setTimeout(r,5500));
window.scrollBy(0,1200); await new Promise(r=>setTimeout(r,1500));
const people=[...document.querySelectorAll('a[href*="/in/"]')].map(a=>(a.innerText||'').split('\n').map(s=>s.trim()).filter(Boolean)[0]).filter(n=>n&&n!=='LinkedIn Member'&&n.length<40);
return {totalEmployees:(document.body.innerText.match(/([\d,]+)\s+(employees|associated members)/i)||[])[1], sample:[...new Set(people)].slice(0,15)};
```
For leadership specifically, `grep`/read for titles like "CEO", "Founder", "CTO" — the People page surfaces roles inconsistently, so the site's about/team page or the funding articles often name founders more reliably.

## Funding & news search
`google.com/search?q=<name>+funding+valuation+news` — grab the AI overview, result titles + snippets, AND real article hrefs (for the dossier's links):
```js
await new Promise(r=>setTimeout(r,4000));
const consent=/before you continue|consent|accept all/i.test(document.body.innerText.slice(0,200));
const titles=[...document.querySelectorAll('h3')].map(h=>h.innerText.trim()).filter(t=>t.length>12).slice(0,10);
const snippets=[...document.querySelectorAll('.VwiC3b, div[data-sncf], div[style*="line-clamp"]')].map(e=>e.innerText.trim()).filter(Boolean).slice(0,8);
const links=[];
document.querySelectorAll('a[href^="http"]').forEach(a=>{const h=a.href,t=(a.innerText||'').trim().split('\n')[0];
  if(t.length>12 && !/google\.|gstatic|\/search\?/.test(h) && /crunchbase|sacra|pitchbook|techcrunch|sifted|tracxn|forge|stockanalysis|\/blog\//i.test(h) && !links.find(l=>l[1]===h.split('&')[0])) links.push([t.slice(0,60),h.split('&')[0]]);});
const aiOverview=(document.body.innerText.match(/AI Overview([\s\S]{0,400})/)||[])[1];
return {consent, aiOverview:aiOverview?.replace(/\s+/g,' ').slice(0,400), titles, snippets, links:links.slice(0,10)};
```
If consent-blocked, dismiss the popup (click "Accept all"/"Reject all") and re-run, or fall back to opening a Crunchbase/Sacra/PitchBook profile directly. **Cross-check** the figures — Crunchbase/PitchBook/Sacra and news often disagree; report the most recent and flag the spread.
