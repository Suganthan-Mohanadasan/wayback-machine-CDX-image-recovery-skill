---
name: wayback-machine
description: "When the user wants to recover, retrieve, or view archived web pages, images, or content from the Wayback Machine or Internet Archive. Also use when the user mentions 'Wayback Machine,' 'web archive,' 'archived version,' 'old version of a page,' 'recover deleted content,' 'CDX API,' 'Internet Archive,' 'web.archive.org,' or 'recover images from old site.' This skill covers the CDX API for finding snapshots, downloading archived pages and assets, and bulk recovery of content and images from archived URLs."
metadata:
  version: 1.0.0
---

# Wayback Machine Content Recovery

You are an expert at using the Internet Archive's Wayback Machine APIs to find, retrieve, and recover archived web content. You help users recover deleted pages, extract images from old versions of sites, and bulk download archived assets.

## Core APIs

### 1. CDX API (Finding Snapshots)

The CDX API returns metadata about all archived snapshots of a URL.

**Base URL:** `https://web.archive.org/cdx/search/cdx`

**Key parameters:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `url` | Target URL (supports wildcards) | `suganthan.com/blog/*` |
| `output` | Response format | `json` |
| `fl` | Fields to return | `timestamp,original,statuscode,mimetype` |
| `filter` | Filter results | `statuscode:200` |
| `collapse` | Deduplicate by field | `digest` (unique content only) |
| `limit` | Max results | `50` |
| `from` / `to` | Date range (YYYYMMDD) | `from=20200101&to=20221231` |
| `matchType` | URL matching | `exact`, `prefix`, `host`, `domain` |

**Example: Find all snapshots of a page**
```
curl -s "https://web.archive.org/cdx/search/cdx?url=example.com/blog/my-post/&output=json&fl=timestamp,original,statuscode,mimetype&filter=statuscode:200&collapse=digest"
```

**Example: Find all pages under a path**
```
curl -s "https://web.archive.org/cdx/search/cdx?url=example.com/blog/*&output=json&fl=timestamp,original,statuscode,mimetype&filter=statuscode:200&collapse=urlkey&matchType=prefix"
```

**Example: Find all images on a page**
```
curl -s "https://web.archive.org/cdx/search/cdx?url=example.com/blog/my-post/*&output=json&fl=timestamp,original,mimetype&filter=mimetype:image/.*&collapse=urlkey"
```

**Response format (JSON):** First row is headers, subsequent rows are data.
```json
[["timestamp","original","statuscode","mimetype"],
 ["20220501120000","https://example.com/blog/post/","200","text/html"]]
```

### 2. Retrieving Archived Content

**Standard Wayback URL (includes toolbar):**
```
https://web.archive.org/web/{timestamp}/{url}
```

**Raw content URL (no toolbar, essential for scraping):**
```
https://web.archive.org/web/{timestamp}id_/{url}
```

The `id_` suffix after the timestamp is critical. It returns the original page content without the Wayback Machine toolbar and JavaScript injection. Always use `id_` when fetching content programmatically.

**Example:**
```bash
curl -s "https://web.archive.org/web/20220501120000id_/https://example.com/blog/my-post/" > page.html
```

### 3. Availability API (Quick Check)

Check if a URL has been archived:
```
curl -s "https://archive.org/wayback/available?url=example.com/blog/post/"
```

Returns JSON with the closest snapshot URL and timestamp.

## Common Workflows

### Recover a Deleted Page

1. **Find the best snapshot:**
```bash
curl -s "https://web.archive.org/cdx/search/cdx?url=example.com/page/&output=json&fl=timestamp,statuscode&filter=statuscode:200&limit=10" | python3 -c "import sys,json; data=json.load(sys.stdin); [print(r[0]) for r in data[1:]]"
```

2. **Fetch the raw HTML:**
```bash
curl -s "https://web.archive.org/web/{TIMESTAMP}id_/https://example.com/page/" > recovered.html
```

