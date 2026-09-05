# Apollo x6 bring-up: device detection is resolving the wrong model

Report from a session bringing up an **Apollo x6** (Thunderbolt 3 Option Card) on
Linux. The driver loads and runs audio, but identifies the unit as an **Apollo x8p**.
Investigating that turned up several linked issues in the detection path, plus a
few smaller documentation/behaviour mismatches.

Everything below is measured on hardware or cited to `file:line`. Inferences are
marked as such.

## Test environment

| | |
|---|---|
| Host | Apple iMac18,3, Intel Alpine Ridge JHL6540 TB3 (`8086:15d2` NHI) |
| Kernel | 6.8.0-139-generic, GCC 13.3.0 |
| Device | Apollo x6 via UA Thunderbolt 3 Option Card |
| PCI | `44:00.0` `1a00:0002`, subsystem `1a00:0014` |
| Driver | `ua_apollo` v0.2.0, built from master at `53c86d0` |
| Boot | Dual-boot; macOS on the same machine runs UA `UAD2System` 11.9.0 |

The dual boot matters for reading the rest of this report: macOS had initialised the
device before the Linux boot, so every measurement below is on a **warm** unit (the
driver logs `mixer DSP alive! Skipping DMA reset to preserve state`). Cold behaviour
is unverified throughout.

The Thunderbolt device needed `boltctl enroll` before the PCIe tunnel appeared —
with `security=user` and nothing auto-authorizing, `lspci` showed no `1a00` device
at all and the tunnel bridges had empty bus ranges. Worth a troubleshooting note;
it presents identically to "driver doesn't detect my Apollo".

## Finding 1 — PCI subsystem `0x0014` is a platform ID, not a model ID

`ua_detect_capabilities()` pins `device_type = UA_DEV_APOLLO_X8P` whenever the
subsystem device ID is `0x0014` (`ua_core.c:214-221`), commented as "reliable,
constant per model" and short-circuiting the serial lookup entirely.

On this **x6**, every identifier is indistinguishable from the device the repo's
x8p support was captured from:

| Signal | This x6 (measured) | Repo's "x8p" |
|---|---|---|
| PCI subsystem | `0x0014` | `0x0014` (`ua_apollo.h:39`) |
| FPGA revision `0x2218` | `0xa241c5ac` | `0xa241c5ac` (`ua_apollo.h:39`) |
| DSP count (`ext_caps[15:8]`) | 6 | 6 (`ua_apollo.h:39`) |
| Name string `0xC034` | `"Apollo"` | — |
| Input IO descriptor | 34 entries | `ua_x8p_io_desc_input`, 34 |
| Output IO descriptor | 52 entries | `ua_x8p_io_desc_output`, 52 |

The IO descriptors are not merely the same length — they are **identical entry for
entry**, all 86, in the same order and groups. This x6 reports `MIC/LINE 3-4`,
`LINE 5-8` inputs and `LINE 1-8` outputs, i.e. channels its chassis has no
connectors for.

**Conclusion:** the FPGA image and DMA channel map are constant across this rack
platform; SKUs differ by analog population, not by anything visible over PCIe.
`0x0014` identifies the platform. No heuristic over these inputs can resolve the
model.

Practical consequence: an x6 is driven as an x8p — ring-connect path, x8p routing
tables, 34/32 channel counts, 8 preamp strips.

### The vendor's own driver names this unit an x6

Cross-checked from macOS on the same iMac (dual boot), with no Linux driver in the
path. `ioreg` shows the same node as the table above — `1a00:0002`, subsystem
`0x0014` — and UA's own stack (`com.uaudio.driver.UAD2System` 11.9.0 build 389)
names it. From `~/Library/Logs/Universal Audio/UAD Meter & Control Panel_0.log`,
six occurrences, most recently `2026-09-05 11:27:30` (path elided for width):

```
UADMeterApp::LoadFirmwareUpdate updatePathName .../FirmwareUpdateRene_2.bin
  unit 0 cardType Apollo x6 - HEXA requiresPowerCycle 1 isAudioDevice 1
```

