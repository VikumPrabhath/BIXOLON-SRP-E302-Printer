# BIXOLON SRP-E302 CUPS Installer for Linux

![Shell](https://img.shields.io/badge/shell-bash-4EAA25?logo=gnubash&logoColor=white)
![Platform](https://img.shields.io/badge/platform-Ubuntu%20%7C%20Debian-E95420?logo=ubuntu&logoColor=white)
![CUPS](https://img.shields.io/badge/CUPS-2.4.x-blue)
![License](https://img.shields.io/badge/license-MIT-green)

One-command automated installer for the **BIXOLON POS CUPS driver** and the **BIXOLON SRP-E302** thermal receipt printer (USB) on Ubuntu and Debian-based systems.

The script installs dependencies, extracts the official BIXOLON driver package, registers the printer queue with the correct driver, verifies the configuration and prints a test receipt, with clear step-by-step terminal output and a full log file.

---

## Table of Contents

- [Why this project exists](#why-this-project-exists)
- [Features](#features)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Usage](#usage)
- [What the script does](#what-the-script-does)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Using the printer from applications (ESC/POS)](#using-the-printer-from-applications-escpos)
- [Uninstall](#uninstall)
- [Repository layout](#repository-layout)
- [Security considerations](#security-considerations)
- [Contributing](#contributing)
- [License and disclaimer](#license-and-disclaimer)

---

## Why this project exists

When a USB receipt printer is added through a generic desktop printer wizard, CUPS may pick the wrong driver (for example an HP PostScript driver). The printer then receives PostScript/PJL commands it cannot interpret and prints raw code such as:

```
@PJL ENTER LANGUAGE=POSTSCRIPT
%!PS-Adobe-3.0
```

This project automates the correct setup using BIXOLON's official CUPS driver so that normal text and documents print properly.

## Features

- Fully automated, single-script installation
- Auto-detects the driver package in `~/Downloads`, or accepts `--file` / `--url`
- Installs required packages (`cups`, `libcups2-dev`, `unzip`, `curl`)
- Handles the nested `.zip` -> `.tgz` package structure
- Auto-detects the USB device URI
- Removes stale or wrongly configured queues and recreates them cleanly
- Verifies that the queue uses the BIXOLON driver (not HP)
- Prints a test receipt and waits for the job to complete
- Colored, step-numbered output (`[n/10]`) and a persistent log file
- Idempotent: safe to run multiple times
- Fails fast with actionable error messages (`set -Eeuo pipefail` and an error trap)

## Requirements

### Hardware

| Item | Details |
|---|---|
| Printer | BIXOLON SRP-E302 (other BIXOLON models can work via `--model`, see below) |
| Connection | USB cable, printer powered on, paper loaded, cover closed |

### Software

| Item | Details |
|---|---|
| OS | Ubuntu 20.04 / 22.04 / 24.04 or another Debian-based distribution |
| Architecture | x86_64 (amd64). The BIXOLON package also ships ARM and Raspberry Pi filters |
| Privileges | `sudo` / root access |
| Network | Internet access for `apt` (and for `--url` if used) |
| Driver package | BIXOLON POS Linux CUPS Driver (e.g. `Software_Linux_POS_CUPS_DriverV1.5.9.zip`) |

> **Tested on:** Ubuntu 24.04 LTS (x86_64), CUPS 2.4.7, BIXOLON SRP-E302 over USB, driver package v1.5.9.
> Other distributions and models are expected to work but have not been verified.

### Driver package

The BIXOLON driver is proprietary and is **not included in this repository**. Download it from the official BIXOLON support/download page for your printer model and place the `.zip` in `~/Downloads`.

## Quick start

```bash
# 1. Clone the repository
git clone https://github.com/VikumPrabhath/BIXOLON-SRP-E302-Printer
cd BIXOLON-SRP-E302-Printer

# 2. Put the BIXOLON driver zip in ~/Downloads, then run:
chmod +x install-bixolon-srp-e302.sh
sudo ./install-bixolon-srp-e302.sh
```

At the end of a successful run, a test receipt is printed and a summary is shown.

## Usage

```
sudo ./install-bixolon-srp-e302.sh [options]
```

| Option | Description | Default |
|---|---|---|
| `-f, --file PATH` | Local driver package (`.zip` or `.tgz`) | Auto-search in `~/Downloads`, current directory |
| `-u, --url URL` | Download the driver package from a URL | none |
| `-n, --name NAME` | Printer queue name | `SRP-E302` |
| `-m, --model PPD` | PPD name known to CUPS | `Bixolon/SRPE302_v1.0.3.ppd` |
| `-d, --uri URI` | Device URI | Auto-detected USB URI |
| `--no-test` | Skip the test print | off |
| `--skip-deps` | Skip `apt` dependency installation | off |
| `-h, --help` | Show help | |

### Examples

```bash
# Auto-detect everything
sudo ./install-bixolon-srp-e302.sh

# Specify the driver package explicitly
sudo ./install-bixolon-srp-e302.sh --file ~/Downloads/Software_Linux_POS_CUPS_DriverV1.5.9.zip

# Download the package from your own hosting
sudo ./install-bixolon-srp-e302.sh --url "https://example.com/Software_Linux_POS_CUPS_DriverV1.5.9.zip"

# Custom queue name and manual USB URI, no test print
sudo ./install-bixolon-srp-e302.sh --name RECEIPT --uri "usb://BIXOLON/SRP-E302?serial=00000001" --no-test
```

### Using a different BIXOLON model

List the models provided by the installed driver and pass the matching PPD:

```bash
lpinfo -m | grep -i bixolon
sudo ./install-bixolon-srp-e302.sh --name SRP-350 --model "Bixolon/SRP350plusIII_v1.0.7.ppd"
```

> Pick the exact model. Similar names (for example `SRP-E302` vs `SRP-QE302`) are different printers.

## What the script does

| Step | Action |
|---|---|
| 1 | Pre-flight checks: root, OS, CPU architecture |
| 2 | Installs missing dependencies (`cups`, `libcups2-dev`, `unzip`, `curl`) |
| 3 | Locates the driver package or downloads it from `--url` |
| 4 | Extracts the zip and the inner `.tgz` into a temporary directory |
| 5 | Runs BIXOLON's official `setup_v*.sh` installer |
| 6 | Verifies the `rastertoBixolon` filter and PPD files, restarts CUPS |
| 7 | Detects the USB printer URI |
| 8 | Creates the CUPS queue, enables it, accepts jobs and sets it as default |
| 9 | Verifies the driver is BIXOLON and the printer is idle |
| 10 | Prints a test receipt and waits for the job to finish |

Temporary files are removed automatically on exit.

**Files and locations touched**

| Path | Purpose |
|---|---|
| `/usr/lib/cups/filter/rastertoBixolon` | Filter symlink installed by BIXOLON's setup |
| `/usr/bin/rastertoBixolon_v*_x64` | Filter binary |
| `/usr/share/cups/model/Bixolon/` | BIXOLON PPD files |
| `/etc/cups/printers.conf`, `/etc/cups/ppd/` | CUPS queue configuration (managed by CUPS) |
| `/var/log/bixolon-srp-e302-setup.log` | Installer log |

## Verification

```bash
# Queue state: should say "idle. enabled"
lpstat -p SRP-E302

# Device URI: should be usb://BIXOLON/SRP-E302?...
lpstat -v SRP-E302

# Driver: must say Bixolon, NOT HP
lpoptions -p SRP-E302 | grep -o "printer-make-and-model='[^']*'"

# Manual test print
echo "Hello from SRP-E302 - $(date)" | lp -d SRP-E302
```

You can also open the CUPS web interface at <http://localhost:631/printers/SRP-E302> and use **Maintenance -> Print Test Page**.

## Troubleshooting

> **Tip:** the CUPS error log contains thousands of harmless `Bad driver information file` lines. Filter them out:
>
> ```bash
> sudo grep -v cups-driverd /var/log/cups/error_log | tail -n 40
> ```

### Quick diagnosis

```bash
lpstat -t                         # overall status
lsusb | grep -i -E "bixolon|1504" # is the printer visible on USB?
lpinfo -v | grep -i usb           # does CUPS see the USB device?
sudo tail -n 50 /var/log/bixolon-srp-e302-setup.log
```

### Problems and fixes

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | Printer prints `@PJL ... %!PS-Adobe ...` text | Queue uses an HP/PostScript driver | Run this script again; it recreates the queue with the BIXOLON driver. Confirm with `lpoptions -p SRP-E302 \| grep make-and-model` |
| 2 | `cups-config: not found`, `CUPS VERSION:` is empty | `libcups2-dev` is missing | `sudo apt install libcups2-dev`, then re-run the script |
| 3 | `No USB printer detected` / no `usb://` entry in `lpinfo -v` | Cable, port or power problem, or printer is off | Power-cycle the printer, replug USB, try another port/cable, check `lsusb`, restart CUPS: `sudo systemctl restart cups` |
| 4 | Queue shows `paused` / `disabled` | CUPS stopped the queue after an error | `sudo cupsenable SRP-E302 && sudo cupsaccept SRP-E302`, then read the filtered log |
| 5 | `Filter failed` in the log | Filter not executable or wrong architecture | `sudo chmod 755 "$(readlink -f /usr/lib/cups/filter/rastertoBixolon)"`; confirm `uname -m` matches the package |
| 6 | `Permission denied` / `Unable to open device` | User not in `lp` group, or USB access blocked | `sudo usermod -aG lp,lpadmin $USER`, then log out and back in |
| 7 | Job disappears but nothing prints | Paper out, cover open, or wrong media size | Check paper and cover; run the printer self-test (hold **FEED** while powering on) |
| 8 | Blank paper or wrong size | Incorrect media or page type | In CUPS web UI: *Administration -> Set Default Options*: 80 mm media, **Variable Length**, **Partial Cut** |
| 9 | `lpstat: Invalid destination name` / `The printer or class does not exist` | Queue not created (driver install does not create a queue) | Run the script or `sudo lpadmin -p SRP-E302 -E -v "<URI>" -m Bixolon/SRPE302_v1.0.3.ppd` |
| 10 | `No driver package found` | Zip not in a searched folder | Use `--file /full/path/to/driver.zip` |
| 11 | `ls: cannot access '.../rastertobixolon'` | Linux is case-sensitive | The filter is named `rastertoBixolon` (capital **B**) |
| 12 | `unzip: filename not matched` | Destination passed without `-d` | Use `unzip file.zip -d ~/target-folder` |
| 13 | `tar: invalid option -- 'y'` | Wrong tar flag | Use `tar xvf file.tgz` |
| 14 | `Bad driver information file ...` in the error log | Harmless CUPS driver-scan noise | Ignore it; filter with `grep -v cups-driverd` |
| 15 | Printing works from CLI but not from my app | App sends data through the wrong queue or format | See [Using the printer from applications](#using-the-printer-from-applications-escpos) |

### Isolating hardware vs. software

1. **Printer self-test:** power off, hold **FEED**, power on. If it does not print, the problem is paper, cover or hardware.
2. **Raw test** (bypasses the BIXOLON driver):

   ```bash
   sudo lpadmin -p SRP-E302-RAW -E -v "usb://BIXOLON/SRP-E302?serial=00000001" -m raw
   printf 'RAW TEST\n\n\n\n\x1d\x56\x00' | lp -d SRP-E302-RAW -o raw
   ```

   - Raw prints, driver queue does not: check media settings and the BIXOLON filter.
   - Neither prints: check USB, power and paper.

### Collecting information for a bug report

```bash
{
  echo "== OS ==";       cat /etc/os-release | head -3; uname -m
  echo "== CUPS ==";     cups-config --version
  echo "== lpstat ==";   lpstat -t
  echo "== USB ==";      lsusb | grep -i -E "bixolon|1504"
  echo "== lpinfo ==";   lpinfo -v | grep -i usb
  echo "== Log ==";      sudo grep -v cups-driverd /var/log/cups/error_log | tail -n 40
} > bixolon-diagnostics.txt 2>&1
```

Attach `bixolon-diagnostics.txt` and `/var/log/bixolon-srp-e302-setup.log` when opening an issue. Review them for sensitive information first.

## Using the printer from applications (ESC/POS)

The BIXOLON CUPS driver converts pages to raster images. This is ideal for documents, but POS applications (Java, Python, Node, and others) that send **ESC/POS commands directly** (cash drawer, cutter, barcodes) should use a **raw queue**:

```bash
sudo lpadmin -p SRP-E302-RAW -E -v "usb://BIXOLON/SRP-E302?serial=00000001" -m raw
lp -d SRP-E302-RAW -o raw receipt.bin
```

| Use case | Queue |
|---|---|
| Documents, text files, CUPS-aware apps | `SRP-E302` (BIXOLON driver) |
| ESC/POS byte streams, cash drawer, auto-cut | `SRP-E302-RAW` (raw) |

> Both queues share one USB device. Avoid printing to both at the same moment.

## Uninstall

```bash
# Remove the printer queue(s)
sudo lpadmin -x SRP-E302
sudo lpadmin -x SRP-E302-RAW   # if created

# Remove the BIXOLON driver (run from the extracted driver package folder)
sudo sh uninstaller_v1.5.9.sh
```

The script log can be deleted with `sudo rm /var/log/bixolon-srp-e302-setup.log`.

## Repository layout

```
.
├── install-bixolon-srp-e302.sh   # Main installer
├── README.md                                          
└── LICENSE
```                    

## Security considerations

- The script requires root and re-executes itself with `sudo`. **Read it before running it.**
- Only use `--url` with sources you trust, and prefer HTTPS. Verify the checksum of the driver package if BIXOLON provides one.
- The script installs a vendor-provided binary filter that runs as part of the CUPS pipeline. Download it only from the official BIXOLON site.
- The script adds the invoking user to the `lp` and `lpadmin` groups. `lpadmin` allows managing printers; remove it if that is not desired: `sudo gpasswd -d $USER lpadmin`.
- The script does not open network ports or share the printer.

## Contributing

Contributions are welcome.

1. Fork the repository and create a feature branch: `git checkout -b feature/my-change`
2. Keep changes POSIX/Bash compatible and run static checks:
   ```bash
   bash -n install-bixolon-srp-e302.sh
   shellcheck install-bixolon-srp-e302.sh
   ```
3. Test on a clean system or VM where possible and state the OS, CUPS version and printer model in your pull request.
4. Use clear commit messages (for example `fix: handle missing USB backend`).
5. Open a pull request describing the problem and the solution.

Bug reports should include the output described in [Collecting information for a bug report](#collecting-information-for-a-bug-report).

## License and disclaimer

This installer script is released under the **MIT License**.

The BIXOLON CUPS driver, PPD files and manuals are the property of **BIXOLON Co., Ltd.** and are subject to their own license (`SoftwareLicense.txt` in the driver package). They are not distributed with this project.

This project is an independent community tool and is **not affiliated with or endorsed by BIXOLON**. BIXOLON and the BIXOLON logo are trademarks of BIXOLON Co., Ltd. The software is provided "as is", without warranty of any kind. Use it at your own risk.
