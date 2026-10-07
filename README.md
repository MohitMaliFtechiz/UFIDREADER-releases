# UFIDREADER releases: install and test guide

Ready-built software for the **RK3576 UHF RFID reader board** (Android 14). Download everything
from **[Releases](../../releases)**. This page covers installing and testing it from a Linux PC.

| Release file | What it is |
|---|---|
| `GB_RK3576_14_<date>_<time>.img` | Complete board image: Android, the UHF reader platform, the ID TECH console and the web console. About 1.9 GB. |
| `GB_RK3576_14_<date>_<time>.img.md5` | Checksum for the image |
| `upgrade_tool_v2.30_for_linux.zip` | Rockchip's Linux tool for flashing the image |
| `IDTech-console-<version>.apk` | The ID TECH console app, the same build as inside the image (for updating a board without reflashing) |
| `ReadersHub-phone-<version>.apk` | The ReadersHub phone/tablet app, which controls a board over Wi-Fi |

**Contents:** [1. What you need](#1-what-you-need) · [2. Flash the board](#2-flash-the-board) ·
[3. First start](#3-first-start) · [4. Install the phone app](#4-install-the-phone-app) ·
[5. Test](#5-test) · [6. Updating only the console app](#6-updating-only-the-console-app) ·
[7. Troubleshooting](#7-troubleshooting)

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

1. On the board, open **ID TECH** and sign in with **`admin` / `admin@123`**.
2. Choose a new password: at least 8 characters, not the default. The same account signs in to
   the web console.
3. Tap **Connect**. The console connects straight to the reader and finds its port on its own.
4. **Optional: turn on the web console.** In the console, go to **Settings → Web** and turn on the
   switch. The card lists the addresses to open from a PC on the same network:
   `http://idtech.local:8080` and `http://<board-ip>:8080`.

| Default after flashing | Value |
|---|---|
| Console sign-in | `admin` / `admin@123`, forced change at first sign-in |
| Recovery account | `superadmin`, for a forgotten admin password; its password is given out separately |
| Web console | Off (Settings → Web) |
| MQTT | Off (Settings → MQTT) |
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
| A4 | First sign-in with `admin@123` | Forced to choose a new password; `short` is refused (8+ characters) |
| A5 | **Connect** | No dialog. Shows Connected; Reader information is filled in. `adb shell getprop vendor.uhf.active_device` shows the port. |
| A6 | Settings → Wi-Fi → on; join a 2.4 GHz network | Networks listed; connects and gets an IP. `adb shell ip addr show wlan0` shows it. |

### B. Inventory and reader settings

| # | Do | Expect |
|---|---|---|
| B1 | Inventory → **Start**, wait 30 s, then **Stop** | Tags appear, counts and rate rise, the screen stays responsive |
| B2 | Search part of an EPC; then **Export CSV** | List filters; the CSV starts with `EPC,Reads,RSSI,Antenna,TID` |
| B3 | Settings → RF & Band → set power **0** → Set; Disconnect, Connect | A warning that RF power is too low to read tags; "Maximum power 30" |
| B4 | Set power **20** → Set; Single scan | Accepted; tags read again |
| B5 | Settings → RF & Band → change region → Set band (confirm) | Applied, no "Permission denied"; Reader information shows the new band. **Set your country's region back.** |
| B6 | Settings → Reader Config: Q5, S1, pause 10, filter 1000 → Apply; force-stop the app (`adb shell am force-stop com.saurav.uhfreader`), sign in, reopen | Values kept. Restore Q7, S0, 0, 0 afterwards. |

### C. Tag memory (one isolated tag)

| # | Do | Expect |
|---|---|---|
| C1 | Inventory → tap the tag's row (it becomes the target) → Tag Memory → bank **EPC**, address **2**, length = EPC words → Read | The data **equals the tag's EPC** |
| C2 | Bank **User**, address 0, write `11223344` (Verify on), then read it back | Write OK; the read-back matches |
| C3 | Target an EPC that is **not** in the field, then Read | "no tag answered …" (not an empty success) |
| C4 | Write EPC: a new EPC of the same length → Inventory | **Only that tag** now shows the new EPC. Write the original back. |
| C5 | Tag Auditor → Read on the targeted tag | CRC, PC, EPC (= target) and TID filled. Locked banks show a reason, not all blank. |

### D. Destructive tests (disposable tag only)

| # | Do | Expect |
|---|---|---|
| D1 | Set a kill password (non-zero), then **Kill tag** (confirm) | The tag never shows in Inventory again (re-scan 30 s) |
| D2 | On another disposable tag: lock the EPC bank (state "locked", with an access password), then write without the password | Write refused, with a "memory locked" or tag-error message |

### E. Web console (Settings → Web on)

| # | Do | Expect |
|---|---|---|
| E1 | Open `http://<board-ip>:8080` | ReadersHub sign-in page |
| E2 | Sign in with the board's admin account | Signs in. A new password set here also works in the console, and the other way round. |
| E3 | Get Connected → Start → Inventory → Start | Live tags. Stopping on the web does not stop the console's own scan. |
| E4 | Setting → Reader Config: change Q → Set; in the console, open Settings → Reader Config | Same Q in both |
| E5 | Settings → Web → switch off | The page stops loading. Switch it on again, and it survives a reboot. |
| E6 | Five wrong passwords in a row | Locked out for 60 s ("too many attempts") |

### F. MQTT (optional: needs a broker on the same network)

On a PC on the board's network, run Mosquitto (`listener 1883 0.0.0.0` and `allow_anonymous true`
in its config) and watch it with `mosquitto_sub -h <pc-ip> -t 'readershub/#' -v`.

| # | Do | Expect |
|---|---|---|
| F1 | Console → Settings → MQTT: host = PC IP, port 1883 → **Test connection** | "Test passed: Connected to …". Port 1 gives "Test failed: connection refused …". |
| F2 | Switch MQTT on | A retained `…/status {"state":"online"}`, then `…/tags` messages without pressing Start |
| F3 | Reboot the board, don't open the app | Publishing resumes on its own |

### G. Phone app

| # | Do | Expect |
|---|---|---|
| G1 | Connect and sign in (section 4) | Header shows the board |
| G2 | Inventory → Start / Stop | Live tag list |
| G3 | Read and Write a tag's User memory | Works; bad hex is refused |
| G4 | Config: change power | The console shows the same power |

## 6. Updating only the console app

A board already running an image from these releases can take a newer console without
reflashing:

```bash
adb install -r IDTech-console-<version>.apk     # keeps the app's data
```

If it fails with a signature error, the board's image was signed with a different key: flash the
image instead (section 2).

## 7. Troubleshooting

| Problem | Fix |
|---|---|
| `upgrade_tool LD` lists nothing | Use a data cable on the OTG port, and try another USB port on the PC (USB 2.0 is most reliable). Enter Loader mode again. `lsusb \| grep 2207` must show the board. |
| Permission or "open device" errors | Use `sudo`, or set up the udev rule (2.2) |
| Flashing stops part-way | Enter Loader (or Maskrom) mode again and repeat `UF`. Never power off the board while it's writing. |
| Board doesn't boot after flashing | Enter Maskrom (hold the board's Maskrom key while connecting power) and flash again |
| Console: "no reader connected" | Check the reader's cable and power, then **Connect** again |
| Tags read, but writes fail with "no tag answered" | Isolate the tag (section 5) and target it from the Inventory row |
| Web console doesn't open | Is Settings → Web on? Is the PC on the same network? Try the IP address instead of `idtech.local`. |
| Tool logs | `~/upgrade_tool/log/` (or `/root/upgrade_tool/log/` when run with `sudo`) |

Please report problems with the step number, what you expected and what happened, plus
`console.log` (section 5).
