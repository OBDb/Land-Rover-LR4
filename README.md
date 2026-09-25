# Land Rover LR4 (L319) — Signal Reference

OBDb signalset for the 2016 Land Rover LR4 / Discovery 4 — 3.0L supercharged
V6 (AJ126), ZF 8HP70, full-time 4WD with locking rear differential and
electronic air suspension.

Signals here were confirmed against ~13M logged request/response pairs from
this vehicle (VIN `SALAK2V62GA832217`). Anything still named `*_RAW` responds
on the bus but hasn't been decoded yet.

---

## Modules

Thirteen modules answer on this truck. Six have never been mined — they
reject unknown DIDs with a negative response rather than staying silent,
which is how we know they're there. Requests go to `hdr`, replies come back
from `rax`.

| Request | Response | Module |
|---|---|---|
| `7E0` | `7E8` | Engine (ECM) |
| `7E1` | `7E9` | Transmission (TCM) |
| `716` | `71E` | Unmined |
| `726` | `72E` | Unmined — returns the VIN |
| `732` | `73A` | Gear selector |
| `734` | `73C` | Headlamp control module (HCM) |
| `737` | `73F` | Unmined |
| `760` | `768` | ABS / brake control module (probable) |
| `761` | `769` | Body / instrument |
| `792` | `79A` | Unidentified — answers `2A3x` |
| `795` | `79D` | Rear differential |
| `797` | `79F` | Unmined |
| `7D3` | `7DB` | Air suspension (EAS) |

Two addressing quirks are worth knowing:

**Per-ECU DID offsets.** The engine's proprietary DIDs sit `+0x300` from
their base (`00F2` → `03F2`), while the transmission and differential use
`+0x1000` (`0E69` → `1E69`). A DID that returns "out of range" is often just
sitting at the other offset.

**`22F4xx` aliases.** Any standard OBD PID `01XX` is also readable as
`22F4XX` — `010C` (RPM) is the same data as `22F40C`. This matters because
Pelican handles standard PIDs internally, so a signalset entry for `012F`
never surfaces in the app. The `F4xx` alias arrives as a custom signal and
shows up normally. Most of this file uses that trick.

### Module identification (2026-09-19)

A UDS identification probe (`22F18C` ECU serial, `22F191` ECU part number)
was run against all six previously unmined modules:

| Module | `F18C` serial | `F191` part number |
|---|---|---|
| `716` | `0000000004757126` | rejected (NRC 31) |
| `726` | `5224293808` | rejected (NRC 31) |
| `734` | `0080-508580` | `EX53-14C243-AB` |
| `737` | `Q117eeA3516` | rejected (NRC 31) |
| `760` | `1716263MO0507` | `CH32-14C227-AB` |
| `797` | `0-00002C` | rejected (NRC 31) |

`734`'s part number is a Hella `14C243` headlamp levelling / adaptive front
lighting controller, the same base number used across Jaguar Land Rover
platforms of this era (it also shows up as `8W83-14C243-AA` on the Jaguar
XK) — identified as the Headlamp control module (HCM). `760`'s part number
matches the ABS/brake module in a published Discovery 4 (L319) ECU scan
report; that identification rests on a single external scan report, so
treat it as probable rather than confirmed. The other four modules didn't
resolve a part number, but each returned a distinct serial, so they're
confirmed to be real, separate modules even though what they do is still
unknown.

`22F187` was also tried against all six and came back NRC 31 (request out
of range) everywhere except `726`, whose reply was malformed and returned
VIN bytes instead of a rejection — worth a retry.

---

## Engine

| Signal | Address | Notes |
|---|---|---|
| Engine Speed | `7E0` `22F40C` | |
| Calculated Load | `7E0` `22F404` | Percent of available torque at current RPM |
| Absolute Load | `7E0` `22F443` | Percent of *peak possible* airflow — exceeds 100% under boost |
| Coolant Temp | `7E0` `22F405` | Thermostat opens ~88°C; normal running 90–105°C |
| Manifold Pressure | `7E0` `22F40B` | **Absolute**, not gauge. See boost calculation below |
| Barometric Pressure | `7E0` `22F433` | ~101 kPa at sea level, drops with altitude |
| Altitude | `7E0` `22F433` | Barometric converted to feet by lookup table |
| Intake Air Temp | `7E0` `22F40F` | Before the supercharger |
| Charge Air Temp | `7E0` `220520` | **After** the supercharger and intercooler |
| Timing Advance | `7E0` `22F40E` | Degrees before top dead centre |
| Throttle Position | `7E0` `22F411` | Actual plate position |
| Commanded Throttle | `7E0` `22F44C` | What the ECU asked for |
| Accelerator Pedal D / E | `7E0` `22F449` / `22F44A` | Two sensors in one pedal |
| Mass Air Flow | `7E0` `22F410` | Grams of air per second |
| Equivalence Ratio | `7E0` `22F444` | Commanded lambda |
| O2 Lambda B1S1 | `7E0` `22F434` | Measured lambda |
| O2 Pump Current B1S1 | `7E0` `22F434` | Second half of the same response. Current, not voltage — see "Two decodes that were wrong" |
| Catalyst Temp B1S1 / B2S1 | `7E0` `22F43C` / `22F43D` | One per bank |
| Oil Temp | `7E0` `2203F3` | Land Rover's own sensor. The SAE alias `F45C` was dropped — 1,169 samples against this one's 347,015 |
| Oil Level | `7E0` `2203E6` | Millimetres in the sump |
| Oil Volume | `7E0` `2203F2` | |
| O2 Lambda B2S1 | `7E0` `22F438` | Measured lambda, bank 2 — first bank 2 lambda ever read on this truck |
| O2 Pump Current B2S1 | `7E0` `22F438` | Second half of the same response. Current, not voltage |
| O2 Voltage B1S2 | `7E0` `22F415` | Post-catalyst sensor, bank 1 |
| Relative Throttle Position | `7E0` `22F445` | A third throttle-position reading, alongside Throttle Position and Commanded Throttle |
| Absolute Throttle Position B | `7E0` `22F447` | A fourth |
| O2 Sensors Present | `7E0` `22F413` | Bitmask of which O2 sensor positions are physically fitted |

### Lambda

Lambda expresses air-fuel ratio relative to perfect combustion (14.7:1 for
gasoline).

- **1.00** — stoichiometric, the cruise target
- **below 1.00** — rich, extra fuel
- **above 1.00** — lean, extra air

Under boost the ECU deliberately commands rich (~0.85) because extra fuel
cools the charge and suppresses knock. A value of 2.00 means deceleration
fuel cut — injectors fully off while coasting, sensor reading pure air.

---

## Fuel

| Signal | Address | Notes |
|---|---|---|
| Fuel Level | `7E0` `22F42F` | |
| Fuel Rail Pressure | `7E0` `22033E` | Proprietary; ~60 bar idle, 130+ under load. The SAE alias `F423` was dropped — 158 samples against this one's 15,910 |
| Short Term Fuel Trim B1 | `7E0` `22F406` | Immediate correction, swings constantly |
| Long Term Fuel Trim B1 | `7E0` `22F407` | Learned correction, drifts slowly |
| Short Term Fuel Trim B2 | `7E0` `22F408` | First recorded 2026-09-20 |
| Long Term Fuel Trim B2 | `7E0` `22F409` | First recorded 2026-09-20 |
| Fuel System Status B1 | `7E0` `22F403` | Enum: open loop cold / closed loop / open loop, load or decel fuel cut / open loop, system fault / closed loop with a feedback fault |
| Fuel System Status B2 | `7E0` `22F403` | Second half of the same response; not yet mapped to named states |
| Commanded Evap Purge | `7E0` `22F42E` | Percent duty cycle on the purge valve |
| O2 Trim B1S2 | `7E0` `22F415` | Second half of the same response — post-catalyst trim, bank 1 |
| Long Term Secondary O2 Trim B1 | `7E0` `22F456` | |

---

## Duplicates found and removed

Four signals were checked against another reading of the same physical
quantity, using paired samples across logged history rather than a single
drive. Two turned out to be the same measurement and were dropped; two
turned out to be genuinely separate sensors and were kept.

