# Building a new site — the five steps

The order below is not a style preference. Each step creates something the next one needs, and three of the
dependencies fail *silently* if you invert them. Where that's the case it's called out.

## Step 0 — the project exists

`quilium init` **refuses a non-empty directory**. Run it first, in an empty folder, before adding anything of
your own. That folder is the project root; Quilium lives at the root, never in a subfolder.

Decide the **locales now**, even if you'll only author one at first. Turning on i18n later introduces two
regressions that raise no error: the home page stops being `/` and becomes `/fr`, breaking any test that
compares URLs; and every node resolved by URL comparison comes back empty outside the default locale.

Details on the CLI: `local-dev.md`.

## Step 1 — architecture

Decide the shape **before** writing YAML, and write the decision down — a short `ARCHITECTURE.md` listing
pages, content-types, collections and routes is enough. It's what you'll check the built site against.

**Load `practices-conventions-architecture` here, not later**, plus whatever the manifest offers beside it.
It carries the decision matrix — the questions to ask, a use-case table, and the anti-patterns that cause most
rework. Everything you need to choose between a collection, a reusable block, an inline repeat, a composed
flow of typed blocks, and plain rich text is in it.

Work through it per feature and record the outcome in your architecture note. What you write down is a
decision with a reason, not a shape you liked: *"team members → ItemSet, because the client adds people and
wants them filterable"*.

Resist compressing its guidance into a rule of thumb you carry between projects. The matrix exists because
the intuitive answer is wrong often enough to matter.

If the skill isn't in your manifest, the one question that prevents most rework is: **can the editor express
this by typing rich text?** A bullet list, a list of links, a paragraph with a button — those are one
`wysiwyg` field. A `repeat` is for typed mini-records whose shape rich text cannot carry: icon + title +
image + link, repeated.

### Ask the rich-text question per block, and answer the palette question with it

Run it on every candidate block and record the outcome, the same way you record the shape. It has two halves
and the second is the one that gets skipped:

1. **Should this be rich text?** Standard HTML the editor could type → one `wysiwyg`. Mini-record shapes, and
   anything landing in an attribute, URL, slug, anchor or CSS class → typed fields.
2. **Does the style palette need a variant for it?** If the answer to (1) is yes *only if* the editor can apply
   a button, a styled list or a callout, then the palette needs that variant — and that is a decision, not a
   detail. Skipping it is what forks the block back into four rigid `text` fields.

**Load `reference-wysiwyg-config` here, with `practices-conventions-architecture`.** The palette is site-wide
settings, so it is decided once, now, for the whole site. If the skill isn't in your manifest, read the site's
existing `wysiwyg` settings and mirror that shape rather than inventing the syntax.

This has to happen before step 2 because each variant the palette covers is a field you never declare — and
because reversing it later is a migration, not an edit: schema changes plus re-authoring every block whose
content was split across typed fields.

## Step 2 — the CMS base

Write the schemas, then the pages, then the routes.

Before the first key: **every identifier in this YAML is English `camelCase`** — type keys, field keys,
tabsets, `select` option values, image versions, route names. The site's language lives in the labels, never
in the keys. This is the cheapest rule in the project to follow now and the most expensive to retrofit: keys
are what templates address by dot access, so renaming one later means touching every template that reads it
*and* migrating the content already stored under the old key.

Within this step the order is:

**0 · the wysiwyg style palette** — write it before the schemas that assume it. Every variant it carries is a
field the content-types below don't declare, so writing it first is what keeps step 1's decisions from quietly
reverting to typed fields at the keyboard. It lives in site-wide settings, which means the read → merge → write
rule at the end of this step applies to it too. Details in `reference-wysiwyg-config`.

**1 · ItemSets** — including the collection that will store form submissions, which must exist before the form
content-type that writes into it.

**2 · content-types** — every one with an explicit `template:`. A type without one falls back to a filename
derived from its key, which won't exist, and you get a dangling reference that `push` reports later.

**3 · page-types** — before any page is created. ⚠️ **A page freezes its template at creation.** Change the
page-type's `template:` afterwards and existing pages keep the old one: they 500 while `get-page-with-content`
still shows the correct value. The fix is to re-save every affected page; the cure is to define page-types
first.

**4 · global fields** — almost never. The logo is the one widely accepted case. Contact details, social links,
footer text and tracking all *feel* global and are better as content blocks the editor can move and duplicate.
If you're adding a second global field, re-read `practices-conventions-architecture` before you do.

**5 · pages** — created `state: active`. ⚠️ `state` means *routable*, not *published*. An `inactive` page 404s
everywhere including locally, and `publish-page` does not change it. "Leave everything in draft" means don't
publish a live revision — never deactivate.

