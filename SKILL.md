---
name: forum-extraction
description: Use when building scrapers to extract and index forum discussions, blog posts, or community content into markdown files with postgres/vector search integration. Triggers on requests to scrape, sync, index, or extract forum content.
---

# Forum Extraction & Indexing

## Overview

Build scrapers that extract forum discussions into structured markdown files, then index them for semantic search. Each forum platform (Discourse, phpBB, Drupal, Vanilla, static blogs) needs a platform-specific fetcher, but the output format, rate limiting, incremental sync, and indexing pipeline are universal.

**Core principle:** Be a good citizen. Rate-limit aggressively, respect robots.txt, use APIs when available, and never hit a forum harder than a human would browse it.

## Architecture

```
Forum API/HTML ──→ Platform Fetcher ──→ Markdown + YAML Frontmatter ──→ File Storage
                                                                            │
                                                                    Statement-MCP / QMD
                                                                     (semantic search)
```

Every scraper follows the same pipeline:
1. **Fetch** — Platform-specific API calls or HTML scraping
2. **Transform** — Convert to markdown with structured YAML frontmatter
3. **Store** — One file per discussion/post, organized by category
4. **Track** — Incremental sync state (last-fetched timestamps)
5. **Index** — Ingest into statement-mcp or QMD for semantic search

## Output Format

Every discussion/post becomes a markdown file with YAML frontmatter:

```markdown
---
id: 12345
title: "Discussion title"
category: category-slug
author: username
created: '2025-01-15T10:30:00+00:00'
updated: '2025-01-16T08:00:00+00:00'
tags: [nurbs, surface, continuity]
url: https://forum.example.com/t/12345
source: forum-name
platform: discourse
countReplies: 14
countViews: 892
solved: true
---

# Discussion title

[body as markdown]

---

## Replies

### username — 2025-01-15T11:00:00+00:00

[reply body as markdown]
```

**Required frontmatter fields:** id, title, category, author, created, url, source, platform
**Optional but recommended:** tags, countReplies, countViews, solved/accepted, updated

## File Organization

```
{output-dir}/
├── categories/
│   ├── {category-slug}/
│   │   ├── {id}.md
│   │   └── ...
│   └── ...
├── metadata/
│   ├── sync-state.json    ← last sync timestamp per category
│   └── categories.json    ← category list cache
└── README.md              ← auto-generated stats
```

## Rate Limiting

**Non-negotiable rules:**

| Forum size    | Delay between requests | Why                                     |
| ------------- | ---------------------- | --------------------------------------- |
| Small (<1K)   | 3-5 seconds            | Don't overwhelm small communities       |
| Medium (1-10K)| 2-3 seconds            | Sustainable pace                        |
| Large (10K+)  | 1-2 seconds            | API can handle it; still be respectful  |
| Blog (static) | 1-2 seconds            | Low load, but don't slam                |

**Always:**
- Use a `RateLimiter` class (see reference implementation below)
- Set a User-Agent identifying the scraper: `OnshapeAssistant/1.0 (research; contact@example.com)`
- Respect `Retry-After` headers on 429 responses
- Back off exponentially on errors (start 5s, max 60s)
- Stop entirely on 403 or repeated 429s

## Platform-Specific Fetchers

### Discourse (McNeel Forum, Blender devtalk)

**API:** Discourse has a public JSON API. Append `.json` to any URL.

```
GET /categories.json                              → list categories
GET /c/{slug}/{id}.json?page=N                    → topics in category
GET /t/{topic_id}.json                            → topic with posts
GET /t/{topic_id}/posts.json?post_ids[]=N&...     → specific posts
```

**Pagination:** Topics list paginated by `page` param. Posts within a topic may need chunked fetching for long threads (post stream IDs in topic JSON).

**Key fields:** `topic.title`, `topic.posts_count`, `topic.views`, `topic.tags`, `topic.created_at`, `post.cooked` (HTML body), `post.username`, `post.created_at`

**Rate limit:** Discourse default is 200 req/min for anonymous. Use 2s delay to stay well under.

### phpBB (FreeCAD Forum)

**No API.** Must scrape HTML.

```
GET /viewforum.php?f={forum_id}&start={offset}    → topic list (25/page)
GET /viewtopic.php?t={topic_id}&start={offset}    → posts in topic (15/page)
```

**HTML selectors:**
- Topic list: `.topiclist .row` or similar (varies by theme)
- Post body: `.postbody .content`
- Post author: `.postprofile .username`
- Pagination: `.pagination` links

**Key gotcha:** phpBB HTML varies significantly by theme. Test selectors against the specific forum before bulk scraping.

### Drupal (Open CASCADE Dev Forum)

