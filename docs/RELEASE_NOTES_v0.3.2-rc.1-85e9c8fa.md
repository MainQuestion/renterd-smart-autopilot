# renterd v0.3.2-rc.1-85e9c8fa — Release Notes

## Smart Autopilot fix: pruning contracts paused for gouging

### Problem

Contracts placed in a Smart Autopilot gouging pause could retain physical data on hosts even after renterd no longer referenced that data. The database references had been removed, but pruning did not run. At the time of investigation, this affected approximately **8.1 GB across four contracts** (IDs 38, 71, 68, and 69).

There were two blockers:

- Gouging-paused contracts were not included among the contracts eligible for pruning.
- The general gouging check blocked pruning when storage, upload, or download prices exceeded their limits, even though pruning needs to enforce the sector-freeing cost limit and general contract constraints.

### What changed

- In Smart Active mode, contracts with Smart state `paused` and reason `gouging` are now eligible for pruning, including contracts in the legacy lifecycle.
- For these contracts, pruning is blocked by `PruneErr` and `GougingErr`. Storage/upload/download price limits alone no longer block pruning.
- All other contracts retain the existing gouging check.
- Ordinary offline `Paused` contracts, draining contracts, and other states are not added to the eligible set.
- Original and Smart Shadow modes remain unchanged.
- Tests cover the gouging decision and eligible-contract selection. Pruning a gouging-paused contract also emits an Info log with the contract ID, host, and size.

### Result

Smart Autopilot can reclaim data that is no longer referenced on gouging-paused contracts while retaining the checks that directly govern pruning cost and contract validity.

This change applies only in **Smart Active** mode.