**PID 67 sensor 1 is the engine coolant PID.** Mean difference 0.0°C over
135 paired samples. Deleted. Sensor 2 of the same PID is kept, renamed to
Charge Cooler Coolant Temp: it ran 11.7°C above ambient and 24°C below
charge air temperature over those same samples, which is where the AJ126's
separate low-temperature intercooler circuit belongs. A genuinely new
signal, not a rename in name only — but it carries no `suggestedMetric`,
so it will never be stored as history by the app; see "Only signals with a
metric are ever recorded" under Polling.

**PID 70 channel A is manifold pressure, at finer resolution — not a
separate boost sensor.** This corrects "F470 was boost pressure all along,"
decoded earlier the same day. PID 70's support byte reads `0x02`; three of
its four defined channels read constant zero across 525 samples and were
dropped. The fourth, channel A, tracks Manifold Pressure to within 1.3 kPa
mean over 525 pairs (r = 0.85) — the same manifold sensor, reported at 1/32
kPa instead of 1 kPa, not a pre-throttle sensor. Kept, renamed to Manifold
Pressure (Fine), slowed to 120s since the extra resolution isn't worth a
faster poll.

**The proprietary oil temperature DID and standard PID 5C agree to 0.7°C**
over 1,135 paired samples once warm, diverging by up to 15°C during
warm-up with PID 5C reading higher. Dropping PID 5C earlier was correct,
and it stays out.

**PID 68 sensor 2 tracks the proprietary charge air temperature DID to
2.7°C mean** — close to redundant, but kept: sensor 1 of the same PID is
the pre-supercharger inlet, running 9.2°C above ambient, and is a real
second reading the truck doesn't report anywhere else. Both kept at 120s
with descriptions.

---

## Drivetrain

| Signal | Address | Notes |
|---|---|---|
| Gear Selector | `732` `22D928` | 0 Park · 1 Reverse · 2 Neutral · 3 Drive · 7 Sport |
| Gearbox Temp | `7E1` `221E69` | ZF 8HP70 sump |
| Locking Diff Oil Temp | `795` `221E8A` | |

---

## Suspension

All four corners report gauge pressure in the air spring.

| Signal | Address |
|---|---|
| Pressure Front Left / Right | `7D3` `223B04` / `223B03` |
| Pressure Rear Left / Right | `7D3` `223B06` / `223B05` |
| Height Offset | `7D3` `222B12` | Signed, millimetres from nominal |
| Compressor Activity | `7D3` `223B07` | ~110 at rest, over 1,300 while the truck raises |
| Ride Height Mode | `7D3` `223B3C` | Enum: 1 Normal, 2 Off-Road, 4 Access, 13 In Transit. Corrected 2026-09-20 — see "Air suspension notes" under Off-road |
| Height Sensor Front / Rear | `7D3` `223B71` / `223B72` | **Inverted** — falls as the truck rises |
| `D11A` Raw | `7D3` `22D11A` | Was called Module Voltage; it is not battery voltage — see "Two decodes that were wrong" |

Normal standing pressures are roughly 40 psi per corner at normal ride
height, rising with load and with raised height modes.

**None of this module's signals carry a `suggestedMetric`, and the app
only ever stores signals that do** — see "Only signals with a metric are
ever recorded" under Polling. Corner pressures, ride height, compressor
activity and module voltage are all live-only: even when `7D3` is being
polled, nothing from this table will ever show up in the app's own
history, only in the raw scan logs. It is also why the app drifts away
from asking `7D3` anything at all between signalset changes.

---

## Reading signals together

Individual numbers are useful. Pairs are where the diagnostics live.

### Boost — Manifold Pressure minus Barometric Pressure

The LR4 reports manifold pressure as *absolute*, so at idle it reads well
below atmospheric and looks wrong. Subtract barometric and you get true
boost:

```
boost = Manifold Pressure − Barometric Pressure
```

Negative is vacuum (closed throttle, coasting). Zero is atmospheric. Positive
is supercharger boost. Because barometric is a live reading, this stays
correct as you gain or lose altitude.

### Heat-limited power — Charge Air Temp with Timing Advance

Charge air temp is the truck's honest report of whether the intercooler is
keeping up. Watch it with timing advance:

| Charge air temp | Timing advance | Meaning |
|---|---|---|
| Near ambient | Normal | Healthy, full power available |
| Climbing | Retarding | ECU pulling timing to prevent knock — power is fading |
| High, stable after stopping | — | Heat soak; recovers once moving |

Timing retarding *without* a temperature rise points at fuel quality or knock
sensor activity instead.

### Fueling health — Commanded vs Measured Lambda

Equivalence Ratio is the target, O2 Lambda is reality. They should track
closely. A persistent gap means fueling isn't achieving its target — failing
sensor, vacuum leak, or fuel delivery shortfall.

### Total fuel correction — Short Term plus Long Term Trim

Add them. Their sum is how far off the base fuel map the engine is running.

| Sum | Meaning |
|---|---|
| Within ±5% | Normal |
| Around +20% at warm idle | Adding fuel to compensate — vacuum leak or lazy MAF |
| Persistently negative | Running rich — leaking injector or high fuel pressure |

Short term swinging while long term stays near zero is normal closed-loop
operation. Long term drifting away from zero is the engine *learning* a
problem, which is the one to act on.

### Sensor cross-checks

Several values are measured twice by independent paths. Agreement validates
both; disagreement localises the fault.

| Pair | Should agree within |
|---|---|
| Accelerator Pedal D vs E | A percent or two — they're redundant by design |

Oil Temp and Fuel Rail Pressure each had an SAE alias checked against the
proprietary signal early on (0.7°C and similar magnitude respectively);
both aliases were dropped as duplicates and are gone from this file, so
there's nothing left to cross-check them against day to day.

Pedal D and E disagreeing is significant: the ECU compares them itself and
will enter limp mode if they diverge.

### Bank imbalance — Catalyst Temp B1 vs B2

The two banks should run within ~30°C of each other. A persistent split means
one bank is working harder — a misfire, an injector, or an O2 sensor on the
cooler side.

### Load distribution — the four suspension pressures

Front and rear pairs should be near-symmetrical left to right. A single
corner reading consistently low is either a load imbalance or an air spring
losing pressure. Compare against Height Offset: if pressure is low but height
is nominal, the system is compensating for a slow leak.

### Warm-up sequence — Coolant, Oil, Gearbox

These heat at different rates, and the order tells you about circulation.
Coolant reaches operating temperature first, oil follows several minutes
later, gearbox last. Oil temp lagging far behind coolant on a long drive
suggests restricted flow. Gearbox temperature climbing above normal while
towing is the signal to back off.

### Airflow sanity — MAF against RPM and Load

At idle, roughly 3–8 g/s. Under full boost, ten times that or more. MAF
staying flat while load and RPM climb means the sensor is under-reporting —
which the fuel trims will then try to correct, showing up as a rising
positive trim.

---

## Computed signals

The signalset defines twelve synthetic signals. Seven are a **ratio between
two readings that should hold a known value**, which makes them suited to
a display: you learn the normal number once, and anything else is a
signal. The eighth, Fuel Rate, is a different shape — a real physical
quantity rather than a 1.0-normal ratio — and is described separately
below. (An earlier eighth ratio, Rail Pressure Crosscheck, was removed
when its SAE rail-pressure alias was dropped as a duplicate; Fuel Rate
took its slot.)

| Signal | Ratio | Normal | Meaning when it moves |
|---|---|---|---|
| Boost Pressure Ratio | MAP / Barometric | 1.0 idle, up to ~1.8 | Above 1.0 is supercharger boost, self-correcting for altitude |
| Lambda Tracking | Measured / Commanded lambda | 1.0 | Fuelling isn't hitting its target — sensor, leak, or delivery |
| Bank Balance | Cat temp B1 / B2 | 1.0 | One bank working harder — misfire or injector on the low side |
| Pedal Agreement | Pedal D / Pedal E | 1.0 | Redundant pedal sensors disagreeing; the ECU limps if they diverge |
| Throttle Tracking | Actual / Commanded throttle | 1.0 | Plate not following orders — sticky or carbonned throttle body |
| Suspension Balance Front | Front left / right pressure | 1.0 | A corner losing air, before the dash warns |
| Suspension Balance Rear | Rear left / right pressure | 1.0 | Same, rear axle |

All seven sit at **1.0 when healthy**, so a single glance covers fuelling,
ignition balance, pedal and throttle integrity, and air springs.

