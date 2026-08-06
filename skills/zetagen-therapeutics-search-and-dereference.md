---
name: Search the site and dereference results
description: >-
  Use the Zetagen Therapeutics federated search collection as the entry point, then resolve
  lightweight stubs to full posts and pages — the correct traversal pattern for this API.
api: openapi/zetagen-therapeutics-content-openapi.yml
operations: [search, getPost, getPage, listPages]
base_url: https://zetagen.com/wp-json
auth: none
generated: '2026-08-05'
method: generated
---

# Search the site and dereference results

The whole zetagen.com corpus is 57 searchable objects. Paging every collection to find one thing is
wasteful; the API gives you a federated search that returns stubs plus a link to the full record.

## Step 1 — search (`search`)

`GET /wp/v2/search?search=<terms>&per_page=20`

Returns lightweight records: `id`, `title`, `url`, `type`, `subtype`, plus `_links.self` pointing at
the full object. `X-WP-Total` was 57 for an empty query at harvest — that is the entire searchable
corpus, so an unfiltered call is a cheap way to enumerate everything.

Useful queries against this site: `ZetaMet`, `ZetaFuse`, `ZetaMAST`, `Zeta-BC-003`, `Phase 2a`,
`Breakthrough Device`, `Series B`, `patent`.

Filter with `subtype` to restrict to `post` or `page`.

## Step 2 — dereference (`getPost` / `getPage`)

Read `_links.self[0].href` off each stub and follow it. Do not reconstruct the URL from `id` alone —
a stub can be either a post or a page and the collection differs. If you must branch manually:

- `subtype: post` → `GET /wp/v2/posts/{id}` (`getPost`)
- `subtype: page` → `GET /wp/v2/pages/{id}` (`getPage`)

Add `_fields=id,slug,link,title,content,date,modified` to drop the inlined SEO payload.

## Step 3 — walk the page hierarchy instead (`listPages`)

When you want structure rather than a keyword, page the 25 published pages:

`GET /wp/v2/pages?per_page=100&orderby=menu_order&order=asc&_fields=id,parent,menu_order,slug,link,title`

Pages are hierarchical and `parent` is the edge (`0` at the root). Two branches carry the substance:

- `/research-overview/` parents `clinical-trials`, `expanded-access-policy` and `regulatory-status`
- `/news/` parents `press-releases` and `publications`

Build the tree from `parent` and you have the site's information architecture without scraping HTML.

## Error handling

| status | `code` | meaning |
|---|---|---|
| 400 | `rest_invalid_param` | a query param failed validation; read `data.params` for which one |
| 404 | `rest_post_invalid_id` | no such post/page/attachment, or not publicly viewable |
| 404 | `rest_no_route` | wrong path **or** wrong method for a registered path |

Branch on `code`, not on `message`. The envelope is `{code, message, data:{status}}` served as
`application/json` — it is not RFC 9457, so there is no `type` URI to key on.

## Rules

- **Do not call `/wp/v2/users`.** Personal data; excluded on purpose — see `skills/_index.yml`.
- `context=edit` returns 401 anonymously. Use the default `view`, or `embed` for stubs.
- Nothing on this surface is writable without a WordPress account. This skill is read-only.
