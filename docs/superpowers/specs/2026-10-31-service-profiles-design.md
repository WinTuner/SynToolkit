# Service profiles — design

Date: 2026-10-31
Status: proposed

## Intent

Give SynToolkit a one-click way to move Windows services between curated
profiles (Minimal, Balanced) the way Sapphire OS ships service lists, without
the failure modes of a raw "disable everything" script.

The feature is for people who already understand that disabling services can
break Windows. It must be reversible on a machine that no longer boots, and it
must never report success without reading the values back.

## Goals

- Pick a preset, see exactly which services change (and which are blocked),
  apply it, and restore the previous state later.
- Ship two curated presets and let users drop in their own JSON presets.
- Take a snapshot of every service before changing it, and write a
  self-contained offline restore script next to it.
- Refuse changes that would break a dependency or a protected service.
- Work on Windows 10 and 11, on any hardware, without stopping services in the
  running session.

## Non-goals (v1)

- Kernel-mode / file-system drivers.
- Renaming binaries, editing `LowerFilters` / `UpperFilters`, deleting services.
- A whitelist mode ("disable everything not listed").
- A per-build "stock Windows defaults" dataset.
- Starting and stopping services during apply.
- Per-user services with template names.

## Anti-goals

- Applying anything without a preview.
- Applying anything without a restorable snapshot.
- Touching a protected service because a preset listed it.
- Reporting success when a write did not stick.
- Hiding the risk behind a silent one-click.

## What exists today

- `StartupPage` (`Views/StartupPage.xaml`) lists startup entries and toggles
  them one at a time. `StartupPageViewModel` owns scan/filter/toggle.
- `StartupScannerService.ScanServices` only lists **Automatic** services and
  skips Microsoft-signed ones. `ToggleService` writes the `Start` DWORD and
  verifies it.
- `RestorePointService.CreateAsync` creates a checkpoint and reports
  `Success`, `Throttled`, `ProtectionDisabled`, `AccessDenied`, `TimedOut` or
  another failure. PR #41 (open) adds an `Unverified` status.
- Profiles already work as JSON files under
  `%ProgramData%\Synergy\Profiles` (`ProfileSerializing`).
- UI strings live in `SynToolkit/lang/*.json` with English fallback.
- The app runs elevated (`requireAdministrator`).

PR #40 (open) adds a per-service "original start type" store at
`HKLM\SOFTWARE\SynToolkit\Startup\ServiceStart` and lists disabled services.
This design does not depend on it: each apply takes its own snapshot.

## Approaches considered

| | Approach | Trade-off |
|---|---|---|
| A | Generate `Services-Disable.bat` / `Services-Enable.bat` only | Safest and smallest, but not the in-app preset experience |
| B | Full clone of service-list-builder (whitelist, drivers, renames, filters) | Highest power, highest chance of bricking, much larger surface |
| C | In-app presets for user-mode services, explicit disable lists, snapshot + offline restore | Matches the requested UX, keeps the dangerous parts out |

**Chosen: C.**

## Decisions from the conversation

- Presets are authored by SynToolkit (Minimal, Balanced) and users can add
  their own JSON files. No unverified Sapphire list is copied.
- "Restore" means restoring the snapshot taken before an apply, not stock
  Windows defaults.
- UI lives behind a "Service profiles" button on the Startup page, not a new
  navigation item.

## Data model

Preset file (`Assets/ServicePresets/<id>.json`, and drop-ins under
`%ProgramData%\Synergy\ServicePresets\*.json`):

```json
{
  "id": "minimal",
  "nameKey": "ServicePreset_Minimal",
  "descriptionKey": "ServicePreset_MinimalDescription",
  "name": "Minimal",
  "description": "Removes telemetry, store and consumer services.",
  "services": [
    { "name": "DiagTrack", "start": "Disabled", "note": "Telemetry upload" },
    { "name": "MapsBroker", "start": "Disabled" }
  ]
}
```

