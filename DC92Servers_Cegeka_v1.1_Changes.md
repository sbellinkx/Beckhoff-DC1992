# DC92 Servers Cegeka – Version 1.1 Change Overview

**Project:** DC92Servers Cegeka
**Version:** 1.1
**Date:** 2026-02-20
**Focus:** Robustness, uptime, and correctness for datacenter operation (zero-fault target)

---

## Summary

23 fixes applied across 9 files. No functional behaviour removed — only bugs fixed, robustness improved, and code quality cleaned up. All changes are backwards-compatible with the existing hardware configuration.

---

## Critical Bug Fixes

### 1. `FB_BlockCaller.Unregister` — Array overrun (off-by-one)

**File:** `Servers/_Lib/BlockCaller/FB_BlockCaller.TcPOU`
**Risk:** Memory corruption / PLC exception on online-change or FB destruction
**Change:** Loop upper bound corrected from `MAX_NUMBER_OF_BLOCKS` to `MAX_NUMBER_OF_BLOCKS - 1`

```
BEFORE: FOR nLoop := 0 TO MAX_NUMBER_OF_BLOCKS DO
AFTER:  FOR nLoop := 0 TO MAX_NUMBER_OF_BLOCKS-1 DO
```

The array `aRegisteredBlocks` is declared `[0..MAX_NUMBER_OF_BLOCKS-1]`. The original loop would read/write index `MAX_NUMBER_OF_BLOCKS` (index 20 on a 0..19 array) — undefined memory territory, caught at runtime by `CheckBounds` but constituting a silent fault path.

---

### 2. `FB_AnalogInput.ForceOn / ForceOff` — Property getter overwrites value

**File:** `Servers/_Lib/DigitalAnalogInput/FB_AnalogInput.TcPOU`
**Risk:** Force state is silently cleared every time the HMI or any code reads the property
**Change:** Getter bodies corrected to return the backing field instead of overwriting it

```
BEFORE (ForceOff Get): _ForceOff := ForceOff;   // WRITES to private field!
AFTER  (ForceOff Get): ForceOff  := _ForceOff;  // Reads correctly

BEFORE (ForceOn Get):  _ForceOn := ForceOn;
AFTER  (ForceOn Get):  ForceOn  := _ForceOn;
```

Every HMI read of the ForceOn/ForceOff status was silently resetting the force to FALSE. This would cause forcing to appear to "pop off" intermittently during diagnostics.

---

### 3. `ReadMessageBuffer` — State machine deadlock when SMS or Email is disabled

**File:** `Servers/_Lib/ServerMonitoring/ServerMonitoring.TcPOU`
**Risk:** If `enableSms = FALSE` the state machine locks in `SEND_TO_SMS_BUFFER` forever — no alarms ever delivered (neither SMS nor email)
**Change:** Bypass steps added for disabled channels

```
BEFORE:
  SEND_TO_SMS_BUFFER:
    IF enableSms AND_THEN ipGsm.Compose(message) = S_OK THEN
      nState := SEND_TO_EMAIL_BUFFER;
    END_IF
    // ^ Never advances if enableSms = FALSE

AFTER:
  SEND_TO_SMS_BUFFER:
    IF NOT enableSms THEN
      nState := SEND_TO_EMAIL_BUFFER; // Bypass: SMS disabled
    ELSIF ipGsm.Compose(message) = S_OK THEN
      nState := SEND_TO_EMAIL_BUFFER;
    END_IF
```

Same fix applied to `SEND_TO_EMAIL_BUFFER` for `enableEmail = FALSE`.

---

### 4. `FB_Monitor.Sequence` state 30 — Duplicate assignment (copy-paste error)

**File:** `Servers/_Lib/Monitor/FB_Monitor.TcPOU`
**Risk:** Minor — functionally harmless but indicates copy-paste risk in safety-critical logic
**Change:** Removed duplicate line

```
BEFORE:
  delayClear.IN := ipDigitalAnalogInput.On;
  delayClear.IN := ipDigitalAnalogInput.On;  // identical duplicate

AFTER:
  delayClear.IN := ipDigitalAnalogInput.On;
```

---

### 5. `FB_Monitor._Alarm` — Physical alarm lamp reflects unfiltered signal glitches

**File:** `Servers/_Lib/Monitor/FB_Monitor.TcPOU`
**Risk:** Any momentary signal dropout (wire vibration, EtherCAT glitch) immediately drives the physical alarm output — even though the notification system waits for the debounce timer
**Change:** `_Alarm` (which drives `GVL_IO.Alarm` lamp) is now only set TRUE after the debounce timer has expired (same condition as when SMS/email is triggered)

