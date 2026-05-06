---
name: pharmacy-catalogues
description: Find and download the latest weekly/fortnightly catalogues from Australian pharmacy and supermarket websites (Chemist Warehouse, Direct Chemist Outlet, Amcal, TerryWhite Chemmart, Coles, Woolworths, HealthSave) and upload them as PDFs to a designated Google Drive folder. Use when the user asks to "fetch the catalogues", "download this week's specials", "grab the pharmacy catalogues", or similar.
---

# Pharmacy & Supermarket Catalogue Downloader

Downloads the current catalogue(s) from a fixed list of Australian retailers and uploads each as a PDF to the configured Google Drive folder.

## Configuration

- **Google Drive destination folder ID**: `1g-i1fXQ6b2a4_LODkLpjc-Q8ML5gCqd6`
- **Working directory for downloads**: create a fresh temp dir per run, e.g. `mktemp -d -t catalogues-XXXX`. Clean it up at the end.
- **Naming convention for uploaded files**: `<retailer>_<YYYY-MM-DD>.pdf` where the date is today's date in the user's locale (use `date +%Y-%m-%d`). If a retailer publishes multiple concurrent catalogues (e.g. Coles has several state/regional catalogues), suffix with a slug, e.g. `coles_2026-05-06_weekly-specials.pdf`.

## Target sites

| Retailer            | Landing page                                              | Notes |
|---------------------|-----------------------------------------------------------|-------|
| Chemist Warehouse   | https://www.chemistwarehouse.com.au/catalogue             | Usually a direct PDF link or a viewer that loads a PDF. |
| Direct Chemist Outlet | https://www.directchemistoutlet.com.au/catalogue        | PDF served from their CDN. |
| Amcal               | https://www.amcal.com.au/our-latest-catalogue/            | Often a Publitas / Issuu embed; look for an underlying PDF URL. |
| TerryWhite Chemmart | https://terrywhitechemmart.com.au/shop/catalogue          | Lists multiple regional catalogues; download each. |
| Coles               | https://www.coles.com.au/catalogues                       | Multiple state catalogues; pick the user's state if specified, else download all. |
| Woolworths          | https://www.woolworths.com.au/shop/catalogue              | Often uses a Publitas viewer (`view.publitas.com/...`). |
| HealthSave          | https://www.healthsave.com.au/catalogue/                  | Direct PDF link. |

## Procedure

Run these steps in order. Use parallel tool calls within each step where the work is independent (e.g. fetching all landing pages at once, downloading all PDFs at once, uploading all PDFs at once).

### 1. Discover the PDF URL for each retailer

For every retailer in the table above, fetch the landing page with `WebFetch` and ask it to return the direct catalogue PDF URL plus any sub-catalogue links. Run all 7 fetches in parallel.

Tips for finding the PDF when the page uses a viewer:

- **Publitas**: viewer URLs look like `https://view.publitas.com/<account>/<catalogue>`. The PDF is typically downloadable at `https://view.publitas.com/<account>/<catalogue>/pdf` or via the page's "Download PDF" button. Inspect the rendered HTML for `.pdf` URLs or `og:url` / `<link rel="canonical">` metadata.
- **Issuu**: PDFs are usually not directly downloadable; if so, record the viewer URL and skip the upload for that retailer with a clear note.
- **Generic CDN-hosted PDFs**: search the page source for `.pdf` substrings.
- If `WebFetch` can't see the link (JS-rendered), try fetching with `curl -sL -A "Mozilla/5.0" <url>` and grep for `.pdf`.

If a site returns multiple catalogues (Coles, TerryWhite), capture all of them with descriptive slugs.

### 2. Download each PDF locally

For each PDF URL discovered, run in parallel:

```bash
curl -sSL -A "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36" \
  -o "$TMPDIR/<retailer>_<date>[_<slug>].pdf" "<pdf_url>"
```

Verify each file:

```bash
file "$TMPDIR/<retailer>_<date>.pdf"      # should report "PDF document"
[ "$(stat -c%s "$TMPDIR/<retailer>_<date>.pdf")" -gt 10000 ]  # >10KB sanity check
```

If a download fails or the result isn't a real PDF, record the failure and continue with the others — don't abort the whole run.

### 3. Upload each PDF to Google Drive

Use the Google Drive MCP tool `mcp__5b1d5cfd-4eda-4c21-8e4e-f530f060efb5__create_file` for each downloaded PDF, with:

- `name`: the filename from step 2 (e.g. `chemistwarehouse_2026-05-06.pdf`)
- `parent_folder_id`: `1g-i1fXQ6b2a4_LODkLpjc-Q8ML5gCqd6`
- `mime_type`: `application/pdf`
- File contents: read the local PDF (the tool's exact content parameter — `content`, `data`, `file_path`, etc. — depends on the schema; load it via `ToolSearch` with `select:mcp__5b1d5cfd-4eda-4c21-8e4e-f530f060efb5__create_file` before the first call).

Run all uploads in parallel.

If the Drive folder already contains a file with the same name from an earlier run today, prefer overwriting (use `mcp__5b1d5cfd-4eda-4c21-8e4e-f530f060efb5__search_files` to check) or append a counter suffix `_2`, `_3`, etc.

### 4. Report

End with a short summary table:

```
Retailer               Status     Drive file
Chemist Warehouse      uploaded   chemistwarehouse_2026-05-06.pdf
Direct Chemist Outlet  uploaded   directchemistoutlet_2026-05-06.pdf
Amcal                  skipped    (Issuu viewer, no PDF)
...
```

Include the Drive folder link in the final message:
`https://drive.google.com/drive/folders/1g-i1fXQ6b2a4_LODkLpjc-Q8ML5gCqd6`

### 5. Clean up

`rm -rf "$TMPDIR"` after the uploads succeed.

## Failure modes to expect

- **Geo-blocking / bot detection**: some retailers (Coles, Woolworths) may serve a challenge page to non-browser user agents. Try a real browser UA first; if still blocked, report the retailer as `blocked` and continue.
- **Rotating catalogue URLs**: the PDF URL changes each week — never hardcode it; always rediscover via step 1.
- **Multiple regional catalogues**: Coles and TerryWhite — download all of them, named with a region slug.
- **Viewer-only catalogues (Issuu)**: skip with a note; do not attempt to scrape the viewer's image tiles.
