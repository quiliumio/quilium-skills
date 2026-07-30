# Quilium skills

Two agent skills that let a coding agent build and run a [Quilium](https://quilium.app) site properly.

They are plain [SKILL.md](https://skills.sh/) packages — nothing Claude-specific in them. Claude Code and
Codex are the two we use daily; Cursor, Copilot, Gemini CLI, opencode, Amp, Zed, Windsurf and Cline read the
same format and are installable in one command (see [Install](#install)).

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

```bash
npx skills add quiliumio/quilium-skills
```

Installs into the current project, and asks which agents to wire up — each one gets the skills in the
directory it actually reads (`.claude/skills/`, `.codex/`, and so on). To skip the prompts and target one
agent explicitly:

```bash
npx skills add quiliumio/quilium-skills --agent codex --skill '*' -y
```

`--agent` also takes several agents at once (`--agent claude-code codex`) or `'*'` for all of them; `--all`
is the shorthand for every skill on every agent. `-g` installs at user level instead of per project, and
`-l` just lists what's in the repo.

Otherwise: clone anywhere and copy the two directories under `skills/` into whatever directory your agent
reads. Claude Code can also load the repo directly as a plugin.

The skills assume the Quilium MCP server is connected to the agent. Without it they can still describe the
process, but they cannot act.

## Conventions this repo assumes

- **Templates are `.liquid`**, full path always (`elements/hero.liquid`).
- **Nothing publishes itself.** Pages and content stay in draft until a human asks, page by page.
- **No hardcoded URL path or page UUID in a template, ever.** Named routes exist for that.

## Contributing

Found a behaviour that contradicts what's written here? Two things are worth doing: open an issue, and call
the MCP's `submit-feedback` tool. The second one fixes it for everybody at the source.
