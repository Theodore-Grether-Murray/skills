# Person research extraction (for BrowserClaw `evaluate`)

Paste these into `mcp__BrowserClaw__evaluate` against each source's `page` id. Large returns get written to a file — read the returned path. For the person's **employer**, use the `company-dossier` skill's `sources.md` payloads (site + funding/news); you mainly need what-they-do + the latest round/news.

## Contents
- [Disambiguate on LinkedIn people search](#disambiguate-on-linkedin-people-search)
- [LinkedIn profile extraction](#linkedin-profile-extraction)
- [Person web search (with data-broker filter)](#person-web-search-with-data-broker-filter)

## Disambiguate on LinkedIn people search
`linkedin.com/search/results/people/?keywords=<name>` — return candidates with the cues you need to pick (headline, location, mutual-connection/degree):
```js
await new Promise(r=>setTimeout(r,5000));
const people=[];
document.querySelectorAll('a[href*="/in/"]').forEach(a=>{
  const first=(a.innerText||'').split('\n').map(s=>s.trim()).filter(Boolean)[0];
  const slug=(a.href.match(/\/in\/([^\/?]+)/)||[])[1];
  if(!first||first==='LinkedIn Member'||first.length>45||!slug) return;
  const card=a.closest('li')||a.parentElement?.parentElement;
  const ctx=(card?.innerText||'').replace(/\s+/g,' ').slice(0,180); // headline · location · "Nth" degree · mutuals
  if(!people.find(p=>p.slug===slug)) people.push({name:first, slug, ctx});
});
return people.slice(0,10);
```
Pick with every cue: the company/role the user named, **1st-degree** (usually the intended person), mutual connections, location. If several plausible and no cue → confirm with the user before a full pass.

## LinkedIn profile extraction
`linkedin.com/in/<slug>/` — headline, about, experience (with descriptions), education, recent activity:
```js
await new Promise(r=>setTimeout(r,5000));
const t=document.body.innerText.replace(/\r/g,'');
const sections={};
[...document.querySelectorAll('section')].forEach(s=>{
  const h=(s.querySelector('h2,.pvs-header__title')?.innerText||'').trim().split('\n')[0];
  if(/About|Experience|Education|Activity|Featured/i.test(h)) sections[h.slice(0,15)]=(s.innerText||'').replace(/\s+/g,' ').slice(0,900);
});
const pick=re=>Object.entries(sections).find(([k])=>re.test(k))?.[1]||null;
return {
  headlineArea: t.split('\n').map(s=>s.trim()).filter(Boolean).slice(0,14), // name, headline, company, school
  followers:(t.match(/([\d,]+)\s+followers/)||[])[1],
  about: pick(/About/i),
  experience: pick(/Experience/i),   // <- the descriptions here carry the tellable detail
  education: pick(/Education/i),
  activity: pick(/Activity/i)         // recent posts / reposts = current preoccupations
};
```
The **experience descriptions** are the highest-value field ("built X used by 5M students", "led Y"). Don't skip them — they're what makes prep specific instead of generic. If the profile is thin, `grep`/`read` the activity section or open their recent-activity feed for posts.

## Person web search (with data-broker filter)
`google.com/search?q="<name>" <company or role>` — capture notable work/press, and **hard-skip people-search/data-broker domains**:
```js
await new Promise(r=>setTimeout(r,4000));
const BROKER=/nationalpublicdata|whitepages|spokeo|411\.com|classmates|ancestry|beenverified|truthfinder|peoplefinder|fastpeoplesearch|radaris|mylife|intelius|residentdatabase|sortedbyname/i;
const titles=[...document.querySelectorAll('h3')].map(h=>h.innerText.trim()).filter(t=>t.length>10).slice(0,10);
const snippets=[...document.querySelectorAll('.VwiC3b, div[data-sncf], div[style*="line-clamp"]')].map(e=>e.innerText.trim()).filter(Boolean).slice(0,8);
const links=[];
document.querySelectorAll('a[href^="http"]').forEach(a=>{const h=a.href,x=(a.innerText||'').trim().split('\n')[0];
  if(x.length>10 && !/google\.|gstatic|\/search\?/.test(h) && !BROKER.test(h) && !links.find(l=>l[1]===h.split('&')[0])) links.push([x.slice(0,55),h.split('&')[0]]);});
const ai=(document.body.innerText.match(/AI Overview([\s\S]{0,450})/)||[])[1];
return {aiOverview:ai?.replace(/\s+/g,' ').slice(0,450), titles, snippets, links:links.slice(0,10)};
```
Anything from a `BROKER` domain (addresses, phone numbers, age, relatives, records) is **out of scope** — don't open it, don't cite it.
