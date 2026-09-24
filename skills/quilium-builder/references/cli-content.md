# Content from the shell — the CLI's gateway commands

Besides the local loop (`references/local-dev.md`), the `quilium` CLI reads and writes **content** — blocks,
collection rows, media, images — through the same API as the MCP tools, with the same validation. Nothing on
this page needs a project directory except where it says so.

## MCP or CLI — your call, by environment

- **No shell, or `quilium --version` does not answer:** the MCP is all you have. Nothing here applies.
- **The CLI is installed:** both work; pick per task.
  - **CLI** for bulk loops, imports, and any long field (HTML, markdown, a 50 KB body). The value goes from the
    API to a file and back — **it never passes through your context**, which a MCP round trip of the same
    field always does, twice.
  - **MCP** for reads you need to reason about, single short edits, and everything the CLI does not cover
    (page/nav mutations, content add/delete/reorder, routes, settings, global fields, dictionary).

The schema rules still come from the MCP skill library: the CLI does not tell you what a field is called or
what shape an image takes. Read `get-site-settings(type)` first, the same as for a MCP write.

## Targeting a site

Inside a project, the site comes from `quilium.json`. Anywhere else, name it:

```bash
quilium nav list --website <site-uuid>              # production
quilium nav list --website <site-uuid> -p staging   # staging
```

The pool is always `-p`/`--pool` (`quilium login -p staging` too); omit it for production.
`quilium whoami` answers "which account am I, and can it reach this site?" — run it before blaming a 403.

## Finding ids

Ids are not guessable, and a block lives at `<navigation>/<page>/<block>`. The chain is the discovery path:

```bash
quilium nav list --only navigations.0.key              # → main
quilium page get main/<page-uuid> --only contents      # the blocks, with their types
quilium content get main/<page-uuid>/<block-uuid>      # one block
```

## The edit loop — a long field, never through your context

```bash
REF=main/<page-uuid>/<block-uuid>
quilium content get  $REF --only custom.content -o block.html   # raw bytes, not JSON-quoted
# edit block.html
quilium content diff $REF block.html                            # a diff of that one field
quilium content push $REF block.html                            # --at custom.title for another field
```

Several fields at once, or a short one:

```bash
quilium content update $REF --field custom.title=@title.txt --json-field custom.tags='["a","b"]'
```

**Values come from a file (`@file`) or stdin (`-`), not from the command line** — HTML breaks shell
quoting, and a long argument hits the OS limit. A short literal (`--field state=active`) is fine.

## Collections — where a loop beats the MCP

```bash
quilium item list  articles --search galets --sort date --direction desc   # bare JSON array on stdout
quilium item get   articles <id> --only body -o body.md
quilium item update articles <id> --field body=@body.md
quilium item create articles row.json          # or --field flags, or piped JSON
quilium item delete articles <id> --yes
```

Item fields are written **flat** (`--field title=…`), the way they read back; `data.title` still works.
Translations and schema rules are the MCP library's — `reference-translations` before writing a locale.

**Every write is verified**: the API accepts a field the type does not declare and drops it with a 200, so
the CLI checks that each field it sent came back, and fails (exit 6) if one did not. That catches a misspelt
key in the first row of an import instead of the last. `--no-verify` only for a mass import you have already
proven on a sample.

## Media and images

```bash
quilium media upload photo.jpg --alt "…" [--folder <uuid>]   # prints the media uuid
quilium image generate "<prompt>" --save                     # prints the media uuid (consumes credits)
```

The uuid is what a content or item field references — the image-array shape is in the MCP library
(`reference-fields-catalog`). Upload the original, never a resized copy.

## Reading the output

- **stdout is the payload, stderr is status.** `ID=$(quilium media upload x.jpg)` captures only the uuid.
- `--only <dot.path>` extracts one value; a string comes out raw, so it can be edited and pushed back.
- `-o <file>` writes the payload to a file instead of stdout.
- Exit codes are a contract: `3` not signed in, `4` not found, `5` forbidden, `6` validation (the per-field
  errors are printed), `7` unreachable. Branch on the code, not on the text.
- `-v` traces every request (method and URL; never the token).

## Account level

```bash
quilium site list                     # a table — finding a uuid is the point
quilium site create --name "…"        # waits for the deploy, prints the uuid and the `quilium init` line
quilium site maintenance [on|off]     # "on" takes the live site offline (503) — confirm with the user first
quilium site public [on|off]          # draft viewable without signing in (client review); live is always public
```

`site public` replaces `"private": false` in `quilium.json` (deprecated — never add it to a theme). The key
still makes the draft public while the switch is off: `quilium site public` reports it, and the draft stays
public until the key is removed and the theme pushed.

`quilium --help`, `quilium <command> --help` and https://www.quilium.io/en/docs/cli have the rest.
