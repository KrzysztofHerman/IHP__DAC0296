# Physical Verification Workflow (Step-by-Step)

This document explains the GitHub Actions workflow defined in `.github/workflows/physical_verification.yml` in detailed, engineering-focused terms. The workflow performs automated physical verification using the IHP Open PDK rule decks and KLayout.

## Overview

The workflow executes two verification stages:

1. **DRC (Design Rule Check)** using the official IHP SG13G2 KLayout DRC deck.
2. **LVS (Layout vs. Schematic)** using the official IHP SG13G2 KLayout LVS deck, but only when a corresponding netlist is available.

All intermediate data is stored under `/tmp` to avoid polluting the repository workspace.

## Step-by-Step Execution

### 1) Repository Checkout

The workflow checks out the repository so it can access:

- `doc/info.json` for release metadata
- The design data under `release/`

This ensures the workflow always runs against the exact version of the repository that triggered the workflow.

### 2) System Dependencies Installation

The workflow installs minimal system packages required by the subsequent steps:

- `jq` for robust JSON parsing
- `python3-pip` for installing Python dependencies

These are installed using `apt-get` on the GitHub-hosted Ubuntu runner.

### 3) KLayout Download (Ubuntu-Matched)

The workflow determines the runner's Ubuntu major version (e.g., 22, 24) using `lsb_release` and then selects a matching KLayout `.deb` URL directly from `https://www.klayout.de/build.html`. This avoids installing a `.deb` built for a newer Ubuntu release, which would fail due to incompatible libc and toolchain dependencies.

Key safety checks performed:

- If no matching URL is found, the workflow terminates early with a clear error.
- The downloaded file is validated with `dpkg-deb --info` to ensure it is a legitimate Debian package.

### 4) KLayout Installation

The workflow installs KLayout using:

1. `dpkg -i` to unpack the package
2. `apt-get -f -y` to resolve and install any missing dependencies

Finally, `klayout -v` is executed to confirm that the executable is available and functional.

### 5) Python Requirements (IHP Open PDK)

The workflow installs the Python requirements published by the IHP Open PDK:

```
https://raw.githubusercontent.com/IHP-GmbH/IHP-Open-PDK/main/requirements.txt
```

These dependencies support the execution of `run_drc.py` and `run_lvs.py` (e.g., `docopt`, `klayout`, `pyyaml`, `gdstk`).

### 6) Fetch Official DRC/LVS Decks

The workflow clones the official IHP Open PDK repository into `/tmp` using a sparse checkout. Only the required subtrees are downloaded:

- `ihp-sg13g2/libs.tech/klayout/tech/drc/`
- `ihp-sg13g2/libs.tech/klayout/tech/lvs/`
- `ihp-sg13g2/libs.tech/klayout/python/sg13g2_pycell_lib/sg13g2_tech_mod.json`

This keeps the checkout lightweight and deterministic while ensuring the workflow always uses upstream deck versions.

### 7) DRC Execution (Always)

The workflow reads the GDS path from `doc/info.json` (`release.gds`). If the field is missing or empty, the workflow stops with an error.

The DRC run is executed as:

```
python3 /tmp/ihp-open-pdk/ihp-sg13g2/libs.tech/klayout/tech/drc/run_drc.py \
  --path="<gds-path>" \
  --run_dir=/tmp/drc-results \
  --mp=1 \
  --precheck_drc
```

Notes:

- `--precheck_drc` enables the deck’s pre-check phase.
- Results are written to `/tmp/drc-results`.

### 8) LVS Execution (Conditional)

The workflow reads the netlist path from `doc/info.json` (`release.netlist`).

- If `release.netlist` is missing or `null`, LVS is skipped.
- If the netlist path is provided but the file does not exist, LVS is skipped with a clear message.

If the netlist is present, LVS is executed as:

```
python3 /tmp/ihp-open-pdk/ihp-sg13g2/libs.tech/klayout/tech/lvs/run_lvs.py \
  --layout="<gds-path>" \
  --netlist="<netlist-path>" \
  --run_dir=/tmp/lvs-results
```

### 9) Artifact Upload

The workflow always uploads both result directories (even if a stage failed), which simplifies post-mortem debugging:

- `DRC-Results` from `/tmp/drc-results`
- `LVS-Results` from `/tmp/lvs-results`

## Failure Modes and Diagnostics

- **KLayout URL not found**: The workflow exits early if `build.html` does not contain a matching Ubuntu `.deb` link.
- **Wrong Ubuntu `.deb`**: Detected by dependency errors; the URL must match the runner’s Ubuntu major version.
- **Missing `release.gds`**: DRC step fails immediately with a clear error.
- **Missing `release.netlist`**: LVS is skipped intentionally and logged as such.

## Design Rationale

- **No repo contamination**: All runtime files are stored in `/tmp`.
- **Upstream rule decks**: Always pulls the official IHP Open PDK DRC/LVS decks to avoid drift.
- **Deterministic selection**: KLayout `.deb` is selected explicitly based on the runner OS version.

If you need changes to the execution flags (e.g., different LVS modes or extra DRC options), update `.github/workflows/physical_verification.yml` accordingly.
