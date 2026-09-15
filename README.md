# Beacon Attribution Demo

An interactive, self-contained reconstruction of a real production bug: a product's variant (size, color, etc.) view count and sales count silently diverge because a page-view beacon event never identifies which variant a shopper actually selected.

Built for a live technical demo — no build step, no dependencies, no server required.

## What it demonstrates

- **Current (broken) mode:** the page loads at the parent-product level with no variant pre-selected, so the pageview beacon fires with `parentId` and `uid` set to the same (parent) ID. Selecting a size fires no follow-up event, so every view rolls up to whichever variant the backend happens to map the parent to — while sales still attribute correctly per variant. The Products Report ends up showing something like "0 views, 9 sales" for a real, selling variant.
- **Fixed mode:** selecting a size fires an updated pageview event with `uid` set to the specific variant's ID, keeping `parentId` unchanged. Views then reconcile against sales at the variant level.

This mirrors the documented rule in the Beacon Tracking spec's "Mapping Product Data from API Responses" section: `parentId` is always the parent product's identifier; `uid` is the specific variant's identifier, and the two should only be equal when there are no variants.

## Running it

Just open `index.html` in a browser. That's it — no install, no build.

```
open index.html
```

or serve it locally if you prefer:

```
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Why it makes real network requests

Each simulated beacon event fires an actual `fetch()` POST rather than just logging to an in-page array, so the browser's **DevTools → Network tab** shows genuine outgoing requests with real JSON payloads.

**The payload body matches the real Athos Beacon Tracking spec exactly** — same `context`/`data` structure, same field names, as documented for a real event:

```
POST https://analytics.athoscommerce.net/beacon/v2/{siteId}/product/pageview

{
  "context": {
    "timestamp": "...", "pageUrl": "...", "userId": "...",
    "sessionId": "...", "pageLoadId": "...", "initiator": "athos/demo/1.0"
  },
  "data": { "result": { "parentId": "...", "uid": "..." } }
}
```

**The destination is not real**, deliberately: requests are sent to `https://httpbin.org/post` (a public echo endpoint) instead of live Athos production infrastructure, since sending demo traffic to real company systems during a demo isn't appropriate. The payload shape is authentic; the endpoint is a safe stand-in.

**Requires an internet connection** for the network requests to succeed (the on-page event log will still work and show request status even if `httpbin.org` is briefly unreachable).

## Demo script

1. Start in **Current (broken)** mode. Point at the event log: one event fired on page load, `parentId` and `uid` both equal the parent ID.
2. Click a size (e.g. Size 9). Notice: no new request fires — the frontend never told the backend which variant was selected.
3. Point at the Products Report: Size 9 shows 0 views, 9 sales — the real symptom.
4. Switch to **Fixed** mode. Click Size 9 again. A new POST fires with `uid` updated to the variant's own ID — check the Network tab.
5. Point at the report: views now reconcile with sales.
