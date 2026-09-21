# renterd Smart Autopilot
English | [Українська](README.uk.md) | [Русский](README.ru.md)

Experimental Smart Autopilot build for [Sia renterd](https://github.com/SiaFoundation/renterd).

> [!WARNING]
> This is an unofficial experimental build and is **not an official Sia Foundation release**.
>
> The project is currently in **alpha** and should be used with appropriate backups and caution.

## Overview

**renterd Smart Autopilot** is an experimental extension of the Sia `renterd` host-management logic.

The long-term goal is to move away from relying on a single aggregate host score and toward a more transparent host profile that can evaluate hosts across several independent dimensions:

- Reliability
- Performance
- Resources
- Economics
- Risk / collateral
- Technical compatibility
- Network diversity

These groups are intentionally not required to collapse immediately into one "magic number".

For example, a host may be highly reliable and inexpensive while having only average performance. Smart Autopilot is intended to preserve that distinction instead of hiding it inside a single score.

The current alpha release contains the first technical foundation for this architecture.

---

## Current release

### v0.1.0-alpha.1

Platform:

- Windows amd64

Based on:

- Sia renterd `v2.9.4`
- renterd binary version: `v2.9.4-11-g4a69ef90`
- renterd commit: `4a69ef90`
- web commit: `99adb87b`

Download:

https://github.com/MainQuestion/renterd-smart-autopilot/releases/latest

---

## Implemented features

### Configurable host scoring

Several renterd host-scoring parameters that were previously hard-coded can now be configured.

Current configurable parameters include:

- Storage free-space weight
- Storage penalty exponent
- Allocation buffer factor
- Interaction penalty exponent
- Interaction trust prior
- Individual legacy scoring factors can be disabled

Default values are intended to preserve the original renterd behavior.

A dedicated **Scoring** page is available in the renterd web interface.

---

### Performance Score

A separate host **Performance Score** has been added.

It is intentionally kept separate from the original renterd host score.

The current implementation uses historical performance information including:

- Upload performance
- Download performance
- Historical transfer speed
- P95 operation performance
- Amount of collected performance data

Performance is evaluated using a rolling historical window.

This is the first step toward treating performance as an independent host characteristic rather than mixing it directly into the legacy host score.

---

### Raw upload and download speed

The Hosts table displays measured host throughput as:

- Upload Mbps
- Download Mbps

Raw throughput remains visible separately from the normalized Performance Score.

---

### Host history

Additional historical information is stored for future Smart Autopilot decisions:

- Speed history
- Price history
- Scan history

This creates the basis for evaluating how a host behaves over time instead of relying only on its latest observed state.

---

### Paused contracts

A new **Paused** contract state has been introduced.

Paused contracts provide an intermediate state between a healthy active contract and a contract that must be treated as bad or unusable.

The purpose is to allow more controlled contract lifecycle management instead of forcing every temporary problem into a binary good/bad decision.

---

### Web UI improvements

The renterd Hosts interface has been extended with additional performance information and metric grouping.

SPA navigation was also corrected to eliminate React hook-order errors that could occur when switching between renterd pages without performing a full browser refresh.

---

## Planned Smart Autopilot architecture

Future development is intended to evaluate hosts using separate groups rather than only the legacy aggregate score.

### 1. Reliability

How consistently the host operates and fulfills its obligations.

Possible signals include:

- Uptime
- Successful and failed interactions
- Consecutive failures
- Host age and historical stability

### 2. Performance

How quickly the host actually services renter requests.

Possible signals include:

- Response time
- Upload throughput
- Download throughput
- P90/P95 operation latency
- Request queue behavior

### 3. Resources

Whether the host has enough resources for the intended data placement.

Possible signals include:

- Free storage
- Used storage
- Storage available for new contracts
- Contract/resource limitations

### 4. Economics

The expected economic cost of using the host.

Possible signals include:

- Storage price
- Upload price
- Download price
- RPC costs
- Contract price
- Expected total storage cost over the contract period

### 5. Risk / collateral

How much financial and operational risk is associated with the host.

Possible signals include:

- Collateral
- Collateral relative to expected stored data
- Contract renewal history
- Data and financial exposure

### 6. Technical compatibility

Whether the host satisfies renterd protocol and system requirements.

Possible signals include:

- RHP protocol version
- Accepting contracts
- Announcement status
- Scan status
- Required feature support

### 7. Network diversity

Whether using the host increases shared infrastructure risk with other selected hosts.

Possible signals include:

- IP address
- Subnet
- ASN
- Geographic information when available
- Infrastructure overlap with other selected hosts

Network diversity differs from the other groups because it depends not only on the host itself, but also on the rest of the selected host portfolio.

---

## Design principles

Development follows several principles:

1. **Inspect renterd behavior before changing it.**
2. **Preserve existing renterd behavior unless the reason for changing it is understood.**
3. **Keep user configurability wherever practical.**
4. **Separate measured data from policy decisions.**
5. **Separate independent host characteristics instead of hiding everything inside one score.**
6. **Use historical data where a single observation is not sufficient.**
7. **Treat host selection as portfolio management, not only individual-host ranking.**

---

## Installation

This repository currently distributes experimental Windows binaries.

Download the latest release from:

https://github.com/MainQuestion/renterd-smart-autopilot/releases

Before replacing an existing renterd installation:

1. Stop renterd.
2. Back up the existing `renterd.exe`.
3. Back up important renterd application data and databases.
4. Extract the release archive.
5. Replace the existing executable with the provided `renterd.exe`.
6. Start renterd and check the log for migration or startup errors.

Existing configuration and database files should not be deleted when replacing only the executable.

---

## Binary verification

For release `v0.1.0-alpha.1`:

Archive:

`renterd-smart-autopilot-v0.1.0-alpha.1-windows-amd64.zip`

SHA256:

`0F077E70F084256EC6EECE82AB78A89D77C3CA5C7EEE19BE559460055E783F31`

The archive contains:

- `renterd.exe`
- `LICENSE`

---

## About the automatically generated source archives

GitHub automatically creates:

- `Source code (zip)`
- `Source code (tar.gz)`

for every release tag.

Those archives contain the contents of this **distribution repository**.

They should not be confused with the modified renterd application source tree.

---

## Upstream project

Smart Autopilot is based on:

[SiaFoundation/renterd](https://github.com/SiaFoundation/renterd)

The upstream renterd project is developed by the Sia Foundation.

This Smart Autopilot repository and its experimental builds are independent and unofficial.

---

## License

Based on Sia Foundation renterd and distributed under the MIT License.

See [LICENSE](LICENSE).

---

## Project status

**Experimental / Alpha**

Current releases are intended for testing, evaluation, and development.

The Smart Autopilot architecture is still evolving, and additional host-management mechanisms will be introduced incrementally as the underlying renterd behavior is studied and verified.
