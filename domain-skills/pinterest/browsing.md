# Pinterest — search, image resolution, and bulk harvesting

Field-tested against `www.pinterest.com` on 2026-07-25 with Chrome CDP + the harness.

## Quick summary

- **Signed-out search works.** The grid renders and the internal search API answers even when the
  header says "You are signed out". No login wall for `/search/pins/`.
- **Signed in, follow the graph instead of firing more queries.** Related-pin feeds and whole
  curated boards return better-matched, more visually coherent sets than any keyword search.
- **Use the internal JSON API, not the DOM.** One call returns ~50 pins with true original
  dimensions and a machine-generated alt-text description. DOM scraping returns ~11 pins per
  viewport and forces a scroll loop.
- **The API must be called from inside the page.** Replaying the identical request from
  `curl`/`urllib` with the same cookies and headers returns 403 — Pinterest fingerprints the client
  beyond headers. `js(...)` a same-origin `fetch` instead.
- **`/originals/` is not the reliable max-resolution URL. `/1200x/<hash>.jpg` is.**

---

## Internal search API

```
GET /resource/BaseSearchResource/get/
      ?source_url=<url-encoded /search/pins/?q=...>
      &data=<url-encoded JSON>
```

`data` payload:

```json
{"options":{"query":"cozy morning coffee aesthetic","scope":"pins",
            "bookmarks":[""],"page_size":50},"context":{}}
```

### Required headers — both, or 403

| Header | Value | Notes |
|---|---|---|
| `x-csrftoken` | value of the `csrftoken` cookie | read with `document.cookie.match(/csrftoken=([^;]+)/)` |
| `x-pinterest-pws-handler` | `www/search/[scope].js` | **the one everybody misses** |
| `x-app-version` | any recent build hash, e.g. `c96e9a5` | not strictly validated |
| `x-requested-with` | `XMLHttpRequest` | |

Missing either of the first two returns HTTP 403 with the body `Invalid Resource Request`
(plain text, *not* JSON — `await r.json()` will throw `Unexpected token 'I'`, which is the
symptom you actually see first).

Call it with `credentials:"include"` from a tab already on `pinterest.com`.

### Response shape

```
resource_response.data.results[]   # the pins; filter type === "pin"
resource_response.bookmark         # feed back into options.bookmarks[0] for the next page
```

Pagination ends when `bookmark` is empty or `-end-`. Note `data` is an **object**
(`{results, oneBarModules, ...}`), not an array — a `data[]` assumption silently yields nothing.

### Fields worth having

| Field | Why it matters |
|---|---|
| `images.orig.{url,width,height}` | **true** original pixel dimensions without downloading |
| `auto_alt_text` | Pinterest's own generated description ("a cup of coffee sitting on top of a bed next to a book") — an excellent cheap pre-filter |
| `grid_title`, `description` | user-supplied; often emoji-only or empty |
| `domain` | `"Uploaded by user"` for direct uploads; a real hostname when the pin was scraped from the web |
| `link` | original source URL — the only reliable provenance signal |
| `id` | pin URL is `https://www.pinterest.com/pin/{id}/` |

`auto_alt_text` is good enough to filter subject matter and junk, but **it is not reliable for
people** — frames with clearly visible faces are regularly described without any person word.
Verify visually before trusting a no-faces filter.

---

## Other resources on the same transport (signed in)

All take the same two required headers; only `source_url` and `x-pinterest-pws-handler` change.

| Goal | Resource | `options` | handler |
|---|---|---|---|
| Search pins | `BaseSearchResource` | `{query, scope:"pins", bookmarks, page_size}` | `www/search/[scope].js` |
| **Search boards** | `BaseSearchResource` | `{query, scope:"boards", ...}` | `www/search/[scope].js` |
| **Whole board** | `BoardFeedResource` | `{board_id, page_size}` | `www/[username]/[slug].js` |
| **Related pins** | `RelatedPinFeedResource` | `{pin: "<pin id>", page_size}` | `www/pin/[id].js` |

Notes that cost time:

- `BoardFeedResource` needs **`board_id`**. Passing `board_url` instead returns HTTP 400.
  `source_url` should be the board's path (`/<user>/<board-slug>/`).
- The related-pin resource is `RelatedPinFeedResource`. **`RelatedModulesResource` 404s** — it is
  the obvious guess and it is wrong.
- Board search results carry `pin_count`, so you can filter to boards worth harvesting before
  spending a request on each.
- Related feeds occasionally return `data: null` for a given seed pin; treat it as a skip, not a
  failure, and move on to the next seed.

### Board search drifts off-topic

Board titles are user-written, so a topical board search matches the *words*, not the imagery.
Observed: "laundry day aesthetic" returns laundry-**room renovation** boards; "hands holding coffee
mug" returns **ceramics retail** boards (mugs for sale, no hands); "moody film photography home"
returns **wedding and engagement portfolios**. Board search is excellent when the aesthetic itself
is the board name people use ("night drive aesthetic" → six dedicated boards, ~340 pins); it is
close to useless when the subject is a household object.

### Finding the board a pin belongs to

The pin detail page embeds it. `__PWS_DATA__` does **not** carry redux state — the pin object is in
a different script tag:

```js
JSON.parse(document.getElementById('__PWS_INITIAL_PROPS__').textContent)
  .initialReduxState.pins['<pin id>']            // → { images, board:{name,url,id}, auto_alt_text, link, ... }
```

