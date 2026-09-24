# The local loop — CLI, rendering, debugging

Everything on this page happens on your machine. None of it goes through the MCP server, and no MCP skill
covers it.

> **If a loaded reference tells you the CLI is out of scope and must never be invoked, it is describing a
> different setting** — an assistant embedded in the CMS, with no shell and no local checkout. You are working
> in a local project: the CLI *is* the loop, and running it is the point. The instruction applies to the other
> context, not to yours.

## `quilium init`

Scaffolds a project. **It refuses a non-empty directory**, so run it first, in an empty folder. If you've
already put a brief, a mockup or a git repo there, you can't init at the root any more and you end up with a
bastard `quilium/` subfolder — awkward for the rest of the project's life.

The folder it creates is the project root. `run` and `push` are both invoked from there.

### The starter ships demo content

A fresh project contains boilerplate that will otherwise follow you to production: a dummy content-type, a
placeholder content block, a generic favicon, and assorted files under `public/`. Clear it out early — a
dedicated pass right after init is the cheapest moment.

The placeholder block has a trap: once you redefine the page-type that held it, the block becomes orphaned,
and `get-page-with-content` **won't return it** because it filters on the page-type's current sections. Delete
it from the back-office's lost-content panel, or temporarily re-add the old section, fetch, delete, remove the
section again.

### `.quiliumignore`

Excludes files from the **upload** without removing them from git. Anything that isn't part of the site —
briefs, mockups, notes, design sources — belongs there. Templates and `public/` are what should travel.

## `quilium run`

Serves the site locally.

- **Draft pages render.** You do not need to publish, or even push, to see your work.
- **It caches templates *and* schema.** After changing either, stop and restart the server. A surprising
  amount of "my change did nothing" is this.
- **Ghost processes hold the port.** Killing by name isn't always enough, and two servers on one port serve
  inconsistent renders — you end up debugging output from the old one. Check what holds the port, kill it by
  PID, confirm it's free, then start exactly one.

With i18n on, bare paths 404: only locale-prefixed URLs resolve (`/fr/about`, not `/about`). `/` redirects to
the default locale.

### Debugging a render

The DOM is the evidence, not a screenshot. A screenshot taken mid-repaint shows layouts that aren't real, and
an injected panel measured during its transition returns an intermediate value.

To read something the page doesn't display, print it and grep for it:

```liquid
<!--DBG value=[{{ suspect }}] size=[{{ suspect.size }}]-->
```

```
curl -s http://localhost:9000/fr/page | grep DBG
```

Remove the marker once you know. This is the fastest way to settle "is the field empty or is the filter
broken?", and those two look identical in the output.

To measure layout without a browser extension, render with a headless browser and have a temporary script
write `getBoundingClientRect()` values somewhere greppable. `document.documentElement.scrollWidth >
window.innerWidth` is the generic overflow detector.

Test forms with `curl`: an invalid POST should return the field errors, a valid one a **302**. Don't use
`curl -L` — it re-POSTs empty on the redirect and reports errors that aren't there. Delete the records your
tests created.

## `quilium push`

Uploads templates and `public/`. Its report is also **a free structural audit**, and worth running before any
delivery even if you're not shipping:

- content-types or page-types referencing a template file that doesn't exist,
- missing error pages,
- a missing `public/apple-touch-icon.png` — flagged whether or not your `<head>` links one. Supply the file.

Aim for zero warnings. Every one of them is a real dangling reference.

Note that `push` uploads *files*. Content, schemas and settings live in the CMS — pushing does not carry
them. `quilium.json` **is** part of the archive: the platform reads `private` from it, and a theme without it
serves its draft as private. Local endpoint overrides in it travel too.

## `quilium dictionary parse`

Extracts the translatable interface strings from your templates into the CMS dictionary.

**This must run before any interface string can be translated.** The MCP does not create dictionary entries —
`update-dictionary-translations` requires an existing entry id per string. Until extraction has run,
`get-dictionary` returns nothing and there is simply nothing to translate.

The working order is:

1. Write templates using the translation filter.
2. Run `quilium dictionary parse` — it uploads the message ids. `quilium push` does it too, after the
   archive.
3. `get-dictionary missing_locale: <slug>` — returns ids and source strings.
4. `update-dictionary-translations` — batched, one locale per call.

⚠️ **Verify by reading back.** On a site with two or more secondary locales, writing one locale has been
observed to drop the others on the entries it touches, while reporting success. If you hit this, fill the
remaining locales from the back-office and report it with `submit-feedback`.

## Authentication

`quilium login` (`-p staging` for staging) keeps the session under `~/.quilium/`; `quilium whoami` tells you
which account it is and whether it reaches this site. With it present, `quilium run` renders the live site's data
against your local templates — which is what makes the loop fast, and also means **you are looking at real
content**. Treat destructive operations accordingly.
