# Install

This guide walks through a full local setup for building, flashing, provisioning, and monitoring the `Poofer_v2` firmware.

## Prerequisites

- macOS, Linux, or WSL2
- Python 3.11+
- ESP-IDF 5.5.x (this repo uses 5.5.2)
- USB-C data cable
- The ESP32-C3 Super Mini board
- A 12V supply for the solenoid driver PCB
- An existing dedicated Wi-Fi network for the poofer system to join

`Poofer_v2` does not use a self-hosted control AP as its primary deployment model. Bring an external Wi-Fi AP/router and put the browser plus both poofer nodes on that same network.

Optional but recommended development tools:

- `clang-format`
- `clang-tidy`
- `pre-commit`

## Install ESP-IDF

Follow Espressif's official install guide for ESP-IDF 5.5.x.

Make sure `idf.py` is available on your PATH. You should be able to run:

```bash
idf.py --version
```

Set your ESP-IDF path once per shell:

```bash
export ESP_IDF_PATH=/path/to/esp-idf
```

If `idf.py` is not on your PATH in a new shell session, source the ESP-IDF environment:

```bash
source "$ESP_IDF_PATH/export.sh"
```

You can also set local defaults by copying `.env.example` to `.env` and editing it.

## Project Setup

```bash
cd /path/to/Poofer_v2
```

Optional but recommended: install git hooks for local linting.

```bash
python3 -m pip install --user pre-commit
pre-commit install
```

If you want linting to run on push as well:

```bash
pre-commit install --hook-type pre-push
```

## Build

```bash
python3 scripts/build.py
```

## Flash

```bash
python3 scripts/flash.py --port /dev/cu.usbmodemXXXX
```

Flash each poofer node separately.

## Monitor

```bash
python3 scripts/monitor.py --port /dev/cu.usbmodemXXXX
```

Monitor each node separately while provisioning or debugging.

## Lint

```bash
scripts/lint.sh
```

Notes:

- `clang-tidy` requires `firmware/build/compile_commands.json`.
- You can generate it with an IDF build. If it is missing, the lint script will skip `clang-tidy`.

## Wi-Fi And UI

`Poofer_v2` expects an existing external network.

Deployment model:

- Put the browser control device on a dedicated Wi-Fi network.
- Put the `stage-left` and `stage-right` poofer nodes on that same network.
- Discover the nodes over `mDNS`.
- Use `Poofer ID Setup` to assign or verify roles.

There is no requirement in v2 that a poofer node host the operator network.

The old AP-hosted single-node flow should be treated as transitional legacy behavior while the multi-node design is being implemented.
