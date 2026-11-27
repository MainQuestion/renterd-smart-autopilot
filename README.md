# renterd Smart Autopilot

**Languages:** English · [Українська](README_UA.md) · [Русский](README_RU.md)

An unofficial experimental extension of Sia `renterd` for transparent host evaluation, contract portfolio management and smarter placement of new data.

If a host develops a problem (an inflated price, unavailability, insufficient collateral), the standard logic withdraws the contract and moves its data to other hosts, and moving data costs money. Smart Autopilot puts such a contract **on pause** and moves the data only when a replacement is economically justified (details in the "Core idea" section below).

> **Current release:** `v0.3.3-rc.1` — a Release Candidate with contract spending accounting (Σ storage, Σ ingress, Σ egress, Σ spent), a reworked Economics score, a pause for failed candidates, a fix for the Merkle-proof error after a contract renewal and an updated Active contracts table.

## Core idea: pause instead of migration

**A contract that has a problem does not have to be withdrawn and moved immediately. Smart Autopilot puts it on pause and moves the data only when that is really justified.**

### The problem

In the standard logic, a host that went beyond the allowed prices, became unavailable or fails the checks leads to the withdrawal of the contract and the move of its data to other hosts. Moving costs money: the data has to be read from the remaining hosts, reconstructed, written again, and the transfer and storage paid at the prices of the new hosts. A price jump or a short outage can start such moves on several hosts at once, and the expense becomes out of proportion to the problem that started it.

### What a pause does

- A contract on pause **receives no new data**, and the data already on the host **stays in place** and can remain readable: no move is started just because of an inflated price or a temporary failure.
- Paused placements **count as healthy redundancy** as long as there is enough data for recovery; if the available shards drop to `K` or fewer, a safety repair is started.
- A replacement is made **only when it is justified**: Smart Portfolio compares **KeepCost** and **MigrationCost** before moving the data. If a replacement is not justified, the contract gets a recoverable **Economic Pause**.
- A replacement is safe: **the new contract is created first, and only then the old one is withdrawn** through a persistent Smart Drain; data safety is checked before archiving.
- If the problem goes away, the contract returns to Good and no move was needed.

### How it looks

```text
Good
  → Paused (price / economics)
      → the problem went away                  → Good again, no move was needed
      → an eligible replacement appeared       → new contract → then drain of the old one
      → replacement not justified              → stays on the Economic Pause
      → hard threshold / timeout               → Bad → ADD (a new contract is formed)
```

The transitions `Good → Paused → Good` and `Paused → Bad → ADD` have been verified in practice (see "Verification").

### What is configurable

- The Smart mode (Original / Smart Shadow / Smart Active): pauses and replacements are carried out by **Smart Active**; Smart Shadow only writes to the log what the autopilot would have done.
- **Keep Gouging Contract Until Replacement**: whether a contract on pause may wait for a replacement or must be drained after the safety boundary.
- **Allow Existing Data on Overpriced Contracts** (off by default): keep the data already placed on an overpriced contract instead of moving it.
- **Smart Replacement Threshold** (default `15%`) on the **Scoring** page.

Where to see the result: `smart-autopilot.log` (economic pause/recovery, Gouging lifecycle, safety repair), the **Scoring** page. Detailed description: "Contract pauses" below.

## What changed

The original renterd mechanisms remain the low-level executing layer: contract formation, renew/refresh, upload/download, migration and archiving.

Smart Autopilot adds a policy layer on top of them:

```text
host measurements and history
        ↓
multidimensional host profile
        ↓
Smart Overall + policy gates
        ↓
PortfolioPlan
        ↓
ADD / PAUSE / RECOVER / REPLACE
        ↓
existing renterd mechanisms
```

The goal is not to replace renterd with one new "magic score", but to keep the different properties of a host separate and use the suitable criteria for each decision.

## What's new in v0.3.3-rc.1

- **Contract spending accounting.** The DB now keeps separate spending on storage, ingress and egress. The Active contracts table gets the columns **Σ storage, Σ ingress, Σ egress, Σ spent**, as well as My Data, Capacity, Initial/Remaining Allowance, Σ collateral and Used Collateral. The sums are counted along the renewal chain. Σ egress is a lower bound: renterd does not see cancelled and failed reads.
- **Economics reworked.** Collateral corrects the effective storage price, and the Economics Score is the Cost Score of the corrected cost. The cost and collateral anchors are taken only over usable hosts; hosts with zero cost do not distort the anchor.
- **Smart candidate pause.** A candidate whose contract formation failed through the host's fault is excluded from the plan for 24 hours (up to 7 days on repeated failures).
- **Merkle-proof fix.** Fixed a bug (an upstream defect) because of which, after a contract renewal, its revision broadcast was rejected (`file contract revision parent has invalid Merkle proof`). Elements that were already corrupted are repaired once at startup.
- **Faster price history.** A new `price_history` index speeds up the price history query about 35 times.
- **UI.** The column choice is remembered in Hosts and Active contracts, grouped contract columns, colors, sorting, totals under the headers, and tooltips that only repeat the value were removed.

