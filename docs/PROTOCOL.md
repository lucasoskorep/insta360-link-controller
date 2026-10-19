# Insta360 Link / Link 2 / Link 2C USB & UVC Protocol Specification

This document provides the complete, byte-accurate hardware control protocol specification for the **Insta360 Link**, **Insta360 Link 2**, **Insta360 Link 2C**, and **Insta360 Link 2 Pro (PUC3)** webcam family.

All protocol structures, USB descriptors, UVC Extension Unit GUIDs, control selectors, sub-opcodes, payload layouts, and state machines documented here were reverse-engineered directly from the official macOS binary:

- **Binary Path**: `/Applications/Insta360 Link Controller.app/Contents/MacOS/Webcam-desktop`
- **Application Version**: `2.0.2` (`arm64` Mach-O executable)
- **SHA-256 Digest**: `41db6e352223f06fa2fd70989a4d2faf622fd9c14b16a84da2aff39c0dcff22e`

---

## 1. USB Device Identification & Firmware Images

All Insta360 Link family cameras enumerate under Arashi Vision's USB Vendor ID (`0x2e1a`). Device discovery is implemented in `MacUVCCameraControlImp::GetUVCCameraList` (`0x100191478`) and `UVCCameraDeviceWatcherImp::StartWatch` (`0x10019e29c`).

- **USB Vendor ID (`idVendor`)**: `0x2e1a` (`11802` decimal — Arashi Vision Inc. / Insta360)

### 1.1 Product IDs (`idProduct`) and Firmware Targets

Verified at `0x100191480` (`0x4c01..=0x4c06`), `0x100191664` (`0x00c2`), `Webcam::CameraInsta::GetCameraModel()` (`0x10005f92c`), `UVCCameraExtendControllerImp::FirmwareUpgrade` jump table (`0x10286f37d`), and firmware path table (`0x1001a0a0c`):

| PID (`idProduct`) | Camera Model | Internal Code | Physical Gimbal | `SetVideoMode` Payload | Virtual MSC Firmware Target Path |
| :--- | :--- | :--- | :---: | :---: | :--- |
| `0x4c01` | Insta360 Link (1st Gen) | `link1` | Yes (3-axis) | 8 bytes | `a:\Insta360WebCamFW.bin` |
| `0x4c02` | Insta360 Link 2 Pro (PUC3 Gimbal A) | `link2_gimbal` | Yes (2-axis) | 8 bytes | `a:\Insta360PUC3FW_GIMBAL.bin` |
| `0x4c03` | Insta360 Link 2C (Fixed-Lens 4K) | `link2_main` | No | 36 bytes | `a:\Insta360LINK2CFW_HOST.bin` |
| `0x4c04` | Insta360 Link 2 (2-Axis Gimbal 4K) | `link2_gimbal` | Yes (2-axis) | 8 bytes | `a:\Insta360LINK2FW_GIMBAL.bin` |
| `0x4c05` | Insta360 Link 2C Pro (PUC3 Host) | `link2_main` | No | 36 bytes | `a:\Insta360PUC3FW_HOST.bin` |
| `0x4c06` | Insta360 Link 2 Pro (PUC3 Gimbal B) | `link2_gimbal` | Yes (2-axis) | 8 bytes | `a:\Insta360PUC3FW_GIMBAL.bin` |
| `0x00c2` | Insta360 Link Simple-Class / DFU | `link1` | — | 36 bytes | `a:\Insta360WebCamFW.bin` |

> **`SetVideoMode` Payload Length Note (`0x1001970f4`)**:
> In `MacUVCCameraExtendController::SetVideoMode`, when a 36-byte (`0x24`) `VideoModeAuxiliaryData` structure is supplied and `pid` is in `0x4c01..=0x4c06`, the binary tests bit `(pid - 0x4c01)` against the bitmask `0x2b` (`0b101011`, matching PIDs `0x4c01`, `0x4c02`, `0x4c04`, `0x4c06`). If the bit is set, the transfer length is truncated to **8 bytes** (`w4 = 8`). For gimbal-less models (`0x4c03` Link 2C and `0x4c05` PUC3 Host), the full **36-byte** (`0x24`) buffer is transmitted.

---

## 2. Per-Model Hardware Capability Matrix

Derived from `Webcam::CameraDeviceSupporter::UpdateUIStatus` (`0x10004a8b8`), the controller determines per-device feature support directly from the USB Product ID:

| Capability Flag | Condition in Binary (`0x10004a8b8`) | `0x4c01` Link | `0x4c02` PUC3 Gimbal A | `0x4c03` Link 2C | `0x4c04` Link 2 | `0x4c05` PUC3 Host | `0x4c06` PUC3 Gimbal B |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| `has_physical_gimbal` | `pid != 0x4c03 && pid != 0x4c05` | Yes | Yes | No | Yes | No | Yes |
| `support_ptz_drag` | `pid != 0x4c03 && pid != 0x4c05` | Yes | Yes | No | Yes | No | Yes |
| `support_auto_framing` | `pid != 0x4c01` | No | Yes | Yes | Yes | Yes | Yes |
| `support_highlighting` | `pid != 0x4c01` | No | Yes | Yes | Yes | Yes | Yes |
| `support_mirror_vertically` | `pid != 0x4c01` | No | Yes | Yes | Yes | Yes | Yes |
| `support_track_speed` | `pid != 0x4c03 && pid != 0x4c05` | Yes | Yes | No | Yes | No | Yes |
| `support_image_template` | `pid != 0x4c01` | No | Yes | Yes | Yes | Yes | Yes |
| `support_privacy_mode` | `pid != 0x4c01` | No | Yes | Yes | Yes | Yes | Yes |
| `support_area_track` | `(pid & 0xfffd) == 0x4c04` | No | No | No | Yes | No | Yes |
| `support_roll_adjustment` | `pid == 0x4c01` | Yes | No | No | No | No | No |
| `support_ai_denoise` | `pid != 0x4c01` | No | Yes | Yes | Yes | Yes | Yes |
| `support_sound_wall` | `pid == 0x4c05 \|\| pid == 0x4c06` | No | No | No | No | Yes | Yes |
| `support_speak_tracking` | `(pid & 0xfffd) == 0x4c04` | No | No | No | Yes | No | Yes |
| `support_auto_track` | `pid != 0x4c03 && pid != 0x4c05` | Yes | Yes | No | Yes | No | Yes |
| `support_smart_adjustment` | `pid != 0x4c03 && pid != 0x4c05` | Yes | Yes | No | Yes | No | Yes |
| `support_audio_capture_mode` | `pid != 0x4c01` | No | Yes | Yes | Yes | Yes | Yes |
| `support_horizontal_correction` | `pid == 0x4c04` | No | No | No | Yes | No | No |

