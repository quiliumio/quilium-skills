---
name: quilium-builder
description: Load when building or restructuring a Quilium CMS site — designing the architecture (page-types, content-types, ItemSets, global fields), writing YAML schemas, wiring routes and detail pages, scaffolding Liquid templates, porting a static HTML prototype, uploading media, or setting up a project locally. Covers the end-to-end process for a new site, the listing-to-detail chain, images, the 404, and the CLI loop (init, run, push, dictionary). Triggers on "new Quilium site", "build this site", "design the architecture", "create a content type", "add a page type", "scaffold a template", "port this prototype", "detail page", "the links point to the wrong page", "images are missing", "the 404 doesn't work", "set up the project", "quilium init", "quilium push", "the site doesn't render". For content work on a site that already runs, use quilium-webmaster instead.
---

# Quilium builder

You build Quilium sites: the structure underneath them, the templates that render them, and the local loop
that lets you see your work before anyone else does.

This skill is a **guide, not a manual**. Quilium ships a live skill library through its MCP server, and that
library is the source of truth for every schema, field type and API shape. Your job is to know *what to load
and when*, not to remember what it says.

## The skills-first protocol — non-negotiable

Before any non-trivial CMS operation:

1. Call **`get-skills`** to get the live manifest. Cache it for the session.
2. Pick **every** skill that matches the task. A real task needs 2–6, not one.
3. Load each with **`get-skill <slug>`**, in parallel when independent. Follow `related:` transitively.
4. Call **`get-site-settings`** to learn what this site actually has. Never infer structure from another project.
5. Then act.

Two skills are worth loading in almost every session: `meta-getting-started` once, and
`practices-ops-mcp-batching` for anything over a handful of calls.

### The library is smaller than you expect, and that is not an error

**What `get-skills` returns is the whole of what you can load.** The library is gated by profile and by site,
and the set you get may be a fraction of what exists — with whole areas absent: templates, routes, YAML
schema syntax, forms, pagination.

So: if a slug isn't in the manifest, it doesn't exist for you. `get-skill` will refuse it. Don't retry it,
don't work around it by inventing what it probably said, and above all **don't leave the gap unfilled and
improvise the implementation**. That last one is how a detail page ships with routing that returns 200 on
every URL and resolves none of them.

When a skill you wanted isn't there, the bundled references below cover the areas that are most often missing
and most expensive to get wrong. If neither has it, say so plainly and probe the engine (rule zero, in
`references/engine-liquid.md`) instead of guessing.

## The four primitives

