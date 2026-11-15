# Screenshot a list of URLs (incl. full-page/PDF) from an AI agent via Apify API

Give [our screenshot Actor](https://apify.com/samgov-tools/url-screenshot) a list of public URLs and it returns one JSON item per URL, with a direct `screenshotUrl` for a PNG, JPEG, or PDF. That's the same shape an agent can consume: check `error`, then use the link.

You need an [Apify API token](https://console.apify.com/settings/integrations). Replace `YOUR_APIFY_TOKEN`. As of 15 November 2026 the prices are $0.0005 to start the run, $0.002 per successfully stored capture, and an extra $0.001 when the capture is full-page or a PDF. Failed URLs are free, though the start fee still applies. The [Pricing tab](https://apify.com/samgov-tools/url-screenshot) is the live source.

## Capture a few pages

```sh
curl --fail-with-body --max-time 330 -X POST \
  'https://api.apify.com/v2/acts/samgov-tools~url-screenshot/run-sync-get-dataset-items?timeout=300' \
  -H 'Authorization: Bearer YOUR_APIFY_TOKEN' \
  -H 'Content-Type: application/json' \
  --data '{
    "urls": [
      "https://example.com",
      "https://example.org"
    ],
    "format": "png",
    "fullPage": true,
    "maxUrls": 5
  }'
```

The response is a JSON array. A viewport PNG item looks like this (illustrative values, not a live capture from this run):

```json
{
  "url": "https://example.com",
  "finalUrl": "https://example.com/",
  "status": 200,
  "title": "Example Domain",
  "screenshotUrl": "https://api.apify.com/v2/key-value-stores/STORE_ID/records/screenshot-1-example-com",
  "format": "png",
  "width": 1280,
  "height": 800,
  "fullPage": false,
  "bytes": 15432,
  "takenAt": "2026-11-14T12:00:00.000Z",
  "error": null
}
```

If `error` is not null, `screenshotUrl` is null. Don't treat HTTP `status` as capture success. A 404 page can still produce a stored capture. For a full-page PNG, `height` is the actual image height in pixels, up to the 16,384 CSS-pixel clip.

For a PDF, set `"format": "pdf"`. PDF always uses a full-page print layout, and it incurs the extra pricing event. You don't also need `"fullPage": true`; if you set both, you still only pay that extra once. PDF output is limited to 50 pages.

## Python

```python
import requests

TOKEN = "YOUR_APIFY_TOKEN"

response = requests.post(
    "https://api.apify.com/v2/acts/samgov-tools~url-screenshot/run-sync-get-dataset-items",
    params={"timeout": 300},
    headers={
        "Authorization": f"Bearer {TOKEN}",
        "Content-Type": "application/json",
    },
    json={
        "urls": [
            "https://example.com",
            "https://example.org",
        ],
        "format": "pdf",
        "maxUrls": 5,
    },
    timeout=330,
)
response.raise_for_status()

for item in response.json():
    if item.get("error"):
        print(f"{item['url']}: {item['error']}")
    else:
        print(f"{item['url']}: {item['screenshotUrl']}")
```

This uses the `requests` package (`pip install requests`). An agent should do the same thing: walk the array, report `error` when present, and pass `screenshotUrl` through when it isn't. Signed links typically work without a second Apify token, but they can expire with the run's storage retention. Treat a signed URL like a bearer link.

## Input

`urls` is the only field you usually need. An empty body captures `https://example.com`. The API also accepts objects in the list, e.g. `{"url": "https://example.com"}`. URLs are trimmed and deduplicated in order. Extra unique URLs above `maxUrls` are ignored (default 10, hard max 100). Processing is sequential, one browser.

| Field | Default | Notes |
| --- | --- | --- |
| `urls` | `["https://example.com"]` | Public `http`/`https` URLs, ports 80 and 443 only. |
| `maxUrls` | `10` | 1–100. |
| `fullPage` | `false` | Long page capture, clipped at 16,384 CSS pixels of height. This does not lazily scroll to force every image to load. |
| `format` | `png` | `png`, `jpeg`, or `pdf`. |
| `device` | `desktop` | `desktop` (1280×800, DPR 1), `mobile` (390×844, DPR 2), or `tablet` (820×1180, DPR 2). |
| `width` / `height` | Device viewport | Optional CSS-pixel overrides (width 320–3840, height 240–2160). |
| `jpegQuality` | `80` | JPEG only, 1–100. |
| `waitUntil` | `load` | `load`, `domcontentloaded`, or `networkidle`. `networkidle` can time out on busy pages. |
| `delayMs` | `0` | Extra wait after navigation, 0–10,000 ms. Useful for client-rendered pages. |
| `blockCookieBanners` | `true` | Hides common consent banners with CSS. It does not click or accept them, and custom banners may remain. |
| `timeoutSecs` | `30` | Per-URL navigation timeout, 5–60 seconds. |

## Things that will not work

Only public pages. Localhost, private or reserved IPs, URL credentials, nonstandard ports, and login-cookie input are rejected. There is no CAPTCHA solver or stealth mode.

The sync API call itself times out at 300 seconds. A handful of pages is usually fine; a large batch may need an [async run](https://docs.apify.com/api/v2/actors-runs-post) instead.

If you want this from an MCP client rather than raw HTTP, point the client at `https://mcp.apify.com?tools=samgov-tools/url-screenshot` and have it call the Actor with the same input fields.

More detail, including how PDF width and height are reported, is on the [Actor page](https://apify.com/samgov-tools/url-screenshot).
