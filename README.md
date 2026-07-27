# Quilium skills for Claude Code

Two skills that let Claude Code build and run a [Quilium](https://quilium.app) site properly.

| Skill | Use it when |
|---|---|
| `quilium-builder` | The site doesn't exist yet, or you're changing its structure — architecture, YAML types, routes, templates. |
| `quilium-webmaster` | The site is running and you're operating it — content, media, collections, translations, SEO. |

## What's here, and what's delegated

The Quilium MCP server ships its own skill library — atomic, versioned guides that update server-side and
arrive fresh in every session. It is the source of truth for everything it covers, so these skills delegate
to it rather than copying it. When a page here says "load `reference-fields-catalog`", it means exactly
that: read the live one.

But **the library is gated by profile, and the set you can load may be a fraction of what exists.** Whole
areas can be absent — templates, routes, YAML schema syntax, forms. A skill that delegates into a gap
produces an agent that improvises the implementation, which is worse than one that never delegated.

So these skills carry the areas most likely to be missing and most expensive to get wrong:

- the **order** in which a whole site gets built, and why the order is load-bearing;
- the **template engine layer** — Liquid globals, filters and the ones that fail silently;
- the **listing → detail chain**, where every mistake still returns 200;
- **images**, end to end: upload, reference, schema versions, render;
- **porting an approved HTML prototype** without quietly redesigning it;
- the **CLI and local loop** (`init`, `run`, `push`, `dictionary`), which no MCP skill covers.

Where the library does cover something, these skills name the hazard and point at it — deliberately without
restating the rule. A named hazard cannot be wrong; a compressed rule can.

## Install

Clone anywhere and point Claude Code at it as a plugin, or copy the two directories under `skills/` into
your project's `.claude/skills/`.

The skills assume the Quilium MCP server is connected. Without it they can still describe the process, but
they cannot act.

## Conventions this repo assumes

- **Templates are `.liquid`**, full path always (`elements/hero.liquid`).
- **Nothing publishes itself.** Pages and content stay in draft until a human asks, page by page.
- **No hardcoded URL path or page UUID in a template, ever.** Named routes exist for that.

## Contributing

Found a behaviour that contradicts what's written here? Two things are worth doing: open an issue, and call
the MCP's `submit-feedback` tool. The second one fixes it for everybody at the source.
