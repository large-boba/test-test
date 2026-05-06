---
name: pharmacy-catalogues
description: Find and download the latest weekly/fortnightly catalogues from Australian pharmacy and supermarket websites (Chemist Warehouse, Direct Chemist Outlet, Amcal, TerryWhite Chemmart, Coles, Woolworths, HealthSave) and upload them as PDFs to a designated Google Drive folder. Use when the user asks to "fetch the catalogues", "download this week's specials", "grab the pharmacy catalogues", or similar.
---

# Pharmacy & Supermarket Catalogue Downloader

Downloads the current catalogue(s) from a fixed list of Australian retailers and uploads each as a PDF to the configured Google Drive folder.

## Configuration

- **Google Drive destination folder ID**: `1g-i1fXQ6b2a4_LODkLpjc-Q8ML5gCqd6`
- **Drive folder link**: https://drive.google.com/drive/folders/1g-i1fXQ6b2a4_LODkLpjc-Q8ML5gCqd6
- **Naming convention**: `<retailer>_<YYYY-MM-DD>.pdf` using today's date (`date +%Y-%m-%d`). Multiple catalogues from one retailer get a slug: `coles_2026-05-06_nsw-weekly.pdf`.
- **Working temp dir**: `TMPDIR=$(mktemp -d -t catalogues-XXXX)` — clean up with `rm -rf "$TMPDIR"` after all uploads succeed.

## CRITICAL: No undo on Drive uploads

The Drive MCP has **no delete tool**. Once a file is uploaded it cannot be removed programmatically. Therefore:
- **Always validate** the downloaded file is a real PDF (≥ 10 KB, `file` command reports "PDF document") before uploading.
- **Always check for an existing upload today** before creating a new one (see Step 3).
- If a download fails validation, log it as `failed` and skip — do not upload garbage.

## Target sites

| Retailer | Landing page | Viewer type | Notes |
|---|---|---|---|
| Chemist Warehouse | https://www.chemistwarehouse.com.au/catalogue | Publitas | Look for `view.publitas.com/chemist-warehouse` links |
| Direct Chemist Outlet | https://www.directchemistoutlet.com.au/catalogue | Direct PDF or Publitas | Look for `.pdf` links or Publitas viewer |
| Amcal | https://www.amcal.com.au/our-latest-catalogue/ | Publitas | Look for `view.publitas.com` links |
| TerryWhite Chemmart | https://terrywhitechemmart.com.au/shop/catalogue | Various | May list multiple regional catalogues; download each |
| Coles | https://www.coles.com.au/catalogues | JS-heavy | Multiple state catalogues; download all found |
| Woolworths | https://www.woolworths.com.au/shop/catalogue | Publitas | Look for `view.publitas.com/woolworths` links |
| HealthSave | https://www.healthsave.com.au/catalogue/ | Direct PDF | Look for `.pdf` links |

## Step 1 — Discover catalogue URLs (run all 7 in parallel)

For each retailer, run this `curl` command (NOT `WebFetch` — sites return 403 to non-browser fetchers):

```bash
curl -sSL \
  -A "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36" \
  -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8" \
  -H "Accept-Language: en-AU,en;q=0.9" \
  -H "Connection: keep-alive" \
  "$URL" | grep -Eoi '(https?://[^"'\''<> ]+\.pdf|https?://view\.publitas\.com/[^"'\''<> ]+)' | sort -u
```

Then ask yourself:
- Are there direct `.pdf` links? Use those.
- Are there `view.publitas.com/<account>/<publication>` viewer links? The PDF download URL is `https://view.publitas.com/<account>/<publication>/pdf` — try it (Publitas usually serves a redirect to the CDN PDF).
- Is the page blank or JS-only (no links at all in the HTML)? Try `WebFetch` on the same URL as a fallback (it uses a headless renderer). If `WebFetch` also fails, mark the retailer as `blocked` and continue.

If a site lists multiple catalogues (Coles state catalogues, TerryWhite regional), capture all of them. Create a slug from the catalogue name, e.g. `coles_<date>_weekly-specials.pdf`, `coles_<date>_nsw-health.pdf`.

## Step 2 — Download each PDF (run all in parallel)

```bash
curl -sSL \
  -A "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36" \
  -H "Accept: application/pdf,*/*" \
  -L \
  -o "$TMPDIR/<retailer>_<date>[_<slug>].pdf" \
  "<pdf_url>"
```

**Mandatory validation before proceeding to upload:**

```bash
FILE="$TMPDIR/<retailer>_<date>.pdf"
file "$FILE" | grep -q "PDF document"         || { echo "SKIP: not a PDF"; continue; }
[ "$(stat -c%s "$FILE")" -gt 10000 ]          || { echo "SKIP: too small (< 10 KB)"; continue; }
```

## Step 3 — Check for existing upload today

Before uploading each file, query Drive to avoid duplicates:

```
Use mcp__5b1d5cfd-4eda-4c21-8e4e-f530f060efb5__search_files with:
  query: parentId = '1g-i1fXQ6b2a4_LODkLpjc-Q8ML5gCqd6' and title = '<filename>.pdf'
```

- If a file with that exact name already exists in the folder: **skip the upload** and report it as `already uploaded`.
- If not found: proceed to Step 4.

## Step 4 — Upload to Google Drive (run all in parallel)

Read the downloaded PDF and base64-encode it:

```bash
B64=$(base64 -w 0 "$TMPDIR/<retailer>_<date>.pdf")
```

Then call `mcp__5b1d5cfd-4eda-4c21-8e4e-f530f060efb5__create_file` with **exactly these parameters**:

```json
{
  "title": "<retailer>_<YYYY-MM-DD>.pdf",
  "parentId": "1g-i1fXQ6b2a4_LODkLpjc-Q8ML5gCqd6",
  "base64Content": "<base64-encoded PDF bytes>",
  "contentMimeType": "application/pdf",
  "disableConversionToGoogleType": true
}
```

> `disableConversionToGoogleType: true` is **mandatory** — without it Google Drive silently converts the PDF to a Google Doc.

## Step 5 — Clean up and report

```bash
rm -rf "$TMPDIR"
```

End with a summary table:

```
Retailer               Status             Drive file
Chemist Warehouse      uploaded           chemistwarehouse_2026-05-06.pdf
Direct Chemist Outlet  uploaded           directchemistoutlet_2026-05-06.pdf
Amcal                  uploaded           amcal_2026-05-06.pdf
TerryWhite Chemmart    uploaded (2 files) terrywhitechemmart_2026-05-06_qld.pdf, ...
Coles                  uploaded (3 files) coles_2026-05-06_weekly-specials.pdf, ...
Woolworths             already uploaded   woolworths_2026-05-06.pdf
HealthSave             blocked            (site returned 403)

Drive folder: https://drive.google.com/drive/folders/1g-i1fXQ6b2a4_LODkLpjc-Q8ML5gCqd6
```

## Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `curl` returns 403 or empty HTML | Bot detection on that retailer | Try `WebFetch` as fallback; if still blocked, log as `blocked` |
| Downloaded file < 10 KB | Redirect to login/error page saved as "PDF" | Log as `failed`; skip upload |
| `file` reports "HTML document" | Site served a bot-challenge instead of PDF | Log as `blocked`; skip upload |
| Publitas `/pdf` URL returns 404 | Catalogue slug changed this week | Re-scrape the viewer page and look for a `download` button URL |
| Coles/Woolworths page has no links in HTML | Fully JS-rendered | Use `WebFetch` with the landing URL — it uses a headless renderer |