---

## 3. UVC Descriptor Discovery & Extension Unit GUIDs

In `FindControllerInterfaceForServiceObject` (`0x100190fe0`), the host walks the USB Configuration Descriptor (`bDescriptorType = 0x04` Interface Descriptor) looking for the UVC VideoControl interface:

- `bInterfaceClass = 0x0e` (`CC_VIDEO`)
- `bInterfaceSubclass = 0x01` (`SC_VIDEOCONTROL`)

Within the VideoControl interface, class-specific interface descriptors (`bDescriptorType = 0x24`, `CS_INTERFACE`) are parsed by `bDescriptorSubtype`:

1. **`VC_INPUT_TERMINAL` (`0x02`)**:
   - Checks `wTerminalType == 0x0201` (`ITT_CAMERA` at byte offset `4..6`) and records `bTerminalID` (`desc[3]`).
2. **`VC_PROCESSING_UNIT` (`0x05`)**:
   - Records `bUnitID` (`desc[3]`).
3. **`VC_EXTENSION_UNIT` (`0x06`)**:
   - Compares the 16-byte `guidExtensionCode` (`desc[4..20]`) against two proprietary Insta360 Extension Unit GUIDs.

### 3.1 Proprietary Extension Unit GUIDs

| Extension Unit | Canonical UUID String (`0x102a9d098` / `0x102a9d0b8`) | Mixed-Endian 16-Byte USB `guidExtensionCode` (`desc[4..20]`) | Default `bUnitID` |
| :--- | :--- | :--- | :---: |
| **Extension Unit 1 (`XU1`)** | `FAF1672D-B71B-4793-8C91-7B1C9B7F95F8` | `2d 67 f1 fa 1b b7 93 47 8c 91 7b 1c 9b 7f 95 f8` | `9` (`0x09`) |
| **Extension Unit 2 (`XU2`)** | `E307E649-4618-A3FF-82FC-2D8B5F216773` | `49 e6 07 e3 18 46 ff a3 82 fc 2d 8b 5f 21 67 73` | `10` (`0x0a`) |

> **Mixed-Endian UUID Conversion (`0x100191140` & `0x100191168`)**:
> Per the USB Video Class specification, the first three fields of `guidExtensionCode` (`Data1: u32`, `Data2: u16`, `Data3: u16`) are encoded in **little-endian** byte order, while `Data4: [u8; 8]` is encoded in **big-endian** (network) byte order:
> - `FAF1672D-B71B-4793-8C91-7B1C9B7F95F8` → `2D 67 F1 FA` | `1B B7` | `93 47` | `8C 91` | `7B 1C 9B 7F 95 F8`
> - `E307E649-4618-A3FF-82FC-2D8B5F216773` → `49 E6 07 E3` | `18 46` | `FF A3` | `82 FC` | `2D 8B 5F 21 67 73`

### 3.2 Default UVC Unit IDs

When UVC descriptors are not overridden, Insta360 Link family cameras use the following standard unit IDs:

| Unit / Interface | Constant | Default ID |
| :--- | :--- | :---: |
| UVC VideoControl Interface | `DEFAULT_CONTROL_INTERFACE` | `0` (`0x00`) |
| Camera Input Terminal (`CT`) | `DEFAULT_INPUT_TERMINAL_ID` | `1` (`0x01`) |
| Processing Unit (`PU`) | `DEFAULT_PROCESSING_UNIT_ID` | `2` (`0x02`) |
| Insta360 Extension Unit 1 (`XU1`) | `DEFAULT_EXTENSION_UNIT_ID` | `9` (`0x09`) |
| Insta360 Extension Unit 2 (`XU2`) | `DEFAULT_EXTENSION_UNIT2_ID` | `10` (`0x0a`) |

---

## 4. UVC Control Request Framing

All standard and extension unit commands are transmitted as USB Control Transfers to Endpoint 0 targeting the UVC VideoControl interface (`MacUVCCameraControl::SetData` at `0x100192880` and `MacUVCCameraControl::GetData` at `0x10019282c`).

### 4.1 USB Setup Packet Layout

| Field | Size | Value / Formula | Description |
| :--- | :--- | :--- | :--- |
| `bmRequestType` | `u8` | `0x21` (`SET_CUR`) or `0xa1` (`GET_*`) | Class-specific (`0x20`), Interface recipient (`0x01`), Host-to-Device (`0x00`) or Device-to-Host (`0x80`) |
| `bRequest` | `u8` | `0x01` or `0x81..0x87` | UVC Request Code (see table below) |
| `wValue` | `u16` (LE) | `(selector as u16) << 8` | Control Selector (`CS`) in high byte, `0x00` in low byte |
| `wIndex` | `u16` (LE) | `((unit_id as u16) << 8) \| (interface_num as u16)` | Entity/Unit/Terminal ID in high byte, VideoControl `bInterfaceNumber` in low byte |
| `wLength` | `u16` (LE) | `data.len() as u16` | Payload length in bytes |

### 4.2 UVC Request Codes (`bRequest`)

| Request Name | `bmRequestType` | `bRequest` | Direction | Usage |
| :--- | :---: | :---: | :---: | :--- |
| `SET_CUR` | `0x21` | `0x01` | OUT (Host → Camera) | Write control value or execute command |
| `GET_CUR` | `0xa1` | `0x81` | IN (Camera → Host) | Read current control value or status |
| `GET_MIN` | `0xa1` | `0x82` | IN (Camera → Host) | Read minimum supported value |
| `GET_MAX` | `0xa1` | `0x83` | IN (Camera → Host) | Read maximum supported value |
| `GET_RES` | `0xa1` | `0x84` | IN (Camera → Host) | Read step resolution |
| `GET_LEN` | `0xa1` | `0x85` | IN (Camera → Host) | Read payload length (`u16` LE, 2 bytes) for variable-length XU selectors |
| `GET_INFO` | `0xa1` | `0x86` | IN (Camera → Host) | Read control capability flags (1 byte) |
| `GET_DEF` | `0xa1` | `0x87` | IN (Camera → Host) | Read factory default value |

---

## 5. Standard UVC Camera Terminal (`CT_*`) & Processing Unit (`PU_*`) Controls

