# LP-governed invoice funding proposal

## Goal

`pool::fund_invoice` remains permissionless today, but the roadmap explicitly calls for LP-governed capital allocation as a later governance layer. This proposal defines a concrete, incremental design that keeps the emergency admin pause in place while reserving any governance control over funding eligibility for a future LP voting mechanism.

## Design summary

1. LPs stake a governance token or pool share position in a dedicated staking contract.
2. Staking power is aggregated into a voting snapshot for each invoice funding attempt.
3. A quorum threshold and voting window determine whether the `fund_invoice` eligibility check passes.
4. The current permissionless check remains in place as the baseline gate, and the governance vote acts as an additional eligibility layer rather than replacing the existing risk checks.
5. The admin pause circuit breaker stays emergency-only: it can suspend state-changing behavior, but it cannot approve or veto individual funding decisions.

## Proposed staking and voting mechanism

- Introduce an LP staking ledger keyed by `(staking_contract, lp_address)` with:
  - `staked_shares`
  - `last_snapshot_ledger`
  - `voting_power`
- Staking is limited to the pool's LP participation record, so voting power is derived from actual pool participation rather than arbitrary wallet count.
- A proposal is created when a caller requests a funding vote for an invoice ID, with fields:
  - `invoice_id`
  - `proposer`
  - `proposal_ledger`
  - `vote_deadline`
  - `yes_votes`
  - `no_votes`
  - `quorum_required`
- Voting uses a fixed snapshot of stake at proposal creation so a funding vote is not manipulable by a last-minute deposit or withdrawal.

## Quorum threshold and timing

- Quorum is measured as a percentage of the total active staking supply, not a raw share count.
- Suggested conservative default: `quorum_bps = 5000` (50% of active voting power).
- Voting window: `voting_window_ledger = 1440` (roughly one day on Stellar, assuming a 5s ledger time), or a configurable parameter stored at contract initialization.
- A funding proposal passes when:
  - `yes_votes + no_votes >= quorum`
  - `yes_votes > no_votes`
  - `proposal_ledger <= current_ledger <= vote_deadline`
- Otherwise, the proposal is rejected and funding remains blocked until a new proposal is created.

## Composition with the existing permissionless check

The governance layer composes with the current eligibility checks as follows:

1. `invoice` must be in `Listed` state.
2. `invoice` funding asset must match the pool's asset.
3. pool must have sufficient available liquidity.
4. funding must not exceed the utilization cap.
5. issuer and buyer must still be registry-verified at funding time.
6. LP governance vote must approve the invoice or, in the minimal future design, not be rejected by quorum or veto conditions.

This keeps the current risk model intact and adds governance only as an additional gate. The exact governance vote is not a substitute for safety checks already enforced on-chain.

## Proposed storage layout

The following `DataKey` additions are consistent with the existing schema conventions in `docs/STORAGE.md`:

```rust
#[contracttype]
pub enum DataKey {
    Admin,
    InvoiceContract,
    EscrowContract,
    FundingAsset,
    TotalShares,
    TotalDeposits,
    TotalFunded,
    TotalYieldDistributed,
    TotalLossRealised,
    ActiveInvoiceCount,
    LPShares(Address),
    LPDepositCount(Address),
    LPYieldEarned(Address),
    LPInitialDeposit(Address),
    FundedInvoice(BytesN<32>),
    MaxUtilizationBps,
    RegistryContract,
    ProtocolFeeBps,
    TreasuryAddress,
    MinInitialDeposit,
    Allowance(Address, Address),
    ShareName,
    ShareSymbol,
    ShareDecimals,
    // governance additions
    StakingContract,
    GovernanceQuorumBps,
    GovernanceVotingWindow,
    FundingVote(BytesN<32>),
    StakingBalance(Address),
}
```

This keeps contract configuration in instance storage while the vote and staking records remain persistent and per-address or per-invoice, matching the established pattern for LP balances and funded invoice entries.

## Interaction with the admin pause circuit breaker

The emergency admin pause remains a separate control plane:

- `pause()` / `unpause()` must be admin-authenticated.
- When paused, all state-mutating public contract entry points should reject without changing storage.
- The pause mechanism is for emergency shutdown and risk control only.
- Governance voting is not granted power to unpause or approve funding; only the admin retains emergency pause authority.

This matches the roadmap intent: governance can shape funding policy only via explicit vote-gated checks, while the admin keeps a narrow emergency brake.

## Follow-up implementation path

This document intentionally stops at the proposal level. A future implementation issue would add the concrete governance state, vote tallying, snapshotting, and `fund_invoice` gate on top of this design.
