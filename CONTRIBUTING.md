# Contributing a template

Anyone can fork this repo and open a pull request — new templates, fixes
to existing ones, or better zone placement/notes. This doc covers the
template format and how to build and check one before you open a PR. For
what happens to your PR after that, see [Review & publishing](README.md#review--publishing)
in the README.

## The three ways to author a template

Pick whichever fits what you're making:

1. **Procedural (Manifold/TypeScript)** — add an entry to the `templates`
   array in [`scripts/gen-templates.mts`](scripts/gen-templates.mts), then
   run:

   ```bash
   npm install
   npm run gen
   ```

   This (re)writes `templates/<id>/{template.json,mesh(es).glb}` for every
   procedural template, including yours. It fails loudly if a solid comes
   out with an error status, or if `build()` doesn't return a solid for
   every part you declared — treat that failure as a hard gate, not a
   warning.

2. **FreeCAD** — model the part in FreeCAD (see `freecad-src/` for the
   existing example), label the target faces `ZONE_<id>` / `ICON_<id>`,
   and export with:

   ```bash
   python3 tools/freecad_export_template.py your-model.FCStd --out templates \
     --author your-github-username --id your-template-id --name "Display Name" --tags tag1,tag2
   ```

   Full label/property conventions (zone modes, depths, multi-part
   `BODY_<id>` objects, colours) are documented in the docstring at the
   top of that script.

3. **By hand** — write `templates/<id>/template.json` and drop in the
   mesh `.glb` file(s) yourself. Match the schema below. This is fine for
   small edits to an existing template (e.g. tweaking a zone or the
   printing notes) without re-exporting the mesh.

Whichever route you use, leave `"published": false` in your
`template.json` while you work. The flag is just your note that it isn't
ready; the site hides such templates by default (an admin can override
that), so a draft is safe to push and iterate on in a PR.

## `template.json` schema

```jsonc
{
  "id": "your-template-id",       // matches the folder name, kebab-case
  "name": "Display Name",
  "units": "mm",
  "tags": ["tag1", "tag2"],
  "author": "your-github-username", // your GitHub username/id — threedeeforge links to it from the model page
  "notes": "A short description, shown in the editor.",
  "printInstructions": "**Printing**\n- Orientation, supports, colours, assembly (Markdown, optional).",
  "published": false,             // omit or `true` once ready to go live
  "verified": false,               // maintainer sets this after a real print
  "colours": ["#2980b9", "#d4a017"], // suggested palette (hex), optional

  // Single-solid templates:
  "mesh": "mesh.glb",

  // OR multi-part templates (parts must not overlap):
  "parts": [
    { "id": "body", "label": "Body", "mesh": "body.glb", "colour": 0 }
  ],

  "zones": [
    {
      "id": "name",
      "label": "Name",
      "kind": "text",              // "richtext" for multi-line wrapping text, "symbol" for an icon, "qr" for a QR code, "image" for a picture
      "part": "body",               // which part this zone sits on (multi-part only)
      "origin": [0, 0, 0],          // zone centre, mm
      "normal": [0, 0, 1],          // out of the solid
      "up": [0, 1, 0],               // text-up direction, in the face plane
      "width": 40, "height": 12,
      "mode": "engrave",            // or "emboss"
      "depth": 0.8,
      "maxLines": 1,                // richtext: most wrapped lines before the text shrinks
      "align": "center",            // richtext only: "left" | "center" | "right", the starting alignment
      "default": "Wolfie",
      "font": "OpenSans",           // optional, defaults to the site's own default
      "colour": 1,                  // optional palette index (qr: the tile behind the code)
      "fg": 2,                      // qr only, optional: palette index of the code and caption (default black)
      "layer": 0.4,                 // qr only, optional: thickness of the code layer, mm (default 0.4)
      "arc": { "radius": 12.9, "sweep": 170, "side": "top" },   // optional, curved text
      "backing": { "offset": 1.2, "height": 1 },                 // optional
      "repeat": [[10, 0, 0]],       // optional extra placements, offsets in-plane
      "wrap": { "radius": 20 }      // optional, bend content around a cylinder
    }
  ]
}
```

`author` must be your own GitHub username/id — the one that opened the
PR. It's how threedeeforge attributes the model to you and, until it's verified,
scopes the unverified preview to your account only (see
[Review & publishing](README.md#review--publishing)).

**Zone kinds.** `text` is one short line (or up to `maxLines` lines, split
only where the user presses Enter), centred, edited inline. `richtext` is for
longer text such as an epitaph or a message: the user edits it in a dialog,
it wraps at `width`, and it is sized as large as it can be while the wrapped
block fits `height` and `maxLines` lines (so more text means smaller text).
`align` sets the starting alignment, which the user can change. The text
(and your `default`) can use a little markdown: `**bold**`, `*italic*`,
`__underline__`, and lists (`- item`, `1. item`, two spaces to nest; wrapped
items hang under their text). There is one font face, so bold and italic are synthesised
(thickened and slanted): use a font with solid strokes, and keep the text large
enough that bold doesn't close up the counters. Give it a
generous `height` and a `maxLines` that matches what you'd want to print; set
`default` to a realistic sample, as stl-check builds it. A richtext zone can't
also have `arc`. `symbol` is an icon from the library.

`qr` and `image` need threedeeforge v1.1.0 or later.

`qr` is a QR code of the user's text (your `default` is the sample, such as a
URL), with an optional caption line under it that the user can type. `width` ×
`height` is the box the whole thing has to fit in: the code, its four-module quiet
zone and the caption, scaled down by the user's size slider. It prints as two
solids that share no volume, so they can be different filaments: a tile `depth`
mm tall in the zone's `colour` (white by default), and the dark modules and the
caption standing on the tile as a layer `layer` mm thick (0.4 by default) in
`fg` (black by default). The site adds white and black to the palette if it
lacks them. A QR zone is always raised, and `mode` makes no difference; to sink
one, model a recess in the mesh and put the zone's `origin` on its floor. A
`depth` of 0 leaves the tile out, so the code and caption print straight onto
whatever the model has at the zone. Size the zone so that a typical code's
blocks come out at 0.8 mm or more (the editor warns under that), since smaller
ones neither print nor scan well; a long default makes a denser code and
smaller blocks, so keep it short.

`image` is a picture the user picks from their own computer. It is reduced to a
small grey raster in their browser and never uploaded, and share links carry only the
conversion settings, so a template can't ship a default picture. Set
`"default": ""`; the zone starts empty. The picture is redrawn in one colour
(`colour`), raised or engraved as `mode` says, to `depth` mm, within `width` ×
`height`. The user chooses how: *solid* traces two-colour art (line drawings,
logos, silhouettes) as it is, while *dots* and *lines* turn a photo into a
halftone, with marks that grow with how dark it is there. They also get
spacing, noise cutoff, invert, background removal, an oval fade and a paint tool
for erasing or keeping parts. Marks are never narrower than 0.5 mm, and a
zone is limited to 9000 dots. As for any raised detail, say in `printInstructions`
whether supports are needed if the zone sits on a vertical face.

```jsonc
{ "id": "link", "label": "QR code", "kind": "qr", "mode": "emboss", "depth": 1,
  "origin": [0, 0, 3], "normal": [0, 0, 1], "up": [0, 1, 0],
  "width": 30, "height": 34, "maxLines": 1, "default": "https://example.com", "colour": 4, "fg": 2 },
{ "id": "pic", "label": "Picture", "kind": "image", "mode": "emboss", "depth": 0.8,
  "origin": [0, 0, 3], "normal": [0, 0, 1], "up": [0, 1, 0],
  "width": 50, "height": 50, "maxLines": 1, "default": "" }
```

`normal` and `up` must be unit vectors and mutually perpendicular; `up`
is what "upright" text on that zone means. Look at a few existing
`templates/*/template.json` files (or the `Zone` type at the top of
`scripts/gen-templates.mts`) for real examples if anything above is
unclear.

## Checking your work before opening a PR

There's no live preview in this repo — the threedeeforge site only sees what a
maintainer explicitly publishes (see README) — so check what you can
locally:

- **Procedural templates**: `npm run gen` succeeding (no thrown error) is
  itself a correctness check — Manifold validates the solid and that
  every declared part builds.
- **Mesh sanity**: open the `.glb` file(s) in any glTF viewer (e.g.
  [gltf-viewer.donmccurdy.com](https://gltf-viewer.donmccurdy.com), or
  Blender, or FreeCAD) and eyeball that the geometry looks right and
  parts don't overlap.
- **Zone placement**: sanity-check that each zone's `origin`/`normal`/`up`
  actually sit on the face you intend, and that `width`/`height` don't
  overrun the part. For a QR zone, check the block size too (see above).
- **`template.json` is valid JSON** and matches the schema above.
- Give a short **`notes`** description of the model, and put the printing
  and assembly advice in **`printInstructions`** (Markdown): orientation,
  whether supports are needed, multi-colour/AMS guidance, glue vs.
  friction-fit, etc. — see existing templates for the level of detail
  expected. The description shows in the editor; the print instructions are
  behind its "Print instructions…" link (under the download buttons) and in
  the README inside the STL zip. If you leave `printInstructions` out, the
  link shows `notes` instead, as it used to.
  Be explicit about supports. Anything *raised* from a vertical (or
  downward-facing) surface, such as embossed text, adornments, beads or
  dentils, has an unsupported underside and needs supports switched on
  (say "everywhere", since "build plate only" will not reach it). Engraving
  into a vertical wall and anything raised from the top face prints without.
  `npm run check-schema` fails an embossed zone on a non-horizontal face
  whose instructions never mention supports.

## Opening the PR

- One template (or one focused fix) per PR.
- Keep `"published": false` unless you're only editing an already-live
  template's metadata (notes, tags) — geometry changes should land as a
  draft first.
