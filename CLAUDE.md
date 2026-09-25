# Land Rover LR4 (L319) OBDb Signalset

This repo defines the OBD-II/UDS signalset for a 2016 Land Rover LR4 /
Discovery 4 (L319) — 3.0L supercharged V6 (AJ126), ZF 8HP70, full-time 4WD,
electronic air suspension. It follows the [OBDb](https://github.com/OBDb)
v3 signalset format and is consumed by the Sidecar/Pelican OBD-II app.

For signal-level detail (what each DID means, how to read boost, altitude
behavior, off-road usage, module identification) and the ongoing findings
log, read `README.md` — it is the primary reference and this file does not
repeat it.

## Layout

- `signalsets/v3/default.json` — the signalset itself, 91 commands. The
  only file the app actually consumes.
- `tests/test_cases/2016/command_support.yaml` — manifest of which commands
  each ECU supports, used by the test suite.
- `README.md` — signal reference, polling budget, and the findings log.

## Architecture facts

- The vehicle is **11-bit CAN** throughout. Every module answers on a
  `0x7xx` request header with its response at `request + 8` (e.g. `7E0` →
  `7E8`). No 29-bit/extended addressing anywhere on this truck.
- **The `F4xx` convention.** `22 F4<nn>` returns standard OBD-II PID `01<nn>`
  with standard scaling, as a UDS request instead of a mode-01 request. This
  exists because Pelican consumes standard mode-01 PIDs internally and they
  never surface as recordable signals in the app, whereas the same data
  requested via the `F4xx` alias arrives as a custom signal and does show
  up. Most of the signalset uses this trick rather than requesting the
  standard PID directly.
- **Per-ECU DID offsets.** The engine's proprietary DIDs sit `+0x300` from
  their base (e.g. `00F2` → `03F2`). The transmission and rear differential
  sit `+0x1000` (e.g. `0E69` → `1E69`). A DID that comes back "out of range"
  is often just addressed at the wrong offset for that module.
- **`freq` is an interval in seconds, not a rate.** A lower number polls
  harder. Total demand is `sum(1/freq)` across all commands and must stay
  under **4.2 req/s**, which is measured, not assumed. The adapter delivers
  about 13.4 req/s in total, but the app spends roughly 3.3 of that on AT
  commands and 5.9 on its own internal mode-01 polling, which never
  surfaces as a recordable signal. Signalset traffic gets what is left.
  Measured across seven sessions, the UDS service-22 share stayed between
  4.1 and 5.1 req/s no matter how big the signalset was: one session polled
  108 distinct DIDs at 4.19 req/s and another polled 23 at 4.22 req/s.
  Current demand is 4.083. Asking for more than the app can deliver does
  not slow everything down evenly — see trap 5.

## Validation

CI (`.github/workflows/presubmits.yml`) validates `signalsets/v3/` against
the schema in the separate `OBDb/.schemas` repo, which is not checked into
this repo. To validate locally:

```
curl -sfL https://raw.githubusercontent.com/OBDb/.schemas/main/signals.json -o /tmp/signals.schema.json
python3 -c "
import json, jsonschema
schema=json.load(open('/tmp/signals.schema.json'))
inst=json.load(open('signalsets/v3/default.json'))
errs=list(jsonschema.Draft7Validator(schema).iter_errors(inst))
print('errors:', len(errs))
for e in errs[:5]: print(list(e.path), e.message[:200])
"
```

`path` is constrained by the schema to a fixed enum (`Engine`, `Fuel`,
`Drivetrain`, `Suspension`, `Transmission`, `Trips`, `ECU`, `Electrical`,
`Tires`, `Battery`, `Climate`, `Movement`, and a few others) — `Diagnostics`
is not a valid value; use `ECU` for undecoded/diagnostic raw signals.

Print total polling demand:

```
python3 -c "
import json
d=json.load(open('signalsets/v3/default.json'))
print(sum(1/c['freq'] for c in d['commands']))
"
```

Check that the test manifest YAML parses and that no entry silently became
a float or other non-string type (see trap 1 below):

```
python3 -c "
import yaml
d=yaml.safe_load(open('tests/test_cases/2016/command_support.yaml'))
for ecu, entries in d['supported_commands_by_ecu'].items():
    for e in entries:
        if not isinstance(e, str):
            print('NON-STRING ENTRY:', ecu, repr(e))
print('ok')
"
```

The pytest workflow (`.github/workflows/response_tests.yml`) runs inside the
`ghcr.io/obdb/devcontainer` image, and the test sources it runs live inside
that image, not in this checkout. Running `pytest tests/` locally will
report "no tests ran" — that is expected, not a failure. There is no way to
run the actual response tests outside CI.

## Traps

These have all actually bitten:

1. **YAML float trap.** In `command_support.yaml`, a plain-scalar entry made
   up only of digits and dots (e.g. `751.222076`) parses as a YAML float,
   not a string, unless it's quoted. Any entry with no hex letter A–F
   anywhere in it must be double-quoted. After editing this file, verify no
   entry parsed as a non-string (see the validation command above).
2. **Deleting a command can orphan a synthetic.** The top-level `synthetics`
   block in `default.json` references signal ids by name (e.g.
   `LR4_BOOST_RATIO` reads `LR4_MAP` and `LR4_BARO`). Removing the command
   that defines one of those signal ids leaves a dangling reference that is
   still valid JSON but semantically broken — the synthetic will never
   compute. Grep `synthetics` for a signal id before deleting the command
   that defines it.
3. **Entry shape differs per ECU in `command_support.yaml`.** Most ECUs use
   `<hdr>.22<DID>` (e.g. `795.221E8A`). `7E1` in the `supported_commands_by_ecu`
   section uses `<hdr>.<rax>.22<DID>` (e.g. `7E1.7E9.221E69`). There is no
   reason for the difference — `7E1`'s `rax` is `7E9`, exactly `hdr+8`, the
   same as everywhere else. It is just how that block was written. Match
   whatever shape the surrounding block already uses; don't normalize
   across ECUs.
4. **Agent worktrees branch from `main`, not from the current branch.** Work
   that depends on unmerged commits on the current branch must be done in
   the main checkout, or the agent must be told explicitly to check out that
   branch — otherwise it silently operates on stale files and produces
   confident, wrong results.
5. **The app settles on a working set of roughly twenty commands and
   ignores the rest, however big this file is.** Taking the longest session
   of each day since 2026-09-05, coverage has been 12 to 23 distinct DIDs
   every time, including the 137-minute drive on 09-20 that polled 23 — the
   widest of any long session in the period. Coverage spikes right after
   the signalset changes (108 DIDs on the morning of 09-20, 965 during a
   PID detector run) and then settles back within hours.

   The stable core is exactly the metric-carrying commands. Thirteen
   commands appear in every long session since 09-05, and they are
   precisely thirteen of the sixteen that carried a `suggestedMetric` at
   the time. Read with trap 8, the rule is that **the app converges on
   polling what it will store.** Modules with no metric-bearing command on
   them — `7D3`, `792`, `732`, `726`, `761` — drift out of the rotation
   entirely.

   **A command's presence in this file does not mean it is being
   collected.** Check the scan logs before relying on one. But do not read
   a quiet module as a broken or "retired" one: an earlier version of this
   trap claimed the app permanently retires ECUs and told the owner to
   reset the app's vehicle profile. That was wrong. It came from reading
   one day's sessions without the months behind them, and coverage
   recovers on its own every time the file changes.

   A corollary for pruning: a quiet command is not a dead one. Nineteen
   commands were deleted on 2026-09-20 for being constant and restored the
   same day, because most had between 6 and 30 samples and none of those
   samples covered the event the command would report — `1E88`'s six were
   all taken with the differential unlocked. A probe costs 0.0017 req/s at
   `freq` 600. Deleting it costs the answer. Park it, don't cull it.

   The restored probes settled this within hours. `3B4D` — the one deletion
   called conclusive, flat `0x00` across 236 samples — returned `0x04`,
   `0x01` and `0x00` within eight minutes of the off-road features actually
   being used. Six of the eleven `792` `2A3x` DIDs turned out to be
   advancing counters rather than constants.

6. **Standard mode-01 PIDs are already being polled by the app, constantly,
   and a proprietary DID may duplicate one.** Before adding a signal, check
   whether the truck already answers it somewhere cheaper, and check whether
   two DIDs you both poll are the same measurement. Three duplicates have
   been found this way by correlating logged samples: PID 67 sensor 1 equals
   the engine coolant PID to 0.0 C over 135 pairs; PID 70 channel A equals
   manifold pressure to 1.3 kPa over 525 pairs; and the proprietary oil
   temperature DID equals standard PID 5C to 0.7 C over 1,135 pairs once
   warm. Correlate against the scan logs before believing a new DID is new.

7. **Synthetic signals support only `op: ratio`, a plain a/b with no
   constant** — but `fmt.div` on an ordinary signal is a `number`, not an
   integer, so a scaling constant can be folded into a hidden operand and
   the ratio then comes out in real units. `LR4_FUEL_RATE` is built that
   way: `LR4_FUEL_DIVISOR` is commanded lambda premultiplied by
   14.7 x 745 / 3600 x 3.785, so MAF divided by it is US gallons per hour.
   Two operands is also the ceiling: anything needing three (speed, MAF
   and lambda for an exact mpg) has to drop one, which is why the mpg
   synthetics assume lambda 1. Whether a synthetic may read another
   synthetic is untested; `LR4_MPG_LAMBDA` exists to find out.

8. **Only signals carrying a `suggestedMetric` are ever written to the
   app's signal database, but every signal displays live.** Screenshots on
   2026-09-20 confirm non-metric and `hidden: true` signals alike appear
   with current values in the app's section lists — so `hidden` does not
   suppress anything, and bandwidth spent on a metric-less signal still
   buys a readout. What it does not buy is history. Measured over fifteen months of backups: the
   store has held exactly 15 distinct signals, and they are precisely the
   ones with a metric. Everything else is requested, answered, decoded and
   discarded. During a 137-minute drive on 2026-09-20 the 14 signals
   recorded were exactly the metric-carrying signals on commands that were
   being polled; the two metric-carrying signals that were missing
   (`engineSpeed`, `absoluteEngineLoad`) were missing because their
   commands were not polled at all. The other two stores in the backup are
   not alternatives: `records` holds manual service and fuel entries, and
   `tripLogger` holds the phone's own GPS trace.

   The metric enum has 36 values and no slot for manifold pressure, ride
   height, suspension pressure, charge air temperature or anything else
   specific to this truck. So most of what this signalset decodes can be
   watched live but can never be looked at historically except by reading
   the scan logs. Weigh that before spending budget on a signal: a fast
   `freq` on a signal with no metric buys a live readout and nothing else.

9. **Nothing in the signalset controls how many decimal places the app
   shows.** The schema's `fmt` has scaling, range, unit and map fields
   and no precision field, and the synthetics block has even less. The
   app decides: integer-scaled signals print as integers, a Celsius
   signal shown in Fahrenheit picks up one decimal from the conversion,
   and a synthetic ratio prints its full float. There is no `div`, `max`
   or unit trick that changes this; it is a feature request for the app.

## Where the data comes from

On macOS, the Sidecar/Pelican app writes zipped SQLite backups to
`~/Library/Mobile Documents/iCloud~com~featherless~apps~electricsidecar/DataBackups/`,
in four folders:

- `scanSessions` — every raw request/response pair (large, on the order of
  a gigabyte or more). This is the ground truth for whether a command
  answers and what it actually returns.
- `signals` — decoded signal values.
- `tripLogger` — trip-level data.
- `records` — other recorded data.

The backups lag live data by up to a day, so a command that looks dead in
the backups may have started responding again since the last sync.