See the release notes for details. Three database migrations (`00067`, `00068`, `00069`) run automatically and only add columns and an index.

## Implemented host profile

Four of the seven planned groups are fully complete in RC1:

- **G2 Performance** — actual upload/download speed from accumulated history.
- **G3 Resources** — the renter's exposure to a host: our data, free space, foreign data, financial exposure, recovery/migration cost, remaining contract term and share of lost sectors.
- **G4 Economics** — storage/ingress/egress cost corrected for the quality of the collateral: collateral corrects the effective storage price, and the final Economics Score is the Cost Score of the corrected cost (p10 anchor over the usable hosts).
- **G5 Risk Coverage** — historical stability of storage/ingress/egress prices and collateral.

Still planned as full separate scoring groups:

- **G1 Reliability**
- **G6 Technical Compatibility**
- **G7 Network Diversity**

Some compatibility/diversity checks are already used as eligibility/policy constraints, but these are not yet completed 1–10 groups.

## Smart Overall

The four completed groups are combined with user-defined weights:

```text
SmartOverall =
    G2 × W2 / 100 +
    G3 × W3 / 100 +
    G4 × W4 / 100 +
    G5 × W5 / 100
```

Persisted defaults: `25 / 25 / 25 / 25`.

Smart Overall is intentionally not the single criterion for all decisions.

## Three modes

### Original

Uses the regular renterd contract formation and maintenance policy. Smart Portfolio decisions and Smart upload ordering are not applied.

Persistent Smart safety lifecycles that are already running (including pauses) are completed safely.

### Smart Shadow

Computes the same Smart decisions as Active and writes them to `smart-autopilot.log`, but does not perform new Smart portfolio mutations (including starting new pauses).

For new uploads Shadow keeps the original `Uploader.Estimate()` order and logs it next to the Smart order.

### Smart Active

Executes Smart Portfolio decisions (including pauses and replacements) and uses Smart Placement for new user uploads.

Candidates are sorted by `SmartOverall DESC` inside the existing Good/Usable upload pool.

## Contract pauses

Pauses are the main mechanism that prevents unnecessary data moves (see "Core idea"). New pauses are started by Smart Active.

### The Paused state

A full `Paused` state was added between Good and Bad:

- new uploads are not accepted;
- existing data can remain readable;
- Paused placements count as healthy redundancy;
- recovery can return the contract to Good;
- a hard threshold/timeout moves it to Bad.

### Price pause (Gouging lifecycle)

Smart Active adds a separate safe lifecycle for an existing contract when the host's only problem is Gouging:

```text
Good
→ Gouging Paused
→ recover / replace / timeout drain
```

A Gouging-Paused contract:

- does not receive new uploads;
- can remain readable if the download price is acceptable;
- keeps its persistent Smart state;
- can be replaced as soon as an eligible replacement appears;
- can start a safety repair when the available shards drop to `K` or fewer.

The **Keep Gouging Contract Until Replacement** setting determines whether such a contract may keep waiting for/renewing a replacement or must be drained after the safety boundary.

#### Allow Existing Data on Overpriced Contracts

The **Allow Existing Data on Overpriced Contracts** setting lets Smart Active keep the data already placed on a Gouging contract instead of moving the data only because of an inflated price.

- The setting is on the **Scoring** page and is **off by default**, so the previous behavior is kept.
- With the setting on, Smart can still form a replacement, but the old contract is not put into drain just because a replacement has already been created.
- After a successful Gouging replacement the old contract becomes a read-only **legacy** contract while its host remains Gouging: new uploads do not go there, renew/refresh to keep the Gouging contract are not performed, and it does not occupy a Smart Portfolio slot.
- Existing data can remain available through the active read/download checks. If the host stops being Gouging, the legacy contract can recover through the regular Gouging recovery lifecycle.
- If the setting is turned off again, **Smart Active** can start a drain of the retained legacy data.
- Contract expiration protection is independent of the price policy: for a Gouging/legacy contract that will no longer be renewed, the existing expiration-safety drain starts on entering the second half of the renew window.
- Gouging safety repair does not force data to be moved while the policy allows keeping it; in the corresponding case the decision is recorded as `allowed_existing_data`.

