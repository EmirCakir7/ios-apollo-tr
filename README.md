# ios-apollo-notes

Field notes from running iOS 16.7 (iPhone X userland on patched xnu-8796) on a Xiaomi Mi 10T
(apollo, SM8250), built on top of [aTechGeekDev/ios-apollo](https://github.com/aTechGeekDev/ios-apollo)
through the fork [EmirCakir7/ios-apollo](https://github.com/EmirCakir7/ios-apollo).
These are findings and open problems, not a how-to. Everything here was measured on one phone;
treat it as a lead, not as a fact about your device.

## Idle / power saving (works)
- Patch 0197 `apollo_idle=1` lowers the polling that kept every core 40-70 % busy with the screen off.
- Patch 0198 adds DDR vote hysteresis: raise after 3 busy samples, lower after 10 quiet ones.
- Real battery state of charge comes from the fuel gauge (`sweetbattery_fg=1`); charging limit `apollo_chg=30` (about 12 W).
- Not yet measured: an overnight battery test with the screen off.

## Weather shows "no data" (not fixable here)
Weather asks the FairPlay daemon for a hardware signature (`AMSFairPlayGetHardwareInfo`, "Mescal Failed").
That is a device-bound attestation. With `fairplayd.H2` disabled the call fails at once; with it enabled it
exits at launch and launchd respawns it forever. We do not emulate it.

## Siri fails immediately
Speech assets are missing: Trial / MobileAsset (error 1601) and
`/System/Library/Assistant/SignalProcessingPlugins`. Microphone and speakers work.

## Voice Memos / "cannot return to the home screen" after a recording (open)
UIKit's system-gesture gate waits for a `systemGestureStateChange` HID event from backboardd. After a
recording ends VoiceMemos logs "System gesture gate timed out" (27 times, only this app). Root cause unknown.
The "Unbalanced calls to begin/end appearance transitions for SBSwitcherViewController" line is benign.
`apollo_syslog=1` (patch 0201) mirrors the os_log of the relevant daemons into the kernel log for debugging.

## Battery Health screen (patch 0200, untested)
`sweetbattery_health=1` supplies a placeholder serial and capacities so Settings > Battery > Battery Health
has data. Off by default.

## Sensors (planned)
Auto-brightness needs an ALS; patch 0199 adds diagnostics. On the Mi 10T the sensors sit on the SLPI
(not the ADSP as on the Pixel 5 port), so the Pixel 5 approach (FastRPC + QMI SNS) needs a different
remoteproc, GLINK edge and QRTR node. Estimated 2-3 weeks of work.

## Camera (early bring-up)
- Patch 0202 `apollo_cam=1`: opt-in, fully undone camera clock / GDSC / CCI probe. It powers no sensor
  and talks to no I2C slave. Not yet booted.
- Prior art: Galaxy Note FE / S7 ports (front camera preview only, RAW10 through a userland capture device in
  mediaserverd, CPU debayer, about 15 fps), S9 ports (HakiOS camera drivers). A Redmi/Xiaomi popsicle port
  documents the camera at 0 %. No public Snapdragon iOS camera pipeline was found.
- iOS 16 traps seen elsewhere: IOSurface plane bytes-per-row at +0x10 (fixed here by patch 0192),
  `DisplayPipePlaneBaseAlignment` must be a dictionary, `BWAudioSourceNode _generateSamples` zero-fills
  about 820 MB and gets jetsam-killed.

## Apps
Apps can be packed into the boot image (`tools/app-pack.sh`, limit 64 files / 48 MiB). App Store apps are
FairPlay-encrypted, so they need decrypted IPAs. TrollStore does not support 16.7.x.

## Cellular / calls (long term)
Prior art on a Pixel 5 port: fake baseband in the kernel, a QMUX/QMI router, voice over the ADSP. VoLTE/IMS
is not documented anywhere we found, and emergency calls must not be promised. Months of work.

## Practical notes
- Forks get their GitHub Actions disabled by GitHub; re-enable them on the Actions tab.
- Boot images are flashed with `fastboot boot` (temporary) or `fastboot flash boot` (permanent, keep a backup).
- Device dumps, firmware, logs and serial/IMEI data are deliberately not published.
