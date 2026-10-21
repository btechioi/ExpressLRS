# Dual-Band (BOTHBANDS) Work — Agent Handoff

> Status as of 2026-10-21. This branch is **WORK IN PROGRESS and does not compile**.
> Another agent may take over from this document. It captures the full state:
> objective, architecture decisions, completed work, remaining work, and known pitfalls.

## Objective

Enable a single ExpressLRS ESP32-C3 firmware image to contain both the
900 MHz (SX127X) and 2.4 GHz (SX128X) links, switchable by command over
UART or wireless (WiFi/BLE), while existing single-band images keep building.

- Both bands compiled into one image.
- Band/links switchable at runtime by command (UART or wireless).
- Existing single-band environments must keep building unchanged.

### Known design limitation (deliberate, for now)

Even when both halves compile into one image, only one link actually runs at
a time — the combined `setup()`/`loop()` dispatch exists, but true
simultaneous full-duplex on both bands requires a second router/link state
and per-link radio/FHSS config that has **not** been implemented yet.

## How to build / verify

Run from `src/`:

```sh
cd src
# BOTHBANDS (the WIP target — currently FAILS to compile):
PLATFORMIO_BUILD_FLAGS="-DRegulatory_Domain_EU_CE_2400 -DRegulatory_Domain_FCC_915" \
  pio run -e Unified_ESP32C3_BOTHBANDS_via_UART

# Single-band regression checks (currently PASS):
PLATFORMIO_BUILD_FLAGS="-DRegulatory_Domain_EU_CE_2400 -DRegulatory_Domain_FCC_915" \
  pio run -e Unified_ESP32C3_900_RX_via_UART   # ~16s, use as smoke test
# Also: Unified_ESP32C3_2400_RX_via_UART, Unified_ESP32C3_2400_TX_via_UART,
#       Unified_ESP32C3_900_TX_via_UART, and _via_WIFI variants.
```

**Note:** all envs currently need the `PLATFORMIO_BUILD_FLAGS` reg-domain pair
set explicitly; both regulatory domains are set together for BOTHBANDS.

Build iteration is fast (~15–25s per run). Grep for real errors with:
`... 2>&1 | grep -E "error:" | sort -u`

## Architecture decisions (do not undo without reason)

1. **One image, per-file `#if` guards.** Files compile both their
   `TARGET_RX` and `TARGET_TX` halves when `BOTHBANDS` is set; combined
   dispatch lives in `rx_main.cpp` (`setup()`/`loop()` call
   `setupRxLink(); setupTxLink();` / `loopRxLink(); loopTxLink();`).
   In `tx_main.cpp`, the original `setup()`/`loop()` are renamed to
   `setupTxLink()`/`loopTxLink()` **only** when both roles are defined
   (guards at tx_main.cpp lines ~1436 and ~1539 — KEEP these two guards).

2. **Config instance naming (central to everything).** In
   `src/lib/CONFIG/config.h`, inside the `#if defined(TARGET_RX)` region:
   - `#if defined(TARGET_TX)` (i.e. both-roles build):
     `extern RxConfig rxConfig;` — a **distinct name** so the TX-side
     `extern TxConfig config;` can coexist in the same TU.
   - `#else` (RX-only build): `extern RxConfig config;` plus
     `#define rxConfig config` so RX sources can uniformly use `rxConfig`.
   - The TX instance keeps the name `config` (type `TxConfig`, defined in
     tx_main.cpp).
   - Consequence: **all RX translation units must use `rxConfig.`, never
     `config.`** — this rename is the single biggest remaining task.

3. **Storage separation is intact:** `TxConfig` uses NVS namespace "ELRS",
   `RxConfig` uses the Arduino EEPROM lib (separate NVS namespace "eeprom").
   No change needed there.

4. **`lib_exclusions.py`** recognizes `BOTHBANDS` env names → target type
   `"BOTH"` with `"esp32c3": ("GFX Library for Arduino","U8g2","GSENSOR","SCREEN")`.

5. **`build_flags.py`** sets both regulatory domains
   (`Regulatory_Domain_EU_CE_2400` + `Regulatory_Domain_FCC_915`) and both
   radio drivers for BOTHBANDS.

6. Radio guard pattern: `#if defined(RADIO_SX128X) && !defined(RADIO_SX127X)`
   (used where only one radio semantics is valid).

