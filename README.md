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

Each simulated beacon event fires an actual `fetch()` POST to the **real Athos Beacon Tracking endpoint** — not a mock, not an echo service:

```
POST https://analytics.athoscommerce.net/beacon/v2/atdtdp/product/pageview

{
  "context": {
    "timestamp": "...", "pageUrl": "...", "userId": "...",
    "sessionId": "...", "pageLoadId": "...", "initiator": "athos/demo/1.0",
    "dev": true
  },
  "data": { "result": { "parentId": "...", "uid": "..." } }
}
```

- **`atdtdp`** is an internal Athos demo site, not a live paying customer.
- **`context.dev: true`** is the officially documented flag (Beacon Tracking spec, "Testing" section) that excludes test traffic from real reporting — the sanctioned way to test against live infrastructure without polluting real analytics.

This means the browser's **DevTools → Network tab** shows a genuine outgoing request to production Athos infrastructure, with a real `200 OK` response — as authentic as this gets without touching a real customer's data.

**Requires an internet connection** for the network requests to succeed (the on-page event log will still work and show request status if the endpoint is briefly unreachable).

## Demo script

1. Start in **Current (broken)** mode. Point at the event log: one event fired on page load, `parentId` and `uid` both equal the parent ID.
2. Click a size (e.g. Size 9). Notice: no new request fires — the frontend never told the backend which variant was selected.
3. Point at the Products Report: Size 9 shows 0 views, 9 sales — the real symptom.
4. Switch to **Fixed** mode. Click Size 9 again. A new POST fires with `uid` updated to the variant's own ID — check the Network tab.
5. Point at the report: views now reconcile with sales.
