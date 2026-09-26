# renterd Smart Autopilot

[English](README.md) | [Українська](README.uk.md) | [Русский](README.ru.md)

> **Unofficial experimental build**
>
> This project is an experimental modification of
> [SiaFoundation/renterd](https://github.com/SiaFoundation/renterd).
>
> It is not an official Sia Foundation release and should currently be
> considered alpha software.

## Overview

**renterd Smart Autopilot** is an experimental extension of Sia renterd focused
on improving host evaluation, contract management, and eventually data
placement decisions.

The long-term goal is to move away from relying on a single opaque host score
and instead build a transparent multidimensional host profile.

The target model is:

```text
Host =
    Reliability
  + Performance
  + Resources
  + Economics
  + Risk
  + Technical Compatibility
  + Network Diversity
```

These dimensions are intentionally kept separate.

A host may, for example, be very reliable and inexpensive while being slower
than the rest of the pool. Another host may be fast but introduce excessive
infrastructure concentration risk.

Smart Autopilot is intended to preserve these differences instead of hiding
them inside one aggregate number.

---

## Current release

### v0.2.0-alpha.1

This release is based on:

```text
renterd: v2.9.4-17-g240bd51e
Commit:  240bd51e
Network: mainnet
```

Windows build:

```text
renterd-smart-autopilot-v0.2.0-alpha.1-windows-amd64.zip
```

SHA256:

```text
B7C451CB4C52447FE2CCC1FAD37E48648C1CE259363D7E380C68E7CE11C7CE30
```

Contained executable:

```text
renterd.exe
Size:   61,056,199 bytes
SHA256: C91A6F16E1641E63AC6513350B0B64563E6DFFAFEBAD3233E6737EB544DE3C84
```

Release page:

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.2.0-alpha.1

Direct Windows archive:

https://github.com/MainQuestion/renterd-smart-autopilot/releases/download/v0.2.0-alpha.1/renterd-smart-autopilot-v0.2.0-alpha.1-windows-amd64.zip

---

# Implemented features

## Configurable renterd host scoring

Several parameters of renterd's existing host scoring mechanism that were
previously effectively fixed can now be configured by the user.

Current configurable parameters include:

- Storage free-space weight
- Storage penalty exponent
- Allocation buffer factor
- Interaction penalty exponent
- Interaction trust prior
- Individual legacy scoring factors can be disabled

The default values are intended to preserve the original renterd behavior.

These settings are exposed through a dedicated **Scoring** page in the renterd
web interface.

The existing renterd score is not removed. Smart Autopilot is being developed
alongside it so that existing behavior remains understandable and
configurable while the new profile-based model evolves.

---

## Performance

A separate **Performance Score** evaluates actual host transfer performance.

It is deliberately maintained independently from renterd's original aggregate
host score.

The current implementation uses historical upload and download measurements
and compares each host with the current pool.

The model includes:

- Historical upload speed
- Historical download speed
- Separate upload and download evaluation
- Pool-relative normalization
- P95 pool reference
- Logarithmic score compression
- Amount of available measurement data
- A combined Performance Score

Performance history is evaluated over a rolling historical window rather than
using only the most recent transfer.

The Hosts table exposes:

```text
Performance
Upload
Download
```

`Performance` is the normalized score.

`Upload` and `Download` show measured raw throughput in Mbps so that the score
can be inspected against the underlying real-world measurements.

Performance remains an independent dimension and is not multiplied into the
legacy renterd host score.

---

## Economics

A separate **Economics Score** estimates the expected cost of using each host
for the renter's actual storage and bandwidth profile.

The calculation considers:

- Storage price
- Upload / ingress price
- Download / egress price
- Expected storage volume
- Expected monthly upload volume
- Expected monthly download volume

The usage profile is taken from renterd's existing contract configuration:

```text
Contracts.Storage
Contracts.Upload
Contracts.Download
Contracts.Period
```

This keeps one source of truth for the renter's expected storage and traffic
requirements instead of maintaining duplicate Economics-specific settings.

Upload and download values stored for the entire contract period are converted
to their monthly equivalents before being used by the Economics model.

For each host, Smart Autopilot first estimates the total expected cost:

```text
TotalCost =
    StoragePrice × StorageVolume
  + IngressPrice × MonthlyUpload
  + EgressPrice × MonthlyDownload
```

The result is then normalized against the current host pool.

The lower-cost end of the pool uses a percentile-based reference instead of a
single raw minimum so that one anomalously cheap host does not distort the
entire scale.

The resulting score ranges from approximately:

```text
1 = expensive relative to the pool
10 = among the cheapest hosts in the pool
```

The Hosts table also exposes the underlying raw:

```text
Storage price
Ingress price
Egress price
```

This keeps the Economics Score auditable instead of presenting only the final
number.

---

## Resources

A separate **Resources Score** evaluates the renter's current exposure to a
host and the potential consequences if that host becomes unavailable.

This group is not simply a measurement of how much free disk space the host
advertises.

The current model evaluates seven signals.

### Data and capacity

```text
Renter data stored on the host
Free storage
Storage occupied by other data
```

### Financial and contract exposure

```text
Financial exposure for the remaining contract lifetime
Estimated data recovery / migration cost
Remaining contract duration
```

### Sector-loss history

```text
Lost-sector share
```

The seven parameters are normalized relative to the current pool.

They are then combined as three conceptual blocks rather than using a flat
1/7 weighting:

```text
Block A = data / capacity
Block B = money / time
Block C = sector-loss share

Resources Score =
    (Block A + Block B + Block C) / 3
```

This structure intentionally gives sector-loss history enough influence that a
serious loss rate cannot disappear inside an average of many otherwise good
metrics.

The Resources group therefore answers a different question from the original
renterd storage-remaining factor:

> How costly and disruptive would losing this particular host be right now?

It exists alongside the legacy score rather than replacing it.

The Hosts table can expose the underlying values, including:

```text
Own data
Free space
Foreign data
Exposure
Recovery compensation estimate
Contract time remaining
Lost-sector share
```

---

## Risk Coverage

A separate **Risk Coverage Score** evaluates historical changes in host pricing
and collateral behavior.

The current implementation tracks events affecting:

```text
Storage price
Ingress price
Egress price
Collateral
```

The number of observed events is evaluated over a configurable historical
window.

The analysis window is available on the **Scoring** page through the
Risk Price Window setting.

The current default window is:

```text
14 days
```

The Hosts table exposes both the resulting Risk Coverage Score and the raw
event counters so that the reason behind the score remains visible.

This group is intentionally separate from Economics.

Economics answers:

> How expensive is the host now?

Risk Coverage is intended to answer:

> How stable has the host's economic behavior been over time?

---

## Host history

Smart Autopilot stores historical host observations that can be used by the
new scoring mechanisms.

Current history tables include:

```text
speed_history
price_history
scan_history
```

This provides a foundation for decisions based on host behavior over time
instead of relying exclusively on the latest scan or latest settings snapshot.

---

## Paused contracts

Smart Autopilot adds a **Paused** contract state between a healthy contract and
a contract that must immediately be treated as bad.

The purpose is to avoid turning every temporary host problem into an immediate
hard failure.

The contract state model can therefore distinguish:

```text
Good
Paused
Bad
```

A temporarily problematic contract may remain paused while the system waits
for recovery.

If the configured timeout or failure conditions are exceeded, it can progress
to `Bad`.

The Active Contracts interface exposes this state separately from the existing
good/bad contract handling.

---

## Host table groups

The Hosts table has been extended to present Smart Autopilot information as
separate expandable groups.

Currently implemented groups are:

```text
Performance
Economics
Resources
Risk Coverage
```

The main score for each group remains visible while its underlying raw metrics
can be expanded when more detail is needed.

This is an important part of the Smart Autopilot design: the user should be
able to understand *why* a host received a particular evaluation.

---

## Table display preferences

Additional table presentation controls are available through
**App preferences**.

Current settings include:

```text
Row size
Number size
Number weight
Number font
```

For number fonts, both normal sans-serif and monospaced presentation are
available.

The default Smart Autopilot presentation uses a relatively dense layout to make
large host and contract tables easier to inspect.

These settings are currently applied to renterd's:

```text
Hosts
Active Contracts
```

tables.

The settings infrastructure is shared by the web applications, although this
development phase applies the behavior specifically to the renterd tables
above.

---

## Web UI improvements

Several UI changes accompany the Smart Autopilot work.

These include:

- Dedicated Scoring configuration page
- Expandable host-profile groups
- Raw metrics displayed alongside normalized scores
- Denser Hosts and Active Contracts tables
- Configurable numeric typography
- Monospaced numeric display option
- Improved visibility of contract states
- Persistent table and group preferences

The renterd SPA layout handling was also corrected to prevent React hook-order
errors that could previously occur during client-side navigation between
pages.

---

# Smart Autopilot architecture

The target architecture evaluates hosts through seven independent dimensions
instead of relying exclusively on one aggregate score.

Four profile groups are already implemented in the current alpha line:

```text
Performance
Resources
Economics
Risk Coverage
```

The remaining major groups are still under development:

```text
Reliability
Technical Compatibility
Network Diversity
```

The exact boundaries of these groups may continue to evolve as the real
renterd code paths and available data are studied.

---

## 1. Reliability

Intended to describe how consistently a host remains available and fulfills
its obligations.

Potential inputs include:

```text
Uptime
Interaction history
Consecutive failures
Host age / historical presence
Scan history
```

The goal is to distinguish long-term operational reliability from raw
performance.

---

## 2. Performance

Implemented.

Measures how quickly the host actually serves renter workloads.

Current inputs include historical:

```text
Upload speed
Download speed
```

with pool-relative normalization and a separate visible Performance Score.

Future versions may extend this dimension with additional latency and
operation-time measurements where renterd exposes reliable data.

---

## 3. Resources

Implemented.

Evaluates current renter exposure, capacity, migration burden, contract
remaining time, and sector-loss history.

It is intentionally broader than simply measuring advertised free space.

---

## 4. Economics

Implemented.

Estimates expected host cost using the renter's actual configured storage,
upload, and download profile and compares it with the current host pool.

---

## 5. Risk

Partially implemented through the current **Risk Coverage** model.

The current mechanism evaluates historical price and collateral change events.

This dimension may later contain additional risk mechanisms as they are
designed and validated against renterd's actual data and contract behavior.

---

## 6. Technical Compatibility

Planned.

This group is intended to determine whether a host is technically usable by the
renter before ranking considerations are applied.

Potential conditions include:

```text
Contract acceptance
Host announcement state
Successful scan completion
Required protocol support
Required host functionality
Other renterd usability gates
```

Some of these checks already exist inside renterd today.

The Smart Autopilot work will first identify and preserve their current
semantics before deciding how they should participate in the new profile.

---

## 7. Network Diversity

Planned.

This group is intended to measure shared infrastructure risk between hosts
rather than evaluate each host entirely in isolation.

Potential signals include:

```text
IP address
Subnet
ASN
Geographic information, where reliably available
Infrastructure overlap with already selected hosts
```

The purpose is to reduce correlated failure risk when distributing redundant
data.

---

# Design principles

## Research the existing renterd behavior first

Changes should be based on the actual renterd implementation rather than on
assumptions about how the system probably works.

The existing mechanism is studied before changing it.

---

## Preserve existing behavior where possible

Smart Autopilot is being developed incrementally.

The intention is not to silently replace existing renterd behavior without
understanding why that behavior exists.

New profile dimensions are therefore being introduced alongside existing
mechanisms first.

---

## Keep scoring transparent

A single score can hide important trade-offs.

For example:

```text
Host A
Reliability:  9/10
Performance:  4/10
Economics:    8/10
Risk:         9/10
Resources:   10/10
```

This is often more useful than reducing all of those characteristics to one
number.

Raw measurements are exposed where practical so that normalized scores can be
checked against their source data.

---

## Preserve user configurability

renterd already gives users significant control over how their renter should
operate.

Smart Autopilot should extend that philosophy rather than replace it with
another collection of hidden constants.

Parameters that meaningfully affect policy or scoring should be configurable
where doing so is practical and safe.

---

## Separate measurement from decision-making

Measuring a host and deciding what to do with that host are different
problems.

The current work primarily builds a richer host profile.

Future decision logic can then use that profile for:

```text
Host selection
Contract formation
Contract renewal
Contract replacement
Data placement
Migration decisions
Network-diversity management
```

without requiring every measurement to be collapsed into one global score.

---

# Installation

## Windows amd64

Download:

```text
renterd-smart-autopilot-v0.2.0-alpha.1-windows-amd64.zip
```

from the GitHub Releases page:

https://github.com/MainQuestion/renterd-smart-autopilot/releases/tag/v0.2.0-alpha.1

The archive contains:

```text
renterd.exe
LICENSE
```

This is an experimental alpha build.

Back up your renterd configuration and data before replacing an existing
installation.

---

## Verify the archive

Expected SHA256:

```text
B7C451CB4C52447FE2CCC1FAD37E48648C1CE259363D7E380C68E7CE11C7CE30
```

PowerShell:

```powershell
Get-FileHash .\renterd-smart-autopilot-v0.2.0-alpha.1-windows-amd64.zip -Algorithm SHA256
```

Expected result:

```text
B7C451CB4C52447FE2CCC1FAD37E48648C1CE259363D7E380C68E7CE11C7CE30
```

---

## Verify the executable

Expected executable SHA256:

```text
C91A6F16E1641E63AC6513350B0B64563E6DFFAFEBAD3233E6737EB544DE3C84
```

PowerShell:

```powershell
Get-FileHash .\renterd.exe -Algorithm SHA256
```

Version check:

```powershell
.\renterd.exe version
```

Expected build identification:

```text
renterd v2.9.4-17-g240bd51e
Network mainnet
Commit: 240bd51e
```

---

# GitHub-generated source archives

GitHub automatically adds:

```text
Source code (zip)
Source code (tar.gz)
```

to every Release.

Those archives contain the contents of this public distribution repository.

They are generated automatically by GitHub and are **not** the Windows
Smart Autopilot binary package.

Use:

```text
renterd-smart-autopilot-v0.2.0-alpha.1-windows-amd64.zip
```

if you want the prebuilt Windows executable.

---

# Development status

This project is currently experimental alpha software.

The implementation is being developed incrementally while the actual renterd
mechanisms are studied.

The current emphasis is on:

```text
Host measurement
Transparent scoring
Historical observations
Contract state handling
User-visible diagnostics
Configurable behavior
```

Later development is intended to build an intelligent decision layer on top of
these measurements.

That layer may eventually manage:

```text
Host portfolio composition
Contract lifecycle decisions
Host replacement
Data migration
Data placement
Infrastructure diversity
```

The goal is not merely to create another `hostScore()` formula.

The goal is to build a transparent host-management layer that can reason about
different kinds of host quality and risk independently.

---

# Upstream project

Smart Autopilot is based on:

https://github.com/SiaFoundation/renterd

renterd is developed by the Sia Foundation and its contributors.

This repository is an independent experimental project and is not affiliated
with or endorsed by the Sia Foundation.

---

# License

This project is distributed under the MIT License, consistent with the upstream
renterd project.

See:

```text
LICENSE
```

for the full license text.