- `start` is `Disabled` (4), `Manual` (3) or `Automatic` (2).
- Only explicit entries change. A service missing from the file is untouched.
- Unknown service names are reported in the preview and skipped. This also
  covers services that differ between Windows builds, so no build field is
  needed.
- `nameKey` / `descriptionKey` resolve through the language files; when a
  drop-in preset has no keys, the literal `name` / `description` is used, and
  the `id` is the last fallback.
- Optional per-service `note` is shown in the preview as the reason.
- Invalid files are logged and ignored, like invalid profiles.

Snapshot (`%ProgramData%\Synergy\ServiceRestore\<yyyyMMdd-HHmmss>\`):

```json
{
  "createdUtc": "2026-10-31T12:00:00Z",
  "presetId": "minimal",
  "services": [
    { "name": "DiagTrack", "start": 2, "delayedAutoStart": false }
  ]
}
```

The same folder holds `Restore.bat`, generated from that snapshot. A restore
writes its own snapshot with a `restore-<timestamp>-<4 hex>` name, so a failed
restore can be rolled back. Snapshots are not pruned automatically in v1.

## Service catalog

A new read layer over
`HKLM\SYSTEM\CurrentControlSet\Services\<name>` and `ServiceController`:

- name, display name, `Start`, `DelayedAutoStart`, `Type`, `ImagePath`
- dependencies (`DependOnService`)
- dependents (reverse of `DependOnService`, computed per scan)
- driver flag: `(Type & 0x0F) != 0`
- protected flag: name in the hard-coded protected list
- installed flag: service key exists
- **writable flag**: the service key can be opened for write by this process.
  Services owned by TrustedInstaller that deny Administrators are reported as
  blocked in the preview instead of failing the apply.

The catalog is injected behind an interface so the apply logic can be tested
without touching the registry.

## Apply pipeline

1. **Load and validate the preset.** Parse, drop unknown services, drop
   drivers, drop protected services, drop services whose key is not writable.
   Report each drop in the preview.
2. **Snapshot.** Read `Start` and `DelayedAutoStart` for every service that
   will change. Write the snapshot folder and `Restore.bat` **before** any
   write.
3. **Restore point.** Call `RestorePointService.CreateAsync`.
   - `Success` or `Throttled`: continue.
   - `Unverified` (once #41 lands) or any environmental failure such as
     `ProtectionDisabled`: continue only after an explicit confirmation that
     names the reason. The snapshot and `Restore.bat` are the primary recovery
     path, so a machine with System Protection off must still be able to use
     the feature.
   - An unexpected exception while creating the point blocks the apply.
4. **Dependency guard.**
   - Disabling: for every service set to `Disabled`, find services that depend
     on it that are not being disabled and are not already `Disabled`. If any
     exists, block that service and name the dependent in the preview.
   - Enabling: for every service set to `Automatic`, if one of its own
     dependencies stays `Disabled`, block it. `Manual` targets only warn,
     because they can be started on demand.
5. **Write.** Set `Start` for each service. Verify by reading back. Never stop
   a running service; the new start type takes effect after restart.
6. **Rollback on mismatch.** If any read-back differs, write the snapshot
   values back for every service this apply touched, verify, and report
   failure with the service names. A service rejected by Windows during the
   write is treated the same way: the whole apply is rolled back.
7. **Result.** Show what changed, what was skipped, and that a restart is
   required.

## Safety rails

1. Protected list always wins over any preset: RpcSs, DcomLaunch, RpcEptMapper,
   PlugPlay, Power, gpsvc, EventLog, ProfSvc, Schedule, SENS, Winmgmt,
   CryptSvc, nsi, mpssvc, SamSs, Winlogon, LSM, and anything in the
   `Type` driver range.
2. A service with an enabled dependent cannot be disabled. A service set to
   `Automatic` cannot have a dependency that stays `Disabled`.
3. No write happens before the snapshot and `Restore.bat` exist on disk.
4. Every write is read back; any mismatch or denied write rolls the whole apply
   back, but only for the services that apply actually touched.
5. No service is stopped or started during apply.
6. The preview is mandatory and lists every change and every block, including
   "denied by Windows".
7. The first use shows a one-time risk notice; it is not dismissible by
   accident.

## Restore

- **In-app:** list snapshots newest first, show the diff (current vs snapshot),
  apply the snapshot with the same write/verify/rollback path, and write a
  fresh snapshot for that restore first.
- **Offline:** `Restore.bat` supports two modes in one file, modelled on
  service-list-builder: live mode writes
  `HKLM\SYSTEM\CurrentControlSet\Services\...`; offline mode takes a
  `DRIVE_LETTER`, loads the offline hive with `reg load`, writes, then
  `reg unload`. The header explains both, and the script needs elevation.

## UI

- A "Service profiles" button in the Startup page header opens a
  `ContentDialog`:
  - preset list (name, description, service count)
  - preview: service, current start type, target start type, reason (`note`),
    status (change / no change / blocked with the dependent or reason /
    denied by Windows)
  - checkbox: "I understand a restart is required"
  - Apply / Cancel
- A second view in the same dialog lists snapshots with Restore.
- All new strings go through `lang/en_us.json`; other languages fall back.

## Failure handling

- Fail loud: a partial apply is rolled back and reported as a failure.
- Log every apply, rollback and restore with the snapshot path.
- A missing preset file or service is a preview warning, not a crash.

## Preset content plan

- `balanced`: services that are safe to disable on a normal desktop, with no
  network, audio, input or update impact. Small, conservative list.
- `minimal`: `balanced` plus telemetry, store, search-indexing-adjacent and
  consumer-app services that are commonly disabled in minimal images.
- Every entry carries a one-line note in the preset file so future curation has
  a reason attached.
- Curation is tested on a VM with a snapshot and a reboot before shipping.

## Tests

- Preset parsing: valid, unknown service, unknown start value, invalid JSON,
  duplicate service, driver entry, protected entry, non-writable entry,
  literal `name` fallback for drop-ins.
- Dependency guard: blocked when an enabled dependent exists; allowed when the
  dependent is also in the preset; ignored when the dependent is disabled;
  blocked when an `Automatic` target depends on a service that stays
  `Disabled`.
- Diff: current vs target classification.
- Restore script rendering: live and offline variants, escaping.
- Apply: read-back mismatch or denied write triggers rollback of all touched
  services.
- Snapshot round-trip: write, read, restore; restore writes its own snapshot.

Tests use fake catalog/store implementations; no test writes to real services.

## Rollout

- Ships with the two presets only. Users can add JSON files.
- No new navigation item and no hidden auto-apply.
- Preset ids and snapshot format are versioned by the folder/file names so a
  later format change does not break old snapshots.

## My calls (review these)

- Restore point is best-effort: when Windows cannot create one (protection
  off, throttle, unverified), the user continues only after a confirmation
  that names the reason, because the snapshot + `Restore.bat` is the real
  safety net and the feature would otherwise be unusable on many machines.
- Services whose registry key denies Administrators are blocked in the
  preview, not discovered as a failure during apply.
- A restore writes its own snapshot first, so a failed restore is undoable.
- Snapshots are not pruned in v1; each apply is a few KB.
- Snapshots live in `ProgramData`, not the registry, so the offline script and
  the app read the same data.
- The dialog handles both apply and restore instead of two entry points.
- No Settings feature flag; the risk notice is the gate.
- No `minimumBuild` field: unknown services are skipped, which covers build
  differences without a second dataset.

## Left to the builder

- Final protected list.
- Final Minimal/Balanced contents, curated and VM-tested.
- Localized copy and the risk-notice wording.
- Icon/glyph for the button.