Boost Pressure Ratio is the exception and the one to watch for fun: it's the
closest thing to a boost gauge this vehicle exposes. The schema's only
operation is division, so true gauge boost (`MAP − Barometric`) isn't
expressible as a synthetic — but the ratio carries the same information and
needs no altitude correction.

### Fuel rate, now computed

The ECM's own supported-PID bitmasks (`0100 = BFBFACD3`, `0120 = A007B119`,
`0140 = FED08511`, `0160 = 07010000`) say PIDs `5E` and `9D` are both
unsupported. The truck does not report fuel rate. Any app showing one is
computing it, and now this signalset does too — added 2026-09-20 in litres
per hour, switched to US gallons per hour on 2026-09-24.

```
Gallons per hour = MAF (g/s) / (Lambda × 14.7 × 745 / 3600 × 3.785)
                 = MAF / (Lambda × 11.516)
```

MAF is grams of air per second; 14.7 is the stoichiometric air-to-fuel
ratio for gasoline; 745 g/L is the density of gasoline; 3600 converts
seconds to hours; 3.785 L/gal converts to US gallons. Commanded lambda
scales the stoichiometric ratio to whatever the ECU is actually targeting.

The schema's synthetic operation is a plain ratio with no constant, so the
constant is folded into a hidden operand: `LR4_FUEL_DIVISOR` reads Commanded
Equivalence Ratio (`22F444`) with `div` set to `2845.54` (`32768 / 11.516`),
which yields `Lambda × 11.516` directly. `LR4_FUEL_RATE` is Mass Air Flow
divided by that, already in gallons per hour, and carries
`suggestedMetric: fuelRate`. The unit is declared `gallonsPerHour`, one of
the schema's enum values, rather than relying on the app to convert.

#### The other ways to look at it

Five fuel synthetics now exist. Every one is built from the same three
polled DIDs, so none costs any bandwidth; the budget is unchanged at
4.100 req/s.

| Signal | Formula | Unit | Lambda? |
|---|---|---|---|
| Fuel Rate | MAF / (λ × 11.516) | gal/h | yes |
| Fuel Rate (gal/min) | MAF / (λ × 690.9) | scalar, labelled | yes |
| Fuel Economy (mpg) | (mph × 11.516) / MAF | scalar, labelled | assumed 1 |
| Fuel Use (gal/100 mi) | MAF / (mph × 0.1152) | scalar, labelled | assumed 1 |
| Fuel Economy (mpg, lambda-corrected, test) | mph / Fuel Rate | scalar, labelled | yes, if it works |

Two constraints shape that table. The unit enum has `gallonsPerHour` and
nothing else for fuel: no gallons per minute, no miles per gallon, no
gallons per hundred miles, no litres per 100 km. Those signals are declared
`scalar` and carry the unit in their name. And a ratio has room for only
two operands, so an exact mpg (speed, MAF *and* lambda) is one operand too
many. The mpg and gal/100 mi signals drop lambda and assume 1.0, which is
true whenever the ECU is in closed loop — most of any drive — and wrong
under enrichment (lambda 0.8 at full throttle, so the readout is 25%
optimistic) and during decel fuel cut (commanded lambda reads 2.0, the
real consumption is zero, the readout is finite). On the 2026-09-23 drive
the stoichiometric assumption cost 3% over the trip: 17.3 mpg against 17.8
with lambda.

The lambda-corrected mpg is an experiment. It divides a hidden mph copy of
vehicle speed by the Fuel Rate synthetic itself, which only works if the
app resolves a synthetic operand that is another synthetic. Nothing in the
schema says either way, and no other OBDb signalset tries it. If it shows a
number on the next drive, it is the mpg to keep and the stoichiometric one
can go. If it stays blank, the app evaluates synthetics against raw
signals only, and the stoichiometric version is as good as a ratio gets.

Ways that were considered and do not work with what this truck offers:
injector duty or pulse width (no DID found), the SAE fuel rate PIDs `5E`
and `9D` (unsupported, above), and fuel level delta over distance. The last
one is real but coarse — the gauge is 8-bit, one step is 0.34 L, and the
trip logger already records tank level at the start and end of every
journey in its own store, so it makes a check on the formula rather than a
live signal.

Validated twice. Integrating the formula over 1,480 MAF samples on the
2026-09-20 drive gives 11.21 L (2.96 gal) burned; the tank gauge fell from
87.8% to 76.9%, which on the 86.3 L tank is 9.48 L (2.50 gal) — agreement to
about 18%, on the pessimistic side, over 91.5 km (19.2 mpg by the
formula). On the 2026-09-23 drive, 15.2 km in 21 minutes, the formula gives
0.54 gal against 0.45 gal on the gauge, where a single gauge step is
0.09 gal. Treat all five as directionally useful, not a trip computer.

#### Decimal places are the app's call

The stored fuel-rate values are full floats (`44.309198338607594`), and
that is what the app prints for a synthetic. Nothing in the signalset
changes it: the schema's `fmt` block has scaling, range, unit and map
fields and no precision field, and a synthetic has fewer fields still. The
same is true of the one decimal that appears on temperatures once the app
is set to Fahrenheit: the signal is an integer in Celsius, the conversion
makes it fractional, and the app's formatter keeps one place. Both are
requests for the app (support@clutch.engineering), not for this file.

---

## Altitude and mountain driving

### Reading altitude

There's no altitude signal, but **Barometric Pressure is one** — air thins
predictably with height. Read `22F433` directly:

| Altitude | Barometric | In psi |
|---|---|---|
| Sea level | 101 kPa | 14.7 |
| 2,000 ft | 94 kPa | 13.7 |
| 4,000 ft | 88 kPa | 12.7 |
| 6,000 ft | 81 kPa | 11.8 |
| 8,000 ft | 75 kPa | 10.9 |
| 10,000 ft | 70 kPa | 10.1 |
| 12,000 ft | 64 kPa | 9.3 |

Roughly 1 kPa per 300 ft near sea level, stretching to about 340 ft
per kPa above 8,000 ft. Weather shifts it a few kPa, so treat
it as approximate unless you calibrate against a known elevation.

### Measured on this truck

From the September 2025 mountain trip, barometric pressure ranged **65–76 kPa,
median 70** — roughly 7,800 to 11,800 ft, spending most of its time near
10,000. Those runs give real baselines to compare against:

| Signal | Mountains, Sept 2025 | Notes |
|---|---|---|
| Charge Air Temp | 6–81°C, median 47 | Wide swing with grade and airflow |
| Gearbox Temp | 8–85°C, median 64 | Never troubling |
| Diff Temp | 0–60°C, median 35 | Cooler than a hot-weather lowland drive |
| Coolant Temp | up to 103°C | Normal ceiling on sustained climbs |
| Mass Air Flow | peak 216 g/s | Full-load airflow |

Worth noting the differential ran *hotter* on flat August driving (median
71°C) than in the mountains (35°C). Ambient temperature matters more to it
than terrain does.

### Fuel trim climbs with altitude

Long term fuel trim tracks altitude clearly across ~76,000 logged samples:

| Barometric | Altitude | Median LTFT |
|---|---|---|
| 100 kPa | ~400 ft | +8.6% |
| 80 kPa | ~6,400 ft | +7.8% |
| 75 kPa | ~7,900 ft | +12.5% |
| 70 kPa | ~9,400 ft | +14.1% |
| 65 kPa | ~10,900 ft | +14.8% |

Altitude adding fuel trim is expected — the mass air flow sensor tends to
under-report in thin air and the ECU compensates. **The part worth watching
is the low-altitude baseline of around +8%.** A healthy engine sits within
±5%, so there's a mild lean condition present before altitude adds anything,
and the two stack: near +15% at 11,000 ft is approaching where a lean code
would set.

Most likely cause on this engine is a small unmetered air leak — the
supercharged V6 has a lot of intake plumbing and hose joints. Not urgent, but
if the truck ever feels flat at altitude, this is the reason to look at
first.

### What altitude does to the engine

A naturally aspirated engine loses about 3% power per 1,000 ft. Yours loses
much less, because the supercharger is a fixed-displacement blower geared to
the crank — it keeps stuffing the same volume in regardless of ambient
pressure. What you'll see instead:

- **Boost Pressure Ratio holds steady.** It's normalised against barometric,
  so it reads the same at 10,000 ft as at sea level. That's the point of
  using the ratio rather than raw manifold pressure.
