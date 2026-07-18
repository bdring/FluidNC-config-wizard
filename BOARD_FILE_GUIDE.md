# Writing a board.yaml

This guide explains `boards/<id>.board.yaml` -- the file that teaches the
config wizard (and, before that, a human or AI assistant transcribing board
documentation) what physical resources a given FluidNC-compatible controller
board exposes: which MCU pins are brought out to which connectors, what each
connector is for, and any fixed/onboard peripherals.

It is written for two audiences at once: a person doing this by hand, and an
AI assistant doing it on a person's behalf from source documentation (the
FluidNC wiki, a vendor's product page, a schematic). That second case is the
expected common path -- see "Using an AI assistant" below.

A board.yaml is **not** a `config.yaml`. It never picks a FluidNC role for a
pin (which axis, which spindle type, etc.) -- it only records physical facts:
what's there, and how it's labeled. The wizard reads a board.yaml and lets a
user (or another assistant session) make those role decisions interactively,
producing an actual `config.yaml` as output.

## Files involved

- `schema/board.schema.json` -- the formal JSON Schema. Validate against
  this; it is the ground truth for field names, types, and required-ness.
- `boards/<id>.board.yaml` -- one file per board. Filename must match
  `board.id` (snake_case), e.g. `corgi_cnc_controller.board.yaml`.
- `chips/<chip_id>.chip.yaml` -- lower-level MCU/chip facts (referenced by
  `board.mcu_chip`, e.g. `esp32`), not covered by this guide.
- `boards/corgi_cnc_controller.board.yaml` -- the most complete real example
  in this repo; worth reading alongside this guide.

## Using an AI assistant

If you're asking an AI assistant to produce a board.yaml from a board's wiki
page and/or vendor documentation, give it:

1. The board's FluidNC wiki URL (`wiki.fluidnc.com/en/hardware/...`) if one
   exists -- this is normally the single best source, since it's usually
   written specifically for FluidNC config authors and often includes a
   worked example config.yaml, which is extremely useful for cross-checking
   pin assignments.
2. Any vendor product page, schematic, or existing example config.yaml you
   have.