---

## Image CDN — getting full resolution

Thumbnail URLs look like `https://i.pinimg.com/236x/ab/cd/ef/<hash>.jpg`. The path segment is the
size. Observed ladder:

```
/236x/  /474x/  /564x/  /736x/  /1200x/  /originals/
```

### The `/originals/` trap

**The original may be stored under a different extension than the thumbnail.** A pin whose grid
image is `.jpg` frequently has a `.png` original. Requesting `/originals/<hash>.jpg` for such a pin
returns an **S3 `AccessDenied` XML document** rather than an image or a clean 404:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Error><Code>AccessDenied</Code><Message>Access Denied</Message>...
```

Originals also appear as `.heic` and `.webp`, so extension guessing is a losing game.

### Use `/1200x/<hash>.jpg`

`/1200x/` is always served as JPEG regardless of the stored original's format, and returns the
**original** when the original is narrower than 1200px. Measured across a sample:

| pin | `/736x/` | `/1200x/` | `/originals/` |
|---|---|---|---|
| A | 736x1311 | 768x1368 | 768x1368 (`.jpg`) |
| B | 736x1307 | 941x1672 | 941x1672 (`.png`) |
| C | 736x1104 | 1024x1536 | 1024x1536 (`.png`) |
| D | 736x1308 | 1080x1920 | 1080x1920 (`.jpg`) |

So `/1200x/<hash>.jpg` gets you the true original in every case here, with no extension probing.
Fall back to `/736x/<hash>.jpg` if it fails. Typical Pinterest originals top out around
1000–1200px wide; 3000–6000px frames exist but are the exception.

**Never trust the URL for dimensions** — `/1200x/` returns whatever the original was. Measure the
downloaded bytes.

### Downloading

Plain `curl`/`urllib` against `i.pinimg.com` works fine with any User-Agent — the bot protection is
on `www.pinterest.com`, not the image CDN. Do **not** use the harness's `http_get` for images: it
`.decode()`s to text and corrupts binaries.

---

## DOM fallback (if the API shape changes)

Pins in the grid:

```js
[...document.querySelectorAll('img[src*="i.pinimg.com"]')].map(i => ({
  src: i.src, alt: i.alt, href: i.closest('a')?.getAttribute('href')
}))
```

Two quirks:

- **The grid is virtualised.** Only ~11 pins exist in the DOM at once; off-screen pins are removed,
  not just hidden. You must collect after every scroll step, not once at the end.
- **CDP `Input.dispatchMouseEvent` mouseWheel does not scroll the grid.** The harness's
  `scroll(x, y, dy=...)` leaves `scrollY` at 0. `js("window.scrollBy(0, 2000)")` works and the page
  height grows as new pins load. Use the JS path on Pinterest.

The `img.alt` attribute in the DOM carries the same `auto_alt_text` string the API returns.

---

## URL patterns

| Purpose | URL |
|---|---|
| Pin search | `/search/pins/?q=<urlencoded>&rs=typed` |
| Pin detail | `/pin/<id>/` |
| Board | `/<user>/<board-slug>/` |
| "More like this" feed | pin detail page, related section |

---

## Traps

- `resource_response.data` is an object with `.results`, not an array.
- 403 body is plain text, so the first error you see is a JSON parse failure, not a status check.
  Read `r.status` before `r.json()`.
- A killed harness call can leave the daemon wedged (every later call hangs with no output).
  `from admin import restart_daemon; restart_daemon()` clears it; re-`ensure_real_tab()` after.
- Search results for "aesthetic"-style queries are heavily polluted with AI-generated stock that has
  **quote text burned into the image**. `auto_alt_text` rarely mentions it. If you need clean plates,
  inspect at full resolution — small light-on-light captions are invisible in a contact sheet.
- Watermarks (e.g. Xiaohongshu IDs) appear in corners at low contrast; same problem, same fix.
- **`domain` and `link` are the only provenance you get, and they matter.** A frame that looks like
  a nice unmade bed turned out to be an H&M catalogue shot — `domain: www2.hm.com` with a
  `go.skimresources.com` affiliate link. Retail catalogue and brand product photography is common in
  "aesthetic" results and is invisible in the image itself. Check `domain`/`link` before assuming a
  pin is an amateur snap. A cheap reject list that removes most of it:

  ```
  shop store product collection etsy amazon amzn hm.com zara ikea westelm
  target walmart shopify skimresources rstyle ltk ceramics
  ```

- **Object-noun searches return the object for sale, not the object in a life.** Anything of the
  form "<household object> aesthetic" is heavily contaminated by e-commerce: querying around mugs and
  coffee returns overwhelmingly studio ceramics on seamless white. If you need the object *in situ*,
  search the room, the light or the time of day instead of the object, and reject frames that are a
  centred object on a near-white background (very high brightness + near-zero saturation is a good
  automatic proxy for "product shot").

## Instrumenting the page (what does not work)

If you try to discover these endpoints by watching traffic rather than reading this file:

- **Patching `window.fetch` captures nothing.** Pinterest's app issues these as XHR, not `fetch`.
- **CDP `Network.enable` + the harness's `drain_events()` returns zero events.** The daemon does not
  forward them, so a request log built that way comes back empty and looks like the page made no
  calls at all.

Probing candidate resource names directly against the API (as tabulated above) is faster than
either.
