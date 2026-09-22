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
2. Load the **smallest set** that covers the task — usually one skill, two when it really spans two areas.
   Never follow `related:` by reflex: load another skill only when a question comes up that the loaded ones
   don't answer. A skill already loaded in this conversation is still in it — don't reload it.
3. **`get-skill <slug>`** for each, in parallel when independent.
4. Read what *this* site has — only what the task needs, with the narrowest call: `get-site-settings` with
   the one `type` you need (not `all`), `get-page-with-content` when you know the page. Never infer
   structure from another project.
5. Act.

Everything you read stays in the conversation until it ends, and a conversation has a size limit: a skill
or a settings dump loaded "just in case" costs the user turns.

If a skill is refused for your profile, say so and continue with what you can load rather than guessing at
its contents.

### The site's own procedures come first

When a site is in scope, the manifest opens with a group **`site` — « Procédures de ce site »**: procedures
the site's own team wrote in the CMS (Settings → AI), with slugs prefixed `site:`. They describe how *this*
site does a recurring task — importing the weekly menu, publishing a job offer, the house style of a news
item. When the task matches one, **load it first** (`get-skill site:<slug>`) and follow it over the generic
skills wherever the two disagree, except on safety and on the hazards below. They are not filtered by
profile; a site without procedures simply has no `site` group.

You can write them too. When the user asks you to remember how *they* do something — or when a task you
just completed is clearly going to come back — offer to turn it into a procedure with `create-ai-skill`
(`name`, `slug`, `description`, `instructions`). It is how the site's team makes the next agent, or the next
session, do the task their way. `get-ai-skills` lists what exists (drafts included), `get-ai-skill` reads
one in full, `update-ai-skill` patches it (only the fields you send change), `delete-ai-skill` removes it.
Two things to get right:

- **`site:` is not part of the slug.** It is a prefix `get-skills` adds to tell a site procedure from a
  library skill. Store `menu-semaine`, load `site:menu-semaine`. The tools strip the prefix if you send it,
  but a slug you invent must be plain: lowercase, digits, single hyphens.
- **The description is what makes it findable.** Say *when* to use the procedure, not what it is —
  « Importer le menu de la semaine depuis le PDF du chef » beats « Procédure menu ».
- **The instructions are as long as the rule, no longer.** They are loaded into every conversation that
  uses them. Write only what is specific to this site — the order of the steps, the field keys, the
  conventions, what the site never wants — as terse Markdown bullets. Never re-explain how the tools work
  or restate a library skill. A one-sentence rule stays one sentence: « Demande dans l'ordre l'image, le
  nom, puis un tag ; refuse une image floue » is a complete procedure. No intro, no examples, no checklist
  unless the user asked for a long process.

`enabled: false` keeps a draft out of `get-skills` while the user reviews it — the same list you'd get
from `get-site-context` under `ai_procedures`. A written procedure is not fixed: when one leads you astray,
say so and offer the fix with `update-ai-skill` rather than silently working around it.

`get-navigations-with-pages` returns the whole site tree — on a large site it is the heaviest read you can
make. Call it to locate a page or to work on the menus, not by reflex; when the screen context or the user
already gives you the page, start from that.

## Hazard index — what corrupts data quietly

Six places where a wrong call **succeeds** and writes bad data. This list exists so you recognise the risk
*before* you've loaded the skill that covers it. It deliberately names hazards without restating their rules:
a compressed rule is how you get a confident, wrong answer. Load the skill.

| Hazard | Covered by |
|---|---|
| The write wrapper differs between content blocks and collection items | `practices-ops-mcp-batching` §4 |
| Media, page and relation references have one required shape | `practices-ops-mcp-batching` §5 |
| **A locale's translated `custom` / `metas` is stored whole** — sending only the changed keys erases that locale's other translations | `practices-ops-mcp-batching` §6 |
| A settings write replaces its whole target, not just the keys you sent | read → merge → write complete |
| Page `state` is routability, not publication — and publishing doesn't change it | `tasks-pages-crud` |
| A list field you send (relations, pages, tags, media, repeat) replaces the whole list — omitted fields are kept, but a partial list loses the rest | send it complete, or use `array_edits` (`update-content` / `update-page` / `update-itemset-item`) |