Implemented in `MacUVCCameraController` (`0x100193694`–`0x100195800`).

### 5.1 Camera Terminal Controls (`unit_id = input_terminal_id`, default `1`)

| Selector | UVC Control Name | Size | Type (LE) | Description & Typical Values |
| :---: | :--- | :---: | :---: | :--- |
| `0x02` | `CT_AE_MODE_CONTROL` | 1B | `u8` | Auto-Exposure mode bitmap: `1` = Manual, `2` = Auto, `4` = Shutter Priority, `8` = Aperture Priority |
| `0x04` | `CT_EXPOSURE_TIME_ABSOLUTE_CONTROL` | 4B | `u32` | Shutter speed / exposure duration in `100 µs` units (e.g. `10..1000`, default `200`) |
| `0x06` | `CT_FOCUS_ABSOLUTE_CONTROL` | 2B | `u16` | Manual focus distance (`0..100`, step `1`, default `50`) |
| `0x08` | `CT_FOCUS_AUTO_CONTROL` | 1B | `u8` | Auto-focus enable: `0` = Manual Focus, `1` = Auto Focus |
| `0x0b` | `CT_ZOOM_ABSOLUTE_CONTROL` | 2B | `u16` | Digital zoom multiplier (`100` = 1.0x .. `400` = 4.0x, step `1`, default `100`) |
| `0x0d` | `CT_PANTILT_ABSOLUTE_CONTROL` | 8B | `(i32, i32)` | `[0..4]` Pan angle in arcseconds (`-468000..+468000`), `[4..8]` Tilt angle in arcseconds (`-324000..+324000`); `1° = 3600 arcsec` |
| `0x0e` | `CT_PANTILT_RELATIVE_CONTROL` | 4B | `[i8, u8, i8, u8]` | `[0]` `pan_dir` (`-1`, `0`, `+1`), `[1]` `pan_speed`, `[2]` `tilt_dir` (`-1`, `0`, `+1`), `[3]` `tilt_speed` |
| `0x0f` | `CT_ROLL_ABSOLUTE_CONTROL` | 2B | `i16` | Gimbal roll angle in tenths of a degree (Link 1 `0x4c01` only, e.g. `-1800..+1800`) |

### 5.2 Processing Unit Controls (`unit_id = processing_unit_id`, default `2`)

| Selector | UVC Control Name | Size | Type (LE) | Description & Typical Values |
| :---: | :--- | :---: | :---: | :--- |
| `0x02` | `PU_BRIGHTNESS_CONTROL` | 2B | `i16` | Image brightness (`0..100`, step `1`, default `50`) |
| `0x03` | `PU_CONTRAST_CONTROL` | 2B | `u16` | Image contrast (`0..100`, step `1`, default `50`) |
| `0x05` | `PU_POWER_LINE_FREQUENCY_CONTROL` | 1B | `u8` | Anti-flicker frequency: `0` = Disabled, `1` = 50 Hz, `2` = 60 Hz, `3` = Auto |
| `0x07` | `PU_SATURATION_CONTROL` | 2B | `u16` | Color saturation (`0..100`, step `1`, default `50`) |
| `0x08` | `PU_SHARPNESS_CONTROL` | 2B | `u16` | Image sharpness (`0..100`, step `1`, default `50`) |
| `0x0a` | `PU_WHITE_BALANCE_TEMPERATURE_CONTROL` | 2B | `u16` | Color temperature in Kelvin (`2000..10000`, step `50`, default `5000`) |
| `0x0c` | `PU_WHITE_BALANCE_TEMPERATURE_AUTO_CONTROL` | 1B | `u8` | Auto White Balance (AWB): `0` = Manual, `1` = Auto |

---

## 6. Proprietary Extension Unit 1 (`XU1`) Selectors (`0x01` – `0x0f`)

- **GUID**: `FAF1672D-B71B-4793-8C91-7B1C9B7F95F8`
- **Default Unit ID**: `9` (`0x09`)
- **Binary Implementation**: `MacUVCCameraExtendController` (`0x100195a18`–`0x10019bd00`)

### Summary Table of `XU1` Selectors

| Selector | Constant Name | Size | Requests | Primary Function |
| :---: | :--- | :---: | :---: | :--- |
| `0x01` | `SYSTEM_AND_PRESET_CMD` | 4B / 5B | `SET_CUR` | Preset recall/add/delete, Gimbal reset, Factory default, Composition bias, Forbidden area mode |
| `0x02` | `VIDEO_MODE` | 8B / 36B | `SET_CUR`, `GET_CUR` | Video mode (`Normal`, `AutoTracking`, `WhiteBoard`, `OverheadShot`, `DeskView`, `AutoFraming`, `SmartWhiteBoard`) & ROI/pitch/zoom |
| `0x03` | `TRACKING_AND_GESTURE` | 6B / 16B | `SET_CUR`, `GET_CUR` | Hand gesture enables, AI tracking speed, Composition style, Smart Whiteboard force |
| `0x04` | `AUDIO_CONTROL` | 2B | `SET_CUR`, `GET_CUR` | Audio AI noise reduction (`0..1`) & Audio capture mode (`1..5`) |
| `0x05` | `TRACK_OBJECT_SELECT` | 10B | `SET_CUR` | Select target person/object bounding box (`UVCRect`) for AI tracking |
| `0x06` | `TRACK_OBJECT_LIST_V1` | `N * 10B` | `GET_LEN`, `GET_CUR` | Detected trackable objects bounding box array (Insta360 Link 1 `0x4c01`) |
| `0x07` | `EXPOSURE_CURVE_AND_ISO` | 259B | `SET_CUR`, `GET_CUR` | Auto/Manual exposure mode (`u8`), Manual ISO (`u16`), and 128-point tone curve (`[u16; 128]`) |
| `0x08` | `EXPOSURE_COMPENSATION` | 4B | `SET_CUR`, `GET_CUR` | Exposure compensation EV (`f32` IEEE-754 little-endian) |
| `0x09` | `WHITE_BALANCE_EXT` | 129B | `SET_CUR`, `GET_CUR` | Extended white balance mode (`u8`) and 32-channel gain array (`[u32; 32]`) |
| `0x0a` | `EXTEND_FUNC_BITMASK` | 8B | `SET_CUR`, `GET_CUR` | 15-bit Extended Function bitmask (`u32 mask`, `u32 values`) |
| `0x0b` | `PANTILT_RELATIVE_EXT` | 4B | `SET_CUR`, `GET_CUR` | Proprietary gimbal relative pan/tilt (`[pan_speed, pan_dir, tilt_speed, tilt_dir]`) |
| `0x0c` | `DEVICE_STATUS` | 4B | `GET_CUR` | Runtime device status bitflags (`u32`) |
| `0x0d` | `PTZ_VERSION_AND_ABS` | 30B | `GET_CUR` | Gimbal firmware version string (`[0..20]`) and current pan/tilt/zoom (`[20..27]`) |
| `0x0e` | `PRESET_POSITION_FLAGS` | 8B | `GET_CUR` | Populated gimbal preset slot bitmask, active slot, and max slots |
| `0x0f` | `USB_MODE_SWITCH` | 4B | `SET_CUR` | Switch to Read-Only Mass Storage Class (`[2, 0, 0, 0]`) or camera file download (`[1, 0, 0, 0]`) |

