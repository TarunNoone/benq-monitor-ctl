# benq-monitor-ctl

Linux command-line control for the **BenQ Mobiuz EX2510S** (and likely sibling Mobiuz models) over **DDC/CI** — no OSD button mashing required.

## ⚠️ Disclaimer

**This project was made by an AI.** It was developed end-to-end by the [OpenCode](https://opencode.ai) coding agent running the **GLM-5.3-Flash** model, with a human providing direction and verifying every hardware write on the physical monitor. Treat it accordingly: review the code, test carefully, and use at your own risk. No warranty.

## What it controls

| Control | VCP code | Range | Notes |
|---|---|---|---|
| Picture mode (all 10) | `0xDC` | Custom, sRGB, M-Book, FPS, Racing Game, RPG, Game HDRi, Cinema HDRi, ePaper, HDR | BenQ-private enum, reverse-engineered |
| Low Blue Light filter | `0x19` | 0–20 | BenQ-private; **write-clamped in some modes** (e.g. max 12 in M-Book) |
| Color temperature | `0x14` | 5000K / 6500K / 9300K / User 1 | Only honored in **Custom** mode |
| Black Equalizer | `0x7A` | 0–10 | BenQ-private |
| Color Vibrance | `0x8A` | 0–20 | BenQ-private |
| Brightness | `0x10` | 0–100 | |
| Contrast | `0x12` | 0–100 | |
| Sharpness | `0x87` | 0–10 | |
| Volume / Mute | `0x62` / `0x8D` | | |
| Input source | `0x60` | DP / HDMI-1 / HDMI-2 | ⚠ see quirks below |
| Power | `0xD6` | on / off (DPM) | |
| OSD lock | `0xCA` | | write ignored by this firmware; read-only status |
| Factory reset | `0x04` | | |

Read-only info also exposed: current refresh rate (`0xAE`), usage hours (`0xC0`), controller/firmware (`0xC8`/`0xC9`), whether the OSD menu is currently open (`0xCA`).

**Confirmed NOT exposed over DDC/CI on this monitor:** B.I.+ (Bright Intelligence) sensor behavior, HDR tone-mapping internals, and the "Standard" mode name (the mode BenQ's spec calls Custom is `0x00`).

## Install

Requirements: Linux, [ddcutil](https://www.ddcutil.com/) ≥ 2.x, a DDC/CI-capable connection (direct HDMI/DP; many docks block DDC).

```bash
# 1. the CLI
sudo cp monitor-ctl /usr/local/bin/

# 2. the feature-definition file that names BenQ's private VCP codes
mkdir -p ~/.local/share/ddcutil
cp udf/BNQ-BenQ_EX2510S-32675.mccs ~/.local/share/ddcutil/

# 3. i2c permissions (otherwise run everything with sudo)
#    see https://www.ddcutil.com/i2c_permissions/
```

Verify with `ddcutil detect`, then `monitor-ctl list`.

## Usage

```bash
monitor-ctl list                 # picture modes
monitor-ctl get                  # current picture mode
monitor-ctl set fps              # switch picture mode
monitor-ctl bluelight 20         # low blue light filter (0-20)
monitor-ctl temp warm            # 5000K (Custom mode only)
monitor-ctl blackeq 7            # black equalizer (0-10)
monitor-ctl vibrance 13          # color vibrance (0-20)
monitor-ctl brightness 40        # get or set
monitor-ctl input dp             # dp | hdmi1 | hdmi2
monitor-ctl power off
monitor-ctl save night.cal       # dump color settings
monitor-ctl load night.cal       # restore them
monitor-ctl raw 0x87 3           # escape hatch

MONITOR="-d 2" monitor-ctl get   # select a different ddcutil display
```

## Known quirks (EX2510S, firmware 0.1)

- **Settings are stored per picture mode.** Brightness, black EQ, vibrance, etc. are saved separately for each of the 10 modes; writing a value only affects the *current* mode. The Low Blue Light filter is global, but some modes clamp how high it can go.
- **The input source readout can lie.** `0x60` reported HDMI-2 while the cable was physically in HDMI-1. Verify port mappings by hand before scripting input switches.
- **Mode switches take ~1–2 s** and DDC reads during the transition come back malformed; the script retries.
- **Writing undeclared codes requires the UDF file.** ddcutil refuses `setvcp` for codes absent from the monitor's capabilities string — the shipped `.mccs` file declares the four BenQ-private ones.

## How the private codes were found

1. `ddcutil probe` — full sweep of all 256 VCP codes (84 respond on this unit, 22 in the vendor-reserved range).
2. Capabilities string parsing — what the firmware *claims* (unreliable but useful).
3. OSD-diffing — read all candidate codes, change a setting via the monitor's buttons, diff.
4. **Value fingerprinting** — set the OSD value to something distinctive (e.g. blue light = 17 → `0x11`), full-scan, and grep for the byte. This found `0x19` after diffing alone had missed it.
5. Write-and-verify — every mapping was confirmed bidirectionally against the OSD display.

Negative results were established the same way (full-sweep + commit-on-close re-check).

## Compatibility

Developed and verified on a **BenQ EX2510S** (RealTek scaler, MCCS 2.2). Other Mobiuz/gaming BenQ models likely share some of these codes but nothing is guaranteed — run `ddcutil probe` and the methodology above before trusting any mapping. PRs with per-model `.mccs` files welcome.

## License

[MIT](LICENSE)
