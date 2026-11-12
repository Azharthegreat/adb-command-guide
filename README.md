# ADB Command Guide

A practical, copy-paste reference for Android Debug Bridge (adb) on Windows, macOS and Linux.
It covers setup, USB and wireless connections, the shell, package management, permissions,
files, logs, diagnostics, networking, testing, reboot modes and safe read-only modem checks.

Replace the placeholders `PACKAGE`, `SERIAL`, `LOCAL_PATH`, `REMOTE_PATH`, `IP` and `PORT` with
your own values. Options vary with Android version, OEM firmware, build type and Platform-Tools
version, so when a command behaves differently, check the device's own help: `adb help`,
`adb shell pm help`, `adb shell am help`, `adb logcat --help`, `adb shell dumpsys -h`.

**Scope boundary.** This guide doesn't cover changing or cloning an IMEI, corrupting EFS/NV
data, bypassing a lock screen or FRP, or writing another device's identity. A wrong partition or
firmware operation can leave a phone with no service or hard-bricked.

**Risk legend.** Commands marked **Caution** change app data or settings. Commands marked
**Danger** can reset the device, break boot, lose data or cut connectivity. Everything else is
read-only or easy to undo.

## Contents

1. [ADB basics](#1-adb-basics)
2. [Installation and first connection](#2-installation-and-first-connection)
3. [Device selection and server control](#3-device-selection-and-server-control)
4. [Wireless debugging](#4-wireless-debugging)
5. [Shell and Linux basics](#5-shell-and-linux-basics)
6. [Device and system information](#6-device-and-system-information)
7. [File transfer and storage](#7-file-transfer-and-storage)
8. [APK installation and package management](#8-apk-installation-and-package-management)
9. [Activities, intents, services and broadcasts](#9-activities-intents-services-and-broadcasts)
10. [Permissions, AppOps and special access](#10-permissions-appops-and-special-access)
11. [Users, profiles and device administration](#11-users-profiles-and-device-administration)
12. [Settings, properties and display](#12-settings-properties-and-display)
13. [Screenshots, recording and input](#13-screenshots-recording-and-input)
14. [Logcat, bug reports and dumpsys](#14-logcat-bug-reports-and-dumpsys)
15. [Networking, forward and reverse](#15-networking-forward-and-reverse)
16. [Processes, memory, CPU and performance](#16-processes-memory-cpu-and-performance)
17. [Battery, power and thermal](#17-battery-power-and-thermal)
18. [Reboot, recovery, bootloader and sideload](#18-reboot-recovery-bootloader-and-sideload)
19. [Testing and instrumentation](#19-testing-and-instrumentation)
20. [Root, remount and debug builds](#20-root-remount-and-debug-builds)
21. [Modem, baseband and EFS/NV notes](#21-modem-baseband-and-efsnv-notes)
22. [Troubleshooting](#22-troubleshooting)
23. [Ready-made recipes](#23-ready-made-recipes)
24. [Cheat sheet](#24-cheat-sheet)
25. [Official references](#25-official-references)

### Filtering output on each platform

`grep` isn't available in Windows Command Prompt. Use `findstr` there, or `Select-String` in
PowerShell.

| Command | Shell |
| --- | --- |
| `adb shell getprop \| grep -i samsung` | macOS, Linux, Git Bash |
| `adb shell getprop \| findstr /i samsung` | Windows Command Prompt |
| `adb shell getprop \| Select-String samsung` | Windows PowerShell |

---

## 1. ADB basics

ADB is a client-server tool. The client runs your command on the computer, an ADB server on the
computer manages connected devices, and the `adbd` daemon on the phone or emulator executes it.

| Command | What it does |
| --- | --- |
| `adb version` | Installed ADB version and build details |
| `adb --version` | Same, in the form some versions expect |
| `adb help` | Help for host-side commands |
| `adb shell COMMAND` | Runs one command in the device's shell |
| `adb shell` | Opens an interactive shell. Type `exit` to leave |

**ADB vs Fastboot.** ADB works while Android (and `adbd`) is running. Fastboot runs in
bootloader mode and deals with partitions and boot images. Samsung phones generally use Download
Mode and Odin instead of standard Fastboot flashing.

## 2. Installation and first connection

**Install Platform-Tools.** The easiest route is the SDK Manager in Android Studio, which keeps
Android SDK Platform-Tools up to date. A standalone package is also available from Android
Developers. Add the folder to your `PATH` so `adb` works from any directory.

```bash
# Verify installation
adb version

# Start the server and list devices
adb start-server
adb devices -l
```

**On the phone.** Go to Settings > About phone > Software information and tap *Build number*
seven times. In Developer options, turn on *USB debugging*. When you connect over USB, the phone
shows an RSA authorization prompt: only allow computers you trust.

| Command | What it does |
| --- | --- |
| `adb devices` | Connected device serials and their state |
| `adb devices -l` | Adds model, product and transport ID |
| `adb get-state` | `device`, `offline` or `unknown` |
| `adb get-serialno` | Serial of the selected device |
| `adb wait-for-device` | Pauses a script until a device is available |

**Unauthorized state.** If a device shows as `unauthorized`, unlock the phone and accept the RSA
prompt. If no prompt appears, revoke USB debugging authorizations in Developer options and
reconnect.

## 3. Device selection and server control

### Multiple devices

| Command | What it does |
| --- | --- |
| `adb -s SERIAL shell` | Targets the device with that serial |
| `adb -d shell` | Targets the only physical USB device |
| `adb -e shell` | Targets the only emulator |
| `adb -t TRANSPORT_ID shell` | Targets a transport ID from `adb devices -l` |
| `set ANDROID_SERIAL=SERIAL` | Default device for a Windows CMD session |
| `export ANDROID_SERIAL=SERIAL` | Default device for a macOS/Linux shell |

### Server management

| Command | What it does |
| --- | --- |
| `adb start-server` | Starts the local ADB server |
| `adb kill-server` | Stops the server. The usual first step when connections get stuck |
| `adb reconnect` | Asks offline devices to reconnect |
| `adb reconnect device` | Reconnects from the device side |
| `adb reconnect offline` | Resets offline or unauthorized transports |

```bash
# Example: two phones connected
adb devices -l
adb -s R58M123ABC shell getprop ro.product.model
adb -s 192.168.1.50:5555 install app-debug.apk
```

## 4. Wireless debugging

### Android 11 and later: pairing

On the phone, open Developer options > Wireless debugging > *Pair device with pairing code*. The
screen shows an IP address, a pairing port and a six-digit code.

```bash
adb pair PHONE_IP:PAIRING_PORT
# enter the pairing code when asked
adb connect PHONE_IP:ADB_PORT
adb devices -l
```

| Command | What it does |
| --- | --- |
| `adb pair IP:PORT` | Starts the pairing-code flow |
| `adb connect IP:PORT` | Connects to a paired device's ADB port |
| `adb disconnect IP:PORT` | Disconnects one wireless device |
| `adb disconnect` | Disconnects every TCP/IP device |
| `adb mdns services` | Lists ADB pairing/connect services advertised on the network (newer Platform-Tools) |

### Legacy TCP/IP method

```bash
# Connect by USB first
adb tcpip 5555
adb shell ip route          # find the phone's IP address
adb connect PHONE_IP:5555

# Put adbd back into USB mode
adb usb
```

**Security.** Never expose port 5555 on public or untrusted Wi-Fi. It accepts connections
without pairing. On Android 11+, prefer authenticated Wireless debugging.

## 5. Shell and Linux basics

| Command | What it does |
| --- | --- |
| `adb shell pwd` | Current directory |
| `adb shell ls -la /sdcard` | Directory listing with hidden entries and details |
| `adb shell cd /sdcard` | Changes directory (useful inside an interactive shell) |
| `adb shell cat FILE` | Prints a text file |
| `adb shell head -n 20 FILE` | First 20 lines |
| `adb shell tail -n 50 FILE` | Last 50 lines |
| `adb shell grep -i PATTERN FILE` | Case-insensitive search, if the device toolbox supports it |
| `adb shell find /sdcard -name "*.log"` | Finds matching files (toolbox dependent) |
| `adb shell mkdir -p /sdcard/Test` | Creates a directory |
| `adb shell cp SOURCE DEST` | Copies a file |
| `adb shell mv SOURCE DEST` | Moves or renames |
| `adb shell rm FILE` | Deletes a file. **Caution** |
| `adb shell rm -rf DIR` | Deletes a directory recursively. **Danger** |
| `adb shell chmod 755 FILE` | Changes permissions, within what the shell user is allowed |
| `adb shell id` | Shell UID and groups |
| `adb shell whoami` | Shell user name, if supported |
| `adb shell uname -a` | Kernel information |
| `adb shell date` | Device date and time |

**Quoting.** Your computer's shell parses quotes first. To run pipes or redirection on the
device, quote the whole command: `adb shell "getprop | grep -i radio"`. Quoting rules on Windows
are slightly different.

## 6. Device and system information

| Command | What it does |
| --- | --- |
| `adb shell getprop` | Every Android system property |
| `adb shell getprop ro.product.manufacturer` | Manufacturer |
| `adb shell getprop ro.product.model` | Model |
| `adb shell getprop ro.product.device` | Device codename |
| `adb shell getprop ro.build.version.release` | Android version |
| `adb shell getprop ro.build.version.sdk` | API level |
| `adb shell getprop ro.build.fingerprint` | Full build fingerprint |
| `adb shell getprop ro.bootloader` | Bootloader version |
| `adb shell getprop gsm.version.baseband` | Reported baseband version (OEM dependent) |
| `adb shell settings get secure android_id` | Android ID for the current user |
| `adb shell wm size` | Physical and override display resolution |
| `adb shell wm density` | Physical and override DPI |
| `adb shell df -h` | Mounted filesystems and free space |
| `adb shell cat /proc/meminfo` | Kernel memory information |
| `adb shell cat /proc/cpuinfo` | CPU information (newer devices may restrict it) |
| `adb shell lscpu` | CPU summary, if the binary exists |
| `adb shell ip addr` | Network interfaces and IP addresses |
| `adb shell ip route` | Routing table. A quick way to find the phone's IP |
| `adb shell dumpsys display` | Display system diagnostics |
| `adb shell dumpsys window displays` | Window and display state |

```bash
# A compact device report
adb shell "echo MODEL=\$(getprop ro.product.model); \
  echo ANDROID=\$(getprop ro.build.version.release); \
  echo SDK=\$(getprop ro.build.version.sdk); \
  echo BASEBAND=\$(getprop gsm.version.baseband)"
```

## 7. File transfer and storage

| Command | What it does |
| --- | --- |
| `adb push LOCAL_PATH REMOTE_PATH` | Copies a file or folder to the device |
| `adb push -a LOCAL REMOTE` | Tries to keep timestamps and mode |
| `adb push --sync LOCAL REMOTE` | Only pushes files that are newer or different |
| `adb pull REMOTE_PATH LOCAL_PATH` | Copies a file or folder from the device |
| `adb pull -a REMOTE LOCAL` | Keeps timestamps and mode |
| `adb shell ls -lh /sdcard/Download` | Lists the Download folder |
| `adb shell stat /sdcard/Download/file.zip` | File metadata |
| `adb shell sha256sum /sdcard/Download/file.zip` | SHA-256 on the device, if available |

```bash
adb push app-release.apk /sdcard/Download/
adb pull /sdcard/DCIM/Camera ./CameraBackup
adb shell mkdir -p /sdcard/ADB-Test
adb push ./logs /sdcard/ADB-Test/
```

**App private storage.** On a production phone the shell user can't read
`/data/data/PACKAGE`. For a debuggable app, use `run-as`:

```bash
# Debuggable apps only
adb shell run-as com.example.app pwd
adb shell run-as com.example.app ls -la files
adb exec-out run-as com.example.app cat databases/app.db > app.db
```

## 8. APK installation and package management

### Install

| Command | What it does |
| --- | --- |
| `adb install app.apk` | Installs one APK |
| `adb install -r app.apk` | Replaces an installed app, normally keeping its data |
| `adb install -d app.apk` | Allows a version-code downgrade (usually debuggable apps only) |
| `adb install -g app.apk` | Grants all runtime permissions declared in the manifest |
| `adb install -t app.apk` | Allows test-only APKs |
| `adb install --user 0 app.apk` | Installs for a specific Android user |
| `adb install-multiple base.apk split_config.arm64_v8a.apk split_config.en.apk` | Installs a set of split APKs |
| `adb install-multi-package ...` | Installs several packages atomically (newer Platform-Tools) |

### List and inspect packages

| Command | What it does |
| --- | --- |
| `adb shell pm list packages` | Every known package |
| `adb shell pm list packages -3` | Third-party packages |
| `adb shell pm list packages -s` | System packages |
| `adb shell pm list packages -f` | With APK paths |
| `adb shell pm list packages -U` | With UIDs |
| `adb shell pm path PACKAGE` | Installed APK path(s) |
| `adb shell dumpsys package PACKAGE` | Package state, UID, permissions, activities, version |
| `adb shell cmd package resolve-activity --brief PACKAGE` | Resolves the default launch activity |

### Uninstall, clear, enable and disable

| Command | What it does |
| --- | --- |
| `adb uninstall PACKAGE` | Uninstalls the app and removes its data. **Caution** |
| `adb uninstall -k PACKAGE` | Uninstalls but keeps data and cache where supported |
| `adb shell pm uninstall --user 0 PACKAGE` | Removes the package for user 0. A system APK stays on the system partition. **Caution** |
| `adb shell cmd package install-existing --user 0 PACKAGE` | Restores a system package removed for user 0 |
| `adb shell pm clear PACKAGE` | Wipes all of the app's data and cache. **Caution** |
| `adb shell pm disable-user --user 0 PACKAGE` | Disables the package for that user |
| `adb shell pm enable PACKAGE` | Enables it again |
| `adb shell am force-stop PACKAGE` | Force-stops the app's processes and services |
| `adb shell pm trim-caches 1G` | Asks the system to trim caches (version dependent) |

**System apps.** Disabling or removing a system package for a user can break the launcher,
Settings, telephony or boot features. Find out what the package does first, and note the
command that restores it.

## 9. Activities, intents, services and broadcasts

### Activities

| Command | What it does |
| --- | --- |
| `adb shell monkey -p PACKAGE -c android.intent.category.LAUNCHER 1` | Quickest way to launch an app's main activity |
| `adb shell am start -n PACKAGE/.Activity` | Starts an explicit activity |
| `adb shell am start -a android.intent.action.VIEW -d "https://example.com"` | Opens a URL or deep link |
| `adb shell am start -W -n PACKAGE/.Activity` | Waits and prints launch timing |
| `adb shell am force-stop PACKAGE` | Stops the package |
| `adb shell am kill PACKAGE` | Kills background processes where allowed |

```bash
# Start an activity with extras
adb shell am start -n com.example/.MainActivity \
  --es username alice \
  --ei user_id 123 \
  --ez debug true
```

### Services and broadcasts

| Command | What it does |
| --- | --- |
| `adb shell am startservice -n PACKAGE/.MyService` | Starts a background service (background limits apply) |
| `adb shell am start-foreground-service -n PACKAGE/.MyService` | Requests a foreground service start |
| `adb shell am stopservice -n PACKAGE/.MyService` | Stops a service |
| `adb shell am broadcast -a ACTION` | Sends a broadcast |
| `adb shell am broadcast -a ACTION -n PACKAGE/.Receiver` | Sends it to one receiver |
| `adb shell dumpsys activity activities` | Activity stack and task state |
| `adb shell dumpsys activity services PACKAGE` | Service diagnostics for a package |

Common extras: `--es` string, `--ei` integer, `--el` long, `--ef` float, `--ez` boolean, `--esa`
string array, `-a` action, `-d` data URI, `-t` MIME type, `-c` category, `-f` flags.

## 10. Permissions, AppOps and special access

### Runtime permissions

| Command | What it does |
| --- | --- |
| `adb shell pm list permissions -g -d` | Dangerous permissions, grouped |
| `adb shell pm grant PACKAGE android.permission.CAMERA` | Grants a declared runtime permission |
| `adb shell pm revoke PACKAGE android.permission.CAMERA` | Revokes it |
| `adb shell dumpsys package PACKAGE \| grep permission` | Permission state for a package |
| `adb shell cmd package reset-permissions` | Resets runtime permissions for every app. **Caution** |

**What `pm grant` can't do.** It only grants permissions the app declares in its manifest and
that the platform lets the shell grant. Signature and privileged permissions can't be granted to
a normal app this way.

### AppOps

| Command | What it does |
| --- | --- |
| `adb shell cmd appops get PACKAGE` | AppOps state for a package |
| `adb shell cmd appops get PACKAGE OP` | State of one operation |
| `adb shell cmd appops set PACKAGE OP allow` | Allows an operation |
| `adb shell cmd appops set PACKAGE OP deny` | Denies it |
| `adb shell cmd appops set PACKAGE OP ignore` | Silently ignores it |
| `adb shell cmd appops reset PACKAGE` | Resets the package to defaults |
| `adb shell cmd appops query-op OP allow` | Lists apps by operation and mode (version dependent) |

```bash
# Check which operations your Android version supports before relying on these
adb shell cmd appops get com.example.app
adb shell cmd appops set com.example.app SYSTEM_ALERT_WINDOW allow
adb shell cmd appops set com.example.app WRITE_SETTINGS allow
```

**WRITE_SECURE_SETTINGS.** `adb shell pm grant PACKAGE android.permission.WRITE_SECURE_SETTINGS`
works for development and helper apps that declare it. It lets the app change sensitive system
settings, so only grant it to apps you trust.

## 11. Users, profiles and device administration

| Command | What it does |
| --- | --- |
| `adb shell pm list users` | Android users and profiles |
| `adb shell am get-current-user` | Foreground user ID |
| `adb shell pm create-user NAME` | Creates a secondary user (OEM and policy limits apply) |
| `adb shell pm create-user --profileOf 0 --managed NAME` | Creates a managed (work) profile for testing |
| `adb shell am switch-user USER_ID` | Switches the foreground user |
| `adb shell am start-user USER_ID` | Starts a user |
| `adb shell am stop-user -w USER_ID` | Stops a user and waits |
| `adb shell pm remove-user USER_ID` | Removes a user or profile and all its data. **Danger** |
| `adb shell dpm list-owners` | Device and profile owner info, if supported |
| `adb shell dpm set-device-owner PACKAGE/.Receiver` | Sets a device owner on a fresh, unprovisioned test device. **Danger** |
| `adb shell dpm remove-active-admin PACKAGE/.Receiver` | Removes a test admin (device owner rules apply) |

**Warning.** Don't experiment with device owner commands on your personal phone. Some ownership
states can only be cleared with a factory reset.

## 12. Settings, properties and display

### Settings database

| Command | What it does |
| --- | --- |
| `adb shell settings list system` | System namespace |
| `adb shell settings list secure` | Secure namespace |
| `adb shell settings list global` | Global namespace |
| `adb shell settings get global airplane_mode_on` | Reads one value |
| `adb shell settings put system screen_brightness 120` | Writes a value (keys vary by OEM). **Caution** |
| `adb shell settings delete system KEY` | Deletes an override |
| `adb shell settings get secure enabled_accessibility_services` | Enabled accessibility services |
| `adb shell settings get secure accessibility_enabled` | Accessibility master switch |

```bash
# Screen timeout: 10 minutes
adb shell settings put system screen_off_timeout 600000
# Turn auto-rotate back on
adb shell settings put system accelerometer_rotation 1
```

### System properties

| Command | What it does |
| --- | --- |
| `adb shell getprop KEY` | Reads a property |
| `adb shell setprop KEY VALUE` | Sets a writable runtime property. Most `ro.*` and `persist.*` properties are protected |
| `adb shell getprop \| grep -i KEYWORD` | Searches properties (macOS/Linux) |
| `adb shell resetprop ...` | Not a standard Android command. It belongs to root environments such as Magisk |

**Read-only properties.** `ro.*` properties are fixed at boot. `setprop ro.product.model ...`
won't permanently change the model on a production device.

### Display overrides

| Command | What it does |
| --- | --- |
| `adb shell wm size 1080x2400` | Overrides the display size |
| `adb shell wm size reset` | Removes the override |
| `adb shell wm density 420` | Overrides the density |
| `adb shell wm density reset` | Removes the override |
| `adb shell wm overscan 0,0,0,0` | Legacy overscan reset. Removed in Android 11 |

## 13. Screenshots, recording and input

### Capture

| Command | What it does |
| --- | --- |
| `adb exec-out screencap -p > screenshot.png` | Saves a screenshot straight to the computer |
| `adb shell screencap -p /sdcard/screen.png` | Saves it on the device |
| `adb shell screenrecord /sdcard/video.mp4` | Records the screen. Press Ctrl+C to stop |
| `adb shell screenrecord --time-limit 30 /sdcard/video.mp4` | 30-second recording |
| `adb shell screenrecord --size 720x1280 --bit-rate 4000000 /sdcard/video.mp4` | Sets resolution and bitrate (device dependent) |

### Input events

| Command | What it does |
| --- | --- |
| `adb shell input tap X Y` | Taps a screen coordinate |
| `adb shell input swipe X1 Y1 X2 Y2 DURATION_MS` | Swipes or drags |
| `adb shell input text "Hello%sWorld"` | Types text. `%s` stands for a space |
| `adb shell input keyevent KEYCODE_HOME` | Home |
| `adb shell input keyevent KEYCODE_BACK` | Back |
| `adb shell input keyevent KEYCODE_POWER` | Power |
| `adb shell input keyevent 26` | Power, by numeric key code |
| `adb shell input keyevent KEYCODE_WAKEUP` | Wakes the screen |
| `adb shell input keyevent KEYCODE_SLEEP` | Puts the screen to sleep |
| `adb shell input motionevent ...` | Low-level motion events on newer versions |

```bash
# Finding coordinates
adb shell wm size
adb shell settings put system pointer_location 1   # show the pointer overlay
adb shell settings put system pointer_location 0   # hide it again
```

## 14. Logcat, bug reports and dumpsys

### Logcat

| Command | What it does |
| --- | --- |
| `adb logcat` | Live logs |
| `adb logcat -c` | Clears the log buffers you can access |
| `adb logcat -d` | Dumps current logs and exits |
| `adb logcat -v time` | Adds timestamps |
| `adb logcat -v threadtime` | Date, time, PID, TID, priority and tag |
| `adb logcat *:E` | Errors and fatal messages only |
| `adb logcat MyTag:D *:S` | Debug and above for `MyTag`, everything else silent |
| `adb logcat --pid=PID` | Logs from one process |
| `adb logcat -b crash` | Crash buffer |
| `adb logcat -b all` | Every buffer you can access |
| `adb logcat -f /sdcard/log.txt` | Writes logs to a file on the device |

```bash
# Save logs on the computer
adb logcat -v threadtime > logcat.txt

# Filter crashes on Windows
adb logcat | findstr /i "FATAL EXCEPTION AndroidRuntime"

# Filter crashes on macOS/Linux
adb logcat | grep -Ei "FATAL EXCEPTION|AndroidRuntime"
```

### Bug reports and dumpsys

| Command | What it does |
| --- | --- |
| `adb bugreport` | Full bug report (zip location depends on version) |
| `adb bugreport OUTPUT_PATH` | Bug report to a chosen path |
| `adb shell dumpsys -l` | Lists system services |
| `adb shell dumpsys SERVICE` | Diagnostics for one service |
| `adb shell dumpsys activity` | ActivityManager state |
| `adb shell dumpsys package PACKAGE` | Package details |
| `adb shell dumpsys window` | WindowManager state |
| `adb shell dumpsys input` | Input devices and state |
| `adb shell dumpsys meminfo PACKAGE` | App memory snapshot |
| `adb shell dumpsys procstats --hours 3` | Process stats for the last 3 hours |
| `adb shell dumpsys batterystats --charged PACKAGE` | Battery stats for a package since the last full charge |
| `adb shell dumpsys jobscheduler` | Scheduled jobs |
| `adb shell dumpsys alarm` | AlarmManager state |
| `adb shell dumpsys notification` | Notification service state |
| `adb shell dumpsys connectivity` | Connectivity diagnostics |
| `adb shell dumpsys wifi` | Wi-Fi diagnostics (some fields redacted) |
| `adb shell dumpsys telephony.registry` | Telephony state (OEM and privacy limits) |

**Privacy.** Bug reports and logcat can contain account identifiers, package names, network
details and app logs. Review and redact them before you share one.

## 15. Networking, forward and reverse

### Port forwarding

| Command | What it does |
| --- | --- |
| `adb forward tcp:LOCAL tcp:REMOTE` | Forwards a computer port to a device port |
| `adb forward tcp:9222 localabstract:chrome_devtools_remote` | Chrome/WebView remote debugging (socket name can differ) |
| `adb forward --list` | Lists forwards |
| `adb forward --remove tcp:LOCAL` | Removes one forward |
| `adb forward --remove-all` | Removes all forwards |
| `adb reverse tcp:DEVICE_PORT tcp:HOST_PORT` | Maps a device localhost port to a computer port |
| `adb reverse --list` | Lists reverse mappings |
| `adb reverse --remove tcp:DEVICE_PORT` | Removes one mapping |
| `adb reverse --remove-all` | Removes all mappings |

```bash
# Let an app on the phone reach a dev server on your computer at http://127.0.0.1:3000
adb reverse tcp:3000 tcp:3000
adb reverse --list
```

### Network checks

| Command | What it does |
| --- | --- |
| `adb shell ping -c 4 8.8.8.8` | Basic IP connectivity |
| `adb shell ping -c 4 example.com` | DNS plus connectivity |
| `adb shell ip addr show wlan0` | Wi-Fi interface details |
| `adb shell ip route` | Routes and default gateway |
| `adb shell ss -tunap` | Sockets and owning processes (access dependent) |
| `adb shell netstat -an` | Legacy socket list, if available |
| `adb shell dumpsys netstats` | Network statistics |
| `adb shell dumpsys connectivity` | Network capabilities and transports |

## 16. Processes, memory, CPU and performance

| Command | What it does |
| --- | --- |
| `adb shell ps -A` | Every process |
| `adb shell pidof PACKAGE` | PID(s) for a package |
| `adb shell top -o PID,CPU,RES,ARGS -n 1` | One CPU and memory snapshot |
| `adb shell kill PID` | Signals a process (shell permission limits apply) |
| `adb shell kill -9 PID` | Force-kills a process. **Caution** |
| `adb shell dumpsys meminfo PACKAGE` | Memory breakdown |
| `adb shell dumpsys cpuinfo` | CPU usage summary |
| `adb shell cat /proc/PID/status` | Kernel view of a process |
| `adb shell cat /proc/PID/maps` | Memory mappings (access restricted) |
| `adb shell am profile start PACKAGE /sdcard/profile.trace` | Legacy method profiling (debuggable apps) |
| `adb shell am profile stop PACKAGE` | Stops profiling |
| `adb shell perfetto ...` | System-wide performance trace (needs a config) |

```bash
# Focused diagnostics for one app
adb shell pidof com.example.app
adb shell dumpsys meminfo com.example.app
adb shell dumpsys package com.example.app
adb logcat --pid=$(adb shell pidof com.example.app)   # macOS/Linux shells
```

In PowerShell, assign the PID to a variable first: `$p = adb shell pidof com.example.app`, then
`adb logcat --pid=$p`.

## 17. Battery, power and thermal

| Command | What it does |
| --- | --- |
| `adb shell dumpsys battery` | Level, status, voltage and temperature |
| `adb shell dumpsys battery set level 20` | Fakes the battery level for testing |
| `adb shell dumpsys battery set status 3` | Fakes the charging status (platform-defined values) |
| `adb shell dumpsys battery unplug` | Pretends the charger is unplugged |
| `adb shell dumpsys battery reset` | Clears every battery override. Always run this after testing |
| `adb shell dumpsys batterystats` | Battery usage statistics |
| `adb shell dumpsys batterystats --reset` | Clears collected battery stats. **Caution** |
| `adb shell dumpsys deviceidle` | Doze state |
| `adb shell dumpsys deviceidle force-idle` | Forces Doze for testing |
| `adb shell dumpsys deviceidle unforce` | Releases forced Doze |
| `adb shell cmd deviceidle whitelist +PACKAGE` | Adds an app to the Doze allowlist |
| `adb shell cmd deviceidle whitelist -PACKAGE` | Removes it |
| `adb shell dumpsys thermalservice` | Thermal status (version and OEM dependent) |
| `adb shell cmd power set-fixed-performance-mode-enabled true` | Fixed performance mode for benchmarks (thermal risk) |

**Reset after testing.** Undo battery, Doze and fixed-performance overrides when you're done.
Left in place, they make the device behave oddly and drain the battery.

## 18. Reboot, recovery, bootloader and sideload

| Command | What it does |
| --- | --- |
| `adb reboot` | Normal reboot |
| `adb reboot recovery` | Reboots into recovery |
| `adb reboot bootloader` | Reboots into the bootloader / Fastboot on supported devices |
| `adb reboot fastboot` | Requests fastbootd (userspace fastboot) on supported devices |
| `adb reboot download` | Samsung Download Mode (OEM dependent) |
| `adb sideload update.zip` | Sends a signed OTA package while recovery is in ADB sideload mode |
| `adb reboot sideload` | Asks a supported recovery to enter sideload mode |

**Don't mix up the modes.** Recovery, bootloader Fastboot, fastbootd and Samsung Download Mode
are separate environments, each with its own set of commands.

Fastboot commands are not ADB commands:

```bash
fastboot devices
fastboot getvar all
fastboot reboot
```

**Flashing.** Partition flashing and bootloader unlocking are outside the scope of this guide.
Never flash without the official image for the exact model and bootloader revision.

## 19. Testing and instrumentation

| Command | What it does |
| --- | --- |
| `adb shell am instrument -w PACKAGE.test/androidx.test.runner.AndroidJUnitRunner` | Runs instrumentation tests |
| `adb shell am instrument -w -e class com.example.TestClass PACKAGE.test/RUNNER` | One test class |
| `adb shell am instrument -w -e class com.example.TestClass#testMethod PACKAGE.test/RUNNER` | One test method |
| `adb shell am instrument -w -e package com.example.feature PACKAGE.test/RUNNER` | Tests in one package |
| `adb shell am instrument -w -e annotation com.example.Smoke PACKAGE.test/RUNNER` | Tests with one annotation |
| `adb shell uiautomator dump /sdcard/window.xml` | Dumps the current UI hierarchy |
| `adb pull /sdcard/window.xml` | Copies it to the computer |
| `adb shell uiautomator runtest test.jar -c CLASS` | Legacy UI Automator JAR tests |
| `adb shell monkey -p PACKAGE 500` | 500 random UI events. Use a test device |

```bash
# Reinstall with all permissions, then launch with a clean log
adb install -r -g app-debug.apk
adb shell monkey -p com.example.app -c android.intent.category.LAUNCHER 1
adb logcat -c
adb logcat -v threadtime
```

## 20. Root, remount and debug builds

These mostly work on emulators and on `eng` or `userdebug` builds. On retail phones they
normally fail.

| Command | What it does |
| --- | --- |
| `adb root` | Restarts adbd as root (refused on production builds) |
| `adb unroot` | Restarts adbd as the normal user |
| `adb remount` | Remounts system and vendor as writable on supported debug builds. **Danger** |
| `adb disable-verity` | Disables Verified Boot enforcement on debug/unlocked builds. **Danger** |
| `adb enable-verity` | Turns verity back on |
| `adb sync` | Syncs AOSP build output to device partitions |
| `adb sync system` | Syncs one partition |
| `adb shell su -c "COMMAND"` | Runs a command as root through a third-party root solution |

**Boot and security risk.** Turning off Verified Boot, remounting, rooting and changing
partitions can make OTAs fail, cause a bootloop, wipe data or weaken security. Don't try them on
a phone you depend on.

## 21. Modem, baseband and EFS/NV notes

**What the terms mean.** Baseband or modem firmware is the executable firmware of the cellular
processor. EFS/NV is vendor-specific persistent storage for radio configuration, calibration,
certificates and identity-related data. Neither is ordinary Android user storage.

### Safe read-only checks

| Command | What it does |
| --- | --- |
| `adb shell getprop gsm.version.baseband` | Reported baseband version |
| `adb shell getprop \| grep -Ei "baseband\|radio\|ril\|gsm"` | Radio-related properties (macOS/Linux/Git Bash) |
| `adb shell getprop \| findstr /i "baseband radio ril gsm"` | Windows CMD equivalent |
| `adb shell dumpsys telephony.registry` | Registration and telephony state |
| `adb shell dumpsys phone` | Phone service diagnostics, where available |
| `adb shell service list \| grep -Ei "phone\|radio\|telephony"` | Related Binder services |
| `adb shell ls -l /dev/block/by-name` | Partition names (access varies) |
| `adb shell ls -l /dev/block/by-name \| grep -Ei "modem\|radio\|efs\|nv\|persist"` | Likely radio and EFS partition names, listing only |

**There's no universal ADB write command for this.** ADB doesn't understand modem NV/EFS layouts,
and Qualcomm, MediaTek, Exynos and each OEM implement them differently. A generic
`adb shell setprop` won't permanently change an IMEI, the baseband or NV values.

### Samsung note

```bash
# Read the baseband version
adb shell getprop gsm.version.baseband

# Enter Download Mode (supported Samsung builds)
adb reboot download
```

In Samsung firmware packages the modem/baseband usually ships as the CP file, and official
service flashing goes through Odin and Download Mode. Check the exact SM model number, region/CSC
and bootloader binary before flashing anything.

**Never write random EFS/NV images.** Restoring another phone's EFS/NV backup, writing a random
partition image or overwriting a block device directly can corrupt network certificates,
calibration and identity. The results include *Baseband Unknown*, a null IMEI, no SIM, no service
or a phone that won't boot.

**Not included on purpose:** IMEI changing or cloning, carrier lock bypass, FRP bypass, access to
stolen devices, diagnostic-port abuse and raw EFS/NV writes. For a legitimate repair, use the
manufacturer's authorized service software and that device's own backup and restore process.

## 22. Troubleshooting

| Command | What it does |
| --- | --- |
| `adb devices -l` | First check: device state |
| `adb kill-server && adb start-server` | Restarts the server (macOS/Linux/Git Bash) |
| `adb kill-server & adb start-server` | The same in Windows CMD |
| `adb reconnect offline` | Resets offline transports |
| `adb usb` | Switches adbd from TCP back to USB |
| `adb disconnect` | Disconnects wireless devices |
| `adb nodaemon server` | Runs the server in the foreground to watch its output |
| `adb server-status` | Server status on newer versions |

### What the states mean

| State | Meaning |
| --- | --- |
| `device` | Authorized and ready |
| `unauthorized` | The RSA prompt on the phone hasn't been accepted |
| `offline` | The transport exists but isn't responding |
| `no permissions` (Linux) | udev rules or permissions problem |
| `more than one device/emulator` | Pick one with `-s SERIAL`, `-d` or `-e` |

**USB checklist, in order:** unlock the phone, try File Transfer USB mode, try a different data
cable, try a different USB port, revoke and re-accept RSA authorization, update Platform-Tools,
install the OEM or Google USB driver on Windows, restart the ADB server.

```bash
# Re-authorize from scratch
adb kill-server
# On the phone: Developer options > Revoke USB debugging authorizations
# Unplug and replug the cable, unlock the phone, accept the RSA prompt
adb start-server
adb devices -l
```

## 23. Ready-made recipes

**A. Full app debugging bundle**

```bash
adb devices -l
adb shell getprop ro.build.fingerprint > build.txt
adb shell dumpsys package com.example.app > package.txt
adb shell dumpsys meminfo com.example.app > meminfo.txt
adb logcat -d -v threadtime > logcat.txt
adb bugreport bugreport.zip
```

**B. Clean reinstall**

```bash
adb uninstall com.example.app
adb install -g app-debug.apk
adb shell monkey -p com.example.app -c android.intent.category.LAUNCHER 1
```

**C. Update and keep data**

```bash
adb install -r app-debug.apk
adb shell am force-stop com.example.app
adb shell monkey -p com.example.app -c android.intent.category.LAUNCHER 1
```

**D. Export a debuggable app's database**

```bash
adb exec-out run-as com.example.app cat databases/app.db > app.db
```

**E. Point a phone app at a server on your computer**

```bash
adb reverse tcp:3000 tcp:3000
adb reverse --list
# the app now reaches http://127.0.0.1:3000
```

**F. Capture a crash**

```bash
adb logcat -c
adb shell am force-stop com.example.app
adb shell monkey -p com.example.app -c android.intent.category.LAUNCHER 1
adb logcat -v threadtime AndroidRuntime:E *:S
```

**G. Find the foreground activity**

```bash
adb shell dumpsys activity activities | grep -E "mResumedActivity|topResumedActivity"
# Windows CMD
adb shell dumpsys activity activities | findstr /i "mResumedActivity topResumedActivity"
```

**H. Screenshot plus UI hierarchy**

```bash
adb exec-out screencap -p > screenshot.png
adb shell uiautomator dump /sdcard/window.xml
adb pull /sdcard/window.xml .
```

## 24. Cheat sheet

| Command | What it does |
| --- | --- |
| `adb devices -l` | List devices |
| `adb -s SERIAL shell` | Shell on one device |
| `adb install -r -g app.apk` | Install or update with runtime permissions |
| `adb uninstall PACKAGE` | Uninstall |
| `adb push LOCAL REMOTE` | Upload |
| `adb pull REMOTE LOCAL` | Download |
| `adb shell pm list packages -3` | Third-party packages |
| `adb shell pm clear PACKAGE` | Clear app data |
| `adb shell am force-stop PACKAGE` | Force stop |
| `adb logcat -v threadtime` | Live detailed logs |
| `adb bugreport report.zip` | Full bug report |
| `adb exec-out screencap -p > screen.png` | Screenshot |
| `adb shell screenrecord /sdcard/a.mp4` | Screen recording |
| `adb reverse tcp:3000 tcp:3000` | Phone localhost to computer dev server |
| `adb pair IP:PORT` | Wireless pairing |
| `adb connect IP:PORT` | Wireless connect |
| `adb reboot recovery` | Recovery |
| `adb reboot bootloader` | Bootloader |
| `adb reboot download` | Samsung Download Mode |
| `adb shell getprop gsm.version.baseband` | Baseband version |

## 25. Official references

- [Android Debug Bridge (adb)](https://developer.android.com/tools/adb)
- [Logcat command-line tool](https://developer.android.com/tools/logcat)
- [dumpsys](https://developer.android.com/tools/dumpsys)
- [SDK Platform-Tools release notes and download](https://developer.android.com/tools/releases/platform-tools)
- [Run apps on a hardware device](https://developer.android.com/studio/run/device)
- [Test from the command line](https://developer.android.com/studio/test/command-line)
- [Perfetto command-line tool](https://developer.android.com/tools/perfetto)
- [AOSP debugging](https://source.android.com/docs/core/tests/debug)

## License

[CC BY 4.0](LICENSE). Share and adapt it with credit.