- **Mass Air Flow drops** at the same throttle and RPM. Thinner air, less
  mass, less fuel, less power — the loss the blower can't fully erase.
- **Fuel trims may drift positive** as the ECU adapts. Mild drift climbing a
  long pass is normal, not a fault.

### What to watch climbing

Sustained high load is the hardest thing you'll ask of the truck.

| Signal | Watch for |
|---|---|
| Charge Air Temp | Climbing on a long pull, with Timing Advance retarding — the ECU protecting itself, and where your power goes |
| Gearbox Temp | The one that matters. Sustained climbing in a low gear heats the ZF fast |
| Coolant Temp | Should stay near 100°C. Rising past that on a grade means the cooling system is at its limit |
| Absolute Load | Pinned near 100% for long stretches means you're asking for everything available |

### What to watch descending

Engine braking sends heat somewhere different.

- **Lambda goes to 2.00** on a closed-throttle descent. That's deceleration
  fuel cut, working exactly as intended — you're using no fuel at all.
- **Gearbox Temp still climbs** in a low gear, even off-throttle.
- **Coolant Temp can rise** on a long descent despite low load, because
  airflow is low and the engine is being driven by the wheels.

---

## Off-road

### The mountain drive that tested off-roading collected no off-road data

Session 2525 (2026-09-20, 15:14–17:32, 137.5 minutes, 91.5 km / 56.9 mi at
7,800–11,000 ft) deliberately included light off-roading to exercise the
suspension and rear differential. It produced none of that data. Every
signal the test was aimed at lives on a module the app had already stopped
addressing before the drive started: ride height, height sensors, corner
pressures, compressor activity and Terrain Response all live on `7D3`,
which the app had not addressed since 14:07 that day. The two
rear-differential-lock candidates lived on `795`, which the app was still
addressing, but neither DID was among the 23 commands it polled that
session.

This is not damage and it is not new. Long sessions have settled to
between 12 and 23 commands for six weeks, and this drive's 23 was the
widest coverage of any long session in that period — see "The app
converges on a small working set" under Polling. The suspension module
carries no metric-bearing command, so the app has no reason to keep asking
it anything once the novelty of a signalset change wears off.

Everything below about air suspension and the rear differential was
recovered from earlier sessions already present in the same log file, not
from this drive.

The deeper constraint is separate and permanent: none of the suspension
signals carry a `suggestedMetric`, so none of them will ever be recorded as
*history* by the app — see "Only signals with a metric are ever recorded"
under Polling. Getting `7D3` polled again restores live readouts and raw
scan-log coverage, not a trip history for ride height or corner pressure.

### Air suspension

The four corner pressures are the most useful thing you have off-road. On
uneven ground the truck articulates, and pressures diverge as weight shifts.

- **Suspension Balance Front / Rear** — the two synthetic ratios. At 1.0 the
  axle is evenly loaded. Away from 1.0, weight has moved to one side.
- A corner going very low while its opposite goes high means that wheel is
  unloading — the first hint of lifting a wheel.
- **Height Offset** shows where you are relative to normal ride height.

Back on pavement both balance ratios should settle at 1.0. If one doesn't,
you've either shifted your load or picked up a slow leak.

### Watch on the trail

| Signal | Why |
|---|---|
| Gearbox Temp | Low-range crawling generates heat with almost no airflow. The number to respect |
| Locking Diff Oil Temp | Same reason, and it climbs fast when the diff is locked and working |
| Coolant Temp | Low speed means low airflow; the fan is doing the work alone |
| Suspension Balance | Articulation and weight transfer, live |
| Gear Selector | Confirms what the transmission thinks it's in |

A drive on 2026-09-20 that changed ride height confirmed Ride Height, the
height sensors and the corner pressures all move together and in the
directions their labels predict — see "Air suspension notes" below. `3B4D`,
once considered for Drive Mode or Terrain Response, is neither: a later
audit across 236 samples spanning many drives and several terrain modes
found it constant at `0x00` throughout. It stays in the file as a 600s
probe. Terrain
Response itself remains unidentified on this truck — nothing currently in
the signalset is known to carry it.

### Before you go

Air suspension raises the truck at low speed and drops it as you speed up. If
you're crawling and it lowers unexpectedly, watch Height Offset and the
corner pressures — an EAS fault normally means the truck sits down, which
matters a lot more with rocks under it than in a parking lot.

---

## Undecoded

These respond on the bus but their meaning isn't established. They're in the
signalset so their values get logged during normal driving, which is how the
formulas eventually get worked out.

| Address | Signals |
|---|---|
| `7E1` / `7E9` | `1E68`, `DD01` — `1E68` is a neighbour of the gearbox temp and varies most of anything unmined on this module; `DD01` shares its DID with the engine's odometer but isn't one |
| `761` / `769` | `197C`, `D11C` — the most active unmined DIDs on the truck, four distinct values each in four samples |
| `7D3` / `7DB` | `3B01`, `3B02`, `3B0B` — `3B01` looks like the ride-height state word (see "Air suspension notes" under Off-road); `3B02` answers four single-byte values that look like one per corner |

### Nineteen commands deleted, and restored

These were deleted on 2026-09-20 and restored the same day, because the
deletion was a mistake worth recording.
The reasoning was that each was constant across its logged samples. The
sample counts were the problem:

| Command | Samples | Distinct values | Was the evidence conclusive? |
|---|---|---|---|
| `726` / `0202` | 13 | 1 | No |
| `792` / `2A32`–`2A3C` (eleven) | 10–19 each | 1–3 | No |
| `795` / `1E88` | 6 | 1 | No |
| `795` / `1E89` | 25 | 2 | No |
| `7E1` / `1E6A` | 29 | 1 | No |
| `7D3` / `3B00` | 55 | 2 | Borderline |
| `7D3` / `3B08` | 64 | 1 | Borderline |
| `7D3` / `3B4D` | 236 | 1 | Yes |
| `7E0` / `F458` | 1 | 1 | No, and the reading was wrong |

Six samples is not evidence that a rear differential lock DID is dead; it
is evidence that the differential was never locked while anyone was
looking. A signal that only moves during a rare event looks constant until
the event happens, and deleting it guarantees the event is never caught.

`F458` was worse than weak — it was misread. `62F458 7F` is a *positive*
reply carrying the data byte `0x7F`, not a `7F` negative response. PID 58
is the long term secondary oxygen sensor trim for bank 2, the counterpart
to PID 56 which this file already carried, and `0x7F` decodes to -0.78%.
It is now mapped properly as `LR4_SEC_O2_TRIM_B2`.

All nineteen are back, at exploratory cadences rather than their old ones:
the event-driven candidates (`1E88`, `1E89`, `3B4D`) at 60s, `3B00` and
`3B08` at 120s, and the rest parked at 600s. Restoring the whole set costs
0.03 req/s against a 4.2 req/s ceiling, so the budget was never the real
argument for removing them. `3B4D` stays in despite being the one genuinely
conclusive case, because at 600s it is free and the cost of being wrong
about it is another six weeks of not knowing.

### What the restored probes did within hours

The probes went back in, the change was merged, and one short evening drive
on 2026-09-20 in which the off-road features were deliberately used settled
three of them. The merge also did what it was predicted to: coverage on the
next sessions jumped from 23 distinct DIDs and 10 modules to **70 DIDs and
13 modules**, with 87 of the file's 90 commands polled. Every module that
had drifted out — `726`, `732`, `761`, `792`, `7D3` — came back without
anyone touching the app.

**`3B4D` is alive, and it was the one deletion called conclusive.** Constant
`0x00` across 236 samples, deleted, restored at 60s on the argument that the
samples had never covered the event. Within eight minutes of the features
being used it returned `0x04`, `0x01` and `0x00`. It does not track the ride
height mode PID, so it is reporting something else, and Terrain Response is
back to being the leading candidate. It now polls at 30s and needs one
sample per selectable mode to map.

**Six of the eleven `792` counters advance.** They were written off on 10 to
19 samples each, all taken within one morning. Nine hours and 111 km later:

| DID | Was | Now | Gain | Per km |
|---|---|---|---|---|
| `2A32` | 404,720 | 420,200 | +14,000 | 126 |
| `2A33` | 212,490 | 212,970 | +480 | 4.3 |
| `2A34` | 2,240 | 3,200 | +960 | 8.6 |
| `2A35` | 335,400 | 337,570 | +2,170 | 19.5 |
| `2A36` | 43,275 | 43,363 | +88 | 0.8 |
| `2A37` | 107 | 108 | +1 | 0.01 |

