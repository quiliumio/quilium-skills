# Images — upload, schema, render

Three separate places, and an image only appears when all three agree. Skipping the upload is the most common
version of "the layout is right but every image is missing".

## Two destinations, not one

- **`public/`** is uploaded by `quilium push`. It is for site chrome: stylesheets, scripts, favicons,
  `robots.txt`. Files here are served at a fixed path and are not editable from the back-office.
- **The media library** holds everything an editor may replace: photos, illustrations, logos used in content.
  These do **not** travel with `push`. They are uploaded through the MCP and referenced by uuid.

An image sitting in your project folder that you never uploaded renders as `src=""`. `push` will not warn you:
as far as it is concerned, nothing is missing.

## 1 · Upload

`upload-media` takes a **public URL** (`file_url`). Over a local (stdio) connection it also accepts
`file_absolute_path`, which is what makes porting a folder of files practical. If you have bytes and no URL,
`request-upload-url` gives you a short-lived ticket: request it, PUT the bytes, then call `upload-media` with
the `public_url` it returned.

Always pass `name` and `alt`, and group with `folder_id` (`create-media-folder` first, `get-media-folders` to
list). Alt text written at upload time is what templates read later — filling it afterwards is a second pass
over every file.

**Keep the file → uuid table.** The upload response is the only place that mapping exists, and every content
write afterwards needs it. Write it to a file as you go; losing it means re-deriving which media is which by
name.

## 2 · Reference

Every image field is written as an **array of objects**, never a bare string:

```json
{ "custom": { "image": [{ "id": "<media uuid>" }] } }
```

A raw uuid string is accepted by the API and renders nothing.

On a multilingual site, do **not** repeat the image inside a `translations` payload. Media, layout and icon
fields belong to the root record; translations carry text.

## 3 · Declare the versions in the schema

An `image` field can declare the derivatives Quilium should generate:

```yaml
image:
  type: image
  versions:
    card: { width: 800, height: 600, label: Card }
    hd:   { width: 1600, height: 1200 }
```

Width-only versions (no `height`) preserve the source aspect ratio — the right choice for a detail-page hero.
A fixed ratio is for grids, where alignment matters more than the crop.

Galleries add `multiple: true` and a `max:`.

## 4 · Render

```liquid
<img src="{{ content.custom.image[0].versions.card.url }}"
     alt="{{ content.custom.image[0].alt }}">
```

- The field is an array. Index it or loop it.
- `versions.<key>` are the keys **you declared**. Ask for a key that isn't there and you get `nil` → `src=""`,
  no error. This is the single most common empty-image cause.
- `{{ image[0].url }}` with no version is the original — a fine last-resort fallback.
- Image URLs are `nil` when absent (not `""`), so `| default:` chains work here:
  `{{ img.versions.hd.url | default: img.versions.card.url | default: img.url }}`.

Alt text comes from the media entry (`.alt`), with `.name` as a fallback. A decorative image — one already
described by adjacent text, or a background behind a scrim — takes `alt=""`, empty and present, never absent.

```liquid
alt="{% if img.alt != blank %}{{ img.alt }}{% else %}{{ content.custom.title }}{% endif %}"
```

## Checks that actually catch this

- `curl` the rendered page and grep for `src=""`. It is the generic signature of a missing upload, a wrong
  version key, or an unreferenced field — and it looks like nothing at all in a browser screenshot of a page
  whose images are lazy-loaded.
- Confirm every uuid you referenced exists in your upload table. A reference to a uuid that was never uploaded
  fails the same silent way.
- Remember Liquid truthiness: `""` is **truthy**. `{% if content.custom.caption %}` enters the filled branch on
  an empty text field and emits empty wrappers. Grep for `<p></p>`, `<h2></h2>` and `<iframe src="">` too.

## Lifecycle — retention and `onDelete`

A collection with `retention` deletes its items automatically (trash, then purge 30 days later).
What happens to the media its items reference is decided **per field**, with `config.onDelete` on
`image`/`file` fields:

```yaml
retention:
  duration: P0D          # delete as soon as the date is passed
  basedOn: displayUntil  # a `date` field of the type — empty date = never expires
tabsets:
  main:
    fields:
      displayUntil:
        type: date
      document:
        type: file
        config:
          onDelete: cascade   # media follows the item to the media trash
      photo:
        type: image
        config:
          onDelete: orphan    # only if nothing else references it
          versions:
            card:
              width: 800
```

`keep` (the default) never touches the media — the right choice for reusable images (logos,
photos). `cascade` is for disposable documents owned by the item. `orphan` checks the media's other
references at purge time. Rules apply at the item's **hard delete** (trash purge), so restoring an
item from the trash restores it with its media intact.
