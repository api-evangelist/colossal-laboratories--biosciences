---
name: Walk the Colossal species program pages
description: Retrieve Colossal's species and program pages as JSON and walk the parent/child hierarchy instead of scraping the rendered site.
api: openapi/colossal-laboratories--biosciences-content-openapi.yml
base_url: https://colossal.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getPages, getPagesById, getMedia, getMediaById, getTypes, getTypesByType]
generated: '2026-08-09'
method: generated
---

# Walk the Colossal species program pages

Colossal's de-extinction and conservation programs are published as WordPress **pages**, not
posts — **59 published** at harvest. The species programs, the technology and labs pages, the
glossary, the FAQ set and the legal pages are all in `/wp/v2/pages`.

Everything here is anonymous. Do not send credentials.

## 1. List every page

```
GET https://colossal.com/wp-json/wp/v2/pages?per_page=100&_fields=id,slug,link,title,parent,menu_order
```

`getPages`. One request covers the whole site — `X-WP-Total` was 59 at harvest, under the
`per_page` ceiling of 100.

## 2. Walk the hierarchy

`parent` is `0` for a top-level page and carries the parent page id otherwise. The dire wolf
program is the deepest branch on the site:

```
/direwolf/                 parent: 0
/direwolf/science/         parent: <id of /direwolf/>
/direwolf/biology/
/direwolf/culture/
/direwolf/conservation/
```

Fetch a branch directly:

```
GET /wp/v2/pages?parent=<id>&orderby=menu_order&order=asc
```

`menu_order` is the editorially-set sequence — use it, not alphabetical order, when presenting
a program.

## 3. Resolve a page by slug

Slugs are the stable handle; ids are auto-increment and shared across posts, pages and media in
one table, so an id alone is meaningless without its collection.

```
GET /wp/v2/pages?slug=mammoth&_fields=id,slug,link,title,content
```

The species programs at harvest: `mammoth`, `thylacine`, `dodo`, `direwolf`, `moa`, `bluebuck`.
Supporting programs: `technology`, `labs`, `conservation`, `elephant-conservation`, `education`,
`species`, `de-extinction`, `foundation`, `glossary`, `careers`, `company`, `advisors`,
`george-church`.

## 4. Pull the imagery

```
GET /wp/v2/media?per_page=100&_fields=id,slug,link,title,alt_text,media_type,mime_type,source_url
```

`getMedia` — 990 items at harvest, the largest collection on this API. Each carries `source_url`
(the direct file), `alt_text` and, untrimmed, a `media_details` object with every generated size
variant. Attachments nest under their owning post's permalink, so `link` tells you which piece
of content an asset belongs to.

To get only a page's own attachments, follow `_links["wp:attachment"]` on that page, which
resolves to `/wp/v2/media?parent={id}`.

## 5. Confirm what is actually exposed before you go looking

```
GET /wp/v2/types
```

`getTypes` returns the post types registered with the REST API. On colossal.com that is
`post`, `page`, `attachment` and a set of editor-internal types (`wp_block`, `wp_template`,
`wp_navigation`, `wp_font_family`, plus plugin types `wppopups-templates`, `rm_content_editor`,
`rank_math_schema`).

It does **not** include `paper`, `podcast`, `team`, `faq` or `pdf`. Those content types exist —
they have their own sitemaps at `/paper-sitemap.xml`, `/podcast-sitemap.xml`,
`/team-sitemap.xml`, `/faq-sitemap.xml` and `/pdf-sitemap.xml` — but they are not registered
with the REST API, so **there is no JSON for them**. If you need Colossal's scientific papers,
podcast catalog, team roster or FAQ corpus, the sitemaps plus rendered HTML are the only route,
and that is a gap for Colossal to close, not a request to keep retrying.

## 6. Be a good citizen

- `cache-control: max-age=600, must-revalidate`. Honor it.
- `robots.txt` sets `Crawl-delay: 10`. Treat it as the floor for API polling too — Colossal
  publishes no other pacing signal.
- `x-robots-tag: noindex` on every wp-json response.

## What this skill will not do

- Write. All create/update/delete requires a WordPress Application Password no third party holds.
- Invent an endpoint for the unexposed post types above.