They are counters on a module still not identified. What they count is
unknown — none of the per-km rates is a clean unit and the elapsed window
mixes driving with nine hours parked. They now poll at 120s so a single
drive yields enough samples to regress against distance and running time.
`2A38` through `2A3C` have still never moved.

**`3B01`'s last bit is pinned.** `0x400` was Off-Road by elimination and
unconfirmed. It was caught twice at a front sensor reading of 85 to 89, the
truck at its highest, with the mode PID reading Off-Road. `3B01` is now a
mapped signal rather than a raw word: `0x100` Normal, `0x400` Off-Road,
`0x800` Access.

**Two decodes in this file were wrong, and the short drive exposed both.**

The second word of `F434` and `F438` was being read as a voltage. SAE J1979
PIDs 34 to 3B report equivalence ratio plus sensor *current*; the voltage
variants are PIDs 24 to 2B, which this ECM does not support. Read as a
voltage it produced a flat 4.0 across 28,432 samples, which should have been
the giveaway. Read correctly it spans -0.95 to +1.36 mA around a mean of
+0.16, exactly what a wideband pump cell does. Both are now
`O2 Pump Current`, in milliamps.

`D11A` was called Suspension Module Voltage and scaled to volts. It is not
battery voltage. Across 221 paired samples its raw value swings 57 to 105,
an 84% range, while control module voltage moved only 12.51 to 14.75, an 18%
range, and the implied volts-per-count scatters by 9%. The old scaling put
it at 16.8 V, which no 12 V system reaches. The best correlation found for
it is 0.58 against compressor activity, which is not enough to name
anything, so it goes back to being an undecoded raw byte.

**The two fuel rail pressures are not the same measurement.** `033E` and
PID 23 were expected to be one quantity scaled two ways. Across 157 paired
samples the ratio between them drifts from 0.41 to 0.56 instead of holding
constant, and in one stretch PID 23 sat pinned near 197 bar while `033E`
climbed from 79 to 110. A scaling error gives a fixed ratio; this does not.
Two different quantities, plausibly a commanded rail target against a
measured one, and separating them needs hard sampling under load.

**There is one ride-height state nobody has logged.** Holding the lower
button puts the truck into a held-Access mode that stays down rather than
self-levelling, and it appears in none of the 484 `3B3C` or 142 `3B01`
samples. Both words have exactly one unfilled slot: `3B01` bit 9 (`0x200`),
sitting between Normal at `0x100`, Off-Road at `0x400` and Access at
`0x800`; and `3B3C` bit 3 (`0x08`), which has only ever been seen inside
`0x0D` with the truck in motion between heights. One of them is the likely
home for it.

Because a mapped signal renders nothing for a value it has not been taught,
both DIDs now carry an untranslated twin — `Ride Height State Raw` and
`Ride Height Mode Raw`. They cost no extra request and they mean an
unrecognised state arrives as a number instead of a blank. `3B4D` showing
empty in the app is exactly that failure mode, and it is why the value went
unnoticed for so long.

Two more suspension DIDs turn out not to be flat either. `3B00` reads
`0x03` in 111 of 113 samples with one `0x103`, taken with the truck at
Access; `3B02` carries a bit-8 flag that toggles the same way; and `3B08`
reads `0x00` in 130 of 131 with a single `0x01`, taken with Off-Road
engaged and the truck at its highest. One sample each is not a decode, but
none of the three is dead.

**The ride height map checks out end to end.** A screenshot at 20:19 shows
Ride Height Mode reading **Access** and the rear sensor at 128; the scan log
for the same moment has `3B3C` at `0x04`, `3B01` at `0x800` and the rear
sensor at 128. Access had never once displayed before the map was corrected.

**All six new probes answered.** PID 01 reports the MIL off with zero stored
DTCs. PID 13 returns `0x77`, six oxygen sensors, three per bank. PID 51 is
gasoline, PID 1C is OBD-II. PID 23 ranges from 29 to 197 bar across the
logs, against the proprietary `033E` reading 41 to 46 bar over the same
evening — both are live and they disagree, which is the comparison worth
making under load.

The standard this file now holds deletions to: a DID needs enough samples
to have covered the event it would report, not merely a lot of samples.
Hundreds of readings taken while the differential was never locked say
nothing about a differential lock DID. When in doubt, park it at 600s
instead — a probe costs 0.0017 req/s and deleting it costs the answer.

**The suspension decodes check out.** Replaying every logged sample through
the formats in this file gives corner pressures of 34-50 psi, a ride height
offset correctly signed at -91 to +50 mm, and module voltage of 9.12-14.56 V,
all physically sensible.

**Terrain Response and `3B4D`.** `3B4D` was labelled Drive Mode and was the
candidate for Terrain Response. Across all logged history it returns a
constant `0x00` — 236 samples, spanning many drives and several terrain
modes, not just the one 2026-09-20 drive that first flagged it as flat.
That's not "unconfirmed," it's answered: `3B4D` is not Terrain Response and
appears to be nothing. It is nonetheless still in the file at 600s, because
that costs 0.0017 req/s and leaves the door open — see "Nineteen commands
deleted, and restored" above.

What the audit could not do is say anything about suspension health. Across
fourteen months only 36 minutes have all four corner pressures captured
together, because the `7D3` module receives one or two requests per drive
despite its `freq` of 10.

**Tire pressures remain unsolved, but they exist.** Module `751` returns no
data to any request, and nothing found so far resembles four tire pressures.
The decisive clue is the spare: this truck warns when the *spare* is low, and a
spare does not rotate — so there is no wheel-speed difference to infer it from.
That rules out an indirect system and means real pressure sensors are fitted and
reporting to some module. Six modules answer negative responses but have never
been mined — `716`, `726`, `734`, `737`, `760`, `797` — and each is now probed
with the Jaguar tire-pressure DID as a locator. A positive response from any of
them finds the module.

### Air suspension notes

A 33-minute drive on 2026-09-20 that changed ride height twice gave the first
real test of these fields, and most of them passed it.

`3B71` and `3B72` track the height sensors, and they read **inverted** — the
number falls as the truck rises. Observed: 115/115 at normal height, 93/101
raised; on the 2026-09-20 drive `3B71` fell from `6A` to `56` on the raise,
then rose to `85` by the end. Treat a falling value as the truck going up.

The four corner pressures (`3B03`-`3B06`) tracked the same drive unambiguously:
around `00C5` cruising, down to about `00AC`, up to about `00FE` at 16:22:42
UTC as the truck raised, then down to `0096`-`009F` at 16:23:49 as it lowered.
Compressor activity (`3B07`) jumped from around `00A0` to `05F0` and `0617`
during those changes.

The driver believed the low "access" height didn't register. It did — the
corner pressures and height sensors both recorded a third, lower level at
16:23:49. What was wrong was the map, not the sampling, and a later audit
across all logged history settled it.

**The ride height map was wrong, and that is why Access never registered.**
`3B3C` across all logged history returns `0x01` (216 samples), `0x02` (11),
`0x04` (5) and `0x0D` (4). The map in this file was ordinal, 0 through 3,
with 0 labelled Access — but the value 0 never once occurs, and `0x04` and
`0x0D` weren't in the map at all. Pinned against the front height sensor,
which is inverted so a falling value means the truck is rising:

| Value | Front height sensor mean | State |
|---|---|---|
| `0x01` | 111.4 (range 91–122) | Normal |
| `0x02` | 90.4 (85–95), highest the truck sits | Off-Road |
| `0x04` | 133.0, lowest the truck sits | Access |
| `0x0D` | 120.5, between Normal and Access | In Transit |

The map is now `1, 2, 4, 13`. `3B01` carries the same state in parallel:
`0x100` pins to a front mean of 111.6 (Normal) and `0x800` to 133.0
(Access). `0x400` has been seen once and never alongside a height sample, so
it's Off-Road by elimination — unconfirmed, unlike the other three.

The two balance ratios respond to cargo, not just faults. With the load area
full, front balance read 0.99 and rear read 0.92 — the rear axle carrying more
on one side. Check the ratios unloaded before reading a low number as a leak.

---

## Polling

### The budget was measured against the wrong ceiling

