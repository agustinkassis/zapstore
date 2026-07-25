# WORK-030 — Amber Session Restore vs Device-Key Backup

**Feature:** FEAT-006-device-key.md
**Status:** Complete

## Tasks

- [x] 1. Skip Amber device-key backup decrypt on session restore
  - Files: `lib/main.dart`
  - `attemptAutoSignIn` is not an interactive first sign-in; do not call
    `maybeOfferDeviceBackup` when restoring a saved Amber pubkey.
- [x] 2. Amber backup payload uses `deviceKey` only
  - Files: `lib/services/device_backup_service.dart`, `spec/features/FEAT-006-device-key.md`
  - Ignore older `privateKeyHex` payloads; no dual-read.
- [x] 3. Tests for restore skip and `deviceKey` payload
  - Files: `test/services/device_backup_service_test.dart`
- [x] 4. Self-review against INVARIANTS.md

## Test Coverage

| Scenario | Expected | Status |
|----------|----------|--------|
| Interactive Amber sign-in, no backup | Creates backup | [x] |
| Interactive Amber sign-in, matching backup | No restore prompt | [x] |
| Interactive Amber sign-in, differing backup | Offers restore | [x] |
| Session restore (auto sign-in) | No Amber backup decrypt | [x] |
| Backup JSON uses `deviceKey` | Round-trips; `privateKeyHex` ignored | [x] |

## Decisions

### 2026-07-25 — Session restore is not interactive sign-in

**Context:** Opening the app while Amber is remembered ran `onSignInSuccess` →
`maybeOfferDeviceBackup` → Amber `nip44Decrypt` of the device-key backup every
cold start. Users saw repeated PrivateHexKey decrypt activity in Amber.

**Decision:** Remove backup checks from `onSignInSuccess`. Interactive Amber
sign-in goes through `signInWithAmber`, which signs in then awaits
`maybeOfferDeviceBackup`. Session restore uses `attemptAutoSignIn` only.

**Rationale:** A flag around auto-restore raced with Riverpod listener delivery
(emulator showed `[I/backup] Amber backup already matches…` on cold start).
Explicit call sites avoid that race without artificial delays.

### 2026-07-25 — Backup field renamed to `deviceKey`

**Context:** Payload previously used `privateKeyHex`, which amplified scary
Amber Activity wording around decrypting a private key.

**Decision:** Encrypt/decrypt JSON `{"deviceKey": "<64 hex>"}` only. Do not
accept legacy `privateKeyHex` backups.

**Rationale:** Directed product change. Users with only an old-format backup
can paste nsec or create a new backup after interactive sign-in.

## Spec Issues

_None — FEAT-006 updated for `deviceKey` per product direction._

## Progress Notes

**2026-07-25:** Implemented restore skip + `deviceKey` payload.
**2026-07-25:** Emulator showed flag-based skip still decrypted on cold start;
switched to interactive-only `signInWithAmber`.
