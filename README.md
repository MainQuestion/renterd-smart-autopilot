# renterd Smart Autopilot

**Languages:** English · [Українська](README_UA.md) · [Русский](README_RU.md)

Unofficial experimental extension of Sia `renterd` focused on transparent host evaluation, contract portfolio management, and smarter placement of new data.

> **Current release:** `v0.3.0-rc.1` — first tested Release Candidate with an active Smart Portfolio decision and execution layer.

## What changed

The original renterd mechanisms remain the low-level execution layer: contract formation, renew/refresh, upload/download, migration, and archival.

Smart Autopilot adds a policy layer above them:

```text
host measurements/history
        ↓
multidimensional host profile
        ↓
Smart Overall + policy gates
        ↓
PortfolioPlan
        ↓
ADD / PAUSE / RECOVER / REPLACE
        ↓
existing renterd executors
```

The goal is not to replace renterd with one new "magic score". The goal is to keep separate host properties visible and use the right criteria for each decision.

## Implemented host profile

Four of the seven planned groups are complete in RC1:

- **G2 Performance** — measured upload/download performance from history.
- **G3 Resources** — renter exposure to the host: own data, free space, foreign data, financial exposure, migration/recovery cost, remaining contract lifetime, and lost-sector share.
- **G4 Economics** — storage/ingress/egress cost plus collateral quality. Final Economics uses the weaker of Cost Score and Collateral Score.
- **G5 Risk Coverage** — historical stability of storage/ingress/egress prices and collateral.

Still planned as full standalone scoring groups:

- **G1 Reliability**
- **G6 Technical Compatibility**
- **G7 Network Diversity**

Some compatibility and diversity checks already exist as eligibility/policy constraints, but they are not yet complete 1–10 profile groups.

## Smart Overall

The four completed groups can be combined using user-controlled weights:

```text
SmartOverall =
    G2 × W2 / 100 +
    G3 × W3 / 100 +
    G4 × W4 / 100 +
    G5 × W5 / 100
```

Default persisted weights are `25 / 25 / 25 / 25`.

Smart Overall is intentionally not used as the only criterion for every decision. ADD, REPLACE, economics, safety, and upload placement each use the policy most appropriate to the task.

## Three operating modes

### Original

Uses the ordinary renterd contract formation/maintenance policy. Smart Portfolio decisions and Smart upload ordering are not applied.

Already-started persistent Smart safety lifecycles are allowed to finish safely.

### Smart Shadow

Computes the same Smart decisions as Active and writes them to `smart-autopilot.log`, but does not execute new Smart portfolio mutations.

For new uploads, Shadow keeps the original `Uploader.Estimate()` order and logs it next to the Smart order.

### Smart Active

Executes Smart Portfolio decisions and uses Smart placement for new user uploads.

New upload candidates are ordered by `SmartOverall DESC` within the existing Good/Usable upload pool.

## Smart Portfolio

RC1 can:

- fill missing contract slots with **Smart ADD**;
- identify the weakest current Good incumbent using Smart Overall;
- require a configurable **G4 Economics improvement threshold** for optimization replacement;
- compare **KeepCost** with **MigrationCost** before migrating data;
- place a contract into recoverable **Economic Pause** when replacement is not economically justified;
- perform safe **REPLACE** by forming the new contract first;
- evacuate the old contract through persistent **Smart Drain**;
- verify migration safety before archival.

Default Smart Replacement Threshold: `15%`.

Normal optimization REPLACE uses a 4-hour cadence. ADD is not blocked by that cadence.

## Gouging lifecycle

Smart Active adds a safer lifecycle for an existing contract whose only problem is Gouging:

```text
Good
→ Gouging Paused
→ recover / replace / timeout drain
```

A Gouging-Paused contract:

- receives no new uploads;
- can remain readable when the download price is still acceptable;
- keeps persistent Smart state;
- can be replaced as soon as an eligible replacement exists;
- can trigger safety repair if available shards fall to `K` or below.

The configurable **Keep Gouging Contract Until Replacement** option controls whether a Gouging-Paused contract may continue waiting/renewing for a replacement or is drained after the configured downtime safety boundary.

## Paused contract state

The project also adds a real `Paused` usability state between Good and Bad.

Paused is designed as a grace state:

- no new uploads;
- existing data can remain readable;
- Paused placements still count as healthy redundancy;
- recovery can return the contract to Good;
- hard threshold/timeout can escalate it to Bad.

## History and diagnostics

Smart Autopilot adds/uses history for:

- upload/download speed;
- prices and collateral;
- scan results.

A dedicated:

```text
smart-autopilot.log
```

records Smart decisions such as ADD, REPLACE, economic pause/recovery, Gouging lifecycle decisions, safety repair, and original vs Smart upload placement order.

## UI additions

The renterd UI includes:

- a dedicated **Scoring** page;
- configurable legacy host-score parameters;
- Smart group weights;
- Smart Replacement Threshold;
- Keep Gouging Contract Until Replacement;
- grouped host profile columns;
- raw upload/download Mbps;
- Smart Overall;
- host sorting improvements;
- configurable numeric/table display preferences.

## RC1 validation

Before publication, RC1 passed build/vet/test and targeted/e2e coverage for the Smart paths. It was also tested on a working renterd installation, including:

```text
4-hour optimization REPLACE cadence
ADD before the cadence expires when a contract is lost
Good → Paused → Good
Paused → Bad → ADD
```

The release binary in this archive is byte-identical to the installed binary used for final testing.

## Build information

```text
Release:        v0.3.0-rc.1
renterd:        v2.9.4-28-g8b23ee01
Backend commit: 8b23ee01
Web commit:     8b42f0b5
Network:        mainnet
```

Windows amd64 executable:

```text
renterd.exe
Size: 61,659,108 bytes
SHA256: CE7A9343A1F58C52B44383EFF16C51B4A2D9D3D7DF35E5789ED32577EB988EB2
```

Release archive:

```text
renterd-smart-autopilot-v0.3.0-rc.1-windows-amd64.zip
Size: 22,298,517 bytes
SHA256: E9A016C202E53EEAD39935D9A77B6CFF0A9CB0449A2F1FF2E7349E5A744D5FDD
```

## Installation

1. Back up your renterd data/configuration before testing an RC build.
2. Download `renterd-smart-autopilot-v0.3.0-rc.1-windows-amd64.zip` from the GitHub Release.
3. Extract `renterd.exe` and `LICENSE`.
4. Replace your renterd executable while renterd is stopped.
5. Start in **Original** or **Smart Shadow** first if you want to observe Smart decisions before enabling **Smart Active**.

Existing databases are upgraded through renterd's normal migration mechanism.

## Detailed comparison

See [`ORIGINAL_VS_SMART_AUTOPILOT.md`](ORIGINAL_VS_SMART_AUTOPILOT.md) for the detailed Original vs Smart Autopilot RC1 behavior.

## Release

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.3.0-rc.1

## License and upstream

This project is an unofficial experimental derivative based on Sia Foundation's renterd.

The distributed build includes the project license. See `LICENSE`.

This project is not an official Sia Foundation release. RC builds are intended for testing; keep backups and monitor logs/alerts.