For most of this file's history the budget was set against **13
request/response round trips per second**, treated as the adapter's
ceiling. That number is real, but it's the whole bus, and most of it
belongs to the app, not to this signalset. Measured on 2026-09-20 (session
2525): total adapter throughput was 13.43 req/s, splitting into AT
commands (adapter setup, header changes, filter changes) at 3.27/s, the
app's own internal mode-01/mode-09 polling — which never surfaces as a
recordable signal — at 5.94/s, and this signalset's UDS service-`22`
traffic at 4.22/s.

Across seven sessions on 2026-09-19 and 2026-09-20, the UDS share stayed
pinned between 4.1 and 5.1 req/s no matter how large the signalset was.
Session 2504 polled 108 distinct DIDs and got 4.19 req/s. Session 2525
polled 23 distinct DIDs and got 4.22 req/s — five times fewer commands,
identical throughput. **The signalset's real ceiling is about 4.2 req/s,
not 11.** This file had been asking for 10.53.

Service `22` is read-only. Polling cannot harm anything; it can only
compete with itself, and with the app's own traffic, for a share that
doesn't grow no matter how much is asked of it.

**`freq` is a minimum interval in seconds, not a rate.** Pelican documents it as
the maximum frequency at which a command may be sent, expressed in seconds, so
a *smaller* number polls *harder*. There is no priority field anywhere in the
v3 format — command count and `freq` are the only levers.

The budget in this file, cut to fit the measured ceiling on 2026-09-20 and
rebalanced on 2026-09-25 so the fuel readout updates every few seconds:

| Interval | Commands | Contents |
|---|---|---|
| 2s | 3 | Engine speed, vehicle speed, mass air flow |
| 3s | 1 | Commanded equivalence ratio |
| 5s | 2 | Calculated load, manifold pressure |
| 10s | 3 | Throttle position, gear selector, ride height mode |
| 15s | 6 | Coolant temp, short term trim B1, lambda B1S1, both height sensors, `3B01` |
| 30s | 20 | Remaining temperatures (oil, charge air, gearbox, diff), fuel rail pressure, timing advance, absolute load, both pedal sensors, fuel trims, commanded throttle, fuel level, all four corner pressures, compressor activity, `3B4D` |
| 60s | 17 | Height offset, MAF A/B, lambda B2S1, ambient, barometric/altitude, battery, catalyst temps, IAT, rail gauge pressure, charge cooler coolant, `3B02`, `3B0B`, `D11A`, `1E68`, `1E88`, `1E89` |
| 120s | 20 | Slower diagnostics: fuel system status, evap purge, O2 voltage B1S2, secondary O2 trims, relative/absolute throttle, IAT 1/2, manifold pressure (fine), run time, `197C`, `D11C`, `3B00`, `3B08`, `2A32` to `2A37` |
| 300s | 1 | Distance since codes cleared |
| 600s | 18 | Odometer, oil level, oil volume, distance with MIL on, warm-ups since codes cleared, monitor status, fuel type, OBD standard, `DD01`, `1E6A`, `726 0202`, `2A38` to `2A3C` |

That's **91 commands at 4.083 req/s**, under the measured 4.2 req/s
ceiling. The 2026-09-20 cut took it from 85 commands at 10.528 to 66 at
3.922; the probes restored the same day and the decodes of 2026-09-23 took
it to 91 at 4.100; and the 2026-09-25 rebalance paid for mass air flow at
2s and commanded lambda at 3s (up from 5s each) by slowing throttle
position and the gear selector to 10s, lambda B1S1 to 15s, rail pressure,
timing advance, absolute load and both pedal sensors to 30s, and MAF A/B
to 60s. Engine speed is on a real cadence (2s) and carries the
`engineSpeed` metric; it had not been polled since 2026-08-30 — see below
for why.

**What the drives of 2026-09-22 and 2026-09-23 actually polled.** Three
drives, 14, 10 and 22 minutes, and the app sent 19, 20 and 21 distinct
DIDs: the `7E0` `F4xx` core plus oil temp, charge air temp, gearbox temp,
diff temp and the odometer. Not one request went to `7D3`, `732`, `792`,
`761` or `726`. The last time any suspension, gear-selector or counter
probe was sent was 2026-09-21 02:20 UTC, in the session right after the
probes were restored. So the open items that need a probe to be alive
(held-Access, the Terrain Response map, the `792` counters) got nothing
from these drives, exactly as trap 5 predicts: the app converged on the
metric-bearing set and the probes dropped out of rotation. The 2026-09-23
drive also predates the merge of the ride-height and wideband decodes, so
the reshuffle a file change usually causes has not had a drive yet. The
four sessions logged after it contain no traffic at all.

The cadence the app delivered on that drive is worth having: commands
asked at 5s were answered every 5.7s, at 15s every 15.5s, and the two at 2s
every 3.05s. So a `freq` is honoured within about 15% except at the top,
where 2s buys 3s. The 21 commands it chose add up to about 1.8 req/s, well
under the ceiling; the app was not short of budget, it was short of
interest. The fuel operands were on the 5.7s cadence, which is why the
fuel readout felt slow.


### The app converges on a small working set, and it is roughly what it can store

An earlier version of this section claimed the app "retires ECUs
permanently" in a one-way ratchet, and recommended resetting the app's
vehicle profile. **That was wrong, and it was wrong because it read a
single day's sessions without checking the months behind them.** The
correction is recorded here rather than deleted, because the wrong version
was convincing.

What the daily history actually shows, taking the longest session of each
day since 2026-09-05 — the only sessions comparable to a drive:

| Session | Date | Minutes | Distinct DIDs polled |
|---|---|---|---|
| 2397 | 09-05 | 67.2 | 13 |
| 2402 | 09-06 | 25.9 | 17 |
| 2426 | 09-09 | 63.9 | 17 |
| 2441 | 09-10 | 57.8 | 17 |
| 2460 | 09-11 | 63.9 | 17 |
| 2470 | 09-13 | 35.5 | 17 |
| 2495 | 09-19 | 50.0 | 19 |
| 2497 | 09-19 | 38.7 | 18 |
| 2525 | 09-20 | 137.7 | **23** |

Long sessions have settled to between 12 and 19 commands for six weeks.
The off-road drive polled 23, which is *more than any other long session in
the period*. The drive was not degraded. It was the best long session on
record, and the 61-of-84 figure is simply what this app has always done.

The stable core is the interesting part. Thirteen commands appear in every
long session going back to 09-05: `03F3`, `1E69`, `DD01`, `F404`, `F405`,
`F406`, `F40D`, `F410`, `F42F`, `F431`, `F434`, `F442`, `F444`. Those are
exactly thirteen of the sixteen commands that carried a `suggestedMetric`
at the time. Not approximately — exactly. The three metric-carrying
commands missing from the core are `F40C`, `F411` and `F443`, and `F411`
joined the working set later.

Read alongside "Only signals with a metric are ever recorded" below, the
behaviour is coherent: **the app converges on polling the commands whose
values it is going to keep.** Commands without a metric get exercised for a
while after the signalset changes and then thin out, because the app has
nothing to do with their values.

That also explains the peripheral modules without inventing a ratchet.
`7D3`, `792`, `732`, `726` and `761` do not carry a single metric-bearing
command between them. They are not being punished; there is nothing on them
the app would store.

The one-day pattern that produced the ratchet theory is real but means
something duller. Coverage spikes after the signalset changes and then
settles: 108 distinct DIDs at 09:57 on 09-20, then 54, 58, 19 and 23 as the
day went on, with several PRs merged in between. Session 2493 on 09-19 hit
965 DIDs, which was the PID detector sweep. Coverage recovers on its own
every time the file changes. **Nothing is stuck, and nothing needs
resetting from inside the app.**

**A command's presence in this file still does not mean it is being
collected** — check the scan logs before relying on one. But the reason is
ordinary triage by the app, not damage.

What remains untested is whether cutting the request budget changes which
commands make the working set. Asking for 10.53 req/s against a 4.2 req/s
ceiling meant the app chose the 40% it would serve; asking for 3.922 means
it does not have to choose. Whether it then serves all 66 is exactly what
the next drive measures.

### Eight commands went quiet for three weeks — the first sighting of the working set, not a separate incident

This section originally explained an isolated 2026-08-30 incident as a
stale copy of the signalset on the app's side. It wasn't isolated, and the
stale copy was only half of it. Those eight commands were sitting outside
the working set the app settles on between signalset changes, described
above. The history below is kept as-is because the reasoning in it — ruling
out the request budget and the `freq` tiers before landing on "the app must
be holding stale data" — was a reasonable read of the evidence available
that day, and because the refresh really did bring them back for a while.

