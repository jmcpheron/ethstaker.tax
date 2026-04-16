# Post-Pectra Income Calculation

Status: review / design note. Not a spec — a shared snapshot of where we are, why
we changed what we changed, and what is still open.

## 1. The issue

ethstaker.tax computes per-validator staking income as:

```
income_on_day = (balance_eod - balance_eod_prev) + sum(withdrawals_during_day)
```

Pre-Pectra, validator economics were simple enough for several shortcuts:

- Effective balance was capped at 32 ETH.
- Any balance above 32 ETH was auto-swept within ~8 days as a "partial"
  withdrawal.
- A full exit produced one sweep of roughly `32 ETH + residual reward`.
- One validator index = one stream of rewards. Balances never moved between
  validators.
- All withdrawals originated on the consensus layer (Shapella sweep).

Pectra (live on mainnet, Feb 2026) invalidates each of those:

- **EIP-7251** raises `MAX_EFFECTIVE_BALANCE` from 32 ETH to 2048 ETH for 0x02
  *compounding* validators. A validator can now legitimately hold ~2048 ETH in
  effective balance with no sweep.
- **EIP-7002** introduces execution-layer-triggered partial withdrawals and
  full exits. The resulting transfer still appears in the block's withdrawal
  list, but it is initiated from `tx.origin` rather than by the sweep.
- **EIP-7251 consolidations** let a *source* validator's entire balance move
  into a *target* validator. There is no withdrawal. The target's balance
  jumps; the source's balance goes to zero (and the source is exited).
- A validator can migrate from 0x01 (non-compounding, auto-sweep >32 ETH) to
  0x02 (compounding up to 2048 ETH) by submitting an EL request.

The original income formula is still *mostly* right — principal and rewards
both flow through `balance` and `withdrawal`, and the sum over a day collapses
correctly. But it breaks silently for consolidations and for any code path
that hard-codes 32 ETH as "the validator size".

## 2. What we already fixed on this branch

Branch: `claude/peptra-income-calculation-review-JCuH4`.

### 2.1 `c92f368` — Improve partial withdrawal handling (#124)

Removed a heuristic in `src/api/api_v2/endpoints/rewards.py` that tried to
separate "full" from "partial" withdrawals by amount:

```python
# Before
if w.amount_gwei > 8 * Decimal(1e9):
    # Assume full withdrawal
    if w.amount_gwei < 32 * Decimal(1e9):
        raise HTTPException(500, "... leads to negative income")
    amount_withdrawn_this_day_wei += (w.amount_gwei % (32 * Decimal(1e9))) * Decimal(1e9)
else:
    amount_withdrawn_this_day_wei += w.amount_gwei * Decimal(1e9)

# After
amount_withdrawn_this_day_wei += w.amount_gwei * Decimal(1e9)
```

The pre-Pectra heuristic was already fragile and becomes outright wrong under
Pectra: a compounding validator can produce a 100 ETH partial withdrawal;
`100 % 32 = 4` would be reported as the reward. The simpler sum is correct
because the surrounding formula already handles principal:

> `amount_earned = (balance_eod − balance_eod_prev) + Σ withdrawals`

If 100 ETH of principal leaves the validator, `balance_eod` drops by 100 ETH
and the withdrawal row adds it back. What remains is the day's accrued
rewards. This is true for 0x01 and 0x02 credentials alike, *provided the
balance snapshots are real consensus-layer balances* — they are, see
`src/indexer/balances.py:124` and `src/providers/beacon_node.py:286`
(`/eth/v1/beacon/states/{state_id}/validator_balances`).

Caveat: the assumption breaks on consolidation days (section 3.1).

### 2.2 `ce0e6dc` — Add RocketMinipoolManager v6 (#125)

Appended the v6 manager address
(`0xe54B8C641fd96dE5D6747f47C19964c6b824D62C`, deployed Feb 9 2026) to
`_MINIPOOL_MANAGER_ADDRESSES` in `src/providers/rocket_pool.py:35`. No ABI
changes; we rely on v6 exposing the same getters we already call
(`getNodeDepositBalance`, `getUserDepositBalance`, `getNodeFee`).

### 2.3 `98573de` — RP withdrawal test fixtures for NO share