The translations one deserves the emphasis: pages and blocks take several locales per call, collection
items one per call — and in every case the call succeeds while silently dropping the translated keys you
didn't resend. Read §6 before any translation write.

A seventh, specific to `customform` blocks: **a
validation rule whose key doesn't match an input's `name=`** fails on every submission
while highlighting nothing. The visitor reads "correct the highlighted fields" and sees
none — a dead end, with no signal on the authoring side either. After touching either the
markup or the rules, re-read the other and compare the two lists.

One rule that isn't a hazard, just a habit: **upload originals**. Never pre-resize or crop — Quilium derives
versions from the type's own configuration.

## What you don't do

- **Publish.** `publish-page` runs only when the user asks, page by page. Bulk publication is their decision,
  never a side effect of your work. Collection items have no system publication — a type may declare its
  own field for it (`state`, `active`, a publish date…). Before saving an item: first read the collection's
  schema, then the site's procedures and instructions, then save.
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
| Add a form | by default a form with a hard-coded template + `validate` — `reference-forms`; a **form builder** (customform) only when the user asks for one — then ask whether it renders with the site's form design system (custom HTML) or with a template of fields: `reference-forms-customform-builder` first, with its pre-flight (reuse the field keys of the collection the form saves into, ask redirect-or-message, write the form's `tag:`) |
| Change a customform's rules, markup or emails by hand | `reference-forms-customform-validation`, `-html`, `-emails` |
| A task this site has written a procedure for | the `site:` slugs of the `site` group — before anything else |
| « Remember how we do this » / write it down for next time | `create-ai-skill` — then `get-ai-skill` to check it reads well |
| A large authoring session | `processes-content-ops-bulk-authoring` |
| Anything over a handful of calls | `practices-ops-mcp-batching` |

Load fresh — slugs evolve, and the descriptions are the source of truth.

## `customform` — recognise it, then load the library

`customform` blocks are the ones whose fields, validation rules and email bodies live in
the **content**, so the client changes them without a developer. Four skills cover them,
and they are the only source — they are versioned server-side and this file deliberately
carries no copy of them:

| Skill | Covers |
|---|---|
| `reference-forms-customform-builder` | **start here** — the `builder` model, the derivation recipe, two complete worked examples. Writing the model rather than raw HTML is what keeps the form editable by the client |
| `reference-forms-customform-validation` | the validation YAML — validators, conditional rules, messages |
| `reference-forms-customform-html` | the `{{_field.…}}` / `{{_form.…}}` vocabulary — the form's markup on one route, the email bodies on both |
| `reference-forms-customform-emails` | email bodies, per-instance settings, the two guards that silently block a send |

What you need before loading them: **recognise the block**. A customform is a content-type
exposing a `validate` field with `config.builder: true` (a legacy `type: formvalidate` is read
the same way but refused at the next settings save). Its HTML lives in the sibling field
`formHtml` and its email bodies in `mailAdminBody` / `mailUserBody` — by naming convention,
nothing declares them (`config.htmlField` / `config.mails` are refused). **Read the
content-type** to confirm they exist, never guess from another site.

Read it for one more thing: a `type: save` field. Its `config.key` names the collection every
submission is written to, and **that collection's declared fields are the keys your form must
reuse** — `lastname` + `firstname` when it declares them, never a single `name`. A key the collection
does not declare is saved in its catch-all field, or nowhere, and the declared column stays empty in
every view of the back-office, with no error. The builder skill's pre-flight walks through it.

If the site has none, this is not the block you're looking for — and creating one is
structural work, so it belongs to `quilium-builder`.

## What this skill is not

Not a procedure manual — procedures live in the MCP library.
Not the structural layer — that's `quilium-builder`.
Not site-specific — what a site does its own way lives in its procedures (the `site` group of `get-skills`,
edited in the CMS) and, for a coding project, in its own `CLAUDE.md`.
