# UFIDREADER releases: install and test guide

Ready-built software for the **RK3576 UHF RFID reader board** (Android 14). Download everything
from **[Releases](../../releases)**. This page covers installing and testing it from a Linux PC.

| Release file | What it is |
|---|---|
| `GB_RK3576_14_<date>_<time>.img` | Complete board image: Android, the UHF reader platform, the ID TECH console and the web console. About 1.9 GB. |
| `GB_RK3576_14_<date>_<time>.img.md5` | Checksum for the image |
| `upgrade_tool_v2.30_for_linux.zip` | Rockchip's Linux tool for flashing the image |
| `IDTech-console-<version>.apk` | The ID TECH console app, the same build as inside the image (for updating a board without reflashing) |
| `ReadersHub-phone-<version>.apk` | The ReadersHub phone/tablet app, which controls a board over Wi-Fi (unchanged since v2026.10.09: take it from that release) |
| `UFIDREADER-ota-<build time>.zip` | OTA update package for the same build. Boards running v2026.10.25 or later find and install newer ones themselves (Firmware → Automatic updates); you can also install it from Firmware → Check now. |
| `uhf-sdk-<version>.aar` | The UHF SDK library for your own Android apps |
| `uhf-sdk-sample-app-src-<version>.zip` | A complete sample app using the SDK (Android Studio / Gradle project) |
| `uhf-sdk-sample-<version>-debug.apk` | The sample app, built (runs in demo mode on any phone, or on the board) |
| `uhf-sdk-docs-<version>.zip` | SDK API reference (Javadoc), SDK README and sample README |

