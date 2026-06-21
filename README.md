# KALKI-MTK

A USB-based console for talking to Nokia/MTK feature phones at a low level  raw protocol frames on IF0 and AT commands on IF1. Lets you send/receive raw hex frames, run AT commands, dump the phonebook to JSON/CSV, do a full device backup, and restore contacts.

This repo only ships the **compiled binary**. No build steps here  just run it.

## WARNING
**Always create a backup before attempting restore operations or testing undocumented commands**

## Commands reference

```
-- IF0 (Nokia/MTK protocol) ----------------------------
send <hex> [rx_ms]            TX then monitor RX
delaysend <ms> <hex> [rx_ms]  wait, TX, monitor RX
listen <ms>                   passive RX only
sequence                      run built-in sequence
if0 / if1                     switch active interface

-- IF1 (AT command layer) --------------------------------
at <command>                  send raw AT command
atlist [n]                    browse embedded AT command reference
diag                          run standard fingerprint sequence
atdump <sm|me> [start] [end]  verbose read + parse trace
dumpcontacts sm|me            dump phonebook to JSON+CSV
backupall                     full device backup
restorecontacts <file.json>   restore from backup

quit
```

---

## Supported phone models

`KALKI-MTK` doesn't hardcode a phone model list  there's nothing in the source that whitelists or checks for specific models. It works at the protocol level on two fronts:

- **IF0**  raw Nokia/MTK USB framing (the `send`/`delaysend`/`listen`/`sequence` commands), whatever the phone's USB composite device exposes on its first interface.
- **IF1**  standard 3GPP Hayes/AT commands plus MediaTek (`AT+E...`) and Nokia (`AT*...`) proprietary extensions, on the second interface.

So compatibility comes down to: **any phone built on an MTK chipset that exposes this USB interface layout will work**, not a specific tested list. In practice that's the modern Nokia (HMD-era) feature-phone lineup  things like the Nokia 105/110/130/150/215/216/220/225/3310 (2017) and similar MTK-based models  since they share this USB/AT command architecture. Older true Nokia (Symbian/S40, FBUS-era, pre-MTK) phones and non-MTK Nokia Android phones use a different protocol and are **not** what this tool targets.

If you plug in a phone and `at+clac` or `atlist 0` gets a response, IF1 is working regardless of exact model. If `send`/`sequence` get a reply on IF0, the raw protocol layer is working too.

---

## AT command usage

On IF1 (`if1`), three commands let you talk to the phone's AT layer:

| Command | What it does |
|---|---|
| `at <command>` | Sends one raw AT command and prints the reply (e.g. `at AT+CGMM`). |
| `atlist` | Lists the 16 built-in reference categories below. |
| `atlist <n>` | Prints every command in category `n`, with a one-line explanation each. |
| `diag` | Runs a standard fingerprinting sequence (identity + status commands) in one go. |
| `atdump <sm\|me> [start] [end]` | Selects phonebook storage (`SM` = SIM, `ME` = phone memory), reads entries `start`–`end` with `AT+CPBR`, and traces the parser output verbatim  useful for debugging phonebook parsing. |

Example session:

```
NOKIA> if1
NOKIA> at AT+CGMM
NOKIA> at AT+CGSN
NOKIA> atlist
NOKIA> atlist 2
NOKIA> diag
```

### AT command reference (built into `atlist`)

This is the same reference table embedded in the binary, organized by category. Anything marked **(MTK)** or **(Nokia)** is a chipset/vendor extension rather than a standard 3GPP command.

**0. Discover what your phone supports**
- `AT+CLAC`  dump every AT command the firmware knows (MTK)
- `AT&V`  dump all current settings
- `ATI` / `ATI0`–`ATI7`  basic info / product code / ROM checksums / firmware revision / manufacturer / country-config / capabilities / model info
- `AT+GMI` / `AT+GMM` / `AT+GMR` / `AT+GSN`  standard manufacturer / model / revision / IMEI

**1. Basic / Hayes**
- `AT`  attention/ping
- `AT&F`  factory-reset settings · `ATZ`  reset to saved profile
- `ATE0/1`  echo off/on · `ATQ0/1`  result codes off/on · `ATV0/1`  numeric/verbose result codes
- `ATS0`, `ATS3`, `ATS4`, `ATS5`, `ATS7`, `ATS10`  auto-answer, line terminators, backspace char, timeouts