## Completed

- Both-roles compilation of `rx_main.cpp`/`tx_main.cpp` bodies; combined
  dispatch; per-file `#if`/`#endif` balance verified (conditional-stack
  depth 0) for `rx_main.cpp` and `config.h`.
- Both-driver compilation, both-regulatory-domain flags, both-bands envs in
  `targets/esp32c3-rx.ini`.
- `config.h` externs restructured as above (fixes the old
  `conflicting declaration 'RxConfig config'` — but see Remaining #3, a
  second occurrence still exists at rx_main.cpp:111).
- Mid-expression `#if` damage from earlier regex edits partially fixed by
  splitting directives onto their own lines — **but more damage remains**
  (see Remaining #2).
- RxConfig-only calls guarded where needed in rx_main
  (GetPowerOnCounter/SetPowerOnCounter, IsOnLoan/ReturnLoan,
  GetBindStorage, GetIsBound, GetUID, GetRateInitialIdx, GetSerialProtocol,
  GetPwmChannel; `somSbus` bogus fallback → `(eServoOutputMode)0`).
- `SerialMavlink.cpp` guarded `#if defined(TARGET_RX) && !defined(TARGET_TX)`
  (uses RxConfig-only members) — re-enable after rxConfig rename lands.
- `lib_exclusions.py` "BOTH" type added.
- `Unified_ESP32C3_900_RX_via_UART` regression build SUCCEEDS.

## Remaining work — ordered next moves

### 1. Rename `config.` → `rxConfig.` in RX TUs
Files: `src/src/rx_main.cpp`, `src/src/rx-serial/*` (incl. `SerialSBUS.cpp`,
`devSerialIO.cpp`), `src/lib/rx-crsf/*`, `devSerialIO`, `SerialMavlink`,
`RXParameters`. Also rename the `RxConfig config;` definitions at
rx_main.cpp lines ~109/111 to `RxConfig rxConfig;` (fixes the
`conflicting declaration 'RxConfig config'` error at line 111).

### 2. Revert the `#if defined(TARGET_RX) && !defined(TARGET_TX)` fallback hacks in rx_main.cpp
They compile out real RX behavior and broke expression syntax. Remove the
`(uint8_t)0` / `(0)` / `false` else-branches and stray `#endif == ...`
fragments; restore normal expressions. Current symptom lines (may shift):
871, 974, 989, 1074/1080/1084/1087, 1294/1298/1299, 1344, 1462–1478
(`serialConfig`/`invert` not declared), 1501, 1720/1724.
Verify with a conditional-stack checker (depth must end at 0).

### 3. Fix the `ui_devices[]` guard at rx_main.cpp ~lines 77–80
The first branch is an empty comment and the real definition sits under
`#elif defined(TARGET_RX)` — leaving `ui_devices[]` **undefined in
RX-only builds**. First branch must hold the definition. Do not break
single-band builds here.

### 4. Relax `defined(TARGET_TX) && !defined(TARGET_RX)` guards in TX files
TX TUs now correctly see `TxConfig`, so these guards can open up:
- `tx_main.cpp` config-call sites (lines ~346, 396, 423, 430, 458, 551,
  570, 611, 621, 767, 786, 812, 821, 1063, 1084, 1168, 1174, 1200, 1224).
  **KEEP** the two setup/loop guards at ~1436 and ~1539.
- `src/src/dynpower.cpp` (2 sites, ~lines 92/110, with
  `#else uint8_t boostChannel = 0;` fallback).
- TX lib files: devBackpack, devVTX, devButton, devRGB, devThermal,
  devGsensor, devWIFI, TXModuleParameters, TXUSBConnector.

### 5. Re-enable methods wrapped out for both-roles builds
- `lib/tx-crsf/TXModuleParameters.cpp`: `sendELRSstatus`,
  `supressCriticalErrors`, `registerParameters`, `updateParameters`,
  `devicePingCalled` (has local `extern TxConfig config;` at ~line 321).
- `lib/tx-crsf/TXUSBConnector.cpp`: `forwardMessage` (missing vtable).
- `src/src/rx-serial/SerialMavlink.cpp`: currently guarded out of
  both-roles builds.
- Confirm `setupTxLink`/`loopTxLink` resolve at link time.

### 6. Fix link-stage duplicate symbols
Defined in both mains; guard/rename so exactly one TU defines each:
`ui_devices`, `otaConnector`, `crsfRouter`, `RXdoneISR`, `TXdoneISR`,
`EnterBindingModeSafely` (declared in `src/include/rxtx_intf.h`).

### 7. Build all envs green
`Unified_ESP32C3_BOTHBANDS_via_UART` then `_via_WIFI`, then regression all
single-band envs (all with the reg-domain flag pair):
`Unified_ESP32C3_2400_RX_via_UART`, `Unified_ESP32C3_2400_TX_via_UART`,
`Unified_ESP32C3_900_RX_via_UART`, `Unified_ESP32C3_900_TX_via_UART`.

### 8. Implement actual dual-link runtime (not started)
- Second router/link state so both links can be active.
- Per-link radio config and FHSS.
- Band-switch command handling over UART and wireless (WiFi/BLE).

## Config member reference (which members exist on which class)

RxConfig-only (TxConfig lacks these — will error as
`'class TxConfig' has no member named ...` if `config.` is used in an RX TU):
GetPowerOnCounter, SetPowerOnCounter, IsOnLoan, ReturnLoan, GetPwmChannel,
GetSerialProtocol, SetSerialProtocol, GetSerial1Protocol, GetSourceSysId,
GetTargetSysId, GetTeamraceChannel, GetTeamracePosition, GetFailsafeMode,
GetBindStorage, GetForceTlmOff, GetRateInitialIdx, SetRateInitialIdx,
GetIsBound, GetUID, GetModelId, GetAntennaGroup, SetAntennaGroup.

TxConfig-only (RxConfig lacks these):
Get/SetLinkMode, Get/SetTlm, Get/SetBackpack*, Get/SetDvr*, Get/SetPTR*,
Get/SetVtx*, Get/SetFanMode, Get/SetMotionMode, SetPowerFanThreshold,
GetRate, GetSwitchMode, GetDynamicPower, GetBoostChannel, GetButtonActions,
Get/SetModelMatch, GetModelConfig.

## Recurring self-inflicted bugs — CHECK AFTER ANY SCRIPTED EDIT

- Unclosed `defined(TARGET_RX` without closing `)`.
- Preprocessor directives left mid-line inside expressions.
- `#endif` followed by code tokens (e.g. `#endif == PROTOCOL_CRSF`).
- Conditional-stack depth must end at 0 per file (verify after edits).
- Only one global name `config` can have one type per TU — that
  constraint is what drove the `rxConfig` naming decision.

## Repository / git state

- Branch: `dual-band-bothbands`, tracks `standalone/master`.
- Remotes:
  - `origin` → https://github.com/ExpressLRS/ExpressLRS.git (upstream)
  - `fork` → https://github.com/btechioi/ExpressLRS.git
  - `standalone` → https://github.com/btechioi/ExpressLRS-DualBand.git
    (default branch `master` = this work)
- Git identity set repo-local: `BTech_IOI
  <89692289+btechioi@users.noreply.github.com>`.
- The pre-handoff WIP state was committed as
  `8a703063 "Add BOTHBANDS dual-link build for ESP32-C3 (900MHz + 2.4GHz)"`
  — note it commits the *non-compiling* state; that is expected.

## Key files

| File | Role |
|---|---|
| `src/lib/CONFIG/config.h` | rxConfig/config naming scheme centerpiece |
| `src/src/rx_main.cpp` | biggest damage: rename + guard reverts + dispatch |
| `src/src/tx_main.cpp` | 21 guards to relax (keep ~1436/~1539) |
| `src/src/rx-serial/SerialSBUS.cpp`, `devSerialIO.cpp` | current build errors, need `rxConfig` |
| `src/src/dynpower.cpp` | 2 TX guards to relax |
| `src/lib/tx-crsf/TXModuleParameters.cpp`, `TXUSBConnector.cpp` | wrapped-out methods/vtable |
| `src/python/build_flags.py` | both reg domains + both drivers |
| `src/python/lib_exclusions.py` | "BOTH" target type |
| `src/targets/esp32c3-rx.ini` | BOTHBANDS env definitions |
| `src/include/rxtx_intf.h` | shared `EnterBindingModeSafely()` decl |
| `src/lib/CONFIG/config.cpp` | TxConfig→NVS "ELRS", RxConfig→EEPROM storage |
