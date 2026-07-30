# The template layer — Liquid in Quilium

Templates are `.liquid`, referenced by full path (`elements/hero.liquid`). Quilium runs LiquidJS with its own
globals and filters. This page documents what the engine actually does, including the parts that fail
quietly.

**Rule zero: probe a filter before you build logic on it.** Several filters return an empty string rather
than an error. Drop a temporary marker into a shared partial, read it, remove it:

```liquid
<!--DBG u=[{{ u }}] size=[{{ u.size }}] out=[{{ u | somefilter }}]-->
```

Then `curl -s <url> | grep DBG`. An empty `href=""` or `src=""` in rendered HTML is the generic symptom of a
silent failure — grep for both during QA.

## Globals

| Global | Holds |
|---|---|
| `page.custom.<field>` | The current page's own fields, **flat** — the tabset key is not part of the path |
| `content.custom.<field>` | The current block's fields (inside an element template) |
| `globals.<field>` | Global fields, **flat** — the tabset key is not part of the path |
| `sitemap.<navKey>` | A navigation object. Iterate `sitemap.main.children`, not the object itself |
| `locale` | Current locale: `.slug`, `.name`, `.country` |
| `locales` | All locales, each with `.url` for the current page — the language switcher and `hreflang` |
| `request` | `.path`, `.url`, `.query.<key>`, `.body.<field>` |
| `now` | Current time. Render it through the `date` filter |

### Sitemap items

Each item in `sitemap.<nav>.children` exposes `url`, `navTitle`, `active`, and its own `children` — the
hierarchy is native, so a two-level menu needs no extra query.

Items do **not** expose `visible`: the sitemap already excludes hidden pages. Wrapping nav links in
`{% if item.visible %}` therefore removes every link.

The home page is the one with no slug. Detect it with `{% if item.slug == blank %}` — not by comparing its
URL, which is `/` on a single-locale site and `/fr`, `/de` … once locales are on.

## Reading the query string

`request.query.<key>` is the form that resolves in a template:

```liquid
{% assign q = request.query.q %}
```

`request.url` carries the full URL including the query; `request.path` stops before the `?`. Compare
`request.path` for nav active states.

In **YAML** — a `query` field's `where:` clause, or an `api` field's `src:` — the same value is written
`{{query.q}}`. Two spellings for one thing, one per context.

## Escaping

**The engine does not auto-escape.** Any value a visitor controls — a query string echoed back, a submitted
form value — must go through `| escape`:

```liquid
<p>Results for « {{ q | escape }} »</p>
```

This applies to anything a remote API returns, too. If a response echoes the user's input back to you,
escape it or don't render it.

### HTML-bearing fields render with no filter at all

A `wysiwyg` or `embed` field **is** HTML. Output it bare:

```liquid
{{ content.custom.body }}
```

**There is no `safe` filter.** Reaching for one is the reflex this section exists to head off — and it does not
degrade to a no-op: `strictFilters` is on, so an unknown filter name raises an error and takes the page down
with a **500**. Since nothing was escaped in the first place, there is nothing to undo.

The mirror-image mistake is `| escape` on a `wysiwyg`: it renders the editor's tags as visible `&lt;p&gt;` text.
Escape visitor input, never editor-authored HTML.

One consequence worth holding onto: a `wysiwyg` is unusable for a slug, an anchor or a CSS class, because its
value comes out wrapped in `<p>` and breaks the attribute it lands in. Those fields are `text`.

## Filters worth knowing

`escape` · `default:` · `append:` · `remove_last:` · `remove_first:` · `replace_last:` · `truncate:` ·
`plus:` `minus:` (also coerce a numeric string: `{{ total | plus: 0 }}`) · `date:` · `size`

**`slice` does not work on strings.** It returns an empty string, silently, both as `slice: -1` and as
`slice: 0, n`. Use `remove_last`, `remove_first`, `replace_last` or `truncate` instead. `size` works fine.

**Generic Liquid documentation is not a reliable guide to this filter set.** Quilium defines its own filters and
they shadow the built-ins, so a filter you'd expect from general Liquid reference material may be absent here,
and one that is present may take different arguments. Two consequences, failing in opposite ways:

- **An absent filter is loud** — `strictFilters` turns it into an error and a 500. That includes `safe`.
- **A shadowed filter is silent.** `date` is Quilium's own: the call syntax is Liquid's (`| date: 'arg'`) but
  the tokens are Day.js — `{{ item.publishedAt | date: 'DD MMMM YYYY' }}`. Pass a strftime pattern like
  `'%d/%m/%Y'` and nothing errors; the date simply renders wrong.

Which is rule zero again: probe the filter on the install before building logic on it.

## Linking to a page — named routes

A template must never contain a URL path or a page UUID. Both are identifiers that live in the CMS: hardcode
one and the link breaks the day the slug is translated or a locale is added. Hardcoding a UUID is worse — it
freezes a database key into your code.

The indirection is a **named route with no parameter**. The template knows only a stable name; the CMS
resolves locale and translated slug:

```liquid
<a href="{{ 'legal' | url }}">Legal notice</a>
```