**2. Identity & device info**
- `AT+CGMI/CGMM/CGMR/CGSN`  manufacturer / model / firmware / IMEI
- `AT+CIMI`  IMSI from SIM · `AT+CCID`  ICCID (SIM serial)
- `AT+CSCS`  character set (GSM/UCS2/IRA/8859-1) · `AT+WPCS`  phonebook character set (MTK)

**3. SIM / PIN / security**
- `AT+CPIN?` / `AT+CPIN="1234"`  PIN status / enter PIN1
- `AT+CPIN="PUK","newpin"`  unblock with PUK
- `AT+CLCK`  query/set lock facilities · `AT+CPWD`  change PIN1/PIN2/barring password
- `AT+CPINC?`  PIN/PUK retry counters (MTK)

**4. Network & operator**
- `AT+CREG` / `AT+CGREG` / `AT+CEREG`  2G/GPRS/LTE registration status
- `AT+COPS`  operator select/scan · `AT+CSQ` / `AT+CESQ`  signal strength
- `AT+CIND`  indicator status · `AT+CPOL` / `AT+COPN`  preferred/all operator lists
- `AT+CBAND`, `AT+ENWINFO`, `AT+EINFO`, `AT+ECOPS?`, `AT+ESBP?`  MTK band/network extensions

**5. Calls  voice**
- `ATD<number>;`  dial · `ATA`/`ATH`  answer/hang up · `AT+CHUP`  hang up (3GPP)
- `AT+CLCC`  list current calls · `AT+CHLD`  hold/multiparty/transfer
- `AT+CLIR`/`CLIP`/`COLP`  caller ID controls · `AT+VTS`  send DTMF
- `AT+CALM`/`CRSL`/`CLVL`/`CMUT`  alert mode, ringer, volume, mic mute · `AT+CUSD`  USSD

**6. Calls  data/fax**
- `ATD<number>` (no `;`)  dial data call · `AT+CBST`  bearer type · `AT+FCLASS`  fax class
- `AT+CR`/`CRC`  service/ring reporting · `AT+DS`/`DR`  V.42bis compression

**7. SMS**
- `AT+CMGF`  PDU/text mode · `AT+CSCA`  SMSC address
- `AT+CMGS`/`CMGW`/`CMGD`  send/write/delete · `AT+CMGL`/`CMGR`  list/read
- `AT+CNMI`  new-message URC config · `AT+CPMS`  select SMS storage

**12. MTK proprietary** ⚠️
- `AT+EGMR=0,7`  read IMEI (safe) · `AT+EGMR=1,7,"IMEI"`  **write IMEI, permanent, illegal in many countries**
- `AT+EMMCINFO`  eMMC flash chip info · `AT+EQUERY=?`  query MTK extensions
- `AT+ERFTX`/`ERFRX`  RF TX/RX engineering test (caution) · `AT+EBAND`/`ERAT`  band/RAT select

**13. Nokia proprietary (FBUS/MBUS AT extensions)**
- `AT*NOKIATEST`  test command · `AT*ERINFO`  extended radio info · `AT*ELAM`  LED control
- `AT+CPROT?`  protocol mode · `AT*EBAUDR?`  baud rate query

**14. Supplementary services (SS/USSD)**
- `AT+CUSD`  send/cancel USSD · `AT+CCFC`  call forwarding query/activate/deactivate
- `AT+CSSN`/`COLP`/`CDIP`/`CTUG`  SS notifications & line ID

**15. Audio**
- `AT+CLVL`/`CMUT`/`CRSL`/`CALM`  volume, mute, ringer, alert mode
- `AT+CAMP`  multimedia ringer (MTK) · `AT+CACM`/`CAMM`/`CAOC`  call meter / advice of charge

**16. STK (SIM toolkit)**
- `AT+STKPRO`/`STKCALL`/`STKENV`/`STKTR`/`STKTI`  SIM toolkit profile, call confirm, envelope, terminal response
- `AT+CUSATP?`  USAT profile

**17. Cell broadcast**
- `AT+CSCB`  subscribe/unsubscribe to CB channels · `AT+CNMI`  enable CB+SMS URCs

**18. Packet domain / internet**
- `AT+CGDSCONT`/`CGTFT`/`CGCMOD`  PDP context management
- `AT+CGSMS`  preferred MO SMS bearer · `AT+CGAUTH`  PAP/CHAP auth for context