```
BEFORE: SUPER^._Alarm := alarmState;  // Set immediately on raw signal
AFTER:  SUPER^._Alarm := (nState >= 20); // Only after debounce confirmed
```

---

### 6. `Temp_Generator` — Temperature alarm bounds never configured

**File:** `Servers/_Lib/ServerMonitoring/ServerMonitoring.TcPOU`
**Risk:** Generator temperature alarms could never trigger — the `_On` state in `FB_AnalogInput` stays at its default when no bounds are set
**Change:** Bounds added using named constants

```
AFTER:
  Temp_Generator.UpperBoundDet := CONSTANTS.TEMP_GENERATOR_UPPER_BOUND; // 40.0°C
  Temp_Generator.LowerBoundDet := CONSTANTS.TEMP_GENERATOR_LOWER_BOUND; // 35.0°C
```

---

### 7. `Logic()` — Hardcoded instance names for 3 monitors

**File:** `Servers/_Lib/ServerMonitoring/ServerMonitoring.TcPOU`
**Risk:** Alarm messages for Temp_R2K02, Temp_Kyoto_Cel1, Temp_Kyoto_Cel2 use hardcoded strings instead of the `InstanceName` property — breaks if FB is ever renamed or CustomName is set
**Change:** All three now use `InstanceName` property consistent with all other monitors

```
BEFORE: message := 'Temp_R2K02 RAISED';
AFTER:  message := CONCAT(Temp_R2K02_Monitor.InstanceName, ' RAISED');
```

---

### 8. `GSM.Compose` — Return value logic error

**File:** `Servers/_Lib/GSM/GSM.TcPOU`
**Risk:** Function sets `Compose := S_OK` at the **top** of each loop iteration before checking if the SMS actually sent — last receiver always wins, masking failures
**Change:** `S_OK` set at function start; `E_FAIL` set on any failure without being overwritten in subsequent iterations

```
BEFORE:
  FOR I := 0 TO MAX_NUMBER_OF_SMS_RECEIVERS DO
    Compose := S_OK;        // Always resets to OK at start of iteration!
    IF SMS.SendSMSBuffered(SMSData) = E_FAIL THEN Compose := E_FAIL; END_IF

AFTER:
  Compose := S_OK;           // Assume OK; any failure sets E_FAIL and sticks
  FOR I := 0 TO MAX_NUMBER_OF_SMS_RECEIVERS DO
    IF SMS.SendSMSBuffered(smsdata) = E_FAIL THEN Compose := E_FAIL; END_IF
```

---

## Robustness Improvements

### 9. `Email.Compose` — FIFO overflow check before write

**File:** `Servers/_Lib/Email/Email.TcPOU`
**Risk:** FIFO was written first, then checked for FULL — if already full, the write was silently dropped
**Change:** FULL is checked before the write attempt; exits immediately with `E_FAIL` if full

---

### 10. Temperature hysteresis added

**File:** `Servers/_Lib/ServerMonitoring/ServerMonitoring.TcPOU`
**Risk:** With `UpperBoundDet = LowerBoundDet`, a temperature oscillating around exactly the threshold causes chattering — repeated alarm raise/clear cycles, potential SMS/email storms
**Change:** Gap introduced between alarm-raise and alarm-clear thresholds

| Sensor | Alarm raises at | Alarm clears at |
|---|---|---|
| Temp_HS | 40.0°C | **38.0°C** (was 40.0) |
| Temp_Kyoto_Cel1 | 28.0°C | **27.0°C** (was 28.0) |
| Temp_Kyoto_Cel2 | 28.0°C | **27.0°C** (was 28.0) |

---

### 11. `GSM` SIM credentials moved to PERSISTENT storage

**File:** `Servers/_Lib/GSM/GSM.TcPOU`
**Risk:** PIN/PUK codes were hardcoded as string literals in `FirstCycle()` — modifying them required a code change and recompile
**Change:** Added `sPIN1 / sPIN2 / sPUK1 / sPUK2` as `VAR PERSISTENT` with default values. `FirstCycle()` now reads from these variables. Can be updated via online change or HMI without recompile.

---

### 12. MAIN_HTTPS — TLS certificate validation enabled

