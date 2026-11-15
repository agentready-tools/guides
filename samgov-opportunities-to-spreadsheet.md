# Pull SAM.gov contract opportunities into a spreadsheet with an Apify Actor

This pulls US federal contract opportunities from [our SAM.gov Actor](https://apify.com/samgov-tools/samgov-opportunities) and writes them to an Excel or CSV file. You don't need a SAM.gov API key if you're fine with the official daily extract, which can be up to about a day old.

You do need an [Apify API token](https://console.apify.com/settings/integrations). Put it in the `YOUR_APIFY_TOKEN` placeholders below, or export it as `APIFY_TOKEN`. Runs are billed on Apify. As of 15 November 2026 the Actor charges $0.002 per stored opportunity ($2 per 1,000); check the [Pricing tab](https://apify.com/samgov-tools/samgov-opportunities) for the current rate.

## Download an Excel file

The request below asks for solicitations whose title contains "renovation", with place of performance in Virginia, and caps the file at 25 rows. Change the JSON body to match what you want. `maxItems` defaults to 10 if you leave it out, and the hard cap is 10,000.

```sh
curl --fail-with-body --max-time 330 -X POST \
  'https://api.apify.com/v2/acts/samgov-tools~samgov-opportunities/run-sync-get-dataset-items?format=xlsx&attachment=1&timeout=300' \
  -H 'Authorization: Bearer YOUR_APIFY_TOKEN' \
  -H 'Content-Type: application/json' \
  -o samgov-opportunities.xlsx \
  --data '{
    "keywords": "renovation",
    "state": "VA",
    "noticeType": "o",
    "maxItems": 25
  }'
```

`noticeType` uses SAM.gov's official `ptype` codes. `o` is Solicitation. Other useful ones: `k` (Combined Synopsis/Solicitation), `r` (Sources Sought), `p` (Pre solicitation), `a` (Award Notice).

Open the `.xlsx` file in Excel, Google Sheets, or LibreOffice. If you prefer CSV, switch `format=xlsx` to `format=csv` and change the output filename.

The sync endpoint waits until the run finishes, then returns the dataset. If that takes more than 300 seconds, the HTTP call fails with 408 even though the run may still be going. Keep `maxItems` modest for this pattern. For larger exports, start an [asynchronous run](https://docs.apify.com/api/v2/actors-runs-post) and download the dataset when it finishes.

## Same call in Python

```python
import requests

TOKEN = "YOUR_APIFY_TOKEN"

response = requests.post(
    "https://api.apify.com/v2/acts/samgov-tools~samgov-opportunities/run-sync-get-dataset-items",
    params={
        "format": "xlsx",
        "attachment": 1,
        "timeout": 300,
    },
    headers={
        "Authorization": f"Bearer {TOKEN}",
        "Content-Type": "application/json",
    },
    json={
        "keywords": "renovation",
        "state": "VA",
        "noticeType": "o",
        "maxItems": 25,
    },
    timeout=330,
)
response.raise_for_status()

with open("samgov-opportunities.xlsx", "wb") as f:
    f.write(response.content)
```

This uses the `requests` package (`pip install requests`).

## Input worth knowing

All fields are optional. An empty body (`{}`) returns up to 10 active notices posted in the last 30 days, from the official public daily extract.

| Field | What it does |
| --- | --- |
| `keywords` | Title search only, not the notice body. In extract mode every word must appear in the title (case-insensitive). |
| `naics` | 2–6 digit NAICS code. Extract mode does a prefix match (`236` matches `236220`). |
| `setAside` | Official set-aside code, e.g. `SBA` for Total Small Business. |
| `noticeType` | Official `ptype` code (`u`, `p`, `a`, `r`, `s`, `o`, `g`, `k`, `i`). |
| `state` | Two-letter place-of-performance state, e.g. `VA`. |
| `activeOnly` | Default `true`. The daily extract only contains active notices anyway. |
| `postedFrom` / `postedTo` | Inclusive range, `MM/dd/yyyy` or `YYYY-MM-DD`, at most one calendar year. If both are omitted, the range is the last 30 days (29 days before today UTC through today). |
| `maxItems` | Stored (and billed) row cap, 1–10,000. Default 10. |
| `source` | `auto` (default), `bulk` to force the no-key extract, or `api` to require the live API. |
| `samApiKey` | Your own free SAM.gov public API key, only if you want live data or inactive notices. This is secret input; don't put a real key in a shared command history. |
| `pageSize` | Live API page size, 1–1,000. Ignored for the daily extract. |

`source: "auto"` uses the live API when `samApiKey` is present, otherwise the extract. Each output row has `dataSource` (`sam.gov-daily-extract` or `sam.gov-api`) and, for extract rows, `extractDate`.

In extract mode, `description` is the notice text. In API mode it's a description URL, and you'll need your own key to fetch it later.

## Nested columns in a spreadsheet

Most of the useful triage fields are flat: `noticeId`, `title`, `solicitationNumber`, `type`, `postedDate`, `responseDeadline`, `naicsCode`, `setAside`, `fullParentPathName`, `uiLink`, `description`, `dataSource`, `extractDate`.

`officeAddress`, `placeOfPerformance`, and `award` are objects; `pointOfContact` is an array. In a raw CSV or Excel export those often show up as JSON text in a single cell. You can ask the API to flatten the objects:

```
https://api.apify.com/v2/acts/samgov-tools~samgov-opportunities/run-sync-get-dataset-items?format=csv&flatten=officeAddress,placeOfPerformance,award&attachment=1
```

That turns `{"foo": {"bar": "hello"}}` into a `foo.bar` column, per [Apify's dataset items API](https://docs.apify.com/api/v2/act-run-sync-get-dataset-items-post). Contact arrays don't flatten as neatly; leave them as JSON, or pick the columns you actually need with `fields`.

To get a slimmer sheet, add:

```
&fields=noticeId,title,solicitationNumber,type,postedDate,responseDeadline,naicsCode,setAside,fullParentPathName,uiLink,description,dataSource,extractDate
```

## If you'd rather not use the API

Run the Actor from [its Apify page](https://apify.com/samgov-tools/samgov-opportunities), then export the dataset as Excel or CSV from the run's dataset tab.

## A few limits

- This does not scrape SAM.gov pages. It uses the official public daily extract and, if you supply a key, the official Get Opportunities API.
- Keyword search is title-only.
- You are charged for rows stored in that run. Running the same query again can charge for the same notices again.
- The extract is active notices only and can lag by about a day. Attachments are not downloaded; `resourceLinks` is null in extract mode.

The full input and output field notes are on the [Actor page](https://apify.com/samgov-tools/samgov-opportunities).