`cardType` is a property of the enumerated unit, not part of the firmware filename,
and `HEXA` agrees with the 6 DSPs read from `ext_caps[15:8]` above.

**This settles the identification.** The model is not inferred from chassis
inspection alone: the vendor's own driver, reading this hardware behind subsystem
`0x0014`, calls it an Apollo x6. That falsifies the specific `0x0014` → x8p mapping
at `ua_core.c:214-221` by counterexample.

It does not by itself prove `0x0014` is a *platform* ID. That conclusion needs a
real x8p to report `0x0014` too — which `devices/apollo-x8p.json` asserts, on a unit
verified as an x8p by its 8 discrete preamps (issue #52), but which this session did
not measure. Open question 4 stands.

### The serial is readable — but only after connect

`BAR0+0x20..0x2F` reads **all `0xff`** on a cold device. `ua_detect_capabilities()`
runs immediately after the BAR ioremap with no initialisation in between, so
`ua_read_serial_type()` reads `0xff`, matches nothing, and sets `device_type = 0`
(`ua_core.c:295`).

Re-reading the same registers **after** ACEFACE connect (via the driver's READ_REG
ioctl) returns real data — the whole low register block populates:

```
0x0020 = "2048"  0x0024 = "2019"  0x0028 = "0113"  0x002c = "89"
serial     = 20482019011389
FW_VERSION = 0x03001c00   HW_UID_LO = 0x36094470
```

Critically, this serial **differs** from the reference device's (`2008 2017 00xxxx`)
even though subsystem ID, FPGA revision, DSP count and all 86 IO descriptor entries
are identical. The serial is the only observed signal that distinguishes units on
this platform.

**Detection is therefore fixable by reordering** — resolve the model after
`ua_audio_connect()` instead of at probe. Finding 2 covers the table problem that
remains once you can read it.

## Finding 2 — the serial lookup table has a hardware-verified wrong entry

The unit under test is physically an **Apollo x6**: 2 mic preamps, 6 analog I/O,
confirmed by inspection. Its serial digits 5-8 read `2019`.

`ua_serial_table` (`ua_core.c:250`) maps `"2019"` → `UA_DEV_APOLLO_X8P`, and carries
a separate `"2016"` → `UA_DEV_APOLLO_X6` entry. Nothing observed reads `2016`.

The digits-5-8 convention is sound: `devices/apollo-x4.json` records serial
`2019 2005 01xxxx`, digits 5-8 = `2005` → x4, matching working hardware. So the
extraction is correct and the **mapping** is wrong.

**`2019` is an Apollo x6.** This is hardware-verified and can anchor the table.

Consequences:

- The table was reconstructed from `_deviceTypeFromSerialNumber()` at kext offset
  `0x3E840`. With one entry confirmed wrong, neighbouring entries cannot be trusted
  without hardware confirmation — including `"2017"` → x8, which is precisely the
  entry `devices/apollo-x8p.json` reasons about when it calls the reference
  device's serial a "false match" and pins by subsystem ID instead. That
  justification rests on an unverified entry.
- Only `2005` (x4) and `2019` (x6) are hardware-confirmed today.

### The IO descriptor does not reflect analog population

An earlier reading of this data took the input descriptor's mic-capable groups
(`0x00` MIC/LINE/HIZ ch 1-2, `0x01` MIC/LINE ch 3-4) as evidence of a 4-preamp
device. **That is refuted.** This x6 has 2 physical mic preamps and reports those
same four channels, plus `LINE 5-8` for which it has no connectors.

The descriptor describes the platform's DMA fabric, not the SKU's analog fitment,
and cannot be used to derive preamp count.

Separately, `ua_models[]` carries `preamps = 8` for this device type
(`ua_audio.c:95`), so the ALSA card exposes 8 preamp strips on hardware with 2.

## Finding 3 — module parameters are inert on the ring-connect path

`no_plugins`, `warm_boot`, and `skip_bus_coeff` are all consumed inside
`ua_audio_init()` (`ua_audio.c:4209`), which probe calls at `ua_core.c:2801`.
But the DSP firmware load is at `ua_core.c:2437` and plugin-chain activation at
`ua_core.c:2521`, both inside `ua_dsp_init_and_load()` — called at `ua_core.c:2770`.

Everything those parameters are supposed to gate has already happened.

| Param | AudioExtension path | Ring-connect path |
|---|---|---|
| `no_plugins` | gates only a `dev_info()` at `ua_audio.c:1181`; activation already ran at `ua_audio.c:1027` | not referenced by `ua_core.c` at all — it is `static` in `ua_audio.c` |
| `warm_boot` | read at `ua_audio.c:4349` | runs after both the FW load and the plugin chain; cannot skip either, contrary to `"Skip FW load + ACEFACE"` |
| `skip_bus_coeff` | assigned at `ua_audio.c:4347`, before connect at `ua_audio.c:4372` — **works** | read by `ua_dsp.c:2515/2541` during the `2521` call, *before* assignment; `ua` is `devm_kzalloc` (`ua_core.c:2616`) so it is **`false`**, silently inverting the documented default of `true` |

So on ring-connect devices (types 32-38 per `UA_RING_CONNECT_BITMASK_HI`: x8p, x16,
x16D, Gen2 variants) there is no way to disable the plugin chain, and the
BUS_COEFF skip that defaults on is silently off.

`no_plugins`' description says "Skip plugin chain entirely." On the AudioExtension
path it skips nothing — the block at `ua_audio.c:1165-1185` comments "Plugin chain
DISABLED — it clobbers capture" and logs "plugin chain skipped", but activation
already happened ~150 lines earlier at `ua_audio.c:1027`. The comment and the log
line both assert the opposite of what the code does.

**Suggested fix:** plumb `no_plugins` through `struct ua_device` the way
`skip_bus_coeff` already is (`ua_apollo.h:799`), and assign both before
`ua_dsp_init_and_load()` rather than in `ua_audio_init()`.

## Finding 4 — stale comment; and the guard that is currently masking Finding 3

The plugin-chain blob is absent on this host and activation returned `-ENODATA`:

```
Direct firmware load for ua-apollo-plugin-chain.bin failed with error -2
plugin chain firmware not found; DMA_REF entries will be skipped
plugin chain: payloads not loaded, activation skipped
plugin chain activate failed: -61
```

This is **deliberate and correct**. `ua_dsp.c:2500-2509` explicitly refuses to send
the chain without payloads, reasoning that sending it with every DMA_REF skipped
would leave the DSP partially configured, and that returning an error keeps
`ua->plugins_activated = false` so a later reconnect retries once the blob is
installed.

The stale part is only the comment above `ua_dsp_load_plugin_payloads()`
(`ua_dsp.c:2278`), which still describes the pre-guard behaviour — "`-ENOENT` is a
soft failure: plugin-chain activation will still run but DMA_REF entries will be
skipped." It should be updated to match the guard.

**This matters for Finding 3.** That guard is the only thing preventing the ungated
plugin chain from running on ring-connect devices, and it disappears as soon as
`ua-apollo-plugin-chain.bin` is installed. Activation then proceeds at
`ua_core.c:2521` with `skip_bus_coeff = false` and no module parameter able to stop
it. Anyone on this platform who installs the blob to get PAD/48V/MicLine relays
working loses that incidental protection.

For the avoidance of doubt: no authentication or licensing is involved. `-61` is
`ENODATA` from the guard above, and the driver contains no auth, challenge/response
or licence logic (`grep -niE 'pace|ilok|licen|authent|authoriz|challenge|nonce|
signature|token|entitle' driver/` returns only `MODULE_LICENSE("GPL")`). The blob is
absent from the repo for IP reasons — captured proprietary DSP payload data,
gitignored, regenerated per user by `tools/build-plugin-chain-firmware.py` — not
because anything gates it at runtime.

## Finding 5 — stale metadata

- `ua_routing.h:518` warns that `ua_models[]` "pins the x8p at 26/26 with no cited
  source". `ua_audio.c:95` now reads `34, 32`. The warning outlived the problem.
- `devices/apollo-x8p.json` says *"No routing config for device_type 0x20 —
  `ua_dsp_send_routing()` returns `-ENODATA`"*. `ua_routing.h:1056` returns
  `&ua_x8p_routing_config`. Stale.
- Same file records `"channels": {"play": 26, "rec": 26}` against the driver's
  34/32. Nothing reads that field, but it is wrong.
- `README.md` lists x8p as "Needs Testing" while the descriptor claims `verified`.

## Finding 6 — clock source and rate are readable, not guesses

The notification bank carries a self-describing, backslash-delimited clock-source
table at `0xC090`:

```
[ 0] S/PDIF   [ 5] ADAT   [ 7] Word Clock   [10] Peer2Peer   [12] Internal
```

Index 12 is `Internal`, which is exactly the `clock_source = 0xC` hardcoded at
`ua_core.c:2419` with a comment attributing it to DTrace RE. It can be read from
the device.

Also readable **cold**, before any driver touches the device:

| Register | Value | Meaning |
|---|---|---|
| `CLOCK_CFG 0xC074` | `0x0000020c` | already set to internal @ 48k |
| `RATE_INFO 0xC07C` | `0x00000201` | |
| `CLOCK_INFO 0xC084` | `0x0000bb80` | 48000 Hz |

The `"features"` list in `devices/apollo-x8p.json` is annotated as a guess. The clock
table is a better source — it is published by the device and reports **Word Clock**,
which the current list omits.

**Caveat:** with only one unit measured, it is not established that this table varies
by SKU. Every other signal on this platform proved constant, so it may equally be a
platform constant. Treat it as better-sourced than a guess, not as confirmed
per-model capability, until a second SKU is compared.

## Finding 7 — what the plugin chain actually contains, and a rate mismatch

Parsing `ua_plugin_chain.h` (1317 entries) gives a clear picture of the payload.

Inline opcodes:

| Opcode | Count | Meaning |
|---|---|---|
| `0x001d0004` | 745 | BUS_COEFF |
| `0x001e0004` | 284 | ROUTING |
| `0x00080004` | 140 | SRAM_CFG |
| `0x001f0004` | 37 | SYNCH |
| `0x00030002` | 11 | MODULE_ACTIVATE |
| `0x001a0002` | 1 | WAKE |

The 99 DMA_REF entries carry **15,004 bytes total** (14.7 KiB). Largest single
payload 3,080 bytes; the bulk are 44 B (x38), 36 B (x23) and 16 B (x19).

MODULE_ACTIVATE params are `0xEB, 0xDA, 0x12B, 0xA5, 0xC2, 0xDB` — five of which
match the DSP "Bill" programs named at `ua_core.c:2466`, whose binaries load
separately via `ua_dsp_load_mixer_blocks()`.

**This means the chain is DSP mixer/routing initialisation, not plugin content.**
14.7 KiB cannot hold UAD plugin DSP code; a single plugin binary is far larger, and
plugin code would traverse this same DMA_REF path if any had been instantiated
during the capture. Useful for the project's licensing posture: the blob is
captured vendor *configuration*, so generating it is not gated on owning plugin
authorisations — worth stating plainly, since `install.sh` asks users to produce it
and the name "plugin chain" suggests otherwise. (UAD plugin entitlements are stored
on the device; the binaries come from the host installer. Neither appears here.)

### The capture was taken at 44.1 kHz

The single WAKE command is:

```
001a0002 0000ac44 00000000 00000000
```

`0xac44` = **44100**. The driver replays this chain unconditionally at whatever rate
it is running — 48 kHz on this host (`clock_info=0x0000bb80`, `rate_index=2`).

If any of the 745 BUS_COEFF entries or the larger coefficient tables (3080 B, 1948 B
x3, 736 B x3) are rate-dependent, they are being applied at the wrong sample rate.
This is a concrete candidate mechanism for the capture damage described at
`ua_audio.c:1165-1185` ("Plugin chain DISABLED — it clobbers capture"), and it
requires no licensing explanation.

**Suggested investigation:** re-capture at 48 kHz and diff the BUS_COEFF payloads
against the existing 44.1 kHz capture. If they differ, the chain needs to be
rate-matched (or regenerated per rate) rather than replayed verbatim.

## What works on the x6 — audio confirmed both directions

- ALSA card registers; 34 playback / 32 record, S32_LE @ 48 kHz
- Transport starts; `SAMPLE_POS` and `FRAME_CTR` advance; period IRQs steady ~21 ms
- Capture DMA live: `REC canary check: 0/1048576 dwords still 0xDEDEDEDE`
- 6 DSPs detected, rings programmed for all six (DSP 4/5 at `BAR0+0x6000/0x6080`)
- No oops, no PCIe AER, stable over ~40 minutes

**Playback: audible.** With no configs deployed at all (`configs/deploy.sh` never run),
PipeWire auto-selected the raw node as default sink and mapped stereo FL/FR onto
`playback_AUX0`/`AUX1` — which the output IO descriptor identifies as MON L/R. System
audio reaches the monitor outputs. `Monitor Playback Switch` on, `Monitor Playback
Volume` 192/192.

**Capture: carrying signal.** A -12 dBFS 440 Hz tone played to the monitor outs while
recording all 32 capture channels appeared on capture `AUX26`/`AUX27` at -8.2 dBFS
peak, 99.9% non-zero; the other 30 channels were exact digital zero.

Channel identity for that pair is unresolved: the input descriptor holds 34 entries
against 32 ALSA channels, so two are dropped. If the dropped pair is S/PDIF
(`dma[16..17]`), as `ua_routing.h` states, then AUX26/27 is AUX1 L/R; under a straight
truncation it is MON L/R.

Channel counts reconcile against the descriptors: input 34 - S/PDIF(2) = **32 record**;
output 52 - S/PDIF(2) - 16 unnamed group `0x0c` entries = **34 playback**.

### Mic preamp inputs are silent, and the routing path explains why

Capture `AUX0`/`AUX1` (the two mic preamps) read **exact digital zero**, not self-noise
— the reference device reportedly shows -102/-106 dBFS there. Digital zero indicates
nothing routed, rather than nothing plugged in.

Two contributing facts:

1. `ua_dsp_send_routing()` is reachable **only** via `UA_IOCTL_SEND_ROUTING`
   (`ua_core.c:2134-2141`). Declared at `ua_apollo.h:989`, defined at
   `ua_dsp.c:3165`, and `ua_core.c:2138` is its **sole call site in the tree** —
   `grep -rn ua_dsp_send_routing driver/` returns those three plus one comment
   (`ua_routing.h:522`) and nothing else. So nothing reaches it at probe or connect,
   and a plain `insmod` never sends a routing table. However, this is not the cause:
   the descriptors it would write are byte-identical to what the hardware already
   reports, so sending them is a content no-op.
2. The plugin chain carries **284 ROUTING and 745 BUS_COEFF** commands (Finding 7) and
   never ran, because the firmware blob is absent and the guard at `ua_dsp.c:2500`
   correctly refuses to send a partial chain. Without those, the DSP mixer has no input
   routing configured.

That makes the missing plugin chain the leading candidate for silent preamp capture.
The playback path works regardless because this unit was warm — macOS had already
configured it (Finding: `mixer DSP alive! Skipping DMA reset to preserve state`).

Caveats: tested WARM; cold behaviour unverified. 48 kHz only. Probe logs `audio
extension connect timeout` / `-110`, adding ~21 s to `insmod`, though ACEFACE then
succeeds on stream open.

## Open questions

1. What model is serial prefix `2017` (the reference device)? Its platform
   identifiers are identical to this x6's, so it is a sibling SKU on the same board,
   but the table cannot be trusted to name it.
2. Which other serial-table entries are wrong? Only `2005` and `2019` are confirmed
   against hardware.
3. What is output group `0x0c`, indices 1-16? Unnamed in both the descriptor and the
   device's own routing table.
4. Does a real Apollo x8p also report subsystem `0x0014` and FPGA `0xa241c5ac`?

## Suggested changes

1. Rename `UA_SUBSYS_APOLLO_X8P` to a platform constant; stop mapping it to a
   single model.
2. Move model resolution after connect — the serial is populated there (Finding 1);
   keep the subsystem ID as a family hint only.
2b. Correct `ua_serial_table`: `2019` is x6, verified. Re-verify the rest against
   hardware before relying on it.
3. Correct the x8p-labelled assets to x8, or to platform-neutral names.
4. Fix `preamps` per SKU (an x6 has 2, not 8). Note it cannot be derived from the
   input IO descriptor — that is a platform constant and overstates the count.
5. Plumb `no_plugins`/`skip_bus_coeff` so they apply before `ua_dsp_init_and_load()`.
6. Update the stale `-ENOENT` comment at `ua_dsp.c:2278` to match the guard at
   `ua_dsp.c:2500`, and remove the contradictory "DISABLED" block at
   `ua_audio.c:1165-1185`.
7. Read clock source and features from `0xC090` instead of hardcoding/guessing.
8. Clear the stale comments and descriptor fields listed in Finding 5.

### Correcting detection is not safe on its own

Changes 1, 2 and 2b move this unit from `device_type` `0x20` to `0x1E`. Nothing in
the tree is ready for `0x1E`: the unit works today only because misdetection lands
it on x8p-shaped defaults that happen to fit. Four sites must change together.

| # | Site | Today (as x8p `0x20`) | After the fix (x6 `0x1E`) |
|---|---|---|---|
| 1 | `ua_audio.c:90` `ua_models[]` | 34 play / 32 rec | **24 / 22** |
| 2 | same row | 8 preamps | **4** (chassis has 2) |
| 3 | `ua_routing.h:1051` `ua_get_routing_config()` | `&ua_x8p_routing_config` | **`NULL`** |
| 4 | `mixer-engine/device_maps/` | `device_map_apollo_x8p.json` | **none — falls back to x4** |

1. **Channel counts.** `ua_audio.c:117-122` programs `play_channels`/`rec_channels`
   straight from `ua_models[]`. Record drops 32 → 22, so capture `AUX26/AUX27` — the
   loopback pair verified at −8.2 dBFS above — ceases to exist. That invalidates the
   verified capture path and silently drops 10 record channels; playback on
   `AUX0/AUX1` (MON L/R) likely survives, since `d2d10bb` describes the old 26/26 as
   *hiding* channels rather than failing. The x6 row's `24, 22` cites nothing. This
   unit measures 34/32 and `d2d10bb` measured 34/32 on an independent x8p, so the
   count looks platform-wide — consistent with Finding 1.
2. **Preamp count** — suggested change 4 above. `4` is as wrong as `8` for this unit.
3. **Routing config.** `ua_get_routing_config()` has cases for X4 and X8P only; X6
   falls to `default: return NULL` (`ua_routing.h:1058`). Latent rather than
   immediate — no routing table is sent today anyway, since `ua_dsp_send_routing()`
   is reachable only via `UA_IOCTL_SEND_ROUTING` (`ua_core.c:2134-2141`) — but the
   config would go from available to absent, closing off the fix for that path.
4. **Device map.** `lookup_device(0x1E)` returns `("Apollo x6", None)`, so
   `find_device_map()` logs *"falling back to Apollo x4"* and loads the 4-preamp x4
   map (`ua_mixer_daemon.py:1515-1535`). The control surface moves from 8 strips to
   4; neither fits this chassis.

Landing the detection fix by itself regresses a device that currently passes audio
in both directions.

## Build warnings (pre-existing, GCC 13.3.0, kernel 6.8.0-139)

```
ua_dsp.c:3246   format '%u' expects unsigned int, argument has type long unsigned int
ua_core.c:2266  frame size of 1080 bytes is larger than 1024 bytes
ua_core.c:2684  unused variable 'v'
ua_audio.c:1303 unused variable 'acr'
```