---

### Detailed `XU1` Selector Payloads

#### `XU1 0x01` — `SYSTEM_AND_PRESET_CMD` (4 or 5 Bytes, `SET_CUR`)
Multiplexed command register dispatched by sub-opcode in `byte[0]`:

| Sub-Opcode (`byte[0]`) | Method in Binary | Address | Length | Wire Payload (Hex / Layout) | Description |
| :---: | :--- | :--- | :---: | :--- | :--- |
| `0x00` | `SetCurPresetPosition(u8 index)` | `0x100199918` | 4B | `[0x00, index, 0x00, 0x00]` | Recall gimbal preset slot `index` (`0..=5`) |
| `0x01` | `AddPresetPosition(u8 index)` | `0x1001999b4` | 4B | `[0x01, index, 0x00, 0x00]` | Save current gimbal PTZ to slot `index` (`0..=5`) |
| `0x02` | `DeletePresetPosition(u8 index)` | `0x100199a54` | 4B | `[0x02, index, 0x00, 0x00]` | Delete gimbal preset slot `index` (`0..=5`) |
| `0x03` | `ResetGimbal()` | `0x100196fb0` | 4B | `[0x03, 0x00, 0x00, 0x00]` | Center/home gimbal pan, tilt, roll, and zoom |
| `0x04` | `LoadFactoryDefault()` | `0x10019701c` | 4B | `[0x04, 0x00, 0x00, 0x00]` | Reset image/camera parameters to factory defaults |
| `0x06` | `SetCompositionBias(i16 x, i16 y)` | `0x10019933c` | 5B | `[0x06, x_lo, x_hi, y_lo, y_hi]` | Set AI tracking composition offset (`x = 0x7fff, y = 0x7fff` resets bias) |
| `0x07` | `EnterTrackForbiddenAreaSetting()` | `0x10019bf74` | 4B | `[0x07, 0x01, 0x00, 0x00]` | Enter interactive tracking forbidden-area calibration mode |
| `0x08` | `ExitTrackForbiddenAreaSetting()` | `0x10019bfb4` | 4B | `[0x08, 0x01, 0x00, 0x00]` | Exit tracking forbidden-area calibration mode |

---

#### `XU1 0x02` — `VIDEO_MODE` (8 or 36 Bytes)
Verified in `SetVideoMode` (`0x100197088`) and `GetVideoMode` (`0x1001978b8`).

**`VideoMode` Enum Values (`byte[0]`, verified in `CameraInsta` at `0x1000601d4`–`0x100060234`)**:
| Value | Variant | UI Display Name | Description |
| :---: | :--- | :--- | :--- |
| `0` | `Normal` | Normal | Standard webcam mode |
| `1` | `AutoTracking` | AI Tracking | Single-person gimbal/digital AI tracking |
| `4` | `WhiteBoard` | Whiteboard | Corner-marker whiteboard keystone enhancement |
| `5` | `OverheadShot` | Overhead | Gimbal pitched straight down (-90°) for top-down view |
| `6` | `DeskView` | DeskView | Tilted trapezoid perspective correction for desktop view |
| `7` | `AutoFraming` | Auto Framing | Multi-person group framing (Link 2 / Link 2C / PUC3) |
| `8` (or `10`) | `SmartWhiteBoard` | Smart Whiteboard | Markerless AI whiteboard detection & perspective correction |

**Wire Layout (36-Byte `VideoModeAuxiliaryData`)**:
| Byte Offset | Type | Field | Description |
| :---: | :---: | :--- | :--- |
| `0` | `u8` | `mode` | `VideoMode` enum value (`0, 1, 4, 5, 6, 7, 8`) |
| `1` | `u8` | `flags` | Auxiliary mode flags (`0` by default) |
| `2..6` | `i32` LE | `rect[0]` (`x`) | ROI X or corner coordinate |
| `6..10` | `i32` LE | `rect[1]` (`y`) | ROI Y or corner coordinate |
| `10..14` | `i32` LE | `rect[2]` (`w`) | ROI Width or corner coordinate |
| `14..18` | `i32` LE | `rect[3]` (`h`) | ROI Height or corner coordinate |
| `18..20` | `i16` LE | `pitch` | DeskView / Host pitch angle (`-900..0` or `-61926` sentinel) |
| `20..22` | `u16` LE | `zoom` | Mode zoom percentage (default `100`) |
| `22..36` | `[u8; 14]` | `extra` | Mode-specific auxiliary bytes |

*Note*: On Link (`0x4c01`), Link 2 (`0x4c04`), and PUC3 Gimbal (`0x4c02`, `0x4c06`), `SET_CUR` sends only the first **8 bytes** (`[0..8]`). On Link 2C (`0x4c03`) and PUC3 Host (`0x4c05`), `SET_CUR` sends all **36 bytes**. `GET_CUR` always reads **36 bytes**.

---

#### `XU1 0x03` — `TRACKING_AND_GESTURE` (6 or 16 Bytes)
Multiplexed control for hand gestures, tracking speed, composition style, and Smart Whiteboard:

1. **`SET_CUR` (16 Bytes)** — Dispatched by `byte[0]`:
   - `byte[0] == 0x00` (`EnableAllGesture` at `0x100197b54`): `[0x00, enable_u8, 0x00, ...]` (also toggles `ExtendFunction::AllGestures` bit 4 in `XU1 0x0a`).
   - `byte[0] == 0x01` (`EnableGestureWork` at `0x100197bc8`): Reads 6 bytes `[all, ok, palm, l, v, rock]` from `GET_CUR`, sets `buf[0] = 0x01` and `buf[gesture_idx + 1] = enable_u8`, padded with zeros to 16 bytes.
   - `byte[0] == 0x03` (`ForceSetSmatrWhiteBoard` at `0x1001980d0`): `[0x03, mode_u8, 0x00, ...]`.
   - `byte[0] == 0x05` (`SetTrackSpeed` at `0x1001975bc`): `[0x05, speed_u8, 0x00, ...]`.
   - `byte[0] == 0x06` (`SetCompositionStyle` at `0x100197498`): `[0x06, style_u8, 0x00, ...]`.

2. **`GET_CUR` (6 or 16 Bytes)**:
   - `byte[0]`: Master gesture enable (`0` or `1`)
   - `byte[1]`: `Gesture::Ok` (`0`) enable (`0` or `1`)
   - `byte[2]`: `Gesture::Palm` (`1`) enable (`0` or `1`)
   - `byte[3]`: `Gesture::LShape` (`2`) enable (`0` or `1`)
   - `byte[4]`: `Gesture::VShape` (`3`) enable (`0` or `1`)
   - `byte[5]`: `Gesture::Rock` (`4`) enable (`0` or `1`)
   - `byte[6]`: `TrackSpeed` (`GetTrackSpeed` at `0x100197630`: `0` = `Slow`, `1` = `Normal`, `2` = `Fast`)
   - `byte[7]`: `CompositionStyle` (`GetCompositionStyle` at `0x10019750c`: `0` = `Head`, `1` = `HalfBody`, `2` = `WholeBody`, `3` = `Custom`)

**`Gesture` Indices (`0x100197bc8` & `setUseGesture*` at `0x100067f58`–`0x100068888`)**:
| Index | Enum Variant | Hand Gesture | Default Action |
| :---: | :--- | :--- | :--- |
| `0` | `Gesture::Ok` | 👌 OK Sign | Toggle AI Tracking / Auto Framing (`byte[1]`) |
| `1` | `Gesture::Palm` | ✋ Open Palm | Toggle AI Tracking / Auto Framing (`byte[2]`) |
| `2` | `Gesture::LShape` | 👆 L-Shape | Dynamic Zoom In/Out (`byte[3]`) |
| `3` | `Gesture::VShape` | ✌️ V-Shape (Peace) | Toggle Whiteboard Mode (`byte[4]`) |
| `4` | `Gesture::Rock` | 🤘 Rock Sign | Custom / Reserved (`byte[5]`) |

---

#### `XU1 0x04` — `AUDIO_CONTROL` (2 Bytes)
Verified in `EnableAudioNoiseReduction` (`0x100197798`), `GetAudioNoiseStatus` (`0x100197808`), `SetAudioCaptureMode` (`0x10019bf54`), and `GetAudioCaptureMode` (`0x10019bf64`).
- **Payload**: `[value_u8, 0x00]` (2 bytes)
- For Link 1 noise reduction: `value_u8` is `0` (Disabled) or `1` (Enabled).
- For Link 2 / Link 2C / PUC3 `AudioCaptureMode` (`0x1000702a8`):

| Value | Variant | UI Display Name | Description |
| :---: | :--- | :--- | :--- |
| `1` | `VoiceFocus` | Voice Focus | Directional beamforming microphone mode |
| `2` | `VoiceSuppression` | Noise Suppression | Wide-angle background noise cancellation |
| `3` | `MusicBalance` | Music Balance | Balanced voice and ambient music fidelity |
| `4` | `Original` | Original Sound | Unprocessed raw microphone audio |
| `5` | `SoundWall` | Sound Wall | Virtual audio fence (PUC3 `0x4c05` / `0x4c06` only) |

---

#### `XU1 0x05` & `0x06` — `TRACK_OBJECT_SELECT` & `TRACK_OBJECT_LIST_V1` (10 Bytes per `UVCRect`)
Verified in `SetTrackObj` (`0x100198a38`) and `GetTrackObjLists` (`0x100198adc`).
Each `UVCRect` entry is 10 bytes (5 × `i16` little-endian):

| Byte Offset | Type | Field | Description |
| :---: | :---: | :--- | :--- |
| `0..2` | `i16` LE | `x` | Normalized bounding box X coordinate |
| `2..4` | `i16` LE | `y` | Normalized bounding box Y coordinate |
| `4..6` | `i16` LE | `width` | Normalized bounding box width |
| `6..8` | `i16` LE | `height` | Normalized bounding box height |
| `8..10` | `i16` LE | `track_id` | Object tracking identifier |

---

#### `XU1 0x07` — `EXPOSURE_CURVE_AND_ISO` (259 Bytes)
Verified in `GetExpouseCurve` (`0x10019aaa0`), `SetExpouseCurve` (`0x10019ab74`), `EnableAutoExposure` (`0x10019ac90`), `GetAutoExposureStatus` (`0x10019ad98`), `SetISOValue` (`0x10019ae24`), and `GetISOValue` (`0x10019af50`).

| Byte Offset | Type | Field | Description |
| :---: | :---: | :--- | :--- |
| `0` | `u8` | `mode` | Exposure mode: `0` = Manual Exposure, `1` = Auto Exposure |
| `1..3` | `u16` LE | `iso` | Manual ISO sensitivity (e.g. `100..=3200`) |
| `3..259` | `[u16; 128]` LE | `points` | 128-point tone/exposure lookup curve (`256` bytes) |

---

#### `XU1 0x08` — `EXPOSURE_COMPENSATION` (4 Bytes)
Verified in `SetExposureCompensation` (`0x100198f34`) and `GetExposureCompensation` (`0x100198fc0`).
- **Payload**: 4-byte IEEE-754 single-precision float (`f32` little-endian) representing EV compensation (e.g., `-3.0` to `+3.0`).

---

#### `XU1 0x09` — `WHITE_BALANCE_EXT` (129 Bytes)
Verified in `SetWhiteBalanceParam` (`0x10019afe0`) and `GetWhiteBalanceParam` (`0x10019b264`).
- `byte[0]`: `u8` White balance mode
- `byte[1..129]`: `[u32; 32]` Little-endian channel calibration gains (`128` bytes).

---

#### `XU1 0x0a` — `EXTEND_FUNC_BITMASK` (8 Bytes)
Verified in `EnableExtendFuncWork` (`0x100199afc`), `GetExtendFuncStatus` (`0x100199dd0`), `EnableVerticalScreen` (`0x10019b7d0`), `EnableLowResolution` (`0x10019b854`), and the `CameraInsta::Init` jump table at `0x102864a20`.

