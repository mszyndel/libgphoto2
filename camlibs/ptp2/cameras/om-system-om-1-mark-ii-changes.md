# OM System OM-1 Mark II — libgphoto2 changes

This document describes the work done to improve tethering, capture, and property
support for the **OM System OM-1 Mark II** (`USB 0x33a2:0x0136`) in the ptp2
camlib. The implementation is based on USB traffic captured from **OM Capture**
(`om1.pcapng`) and validated with local debug logs.

## Background

OM-1 generation bodies use vendor extension ID `0xfffd` (`PTP_VENDOR_GP_OLYMPUS_OMD`)
and a set of vendor opcodes (`0x948x`, `0x94xx`, `0x0400`) that differ from older
Olympus PEN/E-M bodies. OM Capture performs a multi-step init handshake before
tethering works reliably. Without that handshake, gphoto2 could connect but
capture, liveview, and `--summary` were unreliable or slow.

## Reference files (not compiled)

| File | Purpose |
|------|---------|
| `om-system-om-1-mark-ii-om-capture-timeline.txt` | Filtered opcode timeline from `om1.pcapng` (polling noise removed) |
| `../olympus-omd-init-data.h` | Property lists and `0x0400` bulk payload extracted from the pcap |
| `om1.pcapng` (repo root) | Original Wireshark capture |

## Source files changed

| File | Summary of changes |
|------|-------------------|
| `ptp.h` | New opcodes, `PTPParams` flags, function declarations |
| `ptp.c` | OMD init/post-connect, liveview, trigger/capture, property name table |
| `usb.c` | `ptp_usb_sendvendorbulk()` for non-standard `0x0400` container |
| `olympus-omd-init-data.h` | **New** — pcap-derived init payloads |
| `library.c` | Init wiring, capture/trigger/preview paths, summary fixes, USB mode warnings |

---

## 1. Vendor identification and property names

### `fixup_cached_deviceinfo()` (`library.c`)

OM System cameras report manufacturer `OMSYSTEM` (not `OLYMPUS`) and often expose
vendor extension `0xffffffff` in `GetDeviceInfo`. For models starting with `OM-`
or `E-M`, the code now sets:

```c
di->VendorExtensionID = PTP_VENDOR_GP_OLYMPUS_OMD;  /* 0xfffd */
```

This routes the camera through the OMD code paths.

### `ptp_device_properties_Olympus[]` (`ptp.c`)

A large property-name table was added/extended for OMD properties (`0xD000`–`0xD1xx`).
`ptp_get_property_description()` now resolves names for both `PTP_VENDOR_GP_OLYMPUS`
and `PTP_VENDOR_GP_OLYMPUS_OMD`.

### `--summary` robustness (`library.c`)

- `APPEND_*` macros use bounded `SPACE_LEFT` to avoid buffer overruns.
- Enumeration values in summary output are capped at 64 entries (with `"... (N values)"`).
- Property loop exits early when the summary buffer is full.

---

## 2. OM Capture init handshake

The handshake was reverse-engineered from the pcap and implemented in three layers.

### 2.1 Startup registration — `ptp_olympus_omd_init()`

Called from `camera_init()` for `PTP_VENDOR_GP_OLYMPUS_OMD` bodies:

1. `0x9489` — register 267 monitored properties (startup list)
2. `0x9486` — query changed properties
3. `0x948b` — register 17 liveview-related properties
4. `0x948a` — poll properties

Sets `params->olympus_omd_registered`.

### 2.2 PC mode switch — `ptp_olympus_init_pc_mode()`

1. Read `0xD052` (Camera Control Mode). If already `1` (PC mode), skip mode
   switch and run post-connect only.
2. Set `0xD0DC` (Capture Target) = `2`.
3. Set `0xD052` = `1` (PC mode). Camera takes ~1.5–3.5 s to complete.
4. Drain events (up to 80 × 50 ms).
5. Call `ptp_olympus_omd_post_connect()`.

**Error handling:** Only failure to set `0xD052` is fatal. Post-connect steps log
warnings but do not fail init.

### 2.3 Post-connect — `ptp_olympus_omd_post_connect()`

Mirrors OM Capture’s tether sequence:

