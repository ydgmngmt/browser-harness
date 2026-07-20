# subliminal-shop.com (Indigo Mind Labs) — downloading purchased audio

Digital downloads are served by the Uplinkly Digital Downloads Shopify app behind the
`/apps/downloads/` app proxy. The customer download page
(`/apps/downloads/product/<variant_id>/<b64-token>/`) has no download buttons — only a
streaming player — but the files are directly downloadable.

## How it works

- Each track on the page is a `div.audio-player[data-stream]` with a `data-url` of the form
  `https://subliminal-shop.com/apps/downloads/download-v2/<b64-signed-token>/`.
  The token embeds `customer_id`, `variant_id`, `product_id`, `expiry` (unix), and file `id`,
  plus an HMAC `s` field. Tokens expire (~1 day).
- `stream.js` sets that URL straight as an `<audio><source src>` — no extra headers.
- **Without the Shopify customer session cookie the same URL renders the download page HTML
  again (HTTP 200, `text/html`)** — even with `Range`, `Sec-Fetch-Dest: audio`, or browser UAs.
  Don't waste time on header tricks; auth is cookie-based.
- **With the customer's `_shopify_essential` cookie, a plain GET returns `302` to a signed
  S3 URL** (`s3.us-west-2.amazonaws.com/indigomindlabs-subliminals-all/...`) which needs no
  cookies at all. So:

```bash
# cookie string pulled from the browser via CDP (Storage.getCookies, domain subliminal-shop)
curl -L -b "$COOKIES" "https://subliminal-shop.com/apps/downloads/download-v2/<token>/" -o out.flac
```

Files are large (~475 MB for a 45-min 24-bit/48 kHz stereo FLAC).

## Traps

- `curl -I` (HEAD) always returns the HTML page headers — probe with a GET.
- `Accept: audio/*` gets the HTML body echoed back with a spoofed `audio/*` content type —
  it is NOT the file (check the size / magic bytes).
- The canonical/og URLs in the page HTML are lowercased by Shopify — the base64 tokens are
  case-sensitive, so never copy tokens from those meta tags; use the `data-url` attributes.