**Wire Layout (8 Bytes)**:
| Byte Offset | Type | Field | Description |
| :---: | :---: | :--- | :--- |
| `0..4` | `u32` LE | `mask` | Bitmask of feature bit(s) being modified (`1u32 << bit_index` on `SET_CUR`) |
| `4..8` | `u32` LE | `values` | Full 32-bit state bitmask of all enabled extended functions |

**Complete `ExtendFunction` Bit Definitions (`0..14`, verified at `0x102864a20`)**:
| Bit Index | Bitmask (`1 << bit`) | Enum Variant | Internal Field in `CameraInsta` | Description |
| :---: | :---: | :--- | :--- | :--- |
| `0` | `0x0000_0001` | `AiZoom` | `ai_zoom_checked_` | Automatic AI Zoom during tracking |
| `1` | `0x0000_0002` | `Reserved1` | — | Reserved |
| `2` | `0x0000_0004` | `Hdr` | `enable_hdr_` | Hardware HDR mode |
| `3` | `0x0000_0008` | `HorizontalMirror` | `mirror_checked_` | Horizontal image flip / mirror |
| `4` | `0x0000_0010` | `AllGestures` | `enable_all_gestures_` | Master hand-gesture recognition flag |
| `5` | `0x0000_0020` | `VerticalMode` | `enable_vertical_mode_` | 9:16 Vertical output resolution |
| `6` | `0x0000_0040` | `StartupParams` | `enable_startup_params_` | Persist startup parameters in camera NVRAM |
| `7` | `0x0000_0080` | `SingleTapTracking` | `enable_single_tap_tracking_` | Single-click target selection tracking |
| `8` | `0x0000_0100` | `AutoTracking` | `enable_auto_tracking_` | Automatic human tracking enable |
| `9` | `0x0000_0200` | `SmartAdjustment` | `smart_adjustment_checked_` | Smart exposure/frame adjustment |
| `10` | `0x0000_0400` | `ForcedVertical` | `forced_vertical_` | Uncropped 9:16 portrait mode (physical 90° rotation) |
| `11` | `0x0000_0800` | `UltimatePrivacyMode` | `enable_privacy_mode_` | 10-second inactivity sleep / gimbal tilt-down privacy |
| `12` | `0x0000_1000` | `VerticalMirror` | `enable_vertically_mirror_` | Vertical image flip (upside-down mount) |
| `13` | `0x0000_2000` | `LowResolution` | `enable_low_resolution_` | Low-resolution compatibility mode |
| `14` | `0x0000_4000` | `TrackForbiddenArea` | `enable_track_forbidden_area_` | Enforce tracking forbidden zones |

---

#### `XU1 0x0b` — `PANTILT_RELATIVE_EXT` (4 Bytes)
Verified in `SetPanTiltRelative` (`0x1001990f4`) and `GetPanTiltRelative`.
- **Payload**: `[pan_speed: u8, pan_dir: i8, tilt_speed: u8, tilt_dir: i8]`
- Note that the byte order (`speed` then `dir`) is reversed compared to standard UVC `CT_PANTILT_RELATIVE_CONTROL` (`0x0e`, which uses `dir` then `speed`).
- `dir`: `0` = Stop, `1` = Positive (Right / Up), `-1` (`0xff`) = Negative (Left / Down).

---

#### `XU1 0x0c` — `DEVICE_STATUS` (4 Bytes, `GET_CUR`)
Verified in `GetDeviceStatus` (`0x10019b8dc`). Returns a 32-bit little-endian status bitmask (`DeviceStatus`):
- Bit 0 (`0x01`): `TRACKING_ACTIVE`
- Bit 1 (`0x02`): `WHITEBOARD_ACTIVE`
- Bit 2 (`0x04`): `GESTURE_DETECTED`
- Bit 3 (`0x08`): `PRIVACY_SLEEP`
- Bit 4 (`0x10`): `OVERHEAD_MODE`
- Bit 5 (`0x20`): `DESKVIEW_MODE`
- Bit 6 (`0x40`): `UPGRADING`

---

#### `XU1 0x0d` — `PTZ_VERSION_AND_ABS` (30 Bytes, `GET_CUR`)
Verified in `GetPTZVersionInfo` (`0x1001993bc`) and `GetPanTiltAbsoluteValue` (`0x1001996cc`).
- `byte[0..20]`: Null-terminated ASCII gimbal firmware version string (e.g. `"v1.3.1.0_build0"`).
- `byte[20..22]`: `i16` LE current Pan angle (in `0.1°` units, `1° = 10` units = `3600 arcsec`).
- `byte[22..24]`: `i16` LE current Tilt angle (in `0.1°` units).
- `byte[24..26]`: `u16` LE current Zoom ratio (`100..400`).
- `byte[26]`: `u8` Gimbal status byte.
- `byte[27..30]`: Reserved.

---

#### `XU1 0x0e` — `PRESET_POSITION_FLAGS` (8 Bytes, `GET_CUR`)
Verified in `GetCurPresetPositionFlag` (`0x1001997c0`).
- `byte[0]`: `slot_mask` (`u8` bitmask of saved presets, bits `0..5`).
- `byte[1]`: `current_slot` (`u8` index of active preset).
- `byte[2]`: `max_slots` (`u8`, typically `6`).
- `byte[3..8]`: Reserved.

---

#### `XU1 0x0f` — `USB_MODE_SWITCH` (4 Bytes, `SET_CUR`)
Verified in `SwitchMscReadOnlyClass` (`0x10019b998`) and `DownloadFileFromCamera` (`0x10019b928`):
- `[0x01, 0x00, 0x00, 0x00]`: Trigger diagnostic log / file download mode.
- `[0x02, 0x00, 0x00, 0x00]`: Switch camera USB personality to Read-Only Mass Storage Class (MSC) / Simple-Class mode for firmware upgrade.

---

## 7. Proprietary Extension Unit 2 (`XU2`) Selectors (`0x01` – `0x0f`)

- **GUID**: `E307E649-4618-A3FF-82FC-2D8B5F216773`
- **Default Unit ID**: `10` (`0x0a`)
- **Binary Implementation**: `MacUVCCameraExtendController` (`0x100195a18`–`0x10019c478`)

### Summary Table of `XU2` Selectors