| Step | Opcode | Notes |
|------|--------|-------|
| Changed props | `0x9486` | GETDATA |
| Batch property set | `0x0400` | 216-byte bulk container; **fire-and-forget** |
| Tether props | `0x9489` | 43 properties |
| Tether LV props | `0x948b` | 15 properties |
| AF target frames | `0x94c4` | GETDATA (~16 KB); discard payload |
| Unknown | `0x94dc` | GETDATA (returns `"OMSYS"` string) |
| Serial / session | `0xD176` | GETDATA (firmware build string) |

Sets `params->olympus_omd_post_connected`. Skipped if already connected.

### 2.4 `0x0400` batch set — critical fix

OM Capture sends `0x0400` and continues within ~8 ms **without waiting for a
response**. An early implementation blocked on `getresp()` for 20 s (USB timeout).

**Fix:** `ptp_olympus_omd_batch_set_properties()` writes the bulk container via
`ptp_usb_sendvendorbulk()` and returns after a 10 ms pause. No response read.

### 2.5 `0x94c4` / `0x94dc` — data-phase fix

Both opcodes return a **data packet**, not a simple OK response.

| Opcode | Wrong | Correct |
|--------|-------|---------|
| `0x94c4` | `PTP_DP_NODATA` → bulk desync, session hang | `PTP_DP_GETDATA`, discard ~16 KB |
| `0x94dc` | `PTP_DP_NODATA` → spurious retry | `PTP_DP_GETDATA`, discard `"OMSYS"` |

Using `NODATA` left unread bytes on the bulk endpoint and caused 20 s write
timeouts on subsequent commands.

### 2.6 USB bulk helper — `ptp_usb_sendvendorbulk()` (`usb.c`)

Sends a pre-built USB bulk container (used for `0x0400`) without going through
the normal request/data/response transaction helpers.

---

## 3. PC mode exit

### `ptp_olympus_exit_pc_mode()` (`ptp.c`)

Called from `camera_exit()` for OMD bodies:

- Restores `0xD052` to the value saved before PC mode (typically `2` = standalone).
- Clears `olympus_omd_post_connected`.