3. **Extract text content from HTML:**
```bash
# Use python3 to strip HTML tags
python3 -c "
from html.parser import HTMLParser
class S(HTMLParser):
    def __init__(self):
        super().__init__()
        self.text = []
        self.skip = False
    def handle_starttag(self, tag, attrs):
        if tag in ('script','style','nav','header','footer'): self.skip = True
    def handle_endtag(self, tag):
        if tag in ('script','style','nav','header','footer'): self.skip = False
    def handle_data(self, data):
        if not self.skip: self.text.append(data.strip())
s = S()
s.feed(open('recovered.html').read())
print('\n'.join(t for t in s.text if t))
"
```

### Recover Images from an Archived Page

1. **Fetch the archived page HTML:**
```bash
curl -s "https://web.archive.org/web/{TIMESTAMP}id_/https://example.com/page/" > page.html
```

2. **Extract image URLs from the HTML:**
```bash
grep -oP 'src="[^"]*\.(png|jpg|jpeg|gif|webp|svg)"' page.html | sed 's/src="//;s/"//' | sort -u
```

3. **Check if original image URLs are still live** (CDN images often persist after site migration):
```bash
curl -sI "https://cdn.example.com/image.png" | head -1
```
If they return HTTP 200, download directly from the original CDN (faster, better quality).

4. **If originals are gone, download from Wayback:**
```bash
# Find archived version of the image
curl -s "https://web.archive.org/cdx/search/cdx?url=cdn.example.com/image.png&output=json&fl=timestamp,original&filter=statuscode:200&limit=1"

# Download it
curl -s "https://web.archive.org/web/{TIMESTAMP}id_/https://cdn.example.com/image.png" -o image.png
```

### Bulk Image Recovery

When recovering many images from an archived page:

1. **Extract all image URLs and save to a file:**
```bash
curl -s "https://web.archive.org/web/{TIMESTAMP}id_/https://example.com/page/" | grep -oP 'src="[^"]*\.(png|jpg|jpeg|gif|webp|svg)"' | sed 's/src="//;s/"//' | sort -u > image_urls.txt
```

2. **Check which originals are still live:**
```bash
while IFS= read -r url; do
  status=$(curl -sI "$url" 2>/dev/null | head -1 | awk '{print $2}')
  echo "$status $url"
done < image_urls.txt
```

3. **Bulk download (from original or Wayback):**
```bash
mkdir -p recovered_images
while IFS= read -r url; do
  fname=$(basename "$url" | sed 's/[?#].*//')
  # Try original first
  if curl -sf "$url" -o "recovered_images/$fname"; then
    echo "OK: $fname (original)"
  else
    # Fall back to Wayback
    ts=$(curl -s "https://web.archive.org/cdx/search/cdx?url=$url&output=json&fl=timestamp&filter=statuscode:200&limit=1" | python3 -c "import sys,json;d=json.load(sys.stdin);print(d[1][0] if len(d)>1 else '')" 2>/dev/null)
    if [ -n "$ts" ]; then
      curl -sf "https://web.archive.org/web/${ts}id_/${url}" -o "recovered_images/$fname" && echo "OK: $fname (wayback)" || echo "FAIL: $fname"
    else
      echo "NOT ARCHIVED: $fname"
    fi
  fi
done < image_urls.txt
```

## Important Notes

- **Rate limiting:** The Wayback Machine will throttle aggressive requests. Add `sleep 1` between bulk downloads.
- **SSL on Mac:** Python's `urllib` often fails with SSL errors on macOS. Use `curl` instead.
- **`id_` is essential:** Without it, you get the Wayback toolbar HTML injected into responses, which corrupts binary files (images, PDFs) and pollutes HTML.
- **CDN persistence:** Images hosted on CDNs (Contentful, Cloudinary, imgix, etc.) often remain accessible long after the site that used them is gone. Always check the original URL first.
- **Wildcard searches:** Use `*` in CDX API URLs for prefix matching (e.g., `example.com/blog/*` finds all blog pages).
- **Deduplication:** Use `collapse=digest` to get only unique versions of a page (skips identical snapshots). Use `collapse=urlkey` to get one result per unique URL.
- **WebFetch limitations:** The WebFetch tool in Claude Code cannot access `web.archive.org`. Always use `curl` via the Bash tool instead.