| Selector | Constant Name | Size | Requests | Primary Function |
| :---: | :--- | :---: | :---: | :--- |
| `0x01` | `GESTURE_BINDING_MAP` | 16B | `SET_CUR`, `GET_CUR` | Map hand gestures (`0..4`) to `GestureFunction` actions (`0..4`) |
| `0x02` | `SYSTEM_TIME_AND_IQ` | 10B | `SET_CUR` | Sync host UTC timestamp, set IQ switches (`D0`/`D1`), or restore factory settings |
| `0x03` | `DEVICE_INFO` | 150B | `GET_CUR` | Read device name (`[0..50]`), firmware version (`[50..100]`), and serial number (`[100..150]`) |
| `0x04` | `IMAGE_TEMPLATE_INFO` | 11B | `SET_CUR`, `GET_CUR` | Image preset template save/apply/delete & populated 64-bit slot bitmask |
| `0x05` | `IMAGE_TEMPLATE_NAME` | 65B | `SET_CUR`, `GET_CUR` | Read/write 64-byte ASCII name for image template `id` (`byte[0]`) |
| `0x06` | `FILTER_MODE` | 1B | `SET_CUR`, `GET_CUR` | Hardware LUT color filter (`0..=8`) |
| `0x07` | *(Reserved)* | — | — | Unused in v2.0.2 |
| `0x08` | `PRIVACY_AND_FILL_LIGHT` | 4B | `SET_CUR`, `GET_CUR` | Hardware privacy mode (`[2, enable, 0, 0]`) and fill light parameters |
| `0x09` | `HOST_PTZ` | 8B | `SET_CUR` | Digital crop box on gimbal-less cameras (`u16 x, y, width, height`, normalized `0..5140`) |
| `0x0a` | `TRACK_OBJECT_LIST_V2` | `N * 10B` | `GET_LEN`, `GET_CUR` | Detected trackable objects bounding box array (`GetNewTrackObjLists` for Link 2 / 2C) |
| `0x0b` | `TRACK_FORBIDDEN_AREA` | 40B | `SET_CUR`, `GET_CUR` | Configure/enable gimbal tracking exclusion zones (`ForbiddenAreaInfo_s`) |
| `0x0c` | `DEFAULT_TRACK_MODE` | 1B | `SET_CUR`, `GET_CUR` | Default AI tracking mode (`0` = Single Person, `1` = Group / Auto Framing) |
| `0x0d` | `HORIZONTAL_CORRECTION` | 1B | `SET_CUR`, `GET_CUR` | Gimbal horizon leveling (`0` = Off, `1` = On; Link 2 `0x4c04`) |
| `0x0e` | `REAL_ZOOM_AND_UDISK` | 4B | `SET_CUR`, `GET_CUR` | Query real optical/digital zoom value (`u16` LE) and UDisk open counter |
| `0x0f` | `REPORT_DATA` | Variable | `GET_LEN`, `GET_CUR` | Telemetry and diagnostic report buffer (`GetReportData`) |

---

### Detailed `XU2` Selector Payloads

#### `XU2 0x01` — `GESTURE_BINDING_MAP` (16 Bytes)
Verified in `GetGestureBindingMap` (`0x100198384`) and `SetGestureBindingFunc` (`0x10019874c`).
Maps each `Gesture` (`0..4`) to a `GestureFunction`:
- `0` = `None`
- `1` = `AutoTracking`
- `2` = `Zoom`
- `3` = `WhiteBoard`
- `4` = `AutoFraming`

---

#### `XU2 0x02` — `SYSTEM_TIME_AND_IQ` (10 Bytes, `SET_CUR`)
Multiplexed system control dispatched by `byte[0]`:

| Sub-Opcode (`byte[0]`) | Method in Binary | Address | Wire Payload (10 Bytes) | Description |
| :---: | :--- | :--- | :--- | :--- |
| `0x01` | `SetTimestamp(u64 ms)` | `0x10019be40` | `[0x01, ts_le[0..8], 0x00]` | Synchronize camera RTC with 64-bit host timestamp (ms) |
| `0x02` | `SetIQSwitch(int bit, bool en)` | `0x100199ce0` | `[0x02, mask_u32_le[0..4], 0..]` | Toggle ISP Image Quality switch bit (`0` = `D0`, `1` = `D1`) |
| `0x04` | `RestoreFactorySet(true)` | `0x10019c1ac` | `[0x04, 0x01, 0x00, ...]` | Full factory reset and reboot camera |
| `0x05` | `RestoreFactorySet(false)` | `0x10019c1ac` | `[0x05, 0x01, 0x00, ...]` | Restore factory settings without rebooting |

---

#### `XU2 0x03` — `DEVICE_INFO` (150 Bytes, `GET_CUR`)
Verified in `GetDeviceInfo` (`0x100196e74`) and `GetSerialNumber` (`0x100196d10`).
Contains three fixed 50-byte null-terminated ASCII strings:
- `byte[0..50]`: `device_name` (e.g. `"Insta360 Link 2"`)
- `byte[50..100]`: `firmware_version` (e.g. `"v2.5.8.8_build1"`)
- `byte[100..150]`: `serial_number` (e.g. `"INK2A240912345"`)

---

#### `XU2 0x04` & `0x05` — `IMAGE_TEMPLATE_INFO` (11 Bytes) & `IMAGE_TEMPLATE_NAME` (65 Bytes)
Verified in `GetImageTemplateInfo` (`0x10019ba4c`), `SetImageTemplate` (`0x10019bb08`), `DeleteTemplate` (`0x10019bbc8`), `GetImageTemplateName` (`0x10019bc74`), and `SetImageTemplateName` (`0x10019bd14`).

**`XU2 0x04` (`IMAGE_TEMPLATE_INFO`, 11 Bytes)**:
| Byte Offset | Type | Field | Description |
| :---: | :---: | :--- | :--- |
| `0` | `i8` | `current_id` | Active template slot ID (`0..63`, or `-1` if none active) |
| `1` | `u8` | `operation` | `ImageOperation`: `0` = `None`, `1` = `SaveOrUpdate`, `2` = `Apply` |
| `2..10` | `u64` LE | `bitmask` | 64-bit mask of populated template slots (`bit N = 1` if slot `N` exists) |
| `10` | `u8` | `max_templates` | Maximum supported templates (typically `10`) |

**`XU2 0x05` (`IMAGE_TEMPLATE_NAME`, 65 Bytes)**:
- `byte[0]`: `i8` `template_id` (`0..63`)
- `byte[1..65]`: 64-byte null-terminated UTF-8/ASCII template name.

