---
name: Track Zetagen's public disclosure record
description: >-
  Pull Zetagen Therapeutics' press releases and peer-reviewed publications from its WordPress
  content API, filter by category and date window, and follow updates over time.
api: openapi/zetagen-therapeutics-content-openapi.yml
operations: [listCategories, listPosts, getPost, listTags]
base_url: https://zetagen.com/wp-json
auth: none
generated: '2026-08-05'
method: generated
---

# Track Zetagen's public disclosure record

Zetagen Therapeutics is a private, clinical-stage biopharmaceutical company. Its entire public
disclosure record — funding rounds, patent grants, FDA and Health Canada regulatory milestones,
trial enrollment and conference presentations, running unbroken from 2018-09-05 to 2026-06-09 —
lives in one WordPress `post` collection. There is no developer program, no API key and no signup.

## Before you start

- Base URL: `https://zetagen.com/wp-json`
- No credentials. Every operation below returned 200 anonymously.
- `per_page` maximum is **100**. Above it the API returns HTTP 400 `rest_invalid_param` — it does
  **not** clamp. See `errors/zetagen-therapeutics-problem-types.yml`.
- Branch on the `code` field of the error envelope, never on the message string.

## Step 1 — resolve the category ids (`listCategories`)

`GET /wp/v2/categories`

There is **no custom post type** on this site. Press releases and publications are both ordinary
posts, and the category term is the only machine-readable discriminator. At harvest:

| id | slug | count | what it is |
|---|---|---|---|
| 2 | `press-release` | 26 | company-issued press releases |
| 7 | `publications` | 6 | peer-reviewed publication and conference-abstract entries |
| 1 | `uncategorized` | 0 | empty |

Resolve these ids at run time rather than hard-coding them — they are WordPress term ids and a
re-import would change them.

## Step 2 — page the disclosure archive (`listPosts`)

`GET /wp/v2/posts?categories=2&per_page=100&orderby=date&order=desc`

- Read `X-WP-Total` and `X-WP-TotalPages` from the response headers to size the walk.
- Follow the RFC 8288 `Link` header `rel="next"` rather than incrementing `page` yourself.
- Narrow the payload with `_fields=id,date,modified,slug,link,title,excerpt,categories` — the full
  object inlines a large `yoast_head` string and a `yoast_head_json` schema.org graph you almost
  never need.
- Swap `categories=2` for `categories=7` to get the publications set instead.

## Step 3 — incremental follow-up

`GET /wp/v2/posts?modified_after=<last-run ISO 8601>&orderby=modified&order=desc`

`date` is site-local (America/Toronto, UTC−4); `date_gmt` and `modified_gmt` are UTC. Compare on the
GMT fields. Posts are edited after publication — the June 2026 ASCO release was published
2026-06-09 and modified 2026-06-19 — so poll on `modified_after`, not `after`.

## Step 4 — fetch one release in full (`getPost`)

`GET /wp/v2/posts/{id}`

`content.rendered` is HTML. `featured_media` is an attachment id (`0` when unset) resolvable at
`/wp/v2/media/{id}`, or inline it with `_embed`.

## Optional — tags (`listTags`)

`GET /wp/v2/tags` — one term is registered site-wide. Tags are not a useful axis here; use
categories.

## Rules

- **Do not call `/wp/v2/users`.** It answers anonymously and returns personal data. It is excluded
  from this skill on purpose — see `skills/_index.yml`.
- No rate limit is signalled (no `RateLimit-*` headers) and none was observed, but this is a small
  corporate site on a single LiteSpeed origin with no CDN. Page politely; do not parallelise the
  whole archive.
- Responses carry no `Cache-Control` or `ETag`, so every request reaches the origin. Cache on your
  side, keyed on `modified_gmt`.
- Everything you retrieve is a **company statement** about its own clinical and regulatory
  programme. Attribute it as such; it is not independent verification.
