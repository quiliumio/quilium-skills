# Porting a static prototype into Quilium

When you're handed a finished HTML prototype, the job is a **port, not a rewrite**. The prototype is the
visual source of truth: it has already been reviewed and approved by someone, and every liberty you take is a
regression they will find before you do.

The failure mode is not dramatic. The site builds, the text is right, the pages render — and the result is a
plausible reinterpretation of the design rather than the design. It happens because generating markup is
easier than transcribing it, and nothing in the toolchain objects.

## Read the prototype before designing the schema

The blocks that repeat across pages are your content-types. A section that appears once, in one place, is
probably page fields. A grid of records the client will add to is an ItemSet.

Decide the shape from what the prototype actually contains, then confirm it against
`practices-conventions-architecture` — see `process-new-site.md`, step 1. Do not start from the schema you
would have designed if you'd been asked first.

## The shell comes from one page, copied identically

Pick the home page as the single reference for `<head>`, header, navigation and footer. Build the layout from
it once.

Prototype pages usually drift from one another — a stray class, a slightly different footer, a nav that lost
an item. **Ignore those divergences. Do not model them.** One header, one footer, one layout, for the whole
site. Reproducing per-page variation gives you conditional branches nobody asked for and a shell that is
subtly different on every page.

Two things in the shell must change, because they stop being static:

- the **navigation**, which now comes from `sitemap.<nav>.children`,
- **internal links**, which become named routes (`{{ 'routename' | url }}`) instead of hardcoded paths.

Everything else is transcription.

## Inside a template: transcribe, then substitute

Copy the block's markup out of the prototype. Keep the tags, the nesting, the classes, the order. Then replace
only the *values* with fields:

```liquid
<h2 class="section-title">{{ content.custom.title }}</h2>
```

Not: rebuild a heading that looks about right.

Two rules that catch most of the drift:

- **Heading levels are load-bearing.** An `h2` in the prototype stays an `h2`. Demoting it to `h3` because it
  reads like a subheading breaks the document outline and the accessibility tree.
- **Use the exact field names from your schema.** Templates and content are usually written at different
  moments — sometimes by different agents. A template reading `subtitle` where the content wrote `subline`
  renders empty and reports nothing.
- **The prototype's language stops at the labels.** A French mockup naming a block *Nos services* gives a
  content-type keyed `services` with fields `title` and `intro` — English `camelCase`, like every other key on
  the site. Transcription applies to the markup and the copy, not to the identifiers.

### The one licensed deviation: collapsing markup into rich text

Everything above says transcribe. There is exactly one place where you may legitimately *not* reproduce the
prototype's structure as template markup: when a block's HTML is **standard HTML the editor could type** — a
`<ul>` of services, a paragraph followed by a link styled as a button, a numbered list of steps — you may drop
the markup from the template and expose **one `wysiwyg` field** instead of transcribing it into typed fields.

**Ask this per block, while reading the prototype, before writing the schema.** It is the same question step 1
asks; here it has teeth, because the prototype is showing you the exact markup and you can see whether a person
could type it.

This is not a liberty with the design, and that distinction is the whole point:

- **The rendered HTML must still come out with the prototype's own tags and classes.** That is the condition,
  not a nice-to-have — and it is what the style palette is for. A custom style whose output carries the
  prototype's own class means the editor applying it reproduces the approved markup exactly.
- **If the palette cannot emit the prototype's markup, you don't get the deviation.** Transcribe it as typed
  fields instead. A wysiwyg that renders a bare `<a>` where the design has a styled button is drift, and it is
  the drift this whole reference exists to prevent.
- **Decide it before the schema, not after.** The palette is site-wide settings; swapping four typed fields for
  one rich-text field once content is authored means re-authoring every instance.

Load `reference-wysiwyg-config` for the palette, and `practices-conventions-architecture` for the shape
decision. If `reference-wysiwyg-config` isn't in your manifest, read the site's existing `wysiwyg` settings and
mirror that shape — do not invent the syntax, and do not silently fall back to typed fields without saying so:
that choice belongs to whoever approved the design.

## Verify against the render, not the data

Reading back what you wrote through the CMS proves the content is correct. It proves nothing about the page.
A section can be entirely absent, a block can be laid out wrong, a CTA can be missing, and the content check
still passes.

Serve the site locally, `curl` each page, and compare that HTML against the prototype's HTML. Not a
screenshot — a headless screenshot of a page with reveal-on-scroll animations or lazy-loaded images shows
blank sections that are perfectly fine, and you will chase them for an hour.

Compare on three axes:

1. **Completeness.** List every `<section>` (by its heading) and every CTA in the prototype page. For each
   one, find its counterpart in the rendered page. Anything in the prototype without a counterpart is a gap —
   never call a page conforming while a section is missing.
2. **Structure.** Same tags, same key classes, same layout, same order, same image slots. Identical text with
   different structure means the template is not reproducing the design.
3. **Content.** The prototype's actual words, not a paraphrase. Lists at the right length. Images wired to real
   media.

When a block diverges, the fix is one of two kinds, and they cost very differently: either the right component
is there with the wrong styling — a tweak — or the wrong component was used altogether, in which case the
block has to be rebuilt. Naming which one you're looking at saves the next person from tweaking the CSS of
something that was never the right block.

## Before you hand it over

- Every page of the prototype exists, and the navigation lists exactly the prototype's nav items — same
  labels, same depth, nothing added, nothing dropped.
- No `src=""` and no `href=""` in any rendered page (see `media.md`).
- No empty wrappers: `<h2></h2>`, `<p></p>`, `<iframe src="">`.
- Detail pages resolve a real item, and an invented slug does not render the listing at 200 (see
  `detail-pages.md`).