A site is built from four things: **page-types** (a page's structure and its own fields), **content-types**
(reusable blocks dropped onto pages), **ItemSets** (managed collections), and **global-fields** (site-wide
singletons).

Choosing between them is the highest-leverage decision in the project and the most expensive one to get
wrong. It also already has a dedicated decision matrix, with a five-question flowchart and a worked list of
anti-patterns:

**Load `practices-conventions-architecture` before you commit to a shape**, along with anything the manifest
offers next to it. Not after. It will tell you things that contradict your instinct — that most pages need no
page-type of their own, that a list you were about to model as a `repeat` is one rich-text field, and that
almost nothing belongs in global fields.

Don't take a compressed version of that matrix from here or anywhere else. Read it live.

## Building a new site

Five steps, in this order. The order encodes dependencies — see `references/process-new-site.md` for the
detail and the reasoning.

1. **Architecture** — decide the shape, write it down before writing YAML.
2. **CMS base** — ItemSets, content-types, page-types, global fields, then routes.
3. **Base layout** — one `layout.liquid` that renders, with header, footer and nav — and the 404, registered
   in the settings.
4. **Templates** — one per content-type and page-type.
5. **Media, then content** — uploads first, since content references them by uuid. In draft.

Templates come **before** content: you want to see a block render before you author fifty of them.

## Absolute rules

These aren't preferences. Each one, broken, produces a failure that looks like something else.

- **Templates are `.liquid`**, full path always (`elements/hero.liquid`, never `hero.liquid`). If a loaded
  reference shows another extension, it is out of date — `.liquid` wins.
- **Never write a URL path or a page UUID in a template.** Both are CMS identifiers; a template that hardcodes
  one breaks the day a slug is translated or a locale is added. The indirection is a **named route**, consumed
  as `{{ 'routename' | url }}`. See `references/engine-liquid.md`.
- **Every wrong way to call the `url` filter returns 200.** A typo'd route name points at the current page; a
  mis-shaped argument dumps the whole record into the query string. Detail pages are the sharp edge —
  `references/detail-pages.md` before you wire one.
- **Images are uploaded, not pushed.** `quilium push` carries `public/`; content images live in the media
  library and reach it only through the MCP. An image you never uploaded renders `src=""` and warns nobody.
- **Register the 404 in the settings.** A `404.liquid` that exists on disk but isn't declared in
  `miscellaneous.rendering.404` is not the site's 404 — every unknown URL errors instead.
- **Define page-types before creating pages.** A page freezes its template at creation; changing the page-type
  afterwards does not propagate, and the page 500s while the settings still look correct.
- **Create pages `state: active`.** `state` means routable, not published. An `inactive` page 404s everywhere,
  including locally. "Leave it in draft" means don't publish a live revision — never deactivate.
- **A settings write replaces its target wholesale.** `update-settings-partial` replaces the whole block you
  aim at, and `update-setting` replaces the entire file. Always read the current value, merge your change into
  it, and write the result back complete — including the parts you have no opinion about.
- **Nothing publishes itself.** Publishing is a human decision, page by page.
- **Probe before building logic on a filter.** Drop `<!--DBG x=[{{ … }}]-->` into a shared partial, `curl | grep
  DBG`, remove it. Some filters return empty without erroring; a wrong assumption here costs an afternoon.

## Where to look — intent to source

Build this table from `get-skills` at the start of the session; the slugs below are the ones commonly
present, not a guarantee. **If the manifest doesn't offer it, use the fallback column** — don't wait for a
skill that isn't coming.

| You're about to… | Load if offered | Otherwise |
|---|---|---|
| Decide the shape of a feature | `practices-conventions-architecture` | ask the wysiwyg question first: a bullet list or a paragraph-with-CTA is one field, not a `repeat` |
| Choose a field type | `reference-fields-catalog` | probe one in the back-office before committing a schema to it |
| Name anything | `practices-conventions-naming` | camelCase keys, consistent site-wide, full template path, `.liquid` |
| Design the editor's experience | `practices-conventions-cms-ux` | — |
| Write a template | — | `references/engine-liquid.md` |
| Wire a listing and its detail pages | — | `references/detail-pages.md` |
| Upload and render images | `tasks-media-crud` (library ops) | `references/media.md` (schema + template side) |
| Port an approved HTML prototype | — | `references/from-a-prototype.md` |
| Set up the 404 | — | `references/process-new-site.md`, step 3 |
| Move a page in the tree | `reference-navigation` | `reorder-navigation`, never `update-page` |
| Batch more than a handful of MCP calls | `practices-ops-mcp-batching` | serialise and read each response |
| SEO metadata | `practices-compliance-seo-basics` | — |
| Run the CLI | — | `references/local-dev.md` |

## Bundled references

- `references/process-new-site.md` — the five steps in full, with the dependency order and why it holds.
- `references/engine-liquid.md` — the template layer: globals, filters, and what fails silently.
- `references/detail-pages.md` — the listing → detail chain: route, query field, hidden page, links.
- `references/media.md` — images end to end: upload, reference, schema versions, render.
- `references/from-a-prototype.md` — porting an approved HTML prototype without redesigning it.
- `references/local-dev.md` — `init`, `run`, `push`, `dictionary`, and the debugging loop.

## What this skill is not

Not a schema reference — that's the MCP library, and it's more current than any copy.
Not a content-authoring guide — that's `quilium-webmaster`.
Not site-specific — per-site conventions belong in the project's own `CLAUDE.md`.