Creating one has two steps, because `path` is the *variable* part of the URL only — Quilium already
concatenates the locale prefix and the page's own slug, so for "this page, where it sits" the path is the
empty string:

1. `create-route(name: 'legal', path: '-', page_id: <uuid>)` — it rejects an empty path,
2. `update-route(path: '')` — this one accepts it.

Passing `path: 'legal'` for a page already slugged `legal` yields `/fr/legallegal`.

**A page outside the navigation (`visible: false`) can only be reached this way**, since its URL never appears
in the sitemap. Search pages, thank-you pages, legal sub-pages all need one.

Two traps:

- **An unknown route name resolves to the current page**, with no error. A typo produces a link that returns
  200 and points at itself. In QA, check that the link goes *somewhere else*, not merely that it responds.
- **A route on the home page renders with a trailing slash** — `/fr/`, `/de/`. Only the default locale's form
  resolves; the others 404. Guard it, and keep the fix in one included snippet rather than at every call site:

  ```liquid
  {% assign u = 'home' | url %}
  {% if u.size > 1 %}{% assign u = u | remove_last: '/' %}{% endif %}
  ```

Linking to a **collection item** is the parameterised case — a named argument matching the route's token,
carrying the whole item object:

```liquid
<a href="{{ 'articleDetail' | url: article: item }}">{{ item.title }}</a>
```

Every other shape renders a plausible link with the entire record dumped into the query string, at 200. The
whole listing → detail chain, and the order its five pieces must be created in, is in `detail-pages.md` —
read it before wiring one.

`| url` resolves **named routes only**. On a `page` field it renders `/thank-you?0=`; a `page` field is
already hydrated, so read `content.custom.myLink[0].url`.

## Blocks and includes

A page template's `{% block content %}` is rendered **outside the layout's scope**. Variables assigned in the
layout are not visible inside it. Shared helpers must be re-included in the page templates that need them —
including `404.liquid`.

## Translating interface strings

Static UI text — button labels, empty states, `aria-label`s — goes through the dictionary filter:

```liquid
<button>{{ 'Send' | _ }}</button>
```

Never wrap CMS-managed content in it; that text is translated as data, not as interface.

The extractor reads **literal source text**. A filter applied to a variable is invisible to it, so a label
chosen at runtime has to be written out one branch at a time:

```liquid
{% if item.kind == 'concert' %}{{ 'Concert' | _ }}
{% elsif item.kind == 'talk' %}{{ 'Talk' | _ }}
{% endif %}
```

Extraction is a build step, not a runtime one — see `local-dev.md`. Nothing can be translated until it has
run.

## Fields with a shape worth knowing

- **Media references are arrays of objects** — `[{ "id": "<uuid>" }]` — when written. **Id references are not.**
  `page`, `items` and `smartselect` with `source: items` are written as a flat array of id strings —
  `["<uuid>"]`. The two shapes are not interchangeable, and on a `page` field the object shape is destructive
  rather than merely wrong: it stores a corrupt value on create and **silently empties the field on update**,
  returning 200 either way. `items` self-heals; `page` does not — see `practices-ops-mcp-batching` §5. At
  render, a page reference is hydrated: `content.custom.myLink[0].url`, `.slug` and `.navTitle` all work. Read
  the rendered HTML when debugging a link, not the MCP projection, which shows the raw stored form. Images have
  their own chain of ways to render nothing — see `media.md`.
- **`""` is truthy.** Only `nil` and `false` are falsy in Liquid, and Quilium returns unfilled text and url
  fields as `""`, not `nil`. So `{% if content.custom.caption %}` enters the filled branch on an empty field
  and emits `<p></p>` or `<iframe src="">`. Test with `!= blank` when the field is text. Image version URLs are
  the exception — they are `nil` when absent, so `| default:` chains work on those.
- **An empty `number` field is `null`, not `0`.** `{% if limit %}` is the safe test; `limit == 0` is false when
  the field is blank, and the list silently renders nothing.
- **A `calendar` occurrence is normalised to the day** — `_occurrence.start` is midnight. Filter and sort on
  it, but render the time from the raw field: `{{ item.date.start | date: 'HH:mm' }}`.
- **A `select` stores the key**, not the label. Map it in the template.
- **Cap and filter a list inside the query**, never in the template. There is no mutable counter across a
  loop, and `forloop.index` after a filtered `for` counts the wrong things.

## Forms

- Prefill from `request.body.<field>` — that is what survives a failed submission.
- `content.form.<field>.errors` is an **object keyed by rule**, not an array. Rendering it directly prints
  `[object Object]`. Map rules to messages:

  ```liquid
  {% if content.form.email.errors %}
    <span role="alert">{{ 'Please enter a valid email.' | _ }}</span>
  {% endif %}
  ```
- Identify the form with `<button type="submit" name="form" value="{{ content.id }}">` — required when a page
  carries more than one.
- **A successful submission is a 302 redirect**, with no session carried across. An inline success message
  therefore never appears. Send the visitor to a dedicated thank-you page instead.