**19. Hardware / engineering** ⚠️
- `AT+CMEC`  mobile equipment control (keypad/display) · `AT+CKPD`  keypad emulation
- `AT+CDIS`/`CIND`  display text / indicator values · `AT+CSVM`  voicemail number

**20. File system (MTK/Nokia)**
- `AT+EFSL?`  file system info · `AT+EFSR`  file system reset · `AT+MFSL?`  memory FS list

**21. Firmware / flashing** ⚠️ **EXTREME CAUTION**
- `AT+SYSCOMP?`  system component versions · `AT+NVRAM?`  NVRAM dump
- `AT+RESTORE=1`  **factory reset NVRAM, destructive**
- `AT+EBOOT`  reboot to download mode / normal reboot
- `AT+BROM`  **enter BROM/DRAM download mode  very dangerous, can brick the device**

> Commands flagged ⚠️ can permanently alter or break the device, or are restricted/illegal in some jurisdictions (e.g. rewriting IMEI). They're included here for reference because the firmware accepts them  that doesn't mean you should run them without knowing exactly what they do on your specific phone.

---

## Termux (Android)

### 1. Prerequisites

```bash
pkg install termux-api libusb
```

Also install the **Termux:API** app from the same source you got Termux from (F-Droid  F-Droid, GitHub  GitHub  don't mix stores). It's a separate APK, not just a package.

Make the binary executable once:

```bash
chmod +x KALKI-MTK
```

### 2. Plug in the phone

Connect the phone to your Android device via a USB-OTG cable. Wait a few seconds for Android to recognize it.

### 3. List the USB device

```bash
termux-usb -l
```

This prints the device path, e.g.:

```
["/dev/bus/usb/001/002"]
```

Copy that path.

### 4. Request permission

```bash
termux-usb -r /dev/bus/usb/001/002
```

Android will pop up a permission dialog asking to allow Termux to access the USB device  **tap "Yes" / "OK"** to grant it (you can also check "use by default" so it doesn't ask again).

### 5. Run the binary against that device

```bash
  ./KALKI-MTK
```
 OR
 
```bash
termux-usb -e ./KALKI-MTK /dev/bus/usb/001/xxx
```

This launches `KALKI-MTK` with a file descriptor for the device handed to it directly, and you'll land in the `NOKIA>` prompt.

> **Note:** `KALKI-MTK` also has its own auto-detect step  if you just run `./KALKI-MTK` with no device attached via the steps above, it will call `termux-usb -l` itself, pick the device (or ask you to pick if there's more than one), and re-launch itself through `termux-usb -e` automatically, triggering the same permission popup. The manual steps above are there for when you want full control, or if auto-detect doesn't find your device.

---

## Linux (desktop)

### 1. Prerequisites

```bash
sudo apt install libusb-1.0-0      # Debian/Ubuntu
# or
sudo dnf install libusb1           # Fedora
# or
sudo pacman -S libusb              # Arch
```

Make the binary executable:

```bash
chmod +x KALKI-MTK
```

### 2. Plug in the phone

Connect the phone via USB cable to your Linux machine.

### 3. Permissions

Raw USB access usually needs elevated permissions. Either:

**Quick way**  run as root:

```bash
sudo ./KALKI-MTK
```

**Better way**  add a udev rule so you don't need root every time. Find the phone's vendor/product ID with `lsusb`, then create `/etc/udev/rules.d/99-kalki-mtk.rules`:

```
SUBSYSTEM=="usb", ATTR{idVendor}=="XXXX", ATTR{idProduct}=="YYYY", MODE="0666"
```

Reload rules and replug the device:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Then run normally:

```bash
./KALKI-MTK
```

### 4. Device selection

On launch, `KALKI-MTK` enumerates connected USB devices itself:

- If exactly one matching device is found, it's used automatically.
- If multiple are found, you'll be prompted to pick one from a numbered list.

To skip the picker and target a specific phone directly, set its vendor/product ID (in hex, from `lsusb`) before running:

```bash
NOKIA_VID=0421 NOKIA_PID=xxxx ./KALKI-MTK
```

---

## Usage

Once connected (either platform), you'll see the banner and prompt:

```
NOKIA> 
```

From there, use the commands listed above. A session log (`session_<timestamp>.log`) is written to the current directory automatically.

Type `quit` to exit cleanly.
