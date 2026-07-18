# Writing a hotspots.yaml

This guide explains `hotspots/<id>.hotspots.yaml` -- the sidecar file that
lets the config wizard show a clickable photo of a board, linking each
photographed connector/pin to the matching resource row in the wizard's UI.

Unlike `board.yaml` (see BOARD_FILE_GUIDE.md), a hotspots.yaml is normally
**not** something to ask an AI assistant to author from documentation alone
-- placing an accurate box over a connector in a photo is a visual task best
done by a person looking at the actual image, using the purpose-built
**hotspot editor** (`hotspot-editor.html`) rather than hand-writing YAML.
This guide is mainly about using that tool well; the file format reference
at the end is there for completeness (and for an AI assistant that's helping
debug or tweak an existing file, or reviewing one for correctness).

A hotspots.yaml is entirely optional -- a board.yaml with no matching
hotspots.yaml simply means the wizard shows its normal resource list without
a photo pane. Nothing else about the wizard depends on it.

## Files involved

- `schema/hotspots.schema.json` -- the formal JSON Schema.
- `hotspots/<board_id>.hotspots.yaml` -- one file per board, named to match
  the board's `board.yaml` `id`.
- `images/<board_id>.<ext>` -- the board photo itself, referenced by the
  hotspots file's `image:` field. Keep the photo in this folder.
- `hotspot-editor.html` -- the standalone authoring tool. Open it the same
  way as the wizard (served over HTTP, e.g. `python3 -m http.server 8000`,
  then `http://localhost:8000/hotspot-editor.html`), since it also needs to
  fetch the board's `board.yaml` to know what connector names are available.

## Before you start: prepare your photo

**Scale your board photo to about 1000 pixels wide using an image editor
before loading it into the hotspot editor.** This isn't a hard requirement
-- the editor and wizard both work with any resolution -- but 1000px wide is
the sweet spot this project has converged on: large enough that connectors
are comfortably distinguishable and labels render crisply, small enough to
keep the file size and page load reasonable. A photo that's much smaller
(a few hundred pixels wide) makes precise box placement fiddly and can leave
on-photo label text looking cramped; a photo that's much larger just adds
load time with no real benefit, since the wizard displays it scaled to fit
its pane anyway. Crop out irrelevant surroundings first if useful, and make
sure the board fills as much of the frame as reasonably possible.

## Using the hotspot editor

1. Open `hotspot-editor.html`, pick the board (it reads that board's
   `board.yaml` to know the available `generic_name` values), and load your
   prepared photo.
2. For each connector you want made clickable, draw a box (`rect`) directly
   over it in the photo, then assign it the matching `generic_name` from the
   board's own resource list -- the editor's dropdown is populated straight
   from `board.yaml`, so a resource with no `generic_name` set there won't
   show a friendly name here either (see BOARD_FILE_GUIDE.md if you need to
   add one).
3. If a connector is small or tightly packed and its own name doesn't fit
   legibly inside the box, use the editor's "+ label box" option to draw a
   *separate* label box (`label_rect`) nearby -- above, below, or to the
   side -- rather than shrinking the text past readability. Choose
   horizontal or vertical text flow for that label box, whichever fits the
   available space better. It's completely normal, and expected, for a
   label box to extend past the edge of the photo (e.g. a label above a
   connector that sits right at the photo's top edge) -- the wizard reserves
   room for this automatically when it displays the photo later; you don't
   need to (and shouldn't try to) work around it by padding the image
   yourself.
4. The editor snaps box edges to a 1% grid as you drag, which is normally
   precise enough; nudge with arrow keys for finer placement if needed.
5. Export/download the finished `<board_id>.hotspots.yaml`, and save your
   prepared photo as `images/<board_id>.<ext>` (matching the `image:` field
   the editor writes). Both files go in this repo, alongside the matching
   `board.yaml` (see the repo's own file-location convention: development
   work happens directly in the repo, not staged in Downloads).
6. Reload the wizard (`index.html`) and confirm the photo pane appears with
   your hotspots in the right places and the right names, and that clicking
   a hotspot highlights the matching resource row.

## File format reference

See `schema/hotspots.schema.json` for the authoritative field-by-field
description. Summary:

```yaml
board_id: corgi_cnc_controller       # must match the board.yaml's id
image: images/corgi_cnc_controller.jpg
hotspots:
  - generic_name: "RS485"            # must match a board.yaml generic_name
    rect: {x: 90.00, y: 36.00, w: 7.00, h: 11.00}
    label_rect: {x: 97.00, y: 36.00, w: 10.00, h: 11.00}   # optional
    label_orientation: horizontal    # only meaningful if label_rect present
```

Key points worth calling out explicitly (they're easy to get wrong if
hand-editing rather than using the editor):

- **Coordinates are percentages of the raw, unmodified photo** -- 0,0 is the
  photo's top-left corner, 100,100 its bottom-right corner. They are
  deliberately **unbounded**: negative values, or values past 100, are valid
  and expected wherever a hotspot or its label needs to extend outside the
  photo itself. Don't clamp them, and don't pad/resize the actual image file
  to try to keep everything "inside" -- the image should be saved exactly as
  captured (only ever downscaled if it's unreasonably huge, never padded).
- `generic_name` is the join key back to `board.yaml` -- it must match
  exactly (case included). If a hotspot's name doesn't match any resource in
  the board file, the wizard simply won't be able to link it to a row.
- `label_rect` is optional; omit it for an ordinary connector where the
  name fits fine directly inside `rect`. Only add it when you need the label
  positioned separately from the clickable box.
- Every hotspot should have exactly one `generic_name` ("one hotspot, one
  name"). If a single physical pin genuinely serves two different resources
  (see BOARD_FILE_GUIDE.md's note on shared pins), give each resource its
  own hotspot -- don't merge two names onto one box.

## Validating

```
python3 -c "
import json, yaml, jsonschema
schema = json.load(open('schema/hotspots.schema.json'))
doc = yaml.safe_load(open('hotspots/your_board.hotspots.yaml'))
jsonschema.validate(doc, schema)
print('ok')
"
```

Structural validation alone won't catch a `generic_name` that fails to match
anything in the board file, or a box that's placed slightly off from its
real connector -- always do a final visual check by loading both files in
the wizard.
