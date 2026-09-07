# Listing → detail pages

The single most failure-prone chain in Quilium. A collection renders fine, the links look right, every URL
returns 200 — and the detail pages show the wrong thing, or nothing, or the listing again.

Nothing here errors when you get it wrong. Read the whole page before wiring your first detail page.

## The chain

Five pieces have to agree, and they are created in this order:

1. an **ItemSet** holding the items, each with a `slug`,
2. a **page-type** for the detail page, carrying a `query` field that resolves one item by slug,
3. a **page** using that page-type — hidden from the navigation, but active,
4. a **named route** pointing at that page, whose path carries the slug token,
5. the **listing template**, which builds its links through that route.

Invert 3 and 4 and nothing works: the route needs the page's id.

## 1 · The route

A route is not a settings file. It is created through `create-route` / `update-route`, read with `get-routes`.
`update-settings-partial` cannot touch routes.

```
create-route(name: 'articleDetail', path: '/{{article.slug}}', page_id: <detail page uuid>)
```

Three things about `path`:

- **It is the variable part of the URL only.** Quilium already prefixes the locale and the host page's own
  path. The final URL is *page URL + path*: a detail page sitting under `/news` with `path: '/{{article.slug}}'`
  serves `/en/news/<page-slug>/my-article`.
- **The token is interpolated, not colon-prefixed.** `{{article.slug}}`, not `:slug` and not `{slug}`. The name
  inside the token is yours to choose — it is the handle you will use everywhere else.
- **The host page always has its own slug — never leave it empty.** The route path is appended to the page's
  URL; it never replaces the page's slug. Two shapes are legal, both with a slugged child:
  - child `article` under `/news`, « Prefix URL with parent slugs » on (`isSlugPrefixed: true`, the default)
    → `/en/news/article/my-article`;
  - child `article` with `isSlugPrefixed: false` → `/en/article/my-article`, wherever it sits in the tree.

  What does **not** work is emptying the child's slug and writing `/news/{{article.slug}}` into the route,
  hoping for `/en/news/my-article`. A child with an empty slug computes the same URL as its parent, and the
  `unique_url` rule rejects the save — the page is never created, and the route has nothing to attach to.
  `/en/news/my-article` is not reachable through a hidden child page at all. The only way to get it is to host
  the route on the listing page itself (`/news` + `/{{article.slug}}`) and let one page-type serve both the
  listing and the detail — which is why section 6 insists on three branches.

A route with **no** parameter — for a fixed page you need to link to — is the empty path, and it takes two
calls because `create-route` rejects `path: ""`:

```
create-route(name: 'legal', path: '-', page_id: <uuid>)
update-route(path: '')
```

Passing `path: 'legal'` for a page already slugged `legal` yields `/en/legallegal`.

## 2 · The detail page-type resolves the item

There is **no `item` global**. The detail page-type declares a `query` field whose `where:` clause matches the
route token, and the template reads the result off `page.custom`.

```yaml
articleDetail:
  name: Article — detail
  template: layouts/articleDetail.liquid
  tabsets:
    config:
      fields:
        article:
          type: query
          itemSet: article
          where: { '&=': { slug: '{{route.article.slug}}', state: Actif } }
          limit: 1
```

`{{route.<token>.slug}}` is the **YAML-side** spelling and works only inside a `query` field's `where:`. In a
template, `route.<token>.slug` is empty — that is not where you read it.

## 3 · The template reads an array, always

```liquid
{%- assign article = page.custom.article[0] -%}   {# correct #}
{%- assign article = page.custom.article -%}      {# renders empty, no error #}
```

`limit: 1` does not unwrap it. The `[0]` is mandatory.

## 4 · Linking from the listing

One form works: **a named argument whose name is exactly the route's token.**

```liquid
<a href="{{ 'articleDetail' | url: article: item }}">{{ item.title }}</a>
```

You pass the **whole item object**, not its slug — the route extracts `.slug` itself. A route with two tokens
takes two arguments, and a token spelled `{{slug}}` takes a scalar:

```liquid
{{ 'eventDetail' | url: event: event, dateStart: eventDateStart }}
{{ 'bookingDetail' | url: slug: box.slug }}
```

Everything else fails **silently and plausibly**:

| What you write | What renders |
|---|---|
| `{{ 'articleDetail' \| url: item }}` | `/news/[object Object]?title=…&body=%3Cp%3E…` — the whole record in the query string |
| `{{ item \| url: 'articleDetail' }}` | right path, **plus** the whole record in the query string |
| `{{ 'articleDetail' \| url }}` | `/news/[object Object]` |

All three return 200. None raise anything.

Two more traps on the same filter:

- **An unknown route name resolves to the current page.** A typo gives you a working link that points at
  itself. In QA, assert the link goes *somewhere else* — not merely that it responds.
- **`| url` is for named routes only.** `{{ content.custom.ctaPage | url }}` on a `page` field renders
  `/thank-you?0=`. A `page` field is already hydrated: read `content.custom.ctaPage[0].url`.

## 5 · The page that hosts the detail

Created like any other page, with two settings that are easy to confuse:

- **`visible: false`** — keeps it out of the navigation and the sitemap. That is exactly why it needs a route:
  a hidden page's URL cannot come from `sitemap.<nav>.children`.
- **`state: active`** — it must stay active. `state` is routability, not publication. An `inactive` page 404s
  everywhere, including locally, and `publish-page` does not change it.

If the detail URL should sit under its listing (`/news/article/my-article`), re-parent the hidden page under
the listing page and keep « Prefix URL with parent slugs » on. Re-parenting is `reorder-navigation` —
`update-page` does not move a page in the tree. If it should sit at the root (`/article/my-article`), set
`isSlugPrefixed: false` on the page instead. Either way the page keeps its own slug (rule 3 above).

An ItemSet's back-office `preview` link only starts working once the route exists.

## 6 · Guard the unresolved case — three branches, not two

If the query matches nothing, `page.custom.article[0]` is empty and the template renders its shell around
nothing. Worse, when one page-type serves both a listing and its details, an unmatched slug renders **the
listing's content under the detail URL, with a 200**: `/news/anything-at-all` looks like a real page.

So branch on three cases, not two:

1. no slug in the URL → the listing,
2. a slug that resolves → the detail,
3. a slug that resolves to nothing → the not-found branch.

Liquid cannot set the HTTP status code. Case 3 renders a not-found *message* at 200 unless your page-type is
built so the CMS itself misses. Say so to whoever asked, rather than reporting a working 404.

## QA for this chain

- Every listing link points somewhere **other than** the listing page.
- Grep the rendered HTML for `[object Object]` and `%5Bobject` — a route parameter with no ItemSet behind it
  serialises that way into `locales[*].url`, which breaks the language switcher and `hreflang` while every
  page still returns 200.
- Hit a detail URL with a slug that does not exist, and check what comes back.
- Check one detail URL in **every** locale, not just the default one.
