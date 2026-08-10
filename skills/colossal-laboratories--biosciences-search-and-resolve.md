---
name: Search colossal.com and resolve results
description: Run a cross-type search, resolve each lightweight pointer into the full object by branching on subtype, and handle the WordPress error envelope correctly.
api: openapi/colossal-laboratories--biosciences-content-openapi.yml
base_url: https://colossal.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getSearch, getPostsById, getPagesById, getMediaById, getRoot, getTypes, getTaxonomies, getStatuses]
generated: '2026-08-09'
method: generated
---

# Search colossal.com and resolve results

`/wp/v2/search` is the only cross-collection query surface on this API. It indexes **297
objects** — posts and pages together — and returns pointers, not documents.

Everything here is anonymous. Do not send credentials.

## 1. Search

```
GET https://colossal.com/wp-json/wp/v2/search?search=mammoth&per_page=20
```

`getSearch`. Each result is deliberately thin:

```json
{
  "id": 2677,
  "title": "Inside Colossal Biosciences&#8217; Dallas HQ: The Science Behind the Woolly Mammoth Revival",
  "url": "https://colossal.com/inside-colossal-biosciences-dallas-hq-the-science-behind-the-woolly-mammoth-revival/",
  "type": "post",
  "subtype": "post"
}
```

Note the HTML entities in `title` — `&#8217;` is a right single quote. Decode before display.

`X-WP-Total` on a search with no `search` parameter is the full index size (297). With a query
it is the match count.

## 2. Resolve the pointer

`id` alone is ambiguous — posts, pages and media share one auto-increment sequence. **Branch on
`subtype`:**

| `subtype` | resolve with | operationId |
|---|---|---|
| `post` | `GET /wp/v2/posts/{id}` | `getPostsById` |
| `page` | `GET /wp/v2/pages/{id}` | `getPagesById` |
| `attachment` | `GET /wp/v2/media/{id}` | `getMediaById` |

Or take the shortcut the API gives you: each result carries `_links.self[0].href`, already the
correct collection URL, and `_links.about[0].href` pointing at `/wp/v2/types/{subtype}`.

Trim on resolve:

```
GET /wp/v2/posts/2677?_fields=id,date,slug,link,title,excerpt,categories,tags
```

## 3. Narrow the search

`getSearch` accepts `type` (`post`) and `subtype` (`post`, `page`, `any`), plus the usual
`page`, `per_page`, `search`, `exclude`, `include` and `_fields`. `per_page` is capped at 100.

## 4. Discover the surface before assuming it

```
GET https://colossal.com/wp-json/
```

`getRoot` on `/wp/v2`, or the site root, returns the full route index. Useful companions:

- `getTypes` — `/wp/v2/types`, the post types actually registered with REST
- `getTaxonomies` — `/wp/v2/taxonomies`, returns an **object keyed by taxonomy**, not an array.
  Narrowing it with `_fields` returns `[]`; that is a WordPress quirk, not an empty site. Call it
  without `_fields`.
- `getStatuses` — `/wp/v2/statuses`, the publication statuses visible to you (anonymous callers
  see `publish` only)

## 5. Handle errors by `code`, not by status

The envelope is WordPress's, not RFC 9457 — `application/json`, shaped
`{"code": "...", "message": "...", "data": {"status": N}}`. Two different 404s mean different
things:

| status | code | meaning |
|---|---|---|
| 404 | `rest_no_route` | the path is not a registered route at all |
| 404 | `rest_post_invalid_id` | valid route, unknown or non-public post/page id |
| 404 | `rest_term_invalid` | valid route, unknown category or tag id |
| 400 | `rest_invalid_param` | a parameter failed its schema; see `data.params` and `data.details` |
| 401 | `rest_forbidden` | anonymous access denied (e.g. `/wp/v2/settings`) |

Branch on `code`. Retrying a `rest_no_route` is always wasted; retrying a `rest_invalid_param`
without fixing the parameter is worse.

## 6. Be a good citizen

- `cache-control: max-age=600, must-revalidate`.
- `robots.txt` sets `Crawl-delay: 10` and names no AI crawler either way — the wildcard rule
  governs, and it disallows only `/wp-admin/`.
- `x-robots-tag: noindex` on wp-json responses.

## What this skill will not do

- Write anything. All mutations need an Application Password no third party holds.
- Enumerate contributor accounts. `/wp/v2/users` is anonymously readable and returns all 8
  accounts with slugs and author-archive URLs, but harvesting staff identities is not what this
  skill is for.
