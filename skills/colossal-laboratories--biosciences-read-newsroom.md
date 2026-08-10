---
name: Read the Colossal newsroom and science insights
description: Page through Colossal's published posts and resolve categories, tags, authors and featured images in a single request.
api: openapi/colossal-laboratories--biosciences-content-openapi.yml
base_url: https://colossal.com/wp-json/wp/v2
auth: none (anonymous read)
operations: [getPosts, getPostsById, getCategories, getCategoriesById, getTags, getTagsById, getUsers]
generated: '2026-08-09'
method: generated
---

# Read the Colossal newsroom and science insights

Colossal keeps everything editorial in the standard WordPress `posts` collection — **238
published** at harvest. There is no separate press-release post type: news releases, conservation
updates and long-form science explainers all live in `/wp/v2/posts` and are separated only by
category.

Everything here is anonymous. Do not send credentials.

## 1. List posts, newest first

```
GET https://colossal.com/wp-json/wp/v2/posts?per_page=20&page=1&orderby=date&order=desc
```

`getPosts`. Read the pagination from the response headers, never from the body:

- `X-WP-Total` — total posts (238 at harvest)
- `X-WP-TotalPages` — pages at the current `per_page`
- `Link: <...>; rel="next"` — the RFC 8288 next page

`per_page` is capped at **100**. Exceeding it returns `400 rest_invalid_param` with
`data.details.per_page.code = rest_out_of_bounds` — see
`errors/colossal-laboratories--biosciences-problem-types.yml`.

## 2. Trim the payload

Post bodies are full rendered HTML and run to tens of kilobytes. Ask for only what you need:

```
GET /wp/v2/posts?per_page=20&_fields=id,date,slug,link,title,excerpt,categories,tags,author,featured_media
```

`_fields` takes a comma-separated list of top-level fields. On a sampled request this returned
exactly those keys and nothing else.

## 3. Filter by category

`getCategories` returns all 15 terms in one call:

```
GET /wp/v2/categories?per_page=100&_fields=id,name,slug,count
```

The ones that matter:

| id | slug | count | what it is |
|---|---|---|---|
| 5 | `news` | 172 | press releases and news coverage |
| 6 | `insight` | 52 | long-form science writing |
| 183 | `george-church` | 9 | co-founder commentary |
| 154 | `dodo` | 8 | dodo program |
| 108 | `thylacine` | 8 | thylacine program |

Then filter:

```
GET /wp/v2/posts?categories=6&per_page=20&orderby=date&order=desc
```

Watch out: `home-page` (12), `wooly-mammoth-page` (22) and `science-technology-page` (23) are
**placement flags**, not subjects. A post carrying `home-page` is not about home pages. Use tags
for subject matter.

## 4. Filter by tag

`getTags` — 156 terms, and this is the real subject vocabulary:

```
GET /wp/v2/tags?per_page=100&orderby=count&order=desc&_fields=id,name,slug,count
```

Top terms at harvest: `de-extinction` (10), `conservation` (8), `woolly-mammoth` (8),
`rewilding` (5), `disruptive-conservation` (4). Filter with `?tags=34`.

## 5. Resolve related objects in one round trip

Add `_embed` instead of making follow-up calls:

```
GET /wp/v2/posts?per_page=10&_embed
```

The `_embedded` object then carries `author`, `wp:featuredmedia` and `wp:term` inline. Without
it you would walk `_links` — a post carries `self`, `collection`, `about`, `author`, `replies`,
`version-history`, `predecessor-version`, `wp:attachment`, `wp:featuredmedia`, `wp:term` and
`curies`.

Note that `author` is usually **16**, a shared organizational account bylined "Colossal
Biosciences", not an individual. `getUsers` enumerates all 8 contributor accounts anonymously;
resolve it only if you actually need the byline.

## 6. Fetch one post

```
GET /wp/v2/posts/2831
```

`getPostsById`. An unknown or non-public id returns `404 rest_post_invalid_id`. An unknown term
id on `/wp/v2/categories/{id}` returns `404 rest_term_invalid` — the two codes differ, so branch
on `code`, not on the status.

## 7. Be a good citizen

- Responses declare `cache-control: max-age=600, must-revalidate`. Honor it; do not re-poll
  inside ten minutes.
- No rate-limit headers are published. `robots.txt` sets `Crawl-delay: 10`, which is the only
  pacing signal Colossal publishes anywhere — treat it as the floor.
- Responses carry `x-robots-tag: noindex`. Do not republish this content as your own index.

## What this skill will not do

- Write. Every create/update/delete on this API requires a WordPress Application Password from
  `https://colossal.com/wp-admin/authorize-application.php`. No third party holds one.
- Reach papers, podcast episodes, team profiles, FAQs or PDFs. Those custom post types exist on
  colossal.com and appear in the sitemap, but they are not registered with the REST API, so there
  is no endpoint for them.
