# qcom-firmware-compat

Names, not files. Every entry here points a Snapdragon laptop's own name at a
file two upstreams already ship for identical hardware, so that a machine
upstream has not been told about gets the sound it already has drivers for.

Nothing vendor-signed and nothing new is shipped: the topology entries are
symlinks into `linux-firmware`, and the UCM entries include a card
configuration `alsa-ucm-conf` already carries. Firmware that cannot be
redistributed is copied from the machine's own Windows installation by
`qcom-firmware-extract` instead.

## Why a laptop needs this

Two names decide whether a Snapdragon laptop has audio:

- **The topology.** The ASoC driver asks for `qcom/<soc>/<model>-tplg.bin`,
  where `<model>` comes from the device tree's `sound` node. If linux-firmware
  has never heard the model name the load fails with `-2` and **no ALSA card
  registers at all** — `aplay -l` prints "no soundcards found", and every
  question about mixers and routing is unanswerable until it is fixed.
- **The card configuration.** `alsa-ucm-conf` picks one by the machine's DMI
  strings, and looks for `conf.d/<card driver>/<card long name>.conf` before
  falling back to the driver's own file. Without a match PipeWire sees a card
  with no speaker or headset path.

Both are per-machine names over hardware that is not per-machine: the X1E80100
laptops share a handful of audio designs, and upstream already ships one blob
under four names and one card configuration for five machines.

## What is here

| Machine | Entry | Points at | Retires when |
|---|---|---|---|
| HP EliteBook Ultra G1q | `/usr/lib/firmware/updates/qcom/x1e80100/X1E80100-HP-ELITEBOOK-ULTRA-G1Q-tplg.bin` | `qcom/x1e80100/X1E80100-Romulus-tplg.bin`, the blob the OmniBook X14, ThinkPad T14s and Zenbook A14 links already resolve to | linux-firmware adds a `Link:` line for the G1q |
| HP EliteBook Ultra G1q | `/usr/share/alsa/ucm2/conf.d/x1e80100/<card long name>.conf` | `/Qualcomm/x1e80100/LENOVO-T14s.conf`, the configuration its baseboard sibling the OmniBook X14 already uses | alsa-ucm-conf widens the x1e80100 DMI regex past `HP.*Omnibook X` |

The topology entry lives under `updates/` so that the day linux-firmware ships
the name, its file wins the firmware search order and this package can never
collide with it in pacman.

## Retirement

Per row, as each upstream lands its line — both are one-line changes, and
alsa-ucm-conf's maintainer has asked for the regex widening in writing (PR
#531). When the last row goes, drop the package.
