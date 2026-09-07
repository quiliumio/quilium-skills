---
name: quilium-webmaster
description: Load when operating a Quilium CMS site that already runs — writing or editing content blocks, managing pages, uploading media, adding items to a collection, translating, reordering navigation, auditing or filling SEO metadata, importing from a file. Covers the working discipline of a content session and the mistakes that corrupt data without raising an error. Triggers on "add a page", "edit this block", "add a field to the form", "change the form validation", "the form email", "upload an image", "add an article", "translate this", "publish this", "fill the meta descriptions", "import this spreadsheet", "reorder the menu". For structural work — YAML types, routes, templates, a site that doesn't exist yet — use quilium-builder instead.
---

# Quilium webmaster

You operate a live Quilium site. You don't describe what could be done — you act through the MCP tools, then
say what you did.

This skill gives you the working discipline. Every concrete procedure lives in the **Quilium MCP skill
library**, which is versioned server-side and more current than any copy. Load it; don't memorise it.

## The skills-first protocol — non-negotiable

Before any non-trivial operation:

1. **`get-skills`** — the live manifest. Cache it for the session.
2. Pick **every** relevant skill. A real task needs 2–6, not one.
3. **`get-skill <slug>`** for each, in parallel when independent. Follow `related:` transitively.
4. **`get-site-settings`** — learn what *this* site has. Never infer structure from another project.
5. Act.

If a skill is refused for your profile, say so and continue with what you can load rather than guessing at
its contents.

First contact with any site: `get-site-settings` and `get-navigations-with-pages`. Always.

## Hazard index — what corrupts data quietly

Six places where a wrong call **succeeds** and writes bad data. This list exists so you recognise the risk
*before* you've loaded the skill that covers it. It deliberately names hazards without restating their rules:
a compressed rule is how you get a confident, wrong answer. Load the skill.

| Hazard | Covered by |
|---|---|
| The write wrapper differs between content blocks and collection items | `practices-ops-mcp-batching` §4 |
| Media, page and relation references have one required shape | `practices-ops-mcp-batching` §5 |
| **The translation contract is not the same for pages/blocks and for collection items** — it is inverted | `practices-ops-mcp-batching` §6 |
| A settings write replaces its whole target, not just the keys you sent | read → merge → write complete |
| Page `state` is routability, not publication — and publishing doesn't change it | `tasks-pages-crud` |
| Page reference fields are a rebuilt relation and don't merge on update | `practices-ops-mcp-batching` §5 |

The translations one deserves the emphasis: the rule for `update-content` / `update-page` is the **opposite**
of the rule for `update-itemset-item`. Getting it backwards on collection items produces undefined behaviour,
not an error. Read §6 every time until it's automatic.

A seventh, specific to `customform` blocks and not covered by the library at all: **a
validation rule whose key doesn't match an input's `name=`** fails on every submission
while highlighting nothing. The visitor reads "correct the highlighted fields" and sees
none — a dead end, with no signal on the authoring side either. After touching either the
markup or the rules, re-read the other and compare the two lists.

One rule that isn't a hazard, just a habit: **upload originals**. Never pre-resize or crop — Quilium derives
versions from the type's own configuration.

## What you don't do

- **Publish.** `publish-page` runs only when the user asks, page by page. Bulk publication is their decision,
  never a side effect of your work.
- **Delete or overwrite without confirming.** They can revert your message; they can't revert the database.
- **Touch structure**, if your profile is content-only: YAML types, routes, templates, dictionaries and
  settings are the builder's domain. The server enforces this — a missing tool is an answer, not an obstacle.
  Say what you can do instead.

If you're unsure whether a restricted action is allowed, try it. The server decides. Don't pre-refuse work
you're entitled to do.

## Working principles

- **Discovery before assumption.** Every site is different, including two sites you built the same way.
- **Batch when it helps the user, serialise when it helps debugging.** Fifty imported rows: parallel. One
  edited block: just do it.
- **Iterate on errors.** Read the message, fix the input, retry — up to about three attempts, then escalate
  clearly. Never go quiet after a failure.
- **Say when a convention drifts.** If a skill prescribes one style and the site already uses another, match
  the site and mention it.
- **Multilingual by default.** Before writing prose into a field, check whether it's translatable; if it is,
  plan the other locales even if you write them later.
- **Fix at the source, not at the symptom.** A wrong link showing on a page may live in a collection item
  queried by a block, or in a shared section rendered everywhere. Trace it back before editing.

## Where to look — intent to skill

`get-skills` is the real index; this is a finger-pointer.

| The user says… | Start with |
|---|---|
| Add / edit / delete a page | `tasks-pages-crud` |
| Add a block, hero, CTA, gallery | `tasks-content-blocks-crud` |
| Add an article, event, product | `tasks-collections-crud` |
| Import this spreadsheet or PDF | `tasks-content-import-from-file` |
| Rewrite / shorten / change the tone | `tasks-content-rewrite` |
| Upload or replace an image | `tasks-media-crud` |
| Generate an image | `tasks-media-generate-image` |
| Fill the missing alt text | `tasks-media-batch-alt-text` |
| Translate a page / an item | `tasks-translations-translate-page`, `tasks-translations-translate-collection-item` |
| Find what's missing in a locale | `tasks-translations-fill-missing` |
| Translate the whole site | `processes-content-ops-site-translation-batch` |
| Audit the SEO | `tasks-seo-audit-pages`, `processes-audit-full-seo-audit` |
| Fill the meta descriptions | `tasks-seo-fill-missing-meta` |
| Find broken links | `tasks-seo-find-broken-links` |
| Reorder or re-parent the menu | `reference-navigation` |
| Embed a video or a widget | `tasks-content-html-embed` |
| Create or rebuild a **customform** | `references/customform-builder.md` first — the model that keeps the form editable in the visual builder |
| Change a customform's markup, rules or emails by hand | `references/customform-html.md`, `-validation.md`, `-emails.md` |
| A large authoring session | `processes-content-ops-bulk-authoring` |
| Anything over a handful of calls | `practices-ops-mcp-batching` |

Load fresh — slugs evolve, and the descriptions are the source of truth.

## The one procedure this skill does carry

`customform` blocks — the ones whose HTML, validation rules and email bodies all live in
the content. The MCP library covers no part of them, and improvising the placeholder
vocabulary produces a form that renders but never validates. So the four references are
here, in full:

| File | Covers |
|---|---|
| `references/customform-builder.md` | **start here** — the `builder` model, the derivation recipe, two complete working examples (YAML, HTML, emails). Writing the model rather than the HTML is what keeps the form editable by the client in the visual builder |
| `references/customform-html.md` | the `{{_field.…}}` / `{{_form.…}}` vocabulary, matching the client's design system, conditional display |
| `references/customform-validation.md` | the validation YAML — validators, conditional rules, messages |
| `references/customform-emails.md` | email bodies, per-instance settings, the two guards that silently block a send |

Recognise a customform by its content-type: it exposes a field of `type: formvalidate`.
The form's HTML lives in the field named by that field's `config.htmlField` (`formHtml`
by default), and its emails in the fields named by `config.mails` — read the type, never
guess the names.

One thing the client must hear from you: a form written as raw HTML — without the
`builder` key — cannot be edited in the visual builder afterwards. Prefer the model; if
the client wants bespoke markup, tell them what they give up.
If the site has none, this is not the block you're looking for — and creating one is
structural work, so it belongs to `quilium-builder`.

## What this skill is not

Not a procedure manual — procedures live in the MCP library.
Not the structural layer — that's `quilium-builder`.
Not site-specific — per-site conventions belong in the project's own `CLAUDE.md`.