Added two partial-withdrawal fixtures in `tests/api/conftest.py` and matching
assertions in `tests/api/api_v2/endpoints/test_rewards_v2.py`, pinning the
node-operator (NO) share formula:

```
NO_share = amount × (bond/32 + (1 − bond/32) × fee)
```

| Validator | Minipool | Bond | Fee | Withdrawal | Expected NO share |
|-----------|----------|------|-----|------------|-------------------|
| 461308    | LEB16    | 16 ETH | 15 % | 15,000,000 gwei | 8.625 × 10¹⁵ wei |
| 584908    | LEB8     |  8 ETH | 14 % | 12,000,000 gwei | 4.26  × 10¹⁵ wei |

Reconciling 461308: `1e9 × 15e6 × (16/32 + (16/32) × 0.15) = 1e9 × 15e6 × 0.575 = 8.625e15` ✓
Reconciling 584908: `1e9 × 12e6 × (8/32 + (24/32) × 0.14) = 1e9 × 12e6 × 0.355 = 4.26e15` ✓

Coverage is limited to small partial withdrawals (the formula's happy path).
No fixture yet for a full exit on a 0x02 minipool or a slashed minipool.

## 3. Open correctness gaps

### 3.1 Consolidations are not modeled

The CL income formula silently misattributes income on the day of a
consolidation: the *target* validator's balance jumps by the *source*'s entire
balance, with no corresponding `Withdrawal` row. Without additional signal we
will report that jump as income on the target and report the source's balance
drop as negative income. They cancel in aggregate across the whole node
operator, but per-validator reports will be wrong — and tax reports are
per-validator.

Mitigation requires new data; see section 5.

### 3.2 0x01 → 0x02 migration is not modeled

The migration itself is not a value event — balances don't change — but any
code that still assumes "validator balance is roughly 32 ETH" (for display,
sanity checks, or future heuristics) will be confused by a validator whose
balance grows to 2048 ETH over time. The CL income formula is unaffected.

### 3.3 RP v6 ABI is assumed, not verified

`ce0e6dc` only adds the manager address. If any ABI changed between v5 and
v6, our on-chain reads will fail at runtime rather than at import. We should
add a smoke test that resolves a known v6-managed minipool and reads
`getNodeDepositBalance`.

### 3.4 Slashing path still raises 500

`src/api/api_v2/endpoints/rewards.py` still returns HTTP 500 when a full
withdrawal amount is less than user-supplied capital
("… leads to negative income"). Not Pectra-specific, but it surfaces more
frequently as mass exits increase. Should degrade gracefully instead of
failing the whole report.

### 3.5 EL-triggered exits are not distinguished

EIP-7002 withdrawals flow through the same `Withdrawal` rows. We don't
capture whether the user initiated the exit via EL vs. the sweep. Not a
correctness issue for the income number; may matter later for the tax-event
classification UI.

### 3.6 Test coverage gaps

- No fixture for a 2048-ETH-effective 0x02 validator.
- No fixture for a large (>8 ETH) partial withdrawal on an RP minipool.
- No fixture for consolidation days.
- No fixture for a slashed full withdrawal — today it raises 500.

## 4. Current data model

Everything relevant lives in `src/db/tables.py`. Full schema history is in
`alembic/versions/`.

| Table | Key columns | Purpose | Pectra readiness |
|-------|-------------|---------|------------------|
| `balance` | `(slot, validator_index)`, `balance` | End-of-day + activation-slot validator balance snapshots (gwei, stored as `Float(asdecimal=True)`) | Stores only the live balance, not the effective balance. Fine for the income formula; insufficient for reporting "rewards-only" to 0x02 users. |
| `withdrawal` | `id`, `slot`, `validator_index`, `amount_gwei`, `withdrawal_address_id` | Every beacon-chain withdrawal from Shapella onward (`src/indexer/withdrawals.py`, starts at slot 6,209,536) | Captures amount and destination. Does not capture credentials type or EL-trigger origin. |
| `withdrawal_address` | `id`, `address` | Normalized 0x-addresses for withdrawals | Fine. |
| `validator` | `validator_index`, `pubkey` | Minimal validator metadata | No credentials info; no consolidation awareness. |
| `block_reward` | `slot`, `proposer_index`, `fee_recipient`, `priority_fees_wei`, `mev_reward_value_wei`, `mev`, `reward_processed_ok` | EL-side reward attribution per proposed block | Unaffected by Pectra. |
| `price` | `(token, currency, timestamp)`, `value` | Fiat conversion | Unaffected. |
| `rocket_pool_minipool` | `minipool_address`, `validator_pubkey`, `initial_bond_value`, `initial_fee_value`, `node_address` | RP minipool registry | Assumes v5/v6 ABI parity. |
| `rocket_pool_bond_reduction` | `(minipool_address, new_bond_amount)`, `timestamp`, `new_fee` | Bond-reduction timeline | Works as-is. |
| `rocket_pool_node` / `rocket_pool_reward_period` / `rocket_pool_reward` | — | RP node + periodic rewards | Unaffected. |

The income computation itself lives in `src/api/api_v2/endpoints/rewards.py`:
- Lines 527–553: CL income per day (the loop `c92f368` simplified).
- Lines 27–162: RP node-operator share calculation for both partial and full
  withdrawals.

## 5. Proposed schema changes

Prioritized by value-per-effort. Each is a standalone Alembic migration.

### Recommended (ship as a pair)

**5.1 `balance.effective_balance` — new nullable column**

- Type: `Numeric(precision=20)` (gwei).
- Source: `/eth/v1/beacon/states/{slot}/validators` at the same slots we
  already fetch balances for.
- Backfill: expensive (one API call per EOD slot), but we can index going
  forward immediately and backfill lazily.
- Unlocks: reporting "rewards-only" as `balance − effective_balance` for 0x02
  validators, and sanity-checking the income loop on days with large
  withdrawals.

**5.2 `validator.withdrawal_credentials_type` — new nullable byte**

- Type: `SmallInteger` (0x00 / 0x01 / 0x02).
- Source: the first byte of `withdrawal_credentials` from
  `/eth/v1/beacon/states/head/validators/{id}`.
- Backfill: cheap — one read per validator, values only change on explicit
  migration events.
- Unlocks: branching on credential type instead of amount heuristics, and a
  place to hang future "migration event" tracking.

### Next, blocked on indexer work

**5.3 `consolidation_event` — new table**

```
consolidation_event
-------------------
slot                    INTEGER   not null  index
source_validator_index  INTEGER   not null  index
target_validator_index  INTEGER   not null  index
amount_gwei             Numeric(18)
```

- Source: processed consolidations in block bodies (and
  `/eth/v1/beacon/pool/consolidations` for the pending queue).
- Required for the CL income formula to stop over-reporting the target and
  under-reporting the source on consolidation days.
- Needs new indexer code in `src/indexer/` (analogous to
  `withdrawals.py`).

### Deferred

**5.4 `withdrawal_request` — EIP-7002 EL-triggered requests**

Low priority: the resulting balance movement is already captured by
`withdrawal`. Only needed if we want to classify exits by trigger in the UI.

**5.5 `withdrawal.amount_gwei` precision**

Already `Numeric(precision=18)` (1e18 gwei headroom). No change needed.

## 6. Verification

Read-only checks a reviewer can run to confirm this doc matches the code:

```bash
# 1. The income-loop simplification is exactly what c92f368 shipped.
git show c92f368 -- src/api/api_v2/endpoints/rewards.py

# 2. Balances come from validator_balances (live), not effective_balance.
grep -n "balances_for_slot\|validator_balances" src/providers/beacon_node.py

# 3. The RP NO-share test fixtures reconcile to the formula in section 2.3.
docker compose -f docker-compose.test.yml up --abort-on-container-exit

# 4. Remaining hard-coded 32-ETH assumptions (expect: RP share math only).
grep -rn "32 \* Decimal(1e9)\|32 ETH" src/

# 5. RP manager address list contains v5 and v6.
grep -n "_MINIPOOL_MANAGER_ADDRESSES" -A 20 src/providers/rocket_pool.py
```

## 7. Out of scope for this doc

- Writing the Alembic migrations for 5.1 / 5.2 / 5.3.
- Implementing the consolidation indexer.
- Any change to the income math in `rewards.py`. This doc describes the gaps;
  the fix is a follow-up.