**6 · routes** — last, because a route points at a page that must already exist. Every page outside the
navigation needs one, since its URL can't come from the sitemap.

⚠️ **If the site has any listing with detail pages, read `detail-pages.md` before this step, not after.** The
ItemSet, the detail page-type's query field, the hidden page, the route and the listing template all have to
agree on one token, they are created in a fixed order, and every way of getting it wrong returns 200.

Writing YAML has one hazard worth naming before you meet it: **a settings write replaces its target
wholesale.** `update-settings-partial` replaces the whole block it aims at — and only handles page-types,
content-types and ItemSet types — while `update-setting` replaces an entire settings file. Read the current
value, merge, write the complete result back. On a fresh project a settings file is often empty, which makes
the first write feel safe and teaches the wrong habit.

## Step 3 — the base layout

One `layout.liquid` that renders end to end: `<head>`, header with navigation, footer, `{% block content %}`.

Get this right before writing any element template, because everything inherits it and every bug in it appears
on every page at once. Build the navigation from `sitemap.<nav>.children` and the language switcher from
`locales` — both are locale-aware already.

Remember that a page template's `{% block content %}` renders outside the layout's scope: helpers assigned in
the layout are not visible inside it.

### The 404, in this step

Write it now, with the layout — it is the one template every unknown URL depends on, and it is the one
everybody forgets until a delivery check finds it.

**It has to be registered in the settings.** The engine renders the template named in
`miscellaneous.rendering.404`, and falls back to a filename that your project does not contain. Without the
key, every unknown URL raises a template error instead of showing your page.

The `miscellaneous` settings are a single object, so this is a full-file write: read the current value, change
only the 404 key, write everything else back unchanged. If it is empty, set the key anyway.

Two things about the 404 template itself:

- **No CMS page backs it.** There is no `page.custom`, no content blocks, no `page.metas` — only the site-wide
  context (sitemap, globals, locales). Its copy is written in the template, through the translation filter so
  it can still be localised.
- **`{% block content %}` renders outside the layout's scope**, so any shared helper the layout assigns has to
  be re-included here. And its "back home" link needs the trailing-slash guard from `engine-liquid.md`, or it
  dies in every secondary locale — a link nobody tests, on a page nobody visits, until both matter.

## Step 4 — templates

One template per content-type and page-type, referenced by full path.

Work against a rendering site: `quilium run` serves draft pages locally, so you can see a block before any
content exists beyond a test instance. Restart the server after changing a template or a schema — it caches
both.

**Working from an approved HTML prototype? Read `from-a-prototype.md` first.** Porting markup faithfully is a
discipline, not a formality, and the drift it prevents is invisible in the content you write back.

## Step 5 — media, then content

**Media first.** Images live in the media library, uploaded through the MCP — they do not travel with
`quilium push`, and content can't reference a media that doesn't exist yet. Upload them with their alt text,
keep the file → uuid table, then wire the fields. `media.md` covers the whole path, including why a missing
version key renders `src=""` without erroring.

This is where the work changes nature: the structure is done, what follows is content authoring. Switch to
`quilium-webmaster` here — its discipline (write wrappers, reference shapes, the inverted translation contract,
never publishing unasked) applies to every call of this step, and to the site's own procedures if it has any.

Then real content, in draft. Templates first, content second — you want to see one block render correctly
before authoring fifty.

For anything beyond a few blocks, load `practices-ops-mcp-batching` first. The wrappers differ by target and
mixing them writes garbage without erroring: content blocks take `custom:`, ItemSet items take a flat `data:`.

Multilingual content: one locale per call, always. For the interface strings in your templates, the dictionary
must be **extracted before it can be translated** — see `local-dev.md`.

## Before you call it done

Run `quilium push`. Beyond uploading, its report is a free structural audit: dangling template references,
missing error pages, types pointing at files that don't exist. Aim for zero warnings.

Then check the things that fail quietly:

- `href=""` and `src=""` anywhere in the rendered HTML — the generic signature of an empty variable, an
  inoperative filter, or an image that was never uploaded.
- Empty wrappers — `<h2></h2>`, `<p></p>`, `<iframe src="">`. In Liquid `""` is truthy, so an unfilled text
  field takes the "filled" branch.
- `[object Object]` or `%5Bobject` anywhere — a mis-called `url` filter, or a route parameter with no ItemSet
  behind it, which also breaks the language switcher while every page still returns 200.
- Every internal link points somewhere *other* than the page it's on (an unknown route name resolves to the
  current page without erroring).
- A detail URL with an invented slug does not render the listing at 200.
- The 404 renders — on an unknown URL, in every locale, not by opening the template.
- Every locale, not just the default one — including the home link in secondary locales.
- Forms: submit an invalid POST and see the errors, submit a valid one and confirm the record, then delete the
  test records.