**Note:** Exiting gphoto2 (or shell) restores standalone mode. The next connect
pays the full `D052=1` mode-switch delay (~1.6 s). OM Capture stays tethered.
See [Performance](#performance) below.

---

## 4. Capture and trigger

### Trigger — `camera_trigger_capture()` / `ptp_olympus_omd_trigger()`

Uses vendor opcode `0x9481`:

| Param | Meaning |
|-------|---------|
| `3` | Shutter down |
| `6` | Shutter up |

`disable_liveview()` is skipped when `params->inliveview` is false (saves a
`0xD06D` round-trip on trigger).

### Capture — `camera_olympus_omd_capture()` (`library.c`)

1. Disable liveview if active.
2. Snapshot object handles before capture.
3. `ptp_olympus_omd_capture()` — `0x9481` (3+6) then `0x9486`.
4. Wait up to 35 s for `ObjectAdded` / `CaptureComplete` events.
5. **Fallback:** `0x9485` (`ptp_olympus_sdram_image`) after 700 ms — SDRAM JPEG
   path used by OM Capture when card events are slow.
6. **Fallback:** diff object-handle list against pre-capture snapshot.
7. Re-enable liveview after successful download.

Card captures produce ORF files (format `0x3800`). Object-handle polling has been
observed to succeed before the `0x9485` path runs.

### Liveview / preview

- **Enable:** `0xD06D` = `0x04000300`, then poll events.
- **Preview:** `ptp_olympus_liveview_image()` with retry loop (25 tries, 40 ms).
- **Disable:** `0xD06D` = `0`.

---

## 5. MTP USB mode detection

`olympus_omd_in_mtp_usb_mode()` detects when the camera is in basic MTP/file
transfer mode (no `0xD002` Aperture property, no `0x9481` opcode). User gets a
status message advising **PC/RAW/Control** USB mode on the camera body.

---

## 6. `camera_init()` wiring (`library.c`)

For `PTP_VENDOR_GP_OLYMPUS_OMD`:

```c
ptp_olympus_omd_init(params);        /* 9489/948b registration */
ptp_olympus_init_pc_mode(params);    /* D0DC + D052 + post-connect */
ptp_getstorageids(...);              /* refresh storage after mode switch */
```

Olympus init runs **after** filesystem folder listing (matching OM Capture ordering).

---

## 7. Key properties

| Code | Name | Role |
|------|------|------|
| `0xD052` | Camera Control Mode | `1` = PC/tether, `2` = standalone |
| `0xD0DC` | Capture Target | Set to `2` before PC mode |
| `0xD06D` | Live View Mode | `0x04000300` = on, `0` = off |
| `0xD008` | Exposure Compensation | Values ÷ 1000 for EV |
| `0xD176` | (unknown) | Firmware/build string read at post-connect |
| `0x9481` | OMD Capture | Params `1`/`5` half-press, `3`/`6` shutter |

---

## 8. Performance

Measured on OM-1 Mark II with local libs (`DYLD_LIBRARY_PATH` + `CAMLIBS`).

### One-shot `gphoto2 --trigger-capture`

| Phase | Time |
|-------|------|
| Camlib scan | ~0.4 s |
| `D052=1` mode switch | ~1.6 s (camera firmware) |
| Post-connect handshake | ~65 ms |
| Trigger (`9481` ×2) | ~7 ms |
| **Total (cold connect)** | **~1.8–2 s** |

Down from ~24 s before the `0x0400` and `0x94c4` fixes.

### `gphoto2 --shell` (recommended for repeat triggers)

| Phase | Time |
|-------|------|
| First `trigger-capture` | ~1.8 s (full init once) |
| Subsequent triggers | **~2–10 ms** each |
| Exit | Restores `D052=2` |

Shell keeps the driver loaded and skips re-init between commands. Repeat trigger
speed matches OM Capture.

### vs OM Capture

OM Capture stays connected; gphoto2 restores standalone mode on exit. Remaining
gap on **cold connect** is almost entirely the inherent `D052` mode-switch delay.

---

## 9. Testing

Build and run with local libraries (Homebrew gphoto2 stubs `gp_log()`):

```bash
# build
cd /path/to/libgphoto2 && make -j4

# env for local camlib + libgphoto2
export DYLD_LIBRARY_PATH=libgphoto2_port/libgphoto2_port/.libs:libgphoto2/.libs
export CAMLIBS=camlibs/.libs

# one-shot trigger
gphoto2 --debug --debug-logfile=gphoto-trigger.log --trigger-capture

# shell (repeat triggers)
gphoto2 --shell
> trigger-capture
> trigger-capture
> quit
```

Camera USB mode must be **PC / RAW / Control**, not MTP-only transfer.

---

## 10. Known limitations and future work

| Item | Notes |
|------|-------|
| Duplicate startup `9486`/`948a` | Also run at post-connect; could trim ~20–30 ms |
| `exit_pc_mode` on every quit | Forces cold `D052` switch next session; optional “stay tethered” mode |
| `0xC108` property-changed events | Logged as unexpected during capture; harmless but noisy |
| `0x94d9` GetLocalObject | Declared; not yet used in capture path |
| ORF mime `0x3800` | Download works; mime mapping warns “unknown” |
| Half-press / AF | `0x9481` params `1`/`5` implemented (`ptp_olympus_omd_half_press`) but not wired to gphoto2 UI |
| Background polling | OM Capture polls `948a`/`9486`/`9484` every ~17 ms; not implemented (not required for basic trigger) |

---

## 11. Debug logs (development)

Logs used during development (repo root):

| Log | Command |
|-----|---------|
| `gphoto-init.log` | `gphoto2 --summary` |
| `gphoto-trigger-capture.log` | `gphoto2 --trigger-capture` (before fixes) |
| `gphoto-trigger3.log` | `gphoto2 --trigger-capture` (after fixes) |
| `gphoto-shell.log` | `gphoto2 --shell` + repeated `trigger-capture` |
| `gphoto-capture-image-and-download.log` | `gphoto2 --capture-image-and-download` |

---

## 12. Opcode quick reference

| Opcode | Name | Direction |
|--------|------|-----------|
| `0x0400` | OMD_BatchSetProperties | Host → camera (bulk, no response) |
| `0x9481` | OMD_Capture | NODATA |
| `0x9485` | OMD_GetImage | GETDATA (SDRAM JPEG) |
| `0x9486` | OMD_ChangedProperties | GETDATA |
| `0x9489` | OMD_SetProperties | SENDDATA |
| `0x948a` | OMD_PollProperties | GETDATA |
| `0x948b` | OMD_SetPropertiesLv | SENDDATA |
| `0x94c4` | OMD_GetAfTargetFrames | GETDATA |
| `0x94dc` | OMD_Unknown_94dc | GETDATA |
| `0x94d9` | OMD_GetLocalObject | GETDATA |