3. This guide and `schema/board.schema.json`.
4. A pointer to an existing board.yaml in this repo (e.g. Corgi's) as a
   style/completeness reference.

Ask the assistant to validate its output against `board.schema.json` before
calling it done, and to flag (in each field's `notes:`) anything it inferred
rather than found explicitly stated -- e.g. "gpio.15 is presumed reusable as
input" vs. "wiki confirms gpio.15 is reusable, verbatim: '...'". This project's
existing files lean heavily on that discipline: prefer a `notes:` field that
quotes or cites the source over a bare unexplained value, since the next
person to touch the file (human or AI) needs to know how confident to be in
each fact, and whether it's worth re-checking against a schematic.

Set `board.status` honestly: `active`/`discontinued` for boards with
solid vendor/official documentation, `community_documented` for boards
whose facts come mainly from user reports rather than official sources, and
`unverified` for anything not yet cross-checked against a schematic or real
hardware. Err toward a lower-confidence status rather than a higher one --
it costs nothing and tells the next reader (human or AI) how much scrutiny
to apply.

## Field-by-field

All top-level sections are documented in detail (with full field lists,
types, and design-rationale notes) in `schema/board.schema.json` -- read that
schema's `description` fields directly; they're written to be as complete a
reference as this guide. What follows is a shorter orientation.

### `board` (required)

Identity: `id` (snake_case, matches filename), `name`, `manufacturer`,
`mcu_chip` (references a `chips/<id>.chip.yaml`), `family` (informational
grouping only -- no inheritance between "family" members), `revision`,
`status`.

### `meta`

Reference links: `wiki_url`, `vendor_docs_url`, `purchase_url`,
`schematic_url`, plus a free-form `notes` field. Fill in whatever you have;
`wiki_url` in particular is worth always including when it exists, since it's
the fastest way for a future reader to verify or extend the file.

### `stepping`

`engines`: an ordered list of FluidNC stepping engines this board's motor
wiring can physically use (`rmt`, `i2s`, `timed`, `pio`). A board whose motor
pins are wired through I2SO shift registers (see `fixed_peripherals.i2so`
below) can only use `i2s` -- check whether the board's motor sockets
reference `i2so.N` pins or plain `gpio.N` pins to tell which applies.

### `fixed_peripherals`

Onboard peripherals with pins that are hardwired and never reassignable:
`spi`, `sdcard`, `i2so` (with `pin_count`, i.e. how many virtual `i2so.N`
pins the board's shift-register chain provides -- state this explicitly per
board rather than assuming a number, since wiki documentation has been wrong
about this before), and `dedicated_uarts` (a UART bus soldered straight to
onboard driver chips, entirely internal, never written into a config.yaml).

### `motor_sockets`

An array of `sockets`, each with a `driver_class`:

- `onboard_fixed` -- a specific driver chip is soldered on
  (`fixed_motor_type` required, e.g. `tmc_2209`).
- `stepstick_socket` -- accepts an interchangeable driver module
  (A4988/DRV8825/TMC2209-standalone/etc.).
- `external_connector` -- step/dir/enable go to a connector for an
  off-board driver; the FluidNC motor type is always `standard_stepper`,
  chosen by the config author later, not knowable from the board alone.

Each socket needs `pins` (`step_pin`/`direction_pin`, or `output_pin` for a
non-stepper fixed actuator like `rc_servo`) and should get a `generic_name`
(see below).

### `generic_pins`, `expansion_sockets`, `rs485_channels`, `expansion_uarts`, `i2c_connectors`

Everything else the board exposes: plain digital I/O (`generic_pins`, each
needing `direction` plus an `input_profile` and/or `output_profile`
describing the real electrical characteristics -- opto-isolated? needs an
external pull-up? MOSFET low-side vs. totem-pole? -- these matter for
correct wiring, don't skip them), general expansion headers
(`expansion_sockets`), RS485/Modbus buses (`rs485_channels`), dedicated
UART expansion connectors like an RJ12 pendant jack (`expansion_uarts`), and
I2C-capable connectors (`i2c_connectors`).

`direction` has two multi-purpose values that are easy to conflate:
`bidirectional` means the pin is driven and read as one shared signal, both
directions active at once (a true I2C SDA line, or any other genuinely
open-drain shared-wire signal). `either` means a bare, uncommitted GPIO that
a config author can choose to wire as an input *or* an output, but never
both at once -- e.g. a repurposable pin on a header with no fixed onboard
role (MKS DLC32's LCD-header pins, for instance). Both require
`input_profile` *and* `output_profile`, but for different reasons: on
`bidirectional` they describe one shared electrical path; on `either` they
describe two independent, mutually-exclusive uses.

A board may legitimately expose the **same physical MCU pin** for two
different purposes -- e.g. an RS485 channel's `tx_pin` that's also
documented as reusable as a plain 5V output. This schema does not enforce
pin uniqueness at the board level (see the schema's top-level description);
record each purpose as its own separate entry, and cross-reference the
sharing in each entry's `notes:` field. This is exactly the shape Corgi's
`gpio.15` takes in the shipped example -- it's simultaneously the RS485
channel's `tx_pin` and a `generic_pins` entry (`Spindle Out15`), each with
its own `notes:` explaining the overlap.

### `generic_name`

Nearly every entry above accepts an optional `generic_name` -- a normalized,
vendor-independent name (e.g. `Driver1`, `In2`, `5V Out4`, `FET4`,
`Spindle Out15`, `Spindle 10V`, `RS485`, `Expander`) used by the wizard's UI
and by hotspots.yaml files (see HOTSPOTS_FILE_GUIDE.md) instead of each
board's own inconsistent silkscreen/vendor label. Apply this fixed scheme
when it's unambiguous:

| Kind of connector/pin                              | generic_name pattern |
|------------------------------------------------------|----------------------|
| Plain digital input                                   | `In<pin>`           |
| 5V push-pull digital output                           | `5V Out<pin>`       |
| MOSFET low/high-side output                           | `FET<pin>`          |
| Open-collector output                                 | `OC Out<pin>`       |
| Onboard-driven motor socket (`onboard_fixed`)         | `Motor<N>`           |
| External-driver-header motor socket (`external_connector`) | `Driver<N>`     |
| Digital spindle-control output (e.g. VFD run/direction)| `Spindle Out<pin>`  |
| Board's 10V analog spindle output                     | `Spindle 10V` (fixed, no suffix) |

For a plain `gpio.N` pin, `<pin>` is just `N` with no separator (e.g.
`FET4` for `gpio.4`). For an `i2so.N` pin (the Nth, 0-indexed, pin of the
board's I2S shift-register chain -- see `fixed_peripherals.i2so`), `<pin>`
is `I2SN`, WITH a preceding space, unlike the bare-number gpio case (e.g.
`OC Out I2S7`, not `OC OutI2S7`, for `i2so.7` -- the space keeps the
rendered label legible) -- needed for any board whose auxiliary outputs are
wired through the I2S chain rather than a native GPIO, which includes some
6-pack-family boards' extra I2SO-driven outputs, not just MKS DLC32.

`N` in `Motor<N>`/`Driver<N>` numbers sockets in silkscreen/physical order,
starting at 1. If a connector doesn't fit any rule above, either omit
`generic_name` entirely or set it equal to `label` -- the wizard falls back
to `label` when `generic_name` is absent, so this is always safe. Don't
invent a new pattern casually; if you think one is needed, flag it rather
than guessing, since consistency across board files is the entire point of
this field.

### Labels vs. generic names -- keep the distinction straight

`label` is always the board's **own** documentation term (silkscreen text,
vendor wiki wording) -- copy it verbatim, including any odd phrasing (e.g.
`"Motor Driver Signals - i2so.1,2,0"`). Never normalize or clean it up.
`generic_name` is the separate, normalized field described above. The wizard
uses `label` only when writing `# <label>` comments into a generated
config.yaml (and when re-reading a config.yaml back in, to disambiguate a
pin shared between two resources) -- everywhere a human looks at the wizard's
UI, or at a hotspots.yaml file, it's `generic_name` that's shown. Getting
these swapped is the most common mistake when hand-authoring or
AI-generating a new board file.

## Validating

There's no bundled validator script in this repo yet; any standard
JSON-Schema-draft-2020-12-compatible tool works, e.g.:

```
python3 -c "
import json, yaml, jsonschema
schema = json.load(open('schema/board.schema.json'))
doc = yaml.safe_load(open('boards/your_board.board.yaml'))
jsonschema.validate(doc, schema)
print('ok')
"
```

(`pip install pyyaml jsonschema` if needed.)

After validating structurally, load the file in the wizard itself
(`index.html`, served over HTTP -- see the main README) and confirm every
resource you expect shows up with a sensible name.
