# Wayback Machine CDX Image Recovery Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for recovering archived web pages, images, and assets from the Internet Archive's Wayback Machine using the CDX API.

## What It Does

When installed, Claude Code automatically uses this skill whenever you mention recovering archived content, the Wayback Machine, or the CDX API. It gives Claude the knowledge to:

- **Find snapshots** of any URL using the CDX API with filtering, date ranges, and deduplication
- **Recover deleted pages** by fetching raw HTML from the best archived snapshot
- **Extract and download images** from archived pages, with smart fallback logic (try original CDN first, then Wayback)
- **Bulk recover assets** from entire sites or URL prefixes using wildcard searches

## Installation

Copy `SKILL.md` into your Claude Code skills directory:

```bash
# Create the skill directory
mkdir -p ~/.claude/skills/wayback-machine

# Clone and install
git clone https://github.com/Suganthan-Mohanadasan/wayback-machine-CDX-image-recovery-skill.git
cp wayback-machine-CDX-image-recovery-skill/SKILL.md ~/.claude/skills/wayback-machine/SKILL.md
```

Claude Code will automatically detect the skill on next launch.

## Usage

Once installed, just ask Claude Code naturally:

```
"Recover the images from this archived page: https://web.archive.org/web/20240101/https://example.com/blog/my-post/"

"Find all archived snapshots of example.com/blog/"

"Download all images from the old version of my site"

"Check if this deleted page is in the Wayback Machine"
```

## What's Inside

The skill covers three Wayback Machine APIs:

| API | Purpose | Use Case |
|-----|---------|----------|
| **CDX API** | Search snapshot metadata | Finding archived URLs, filtering by date/status/type |
| **Raw Content** (`id_` suffix) | Fetch original content | Downloading pages and binary files without Wayback toolbar injection |
| **Availability API** | Quick archive check | Verifying if a URL has been archived |

Key workflows included:

1. **Single page recovery** with HTML extraction and text parsing
2. **Image recovery** with CDN persistence detection (images on Contentful, Cloudinary, etc. often outlive the sites that used them)
3. **Bulk asset recovery** with parallel download and Wayback fallback logic

## Important Notes

- Always use the `id_` suffix when fetching content programmatically (`/web/{timestamp}id_/{url}`). Without it, Wayback injects its toolbar HTML, which corrupts binary files and pollutes scraped content.
- The CDX API throttles aggressive requests. Add `sleep 1` between bulk downloads.
- Use `collapse=digest` to deduplicate identical snapshots. Use `collapse=urlkey` for one result per unique URL.
- Claude Code's `WebFetch` tool cannot access `web.archive.org`. The skill uses `curl` via the Bash tool instead.

## Background

I built this skill after recovering 54 images across two blog posts during a site migration from WordPress to Astro. The original images were lost, but between CDN persistence and the Wayback Machine's CDX API, every single one was recovered.

Full write-up: [How to Use the Wayback Machine to Recover Lost Content](https://suganthan.com/blog/how-to-use-wayback-machine/)

## Licence

MIT