---

#### `XU2 0x06` — `FILTER_MODE` (1 Byte)
Verified in `SetFilter` (`0x10019bdb0`), `GetFilter` (`0x10019bdf8`), and UI mapping at `0x100096f94`:
| Value | Variant | Filter Name |
| :---: | :--- | :--- |
| `0` | `FilterMode::None` | None (Original) |
| `1` | `FilterMode::Style1` | Style 1 |
| `2` | `FilterMode::Style2` | Style 2 |
| `3` | `FilterMode::Fresh` | Fresh |
| `4` | `FilterMode::Amber` | Amber |
| `5` | `FilterMode::Film1` | Film 1 |
| `6` | `FilterMode::Film2` | Film 2 |
| `7` | `FilterMode::Cool` | Cool |
| `8` | `FilterMode::Warm` | Warm |

---

#### `XU2 0x08` — `PRIVACY_AND_FILL_LIGHT` (4 Bytes)
Verified in `SetPrivacyMode` (`0x10019c108`) and `GetPrivacyMode` (`0x10019c168`):
- **`SET_CUR` (4 Bytes)**: `[0x02, enable_u8, 0x00, 0x00]` where `enable_u8` is `1` (Privacy On) or `0` (Privacy Off).
- **`GET_CUR` (4 Bytes)**: `byte[0]` = status code (`i8`), `byte[1]` = `1` if privacy mode is enabled else `0`.

---

#### `XU2 0x09` — `HOST_PTZ` (8 Bytes, `SET_CUR`)
Verified in `SetHostPTZ` (`0x10019bec0`). Used on gimbal-less models (Link 2C `0x4c03` and PUC3 Host `0x4c05`) for digital pan/tilt/zoom cropping within a `5140 × 5140` coordinate space:
- `byte[0..2]`: `u16` LE `x` (`0..5140`)
- `byte[2..4]`: `u16` LE `y` (`0..5140`)
- `byte[4..6]`: `u16` LE `width` (default `5140`)
- `byte[6..8]`: `u16` LE `height` (default `5140`)

---

#### `XU2 0x0b` — `TRACK_FORBIDDEN_AREA` (40 Bytes)
Verified in `SetTrackForbiddenArea` (`0x10019bff4`), `GetTrackForbiddenArea` (`0x10019c038`), and `EnableTrackForbiddenArea`. Stores up to 4 `ForbiddenAreaInfo_s` entries (10 bytes each: `u8 enabled, u8 index, i16 pan_min, i16 pan_max, i16 tilt_min, i16 tilt_max`).

---

#### `XU2 0x0c` — `DEFAULT_TRACK_MODE` (1 Byte)
Verified in `SetDefaultTrackMode` (`0x10019c220`) and `GetDefaultTrackMode` (`0x10019c268`).
- `byte[0]`: `0` = Single-person tracking, `1` = Multi-person / Group tracking.

---

#### `XU2 0x0d` — `HORIZONTAL_CORRECTION` (1 Byte)
Verified in `EnableHorizontalCorrection` (`0x10019c2b0`) and `GetEnableHorizontalCorrection` (`0x10019c2f8`).
- `byte[0]`: `0` = Disabled, `1` = Enabled (supported on Insta360 Link 2 `0x4c04`).

---

#### `XU2 0x0e` — `REAL_ZOOM_AND_UDISK` (4 Bytes)
Verified in `GetRealZoomValue` (`0x10019c340`), `GetUDiskOpenCount`, and `SetUDiskOpenCount`.
- `byte[0..2]`: `u16` LE real zoom factor (`100..400`).

---

## 8. Firmware Upgrade Protocol (`UVCCameraExtendControllerImp::FirmwareUpgrade`)

Verified in `UVCCameraExtendControllerImp::FirmwareUpgrade` (`0x10019fc88`) and `MacSimpleClassControlImp` (`0x1001a0a0c`).

Insta360 cameras support two firmware delivery paths depending on the active USB mode:
1. **Read-Only Mass Storage Class (MSC) Mode**: Triggered via `XU1` selector `0x0f` (`[0x02, 0x00, 0x00, 0x00]`).
2. **USB Vendor Simple-Class Mode (`PID 0x00c2` or vendor interface)**: Transfers the firmware image over raw USB Vendor Control + Bulk OUT transfers.

### 8.1 Step-by-Step USB Simple-Class Upgrade Sequence (`0x10019fc88`)

1. **Initiate Firmware Transfer (Vendor Control Transfer OUT)**:
   - `bmRequestType = 0x41` (Host-to-Device, Vendor, Interface)
   - `bRequest = 0x01`
   - `wValue = 0x0004`
   - `wIndex = 0x0000`
   - `wLength = 137` (`0x0089` bytes)
   - **137-Byte Header Payload**:
     - `byte[0..8]`: `u64` LE total firmware file size in bytes
     - `byte[8]`: `0x5c` (`'\\'` path separator sentinel)
     - `byte[9..137]`: 128-byte null-padded ASCII target path on camera flash (from `0x1001a0a0c`, e.g. `a:\Insta360WebCamFW.bin`, `a:\Insta360LINK2FW_GIMBAL.bin`, `a:\Insta360LINK2CFW_HOST.bin`, `a:\Insta360PUC3FW_GIMBAL.bin`, or `a:\Insta360PUC3FW_HOST.bin`).

2. **Stream Firmware Image in 512 KB Bulk Chunks**:
   - **Endpoint**: Bulk OUT Endpoint `0x01`
   - **Chunk Size**: `0x80000` bytes (`524,288` bytes = 512 KB) per bulk transfer until the entire firmware binary is transmitted.

3. **Verify Firmware Reception (Vendor Control Transfer IN)**:
   - `bmRequestType = 0xc1` (Device-to-Host, Vendor, Interface)
   - `bRequest = 0x81`
   - `wValue = 0x0004`
   - `wIndex = 0x0000`
   - `wLength = 1`
   - **Response**: 1 byte (`0x00` indicates checksum/transfer verification succeeded).

4. **Reboot into Normal UVC Mode (Vendor Control Transfer OUT)**:
   - `bmRequestType = 0x41` (Host-to-Device, Vendor, Interface)
   - `bRequest = 0x01`
   - `wValue = 0x0002`
   - `wIndex = 0x0000`
   - `wLength = 1`
   - **Payload**: `[0x01]`