### Economic pause (Economic Pause)

Smart Portfolio compares **KeepCost** and **MigrationCost** before moving the data. If a replacement is not economically justified, the contract is put on a recoverable **Economic Pause** instead of being moved. More about the replacement rules: "Smart Portfolio".

## Smart Portfolio

Smart Portfolio can:

- fill missing contract slots through **Smart ADD**;
- find the weakest current Good incumbent by Smart Overall;
- apply a configurable **G4 Economics improvement threshold**;
- compare **KeepCost** and **MigrationCost** before moving data;
- put a contract on a recoverable **Economic Pause** if a replacement is not economically justified (see "Contract pauses");
- perform a safe **REPLACE**, creating the new contract first;
- withdraw the old contract through a persistent **Smart Drain**;
- check data safety before archiving;
- temporarily exclude a candidate whose formation failed through the host's fault (24 hours, up to 7 days on repeated failures; kept in memory and reset on restart). This is not a contract pause but an exclusion of the host from the candidates for new contracts.

Default Smart Replacement Threshold: `15%`.

The regular optimization REPLACE has a 4-hour cadence. ADD is not blocked by it.

## History and diagnostics

Smart Autopilot uses history of:

- upload/download speed;
- prices and collateral;
- scan results.

A separate:

```text
smart-autopilot.log
```

records ADD, REPLACE, economic pause/recovery, the Gouging lifecycle, safety repair and the comparison of the Original/Smart upload order.

## UI

Added:

- a separate **Scoring** page;
- configurable legacy host-score parameters;
- Smart group weights;
- Smart Replacement Threshold;
- Keep Gouging Contract Until Replacement;
- Allow Existing Data on Overpriced Contracts;
- grouped host-profile columns;
- raw upload/download Mbps;
- Smart Overall;
- improved host sorting;
- table and number display settings;
- the **Active contracts** table with column groups (General, Time, Contract data, Financial, Financial original), the columns Σ spent, Σ storage, Σ ingress, Σ egress, My Data, Capacity, Allowance and collateral, sorting, totals under the headers and a remembered column choice.

## Verification

Before publishing v0.3.3-rc.1 it passed build/vet/test (including e2e and `-race` on the affected packages), the Merkle-proof fix test on a real chain, and practical testing on a working installation: migrations `00067`–`00069`, a one-time repair of 8 contracts (`invalid: 8` → `repaired: 8`, and `invalid: 0` on the next start), a successful revision broadcast, and a check of the contracts table in the browser. The release build was made from the same commits as the one running on the working installation.

Earlier, before publishing RC1, the Smart paths were checked:

```text
4-hour optimization REPLACE cadence
ADD before the cadence expires when a contract drops out
Good → Paused → Good
Paused → Bad → ADD
```

## Build information

```text
Release:        v0.3.3-rc.1
renterd:        v0.3.3-rc.1
Backend commit: 3738c027
Web commit:     83a2dde8
Network:        mainnet
```

```text
renterd.exe
Size: 62,078,153 bytes
SHA256: C9FB10B32D08BBD12150C2C34ED1C38092B472B55BCC2BD7A797E26423B94F17
```

```text
renterd-smart-autopilot-v0.3.3-rc.1-windows-amd64.zip
Size: 22011895 bytes
SHA256: E0AB8B28DD4A2701535D02E4F177D1E29BE4B83EE7DD6397998C58CC049D8B68
```

## Installation

1. Back up the renterd data and configuration (for SQLite — a `.backup` of the DB and `metrics.sqlite`).
2. Download `renterd-smart-autopilot-v0.3.3-rc.1-windows-amd64.zip` from the GitHub Release.
3. Extract `renterd.exe` and `LICENSE`.
4. Stop renterd and replace the executable.
5. For the first start use **Original** or **Smart Shadow**, and then switch to **Smart Active** if needed.

Database migrations are run by the standard renterd mechanism. On the first start migrations `00067`–`00069` will run (creating the index takes seconds), and the log will show the lines `contract elements checked` and, if needed, `contract elements repaired`. Rollback: put back the previous `renterd.exe`; the old binary starts on a DB with the new migrations.

## Detailed comparison

See [`ORIGINAL_VS_SMART_AUTOPILOT.md`](ORIGINAL_VS_SMART_AUTOPILOT.md).

## Release

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.3.3-rc.1

## License and upstream

This is an unofficial experimental derivative build based on Sia Foundation renterd.

See `LICENSE`.

This is not an official Sia Foundation release. RC versions are intended for testing; keep backups and watch the logs/alerts.