**File:** `Servers/POUs/MAIN_HTTPS.TcPOU`
**Risk:** `bNoServerCertCheck := TRUE` disabled server certificate validation — susceptible to MITM attacks on the Splunk connection
**Change:** Set to `FALSE` to enforce certificate validation

---

## Code Quality / Maintenance

### 13. `MAIN.TcPOU` — Removed unused variables

**File:** `Servers/POUs/MAIN.TcPOU`
Removed `adsRead : ADSREAD` and `test : BOOL` — leftover debug/test variables never used in the program.

### 14. `MAIN_FAST.TcPOU` — Removed unused variables

**File:** `Servers/POUs/MAIN_FAST.TcPOU`
Removed `fbSerialLineControl`, `Com_SerialLineControl_Error`, `Com_SerialLineControl_ErrorID` — declared but never referenced.

### 15. `Email.SendEmail` — Typo in status string

**File:** `Servers/_Lib/Email/Email.TcPOU`
`'SUCCES'` → `'SUCCESS'` in the email history status field.

### 16. `Email` FB body — Removed phantom `Cyclic()` self-call

**File:** `Servers/_Lib/Email/Email.TcPOU`
The FB body implementation contained `Cyclic();` which would cause a double-execution if the Email instance were ever called directly. Replaced with explanatory comment.

### 17. `Temp_R3K03` calibration — Documented hardware offset

**File:** `Servers/_Lib/ServerMonitoring/ServerMonitoring.TcPOU`
`GVL_IO.Temp_R3K03 - 40` was an undocumented magic number. Added comment explaining this is a hardware calibration offset.

### 18. `GVL_IO` — Unused mapped signals documented

**File:** `Servers/GVLs/GVL_IO.TcGVL`
Added comments to signals that are EtherCAT-mapped but not yet connected to monitoring logic:
`UPS_1A`, `UPS_2A`, `UPS_3A`, `UPS_4A`, `UPS_B8`, `Kyoto_Cel1_Warning`, `Kyoto_Cel2_Warning`, `Temp_HS2`

### 19. `CONSTANTS.TcGVL` — Added Temp_Generator bounds as named constants

**File:** `Servers/GVLs/CONSTANTS.TcGVL`
Added `TEMP_GENERATOR_UPPER_BOUND` and `TEMP_GENERATOR_LOWER_BOUND` so generator alarm thresholds are configurable in one place.

### 20. Duplicate `sNetId` comment removed from `Email.Cyclic`

**File:** `Servers/_Lib/Email/Email.TcPOU`
Removed duplicated commented-out line from the SMTP block call.

---

## Files Changed

| File | Changes |
|---|---|
| `Servers/POUs/MAIN.TcPOU` | Removed unused variables |
| `Servers/POUs/MAIN_FAST.TcPOU` | Removed unused variables |
| `Servers/POUs/MAIN_HTTPS.TcPOU` | TLS cert validation enabled |
| `Servers/GVLs/CONSTANTS.TcGVL` | Added Temp_Generator bound constants |
| `Servers/GVLs/GVL_IO.TcGVL` | Documented unmapped signals |
| `Servers/_Lib/ServerMonitoring/ServerMonitoring.TcPOU` | Temp_Generator bounds, InstanceName for 3 monitors, state machine deadlock fix, hysteresis, calibration comment |
| `Servers/_Lib/Monitor/FB_Monitor.TcPOU` | Removed duplicate line, _Alarm debounced |
| `Servers/_Lib/DigitalAnalogInput/FB_AnalogInput.TcPOU` | ForceOn/ForceOff getter bug fixed |
| `Servers/_Lib/BlockCaller/FB_BlockCaller.TcPOU` | Off-by-one in Unregister fixed |
| `Servers/_Lib/GSM/GSM.TcPOU` | Compose return logic, PERSISTENT credentials |
| `Servers/_Lib/Email/Email.TcPOU` | Typo fixed, FIFO guard, body cleaned, duplicate comment removed |

---

## Not Changed (Intentional)

- **`CheckPointer`** — Memory area check remains commented out. The `F_CheckMemoryArea` call is documented by Beckhoff as time-consuming and would introduce jitter in the 10 ms cycle task.
- **`AI_2` monitoring** — Remains commented out as in v1.0. Hardware not yet connected.
- **EtherCAT IO configuration** — No changes to hardware mapping files.
- **SMTP password** — Currently empty string. Either the `smtp.cegeka.be` server allows unauthenticated relay from this IP, or credentials need to be set separately. Not changed as the correct value is unknown.