**No standard API.** Scrape HTML.

```
GET /forums/{forum_id}?page={N}                   → topic list
GET /node/{node_id}                                → single topic with comments
```

**HTML selectors (typical Drupal):**
- Topic list: `.view-content .views-row`
- Topic body: `.field--name-body`
- Comments: `.comment-wrapper .comment`

**Key gotcha:** Older Drupal sites may have different markup. Some content may be at `old.opencascade.com` with a different structure.

### Static Blogs (Quaoar, C3D Labs, Geometric Tools)

**Approach:** Sitemap or manual URL list → fetch each page → extract article content.

```
GET /sitemap.xml                                   → all URLs
GET /feed/ or /rss/                                → RSS feed with recent posts
GET /{post-url}                                    → individual article
```

**HTML selectors:** Highly variable. Use `<article>` tag or `main` content area. Strip nav, sidebar, footer.

**For Quaoar specifically:** Small volume (~100 posts), extraordinary depth. Can be fully indexed in one pass with no incremental sync needed.

## Incremental Sync

Track last-sync timestamps per category in `metadata/sync-state.json`:

```json
{
  "developer": "2026-03-08T10:00:00+00:00",
  "grasshopper": "2026-03-08T10:15:00+00:00"
}
```

**Sync logic:**
1. Load sync state
2. For each category, fetch discussions updated since last sync
3. Overwrite existing files (they may have new replies)
4. Update sync state after each category completes
5. Save state atomically (write to .tmp, rename)

**For APIs (Discourse, Vanilla):** Use sort-by-latest and stop paginating when you hit posts older than the cutoff.

**For HTML scrapers (phpBB, Drupal):** Fetch first N pages of each forum (sorted by last post date), stop when you see topics older than cutoff.

## Indexing into Statement-MCP

After scraping, ingest the collection:

```python
# Using statement-mcp's ingest_collection
mcp__statement-mcp__ingest_collection(
    path="/path/to/forum/categories",
    collection="mcneel-forum",
    glob_pattern="**/*.md"
)
```

**Re-running is safe:** `ingest_collection` uses content hashes to skip unchanged files and marks deleted files as superseded.

## CLI Pattern

Use Click for CLI commands. Each forum gets its own command:

```python
@cli.command()
@click.option("--output-dir", required=True, type=click.Path())
@click.option("--delay", default=2.0, help="Seconds between API requests")
@click.option("--since", default=None, help="ISO timestamp cutoff")
@click.option("--category", default=None, help="Sync only this category")
@click.option("--skip-replies", is_flag=True, help="Skip fetching replies")
@click.option("--max-pages", default=None, type=int, help="Max pages per category")
def sync_mcneel(output_dir, delay, since, category, skip_replies, max_pages):
    """Sync McNeel/Grasshopper forum discussions."""
    ...
```

**Always include `--max-pages`** for initial testing and to cap runtime.

## Reference Implementation

The Onshape forum syncer at `mcp/tools/tools/sync_forums.py` is the reference:
- 273 lines, Vanilla Forums API v2
- `RateLimiter` class in `utils.py`
- `html_to_markdown()` using markdownify + BeautifulSoup
- YAML frontmatter with engagement metadata
- Incremental sync via `sync-state.json`
- Click CLI with `--delay`, `--since`, `--category`, `--skip-comments`

Adapt this pattern for each new platform. The output format and sync infrastructure are reusable; only the API/HTML fetching layer changes.

## Checklist for Adding a New Forum

1. [ ] Identify platform (Discourse/phpBB/Drupal/blog/other)
2. [ ] Check robots.txt and ToS
3. [ ] Find API or determine HTML selectors
4. [ ] Choose appropriate rate limit delay
5. [ ] Write platform fetcher (categories, topics, posts)
6. [ ] Map API/HTML fields to standard frontmatter
7. [ ] Add CLI command with standard options
8. [ ] Test with `--max-pages 1 --skip-replies`
9. [ ] Run full sync
10. [ ] Ingest into statement-mcp
11. [ ] Verify search quality

## Common Mistakes

| Mistake | Fix |
| ------- | --- |
| No rate limiting | Always use RateLimiter. 2s minimum. |
| Scraping without checking robots.txt | Check first. Respect disallow rules. |
| Storing raw HTML | Convert to markdown. HTML is noisy and wastes tokens. |
| No incremental sync | Always track timestamps. Full re-scrape is wasteful. |
| Missing frontmatter | Every file needs at minimum: id, title, author, created, url, source |
| Giant monolithic scraper | One command per platform. Share utils. |
| No error handling on network | Retry with exponential backoff. Log failures. Continue. |
| Running at full speed on small forums | 3-5s delay for <1K post forums |