**Contents:** [1. What you need](#1-what-you-need) · [2. Flash the board](#2-flash-the-board) ·
[3. First start](#3-first-start) · [4. Install the phone app](#4-install-the-phone-app) ·
[5. Test](#5-test) · [6. Firmware updates (OTA)](#6-firmware-updates-ota) ·
[7. Updating only the console app](#7-updating-only-the-console-app) ·
[8. SDK and sample app](#8-sdk-and-sample-app) · [9. Troubleshooting](#9-troubleshooting)

---

## 1. What you need

- **A Linux PC, x86-64.** Ubuntu 22.04 is tested. You also need `adb` (`sudo apt install adb`)
  and `md5sum`/`unzip`, which come with Ubuntu.
- **A USB data cable** to the board's **OTG / USB-flash** port. A charge-only cable will not work.
- **The board**, with the UHF reader attached: on the board UART, or through a USB-serial adapter.
- **For testing:** a few UHF tags, plus **one disposable tag** for the destructive tests (kill,
  lock). An Android 8.0+ phone on the same network is optional, for the phone app.

## 2. Flash the board

> Flashing replaces everything on the board. Treat it as a factory reset: app data (accounts and
> settings) goes back to the defaults.

**2.1 Download and check.** Put the image, its `.md5` file and the tool zip in one folder:

```bash
md5sum -c GB_RK3576_14_*.img.md5          # must print: OK. If not, download again; do not flash.
unzip upgrade_tool_v2.30_for_linux.zip
cd upgrade_tool_v2.30_for_linux && chmod +x upgrade_tool
```

**2.2 Optional: run the tool without `sudo`.** Allow your user to reach Rockchip USB devices
(vendor ID `2207`):

```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="2207", MODE="0666"' | sudo tee /etc/udev/rules.d/99-rockchip.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
```

**2.3 Put the board into flashing (Loader) mode.**

- **With adb**, if the board is running and USB debugging is on: `adb reboot loader`. The screen
  stays dark.
- **Without adb:** hold the board's **Recovery/Update** key while connecting power, then release
  it after about 3 seconds.

Check that the tool sees it:

```bash
sudo ./upgrade_tool LD
```

The output must list one device with **`Mode=Loader`**. `Mode=Maskrom` (the low-level recovery
mode) also works for flashing. If nothing is listed, see [Troubleshooting](#7-troubleshooting).

**2.4 Flash:**

```bash
sudo ./upgrade_tool UF ../GB_RK3576_14_<date>_<time>.img
```

This takes 2–5 minutes and ends with **`Upgrade firmware ok.`** Then the board restarts by itself.
**Don't unplug it while it's writing.**

## 3. First start

The first boot takes 1–3 minutes. Then:

```bash
adb wait-for-device
adb shell getprop ro.build.fingerprint      # the new build
adb shell pm path com.saurav.uhfreader       # /system/priv-app/IDTech/IDTech.apk
```

The console looks and works like the web console: the same sign-in screens, the same seven
tabs and the same Setting sub-tabs.

1. On the board, open **ID TECH** and sign in with **`admin` / `admin@123`**.
2. **Set a New Password:** at least 8 characters, not the default. The same account signs in to
   the web console.
3. **Get Connected:** tap **Connect**. The console finds the reader's port on its own. (**Skip
   (Continue Offline)** opens the console without a reader.)
4. **Optional: turn on the web console.** Go to **Setting → Web** and turn on the switch. The card
   lists the addresses to open from a PC on the same network: `http://idtech.local:8080` and
   `http://<board-ip>:8080`.

| Default after flashing | Value |
|---|---|
| Console sign-in | `admin` / `admin@123`, forced change at first sign-in |
| Recovery account | `superadmin`, for a forgotten admin password; its password is given out separately |
| Web console | Off (Setting → Web) |
| MQTT | Off (Setting → MQTT) |
| Wi-Fi | Built-in AP6212 module: **2.4 GHz only** (5 GHz-only networks don't appear). Settings → Network & internet → Wi-Fi. |
| Permissions for other apps | Any app that declares `UHF_READ` / `UHF_WRITE` can read and write tags. Region, power and antenna changes are limited to privileged system apps. |

## 4. Install the phone app

On a phone (Android 8.0+), connected to the PC with USB debugging on:

```bash
adb uninstall com.idtech.connectclient       # only if an older copy is installed
adb install -r ReadersHub-phone-<version>.apk
```

Or copy the APK to the phone and open it, allowing installs from this source when asked.

To use it:
1. Turn on the web console on the board first (section 3, step 4).
2. On the phone, use the same Wi-Fi as the board.
3. Open **ReadersHub** and tap **Connect**. Pick the board found as `idtech.local`, or enter its IP.
4. Sign in with the board's admin account.

## 5. Test

Do these in order. Note each result as pass or fail, with a screenshot or the error text. When a
step fails, also save the log:

```bash
adb logcat -d -s UhfConsole > console.log
```

**Isolate one tag for sections C and D.** Lower the RF power to about 5–10 dBm and hold the tag
at the antenna, or keep other tags away. With many tags in range, "first tag" operations hit a
random one.

### A. Board and console

| # | Do | Expect |
|---|---|---|
| A1 | Look at the launcher | ID TECH icon showing the full logo ("IDENTITY AND SECURITY") |
| A2 | `adb shell pm list packages \| grep -E "saurav\|idtech"` | Only `com.saurav.uhfreader` |
| A3 | Sign in with a wrong password | "Operator name or password is not correct" |
| A4 | First sign-in with `admin@123` | "Set a New Password" screen; `short` is refused (8+ characters) |
| A5 | Get Connected → **Connect** | The console opens; the header pill shows **Connected**. Setting → Reader Config → Reader Information is filled in. `adb shell getprop vendor.uhf.active_device` shows the port. |
| A6 | Settings → Wi-Fi → on; join a 2.4 GHz network | Networks listed; connects and gets an IP. `adb shell ip addr show wlan0` shows it. |

### B. Inventory and reader settings

| # | Do | Expect |
|---|---|---|
| B1 | Inventory → **Start**, wait 30 s, then **Stop** | Tags appear, counts and rate rise, the screen stays responsive |
| B2 | Search part of an EPC; then **Export CSV** | List filters; the CSV starts with `No.,EPC ID,Len,Times,LastSeen,RSSI,Antenna` |
| B3 | Setting → Antenna → slider to **0** → Set Power; ⓘ menu → Back to Connection → Connect | A warning that RF power is too low to read tags |
| B4 | Slider to **20** → Set Power; Inventory → Single Scan | Accepted; tags read again |
| B5 | Setting → RF & Band → choose a band → Set Band (confirm) → Get Band | Applied, no "Permission denied"; Get Band shows the new band. **Set your country's region back.** |
| B6 | Setting → Reader Config: Q 05, Session 01, Read Pause 2*5ms, Filter 1*1sec → Set Inventory Parameters; reboot the board, sign in, reopen | Values kept. Restore Q 07, Session 00, 0, 0 afterwards. |

### C. Tag memory (one isolated tag)

| # | Do | Expect |
|---|---|---|
| C1 | Inventory → tap the tag's row → Tag Memory → **Tag selected in Inventory page** → bank **EPC**, Start Byte **04**, Tag Length = EPC bytes (12 for a 96-bit EPC) → Read | Data (Hex) **equals the tag's EPC** |
| C2 | Bank **USER**, Start Byte 00, Tag Length 4, Write Data `11223344` (Verify writes on) → Write, then Read | "Data written."; the read-back matches |
| C3 | Select a tag on Inventory, take it out of the field, then Read | "no tag answered …" (not an empty success) |
| C4 | Write EPC card: a new EPC of the same length → Write → Inventory | **Only that tag** now shows the new EPC. Write the original back. |
| C5 | Tag Auditor → **Tag selected in inventory page** → Read | Banks A-D filled: CRC, PC, EPC (= target), TID. A locked bank shows the reason in brackets. |

### D. Destructive tests (disposable tag only)

| # | Do | Expect |
|---|---|---|
| D1 | Tag Memory → **Tag Kill–Lock & Block Erase**: Read Type **Kill Password**, Write Password non-zero → Write; then Tag Kill with that password → Kill Tag (confirm) | The tag never shows in Inventory again (re-scan 30 s) |
| D2 | On another disposable tag: set an access password, then Protect: **EPC Memory**, **Writeable with Password Protection** → Set Protect; then write without the password | Write refused, with a "memory locked" or tag-error message |

### E. Web console (Setting → Web on)

From v2026.10.25 the web console follows the ReadersHub design: a sidebar (collapse it with «),
status chips at the top (Reader, MQTT, Kafka, Webhook, Online) and these pages.

| # | Do | Expect |
|---|---|---|
| E1 | Open `http://<board-ip>:8080` (or `http://idtech.local:8080`) | ReadersHub sign-in page |
| E2 | Sign in with the board's admin account | Signs in. A new password set here also works in the console, and the other way round. |
| E3 | Get Connected → Serial Port → Start | **Device Info**: firmware, region, port. Continue → the console; the **Reader** chip turns green. |
| E4 | Inventory → Start | Live tags. Stopping on the web does not stop the console's own scan. |
| E5 | Settings → Antenna → Number of Antennas: **4** | "Antenna ports enabled: 1, 2, 3, 4". Tags read on every connected antenna; Tag Streaming's **Port** column shows 1-4. |
| E6 | Settings → Antenna: move one antenna's slider, Set Power | "RF power set: A1 … A4 dBm" (each antenna has its own power) |
| E7 | Settings → Antenna → Measure (Antenna Return Loss) | A value in dB for each connected antenna, "-" for an empty port |
| E8 | Tag Streaming → Begin feed | Unique EPCs, reads per second, per-tag RSSI min/avg/max and port; the download button saves a CSV |
| E9 | Settings → Saved profiles → New profile: 2 ports, different powers → Apply changes → Run | "Currently running" shows it; Stop ends it |
| E10 | Settings → Reader Config: change Q → Set; in the console, open Setting → Reader Config | Same Q in both |
| E11 | Event Reporting → Webhook: endpoint URL of any HTTP server that accepts POST (your own, or a request-inspector service) → Send test request; then turn delivery on → Apply | "Test passed: endpoint answered HTTP 200"; then tag batches arrive as JSON and the **Webhook** chip turns green |
| E12 | Network → Network Interfaces | eth0/wlan0, link, IPv4/IPv6, speed, MAC |
| E13 | Network → NTP | The time servers, with stratum/reach as the board polls them |
| E14 | Network → HTTP/HTTPS → turn HTTPS on → Apply | `https://<board-ip>:8443` opens (self-signed certificate: the browser warns once) |
| E15 | Network → HTTP/HTTPS → Create token; `curl -H "Authorization: Bearer <token>" http://<board-ip>:8080/api/status` | JSON status. After Revoke, the same command gives 401. |
| E16 | Debug Log → type `21` → Send | "Status 0x00 (ok): …" (the reader's information reply) |
| E17 | Setting → Web → switch off | The page stops loading. Switch it on again, and it survives a reboot. |
| E18 | Five wrong passwords in a row | Locked out for 60 s ("too many attempts") |

Not yet available: connecting to a network (TCP) reader - this board's reader is on its serial port.

**Please check on the board and report back** (these could not be tested without hardware):
joining Wi-Fi and setting a static Ethernet address (Network → Add connection; changing the
address can drop your browser - reconnect at the new address), changing NTP servers and the
hostname, Kafka delivery to a real broker, and a firmware update (section 6).

### F. MQTT (optional: needs a broker on the same network)

On a PC on the board's network, run Mosquitto (`listener 1883 0.0.0.0` and `allow_anonymous true`
in its config) and watch it with `mosquitto_sub -h <pc-ip> -t 'readershub/#' -v`.

| # | Do | Expect |
|---|---|---|
| F1 | Console → Setting → MQTT: host = PC IP, port 1883 → **Test connection** | "Test passed: Connected to …". Port 1 gives "Test failed: connection refused …". |
| F2 | Switch MQTT on | A retained `…/status {"state":"online"}`, then `…/tags` messages without pressing Start |
| F3 | Reboot the board, don't open the app | Publishing resumes on its own |

### G. Phone app

| # | Do | Expect |
|---|---|---|
| G1 | Connect and sign in (section 4) | Header shows the board |
| G2 | Inventory → Start / Stop | Live tag list |
| G3 | Read and Write a tag's User memory | Works; bad hex is refused |
| G4 | Config: change power | The console shows the same power |

## 6. Firmware updates (OTA)

Boards running **v2026.10.25 or later** update themselves from this page:

- **Firmware → Check now** looks for a release with a newer `UFIDREADER-ota-<build time>.zip`.
  **Install update** downloads it, checks it against the release page's SHA-256, verifies its
  signature and restarts into recovery to apply it (a few minutes; the board is offline meanwhile).
- **Automatic updates** on: the board checks once a day and installs what it finds.
- Boards on an older image have no updater: flash this image once (section 2); later releases can
  then arrive over the air.
- The web console must be switched on (Setting → Web) for updates to be checked.

## 7. Updating only the console app

A board already running an image from these releases can take a newer console without
reflashing:

```bash
adb install -r IDTech-console-<version>.apk     # keeps the app's data
```

If it fails with a signature error, the board's image was signed with a different key: flash the
image instead (section 2).

## 8. SDK and sample app

`uhf-sdk-<version>.aar` is the library for your own apps on the board. Unzip
`uhf-sdk-sample-app-src-<version>.zip` and open the folder in Android Studio (it builds against the
AAR in `libs/`), or install the ready-built sample: `adb install uhf-sdk-sample-<version>-debug.apk`.
The sample shows opening a session, reader information, live inventory (antenna ports 1-4), tag
memory read/write, power/band/antenna settings and raw commands; its **Demo** switch runs it
without a reader. The API reference is in `uhf-sdk-docs-<version>.zip` (`javadoc/index.html`).
Version 2.1.0 adds `transactRaw` (raw reader commands; needs the platform-signed `UHF_RAW`
permission, so ordinary apps get a clear "permission denied").

## 9. Troubleshooting

| Problem | Fix |
|---|---|
| `upgrade_tool LD` lists nothing | Use a data cable on the OTG port, and try another USB port on the PC (USB 2.0 is most reliable). Enter Loader mode again. `lsusb \| grep 2207` must show the board. |
| Permission or "open device" errors | Use `sudo`, or set up the udev rule (2.2) |
| Flashing stops part-way | Enter Loader (or Maskrom) mode again and repeat `UF`. Never power off the board while it's writing. |
| Board doesn't boot after flashing | Enter Maskrom (hold the board's Maskrom key while connecting power) and flash again |
| Console: "no reader connected" | Check the reader's cable and power, then **Connect** again |
| Tags read, but writes fail with "no tag answered" | Isolate the tag (section 5) and target it from the Inventory row |
| Web console doesn't open | Is Setting → Web on? Is the PC on the same network? Try the IP address instead of `idtech.local`. |
| Tool logs | `~/upgrade_tool/log/` (or `/root/upgrade_tool/log/` when run with `sudo`) |

Please report problems with the step number, what you expected and what happened, plus
`console.log` (section 5).