Between 2026-08-30 and 2026-09-19, eight commands in this file were never
requested on a drive: `F40C`, `F411`, `F40E`, `F443`, `F449`, `F44A`, `F407`
and `033E`. The cost was real — `engineSpeed` is wired to `F40C`, and no
engine speed value reached the signal database in fourteen months. Two
synthetics could not compute either, `LR4_THROTTLE_TRACKING` needing `F411`
and `LR4_PEDAL_AGREEMENT` needing `F449` and `F44A`.

**Looked resolved on 2026-09-20, briefly.** Every command in the file was
polled that morning. `F40C` was requested 172 times and answered 172
times, and engine speed was recorded for the first time in fourteen
months. At the time this was credited to the app picking up a fresher copy
of the signalset, and that reading has held up: coverage does spike right
after the file changes. What it does not do is stay there. By the 15:14
session the working set was back to 23 commands, and `F40C` was outside it
again. The eight commands were never broken; they sit outside the set the
app settles on between signalset changes, which is roughly the commands
whose values it stores. `F40C` carries `engineSpeed` and should have been
in that set, and its absence is the one part of this that is still
unexplained.

Getting to the "stale signalset" explanation meant ruling out the two
obvious causes, and both remain worth knowing even though the explanation
built on them was wrong. It was not the request budget: demand was 10.21
req/s against the 11-13 the adapter appeared to deliver — appeared to,
because that 11-13 figure was the whole-bus number, not this signalset's
real share. It was not the `freq` values either, because the dead commands
shared tiers with live ones — `F40C` and `F40D` are both `freq` 1 and only
`F40D` ran; `F411`, `F443`, `F449` and `F44A` sat at `freq` 3 alongside
`F404`, `F434` and `F444`, which all ran.

The lesson that outlives the incident, revised: **a command's presence in
this file means nothing until the scan logs confirm the app is actually
addressing its module this session.** Before concluding a command is
unsupported, or that a fix worked, check which headers the app is
addressing — not just which commands answer when you send them by hand.

### The seven probes, and how they turned out

Seven commands were added on 2026-09-20 to be watched rather than trusted.
The 2026-09-20 drive settled all of them.

**`7E0/F408` and `7E0/F409` answered — the bank-2 fuel trims exist.** They
had never been requested on this vehicle before. They are now real signals
rather than probes, and what they show matters: see below.

**`7E0/1153`, `7E0/1154`, `7E0/113F` and `7E1/2104` returned `7F 22 31`.**
They are session-gated, not dead — see the manual-probe list in
`TESTING.md`. The charging-voltage reading behind `1153`/`1154` still
stands (`1154` pinned at exactly `0E00`, 14.00 V, with both dipping to
5.7-10.1 V on crank, tracking nothing that `F442` does), but it can only be
read by hand in an extended diagnostic session, so those commands have been
removed from this file.

**`726/0202` answered and stayed at `00`** across twelve samples.

`F41F` also recorded, which was the incidental test of whether the `F4xx`
alias mechanism works at all against a standard PID the app already polls.
It does.

### Both banks are running lean, and they agree with each other

First bank-2 data from this truck, measured over the 2026-09-20 drive:

| | Bank 1 | Bank 2 |
|---|---|---|
| Short term | -0.55% (n=305) | -5.36% (n=7) |
| Long term | **+9.31%** (n=13) | **+7.32%** (n=8) |

Two points apart on the long-term trims is not an asymmetry. Both banks are
correcting lean together by 7-9%, which points at something shared — fuel
delivery, MAF calibration, or unmetered air upstream of where the intake
splits — rather than a fault on one side. Any earlier reading of this as a
bank asymmetry came from having only bank 1 to look at.

Sample sizes are small: one drive, and single digits on three of the four
figures. Treat the direction as real and the magnitude as provisional.

`F403`, `F42E` and `F438`, added to the file below, exist specifically to
test this: whether the engine was in closed loop when the trims were
measured, whether a stuck-open purge valve is pulling in unmetered air, and
what bank 2's own oxygen sensor is doing behind its trim.

### The debug batch, and what the first drive did to it

On 2026-09-20, 51 commands went in — one for every DID this truck had ever
answered that nothing here decodes — each a raw scalar at its observed byte
width, named `LR4_<DID>_RAW`. The first real drive cut that back hard.

**Thirty-six of them are session-gated.** The whole `7E0` `10xx`/`11xx`
family and the `7E1` `0302`-`0305`, `0805`, `101A`, `2104`, `210F`,
`2B1B`-`2B1E` block returned `7F 22 31`. In fourteen months of logs they
have answered in exactly two sessions: one in December 2025 that sent
`10 03` to open an extended diagnostic session, and one in June 2025 doing
unusual protocol and flow-control setup. Every session since that did not
open an extended session has been rejected.

The app polls in the default session, so it can never reach them. All 36
are out of this file and listed in `TESTING.md` as manual-probe-only. This
also rewrites the history: `1cf20db` dropped many of them as "dead probes",
and they were never dead — they were session-gated, and nobody had noticed
the December session opened `10 03` first.

What survived, and how it was re-tiered after the drive:

| Command | Change | Why |
|---|---|---|
| `7D3/3B01` | `freq` 600 → 10 | Returned three single-bit values; looks like the ride-height state word |
| `7D3/3B00`, `7D3/3B08` | `freq` 600 → 30 | Still flat through a real height change, but on 3 samples each — one more look before deletion |
| `761/197C`, `761/D11C` | `freq` 60 → 30 | Four distinct values in four samples each; the most active unmined DIDs on the truck |
| `792/2A32`, `792/2A37` | unchanged | Each ticked by a small amount, consistent with counters. The rest of the `2A3x` block stayed flat |
| `726/0202`, `795/1E88`, `795/1E89` | unchanged | Still flat; the diff-lock cycle is what will decide the `795` pair |

The file is now 73 commands at 10.55 req/s.

A DID that moves on a real drive earns a proper decode. One that stays flat
through a full cycle — including a ride-height change and a terrain-mode
switch — can be judged on evidence rather than on suspicion. Do not read a
`7F 22 31` as "unsupported" without checking whether an extended session
would have changed the answer.

### Sixteen more, found by reading the ECM's own PID list, and how they did

The ECM's `0100`/`0120`/`0140`/`0160` bitmasks report 54 supported mode-01
PIDs. Twenty-five had no entry here; after removing metadata PIDs and the
two SAE aliases already dropped, sixteen went in on 2026-09-20 and were
tested the same day. **All sixteen answered.** Ten earn their place.

| Command | Result |
|---|---|
| `22F466` | **Two mass airflow sensors.** A 2.09 g/s against B 1.78 g/s, summing to 3.88 against `F410`'s 4.16 a second later. One idle sample — see the correction below |
| `22F467` | **Two coolant temperature sensors**, 61 °C and 35 °C |
| `22F468` | **Two intake air temperature sensors**, 36 °C and 59 °C |
| `22F403` | Fuel system status, two states over 14 samples: open loop on load or decel ×9, closed loop ×5 |
| `22F42E` | Evap purge duty, 0 to 72.2% |
| `22F438` | Bank 2 lambda 0.853-1.985 over 29 valid samples; railed at `FFFF` on 23 of 52 |
| `22F415` | Post-catalyst O2, 0.13-0.93 V and switching |
| `22F445` / `22F447` | Relative throttle 2.7-22.7%, absolute throttle B 12.5-22.4% |
| `22F470` | Decoded against SAE J1979 PID 70: channel A reads 24.72 kPa at idle. Later shown to be manifold pressure at finer resolution, not a separate boost sensor — see "Duplicates found and removed" above |

**The dual mass airflow sensors exist, but the 17.5% disagreement claim
above was wrong — it rested on a single idle sample.** With 1,474 paired
samples from the 2026-09-20 drive, the least-squares slope of (A+B) against
the single MAF PID through the origin is 0.88, not 1.0, and individual
pairs scatter badly: one pair reads 7.03 g/s against the single PID's
18.16 g/s at the same instant. The split between the two channels is not
trustworthy at this polling rate. Use the single MAF PID for anything
quantitative; treat A and B as present but unvalidated.

