# renterd Smart Autopilot

**Languages:** English · [Українська](README_UA.md) · [Русский](README_RU.md)

An unofficial experimental extension of Sia `renterd` for transparent
host scoring, contract portfolio management and smarter placement of
new data.

> **Current release:** `v0.3.3-rc.1` --- a Release Candidate with
> per-contract spending accounting (Σ storage, Σ ingress, Σ egress,
> Σ spent), a reworked Economics score, a pause for failed candidates,
> a fix for the Merkle-proof error after a contract renewal and an
> updated Active contracts table.

## What changed

The original renterd mechanisms remain the low-level execution layer:
contract formation, renew/refresh, upload/download, migration and
archival.

Smart Autopilot adds a policy layer on top of them:

``` text
host measurements and history
        ↓
multi-dimensional host profile
        ↓
Smart Overall + policy gates
        ↓
PortfolioPlan
        ↓
ADD / PAUSE / RECOVER / REPLACE
        ↓
existing renterd mechanisms
```

The goal is not to replace renterd with one new "magic score", but to
keep the different properties of a host separate and to use the right
criteria for each decision.

## What is new in v0.3.3-rc.1

-   **Per-contract spending accounting.** The database now records
    storage, ingress (data received by the host) and egress (reads)
    spending separately. The Active contracts table gained the columns
    **Σ storage, Σ ingress, Σ egress, Σ spent**, as well as My Data,
    Capacity, Initial/Remaining Allowance, Σ collateral and Used
    Collateral. Sums are taken over the renewal chain. Σ egress is a
    lower bound: cancelled and failed reads are not visible to renterd.
-   **Economics reworked.** Collateral corrects the effective storage
    price, and the Economics Score is the Cost Score of the corrected
    cost. The cost and collateral anchors are taken from the usable
    hosts only; hosts with a zero cost do not distort the anchor.
-   **Smart candidate pause.** A candidate whose contract formation
    failed through the host's fault is left out of the plan for 24 hours
    (up to 7 days after repeated failures).
-   **Merkle-proof fix.** Fixed a bug (an upstream defect) that made
    revision broadcast for a renewed contract be rejected (`file
    contract revision parent has invalid Merkle proof`). Elements that
    were already corrupted are repaired once at startup.
-   **Faster price history.** A new `price_history` index speeds up the
    price history query about 35 times.
-   **UI.** The column choice is remembered in Hosts and Active
    contracts, grouped contract columns, colors, sorting, totals under
    the headers, and tooltips that only repeat the value were removed.

See the release notes for details. Three database migrations (`00067`,
`00068`, `00069`) run automatically and only add columns and an index.

## Implemented host profile

Four of the seven planned groups are fully complete in RC1:

-   **G2 Performance** --- actual upload/download speed from accumulated
    history.
-   **G3 Resources** --- the renter's exposure to a host: our data, free
    space, foreign data, financial exposure, recovery/migration cost,
    remaining contract term and share of lost sectors.
-   **G4 Economics** --- storage/ingress/egress cost corrected for the
    quality of the collateral: collateral corrects the effective storage
    price, and the final Economics Score is the Cost Score of the
    corrected cost (p10 anchor over the usable hosts).
-   **G5 Risk Coverage** --- historical stability of
    storage/ingress/egress prices and collateral.

Still planned as full separate scoring groups:

-   **G1 Reliability**
-   **G6 Technical Compatibility**
-   **G7 Network Diversity**

Some compatibility/diversity checks are already used as
eligibility/policy constraints, but these are not yet completed 1--10
groups.

## Smart Overall

The four completed groups are combined with user-defined weights:

``` text
SmartOverall =
    G2 × W2 / 100 +
    G3 × W3 / 100 +
    G4 × W4 / 100 +
    G5 × W5 / 100
```

Persisted defaults: `25 / 25 / 25 / 25`.

Smart Overall is intentionally not the single criterion for all
decisions.

## Three modes

### Original

Uses the regular renterd contract formation and maintenance policy.
Smart Portfolio decisions and Smart upload ordering are not applied.

Persistent Smart safety lifecycles that are already running are
completed safely.

### Smart Shadow

Computes the same Smart decisions as Active and writes them to
`smart-autopilot.log`, but does not perform new Smart portfolio
mutations.

For new uploads Shadow keeps the original `Uploader.Estimate()` order
and logs it next to the Smart order.

### Smart Active

Executes Smart Portfolio decisions and uses Smart Placement for new
user uploads.

Candidates are sorted by `SmartOverall DESC` inside the existing
Good/Usable upload pool.

## Smart Portfolio

Smart Portfolio can:

-   fill missing contract slots through **Smart ADD**;
-   find the weakest current Good incumbent by Smart Overall;
-   apply a configurable **G4 Economics improvement threshold**;
-   compare **KeepCost** and **MigrationCost** before moving data;
-   put a contract into a recoverable **Economic Pause** if a
    replacement is not economically justified;
-   perform a safe **REPLACE**, creating the new contract first;
-   retire the old contract through a persistent **Smart Drain**;
-   check data safety before archival;
-   temporarily exclude a candidate whose formation failed through the
    host's fault (24 hours, up to 7 days after repeated failures; kept
    in memory and reset on restart).

Default Smart Replacement Threshold: `15%`.

A regular optimization REPLACE has a 4-hour cadence. ADD is not blocked
by it.

## Gouging lifecycle

Smart Active adds a separate safe lifecycle for an existing contract
when the host's only problem is Gouging:

``` text
Good
→ Gouging Paused
→ recover / replace / timeout drain
```

A Gouging-Paused contract:

-   receives no new uploads;
-   can stay readable if the download price is acceptable;
-   keeps a persistent Smart state;
-   can be replaced as soon as an eligible replacement appears;
-   can trigger a safety repair when the number of available shards
    drops to `K` or less.

The **Keep Gouging Contract Until Replacement** setting defines whether
such a contract may keep waiting for / renewing a replacement or must be
drained after the safety boundary.

### Allow Existing Data on Overpriced Contracts

The **Allow Existing Data on Overpriced Contracts** setting lets Smart
Active keep data that is already placed on a Gouging contract instead of
moving it only because of an inflated price.

-   The setting is on the **Scoring** page and is **off by default**, so
    the previous behavior is preserved.
-   With the setting on, Smart can still form a replacement, but the old
    contract is not moved to drain just because a replacement already
    exists.
-   After a successful Gouging replacement the old contract becomes a
    read-only **legacy** contract while its host stays Gouging: no new
    uploads go there, no renew/refresh is performed to keep the Gouging
    contract, and it does not occupy a Smart Portfolio slot.
-   Existing data can stay available through the current read/download
    checks. If the host stops being Gouging, the legacy contract can
    recover through the regular Gouging recovery lifecycle.
-   If the setting is turned off again, **Smart Active** can start a
    drain of the retained legacy data.
-   Expiration protection is independent of the price policy: for a
    Gouging/legacy contract that will not be renewed any more, the
    existing expiration-safety drain starts when it enters the second
    half of the renew window.
-   Gouging safety repair does not force data to move while the policy
    allows keeping it; in the relevant case the decision is recorded as
    `allowed_existing_data`.

## Paused

The project also adds a full `Paused` state between Good and Bad:

-   new uploads are not accepted;
-   existing data can stay readable;
-   Paused placements count as healthy redundancy;
-   recovery can return the contract to Good;
-   a hard threshold/timeout moves it to Bad.

## History and diagnostics

Smart Autopilot uses history of:

-   upload/download speed;
-   prices and collateral;
-   scan results.

A separate:

``` text
smart-autopilot.log
```

records ADD, REPLACE, economic pause/recovery, Gouging lifecycle, safety
repair and the Original/Smart upload order comparison.

## UI

Added:

-   a separate **Scoring** page;
-   configurable legacy host-score parameters;
-   weights of the Smart groups;
-   Smart Replacement Threshold;
-   Keep Gouging Contract Until Replacement;
-   Allow Existing Data on Overpriced Contracts;
-   grouped host-profile columns;
-   raw upload/download Mbps;
-   Smart Overall;
-   improved host sorting;
-   table and number display settings;
-   the **Active contracts** table with column groups (General, Time,
    Contract data, Financial, Financial original), the columns Σ spent,
    Σ storage, Σ ingress, Σ egress, My Data, Capacity, Allowance and
    collateral, sorting, totals under the headers and a remembered
    column choice.

## Verification

Before publishing v0.3.3-rc.1 it passed build/vet/test (including e2e
and `-race` on the affected packages), a Merkle-proof repair test on a
real chain, and practical testing on a production installation:
migrations `00067`--`00069`, the one-time repair of 8 contracts
(`invalid: 8` → `repaired: 8`, `invalid: 0` at the next start),
successful revision broadcast, and a browser check of the contracts
table. The release build is built from the same commits as the one
running on the production installation.

Earlier, before publishing RC1, the Smart paths were verified:

``` text
4-hour optimization REPLACE cadence
ADD before the cadence expires when a contract drops out
Good → Paused → Good
Paused → Bad → ADD
```

## Build information

``` text
Release:        v0.3.3-rc.1
renterd:        v0.3.3-rc.1
Backend commit: 3738c027
Web commit:     83a2dde8
Network:        mainnet
```

``` text
renterd.exe
Size: 62,078,153 bytes
SHA256: C9FB10B32D08BBD12150C2C34ED1C38092B472B55BCC2BD7A797E26423B94F17
```

``` text
renterd-smart-autopilot-v0.3.3-rc.1-windows-amd64.zip
Size: 22011895 bytes
SHA256: E0AB8B28DD4A2701535D02E4F177D1E29BE4B83EE7DD6397998C58CC049D8B68
```

## Installation

1.  Back up the renterd data and configuration (for SQLite: `.backup` of
    the database and `metrics.sqlite`).
2.  Download `renterd-smart-autopilot-v0.3.3-rc.1-windows-amd64.zip`
    from the GitHub Release.
3.  Extract `renterd.exe` and `LICENSE`.
4.  Stop renterd and replace the executable.
5.  For the first run use **Original** or **Smart Shadow**, then switch
    to **Smart Active** if needed.

Database migrations run through the standard renterd mechanism. On the
first start migrations `00067`--`00069` run (creating the index takes
seconds), and the log shows `contract elements checked` and, if needed,
`contract elements repaired`. Rollback: put the previous `renterd.exe`
back; the old binary starts on a database with the new migrations.

## Detailed comparison

See [`ORIGINAL_VS_SMART_AUTOPILOT.md`](ORIGINAL_VS_SMART_AUTOPILOT.md).

## Release

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.3.3-rc.1

## License and upstream

This is an unofficial experimental derivative build based on Sia
Foundation renterd.

See `LICENSE`.

This is not an official Sia Foundation release. RC versions are meant
for testing; keep backups and monitor logs/alerts.