`F403` matters for the same investigation: the engine does reach closed
loop, so the 7-9% long-term trims are real measurements rather than
open-loop artifacts.

**Four were declared wrong and recorded nothing useful.** `F466`, `F467`,
`F468` and `F470` were added at `len` 8 because their payload widths were
unknown. The first byte of each turns out to be a sensor-support bitmask,
not data, so all four logged a constant 3 or 2. The figures above came from
decoding raw frames by hand. Widths are now known and declared properly.

**Three did nothing and are gone.** `F416`, `F419` and `F41A` return `FFFF`,
not-available, on every sample. The ECM's bitmask claims them; the values do
not exist.

**One did its whole job in a single reply and is also gone.** `F413`
returned `0x77`: bank 1 sensors 1, 2 and 3 and bank 2 sensors 1, 2 and 3 are
present. That can never change, so it is recorded here instead of polled.

Two still needed samples at the time: `F456` (three samples, all near
zero) and `F458` (one sample, `7F`, meaning unknown). `F458` is now
resolved — it returns a `7F` negative response consistently and has been
deleted; see "Nineteen commands deleted, and restored" under Undecoded. `F456` stayed in
at `freq` 120.

`F470` was read against the SAE J1979 definition of PID 70: its ten bytes
are commanded boost and measured boost for two channels, at 1/32 kPa. The
support byte is `0x02`, flagging channel A as present, and channel A read
24.72 kPa at idle. At the time this looked like the first direct boost
measurement on this truck. A wider check on 2026-09-20 against 525 paired
samples found channel A tracking Manifold Pressure to within 1.3 kPa mean
(r = 0.85) — it is the same manifold sensor at finer resolution, not a
separate boost sensor. See "Duplicates found and removed" above. Boost is
still only available by subtracting Barometric Pressure from Manifold
Pressure, as below.

Checking the rest of the signalset the same way found nothing else. Against
the canonical SAE signalset, the only other unread fields in replies we
already make are sensor-present bits and channels this engine does not have:
`F468` carries six intake air temperature slots and answers `FF` on four,
`F466` and `F467` expose support bits for sensors we already read, and
`F408`/`F409` reserve a second byte for bank 4, which a V6 does not have.

Barometric pressure moved from `freq` 1 to `freq` 5 to pay for this. It does
not change at 1 Hz, and it was consuming 0.9 req/s of a budget with no slack.

### From 89 commands back down to 66

The narrative above tracks the file growing to 89 commands at 10.93 req/s
over the course of 2026-09-20. Later the same day, session 2525 showed
that budget had been measured against the wrong ceiling — 11 req/s that
was never really available, against a true UDS share of about 4.2 req/s —
and that the app settles on a working set of roughly twenty commands
between signalset changes regardless of what the file asks for. The
response was a re-tiering rather than a cull: nineteen commands were
deleted and then restored as slow probes once it was clear the evidence
against most of them was six to thirty samples (see "Nineteen commands
deleted, and restored" under Undecoded), the duplicate signals resolved under
"Duplicates found and removed" (PID 67 sensor 1 deleted outright, three of
PID 70's four channels dropped as dead alongside it), and a re-tiering of
everything that survived. The result is the 66-command, 3.922 req/s budget
at the top of this section.

### Only signals with a metric are ever recorded

This is the most consequential thing measured on 2026-09-20, and it sets the
boundary on what this whole project can deliver.

**The app writes a signal to its database only if that signal carries a
`suggestedMetric`. Everything else is requested, answered, decoded and
thrown away.**

Across fifteen months of backups, from 2025-06-29 to 2026-09-20, the signal
database has held exactly 15 distinct signals. Not 84, not the 98 the file
carried at its peak. Fifteen.

The 137-minute drive on 2026-09-20 recorded 14, and they are precisely the
metric-carrying signals whose commands were being polled. The signalset held
16 metrics that day. The two missing from the database are `engineSpeed` and
`absoluteEngineLoad`, and they are missing for the unrelated reason that
their commands, `F40C` and `F443`, were among the 61 never sent. The match
is exact in both directions: every metric-carrying polled signal was stored,
and no signal without a metric ever was.

The other two stores in the backup are not alternatives. `records` holds
manual service and fuel-up entries and is empty. `tripLogger` holds the
phone's own CoreLocation trace — latitude, longitude, altitude, speed — with
no OBD content in it.

What that costs, stated plainly: the metric enum has no slot for manifold
pressure, ride height, suspension pressure, corner pressure, charge air
temperature, barometric pressure, ambient temperature, rear differential
temperature or boost. **None of those can ever be looked at historically
through the app.** They exist live, or in the raw scan logs, and nowhere
else.

That includes the suspension work in its entirety. Even when the
`7D3` module is being polled, no ride height and no corner pressure will be
recorded as history. It includes the charge cooler coolant temperature newly
decoded from PID 67 sensor 2 — a real signal, genuinely new, that will never
persist.

One limit on the claim, now resolved: what was measured is that these
signals are never *stored*. Whether the app renders them live was a separate
question the logs could not answer, and screenshots on 2026-09-20 settled it
— **non-metric signals do display live**, with current values, in the app's
section lists. Manifold pressure, charge air temperature, the suspension
pressures and every raw probe are all on screen. So the bandwidth spent on
them buys a real readout; what it does not buy is history.

The practical rule: a fast `freq` on a signal with no metric buys a live
readout and nothing else. That can be worth paying for. It should be a
deliberate choice rather than an accident.

### Metric slots, seventeen of thirty-six

The schema defines 36 `suggestedMetric` values — the slots Pelican treats as
connectable. Per the section above they are also the only signals that get
recorded at all, which makes this table the boundary of what is knowable
about this truck over time, not just a list of gauges. This signalset fills
17, up from 16 on 2026-09-20 when
`fuelRate` was added as a computed signal (see "Fuel rate, now computed"
above). The remaining 19 break down as:

| Slots | Why they stay empty |
|---|---|
| 10 | Electric and hybrid only: state of charge and health, traction battery, charging, electric range, CVT deterioration |
| 8 | Tire pressure and temperature, which this truck has not yet given up |
| 1 | `fuelRange` is a proprietary Ford DID |

`fuelRate` looked like it belonged in this list too — the schema's only
formula operation is `ratio`, a plain a divided by b with no constant and no
scaling — but the constant can be folded into a hidden operand on one side
of the ratio instead of the ratio itself, which is how it got filled. See
above for the mechanism. Tire pressure is the one still worth chasing: eight
of the nineteen empty slots are tires, and `TESTING.md` records what the
Jaguar signalset does and what has never been tried here.

### Hidden signals

Thirty-seven of the file's 119 signals (including synthetics) carry
`hidden: true`: the `*_RAW` probes, the `3B02` byte splits and the
synthetic operands.

**`hidden: true` does not appear to do anything in this app.** Screenshots
taken on 2026-09-20 show `3B00 Raw`, `3B01 Raw`, `3B02 Byte 1`, `3B4D Raw`,
`3B08 Raw`, `2A32 Raw`, `2A3A Raw`, `197C Raw`, `726 0202 Raw`, `D11C Raw`,
`Monitor Status Raw`, `O2 Sensors Present Raw`, `OBD Standard Raw` and
`Fuel Type Raw` all listed with values in the Suspension, Fuel and ECU
sections. Every one of those carries `hidden: true` in this file. Whatever
the flag is for, it is not suppressing them from the section lists, so it
should not be relied on to keep clutter down. It also does not affect
recording: none of these carries a metric, so none was ever stored either
way.

Eighteen signals carry a `description`, concentrated on the probes and on
the decoded signals with a catch — the inverted height sensors, the bank 2
lambda that rails at `FFFF`, altitude being a barometric lookup rather than
GPS.

### Safety

Service `22` is read-only by definition and cannot change vehicle state. It
is safe to send anything at it.

Service `10 03` (extended diagnostic session) does change ECU state. It's a
standard service that auto-reverts after a few seconds, but run it parked —
some modules reduce normal messaging while an extended session is open.

Avoid entirely unless you know exactly what you're doing: `2F` (actuator
control), `31` (routine control), `2E` (write), `11` (ECU reset), `14` (clear
codes), `27` (security access). On this vehicle `2F` and `31` sent to the
suspension module can physically raise or lower the truck, and `14` erases
emissions readiness monitors, which means failing inspection until a full
drive cycle completes.
