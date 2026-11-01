# RentRoll — a native launch mode for Stonk.fun

**Designed by the CONDO team (@CONDOonSol). Public specification, November 2026.**

RentRoll adds two things to a Stonk.fun reward launch:

- **Roll a Lease**: every buy made in RentRoll mode goes into an on-chain vault under a lease of random length. Only the tenant knows when their lease ends.
- **Collect Rent**: the reward tax that Stonk.fun already distributes in the quote asset is paid to locked positions, weighted linearly by *locked amount × time held*.

A third feature, **Sublease**, lets a tenant sell a locked position to someone else without touching the pool.

---

## 1. Context: the Stonk.fun stack today

| Piece | Today |
|---|---|
| Bonding curve | Raydium LaunchLab (`LanMV9sAd7wArD4vJFi2qDdfnVhFxYSUg6eADduJ3uj`), pools attributed to Stonk.fun platform configs |
| Mints | Token-2022, 1B supply, 6 decimals; reward launches carry a TransferFee extension (100 or 300 bps); no transfer hook |
| Graduation | Raydium CPMM (`CPMMoo8L3F4NbTegBCKVNunggL7H1ZpdTHKxQB5qKP1C`) |
| Holder rewards | Transfer tax collected and paid to holders in the quote asset by Stonk.fun's distributor |

RentRoll uses all of this as-is. **No Raydium change is required.**

## 2. Native integration (what Stonk.fun would run)

1. **A "RentRoll" platform config** on LaunchLab, configured like the existing reward config (Token-2022 + transfer fee, quote asset of the creator's choice, graduation to CPMM).
2. **A lease vault program** (Anchor). It holds the positions and routes Roll a Lease buys by CPI:
   - pre-graduation: LaunchLab `buy_exact_in`, with the vault token account as `user_base_token`;
   - post-graduation: CPMM `swap_base_input`, with the vault token account as `output_token_account`.
   The locked amount is measured as `vault_balance_after − vault_balance_before`, which is net of the Token-2022 transfer fee.
3. **Default routing**: the Stonk.fun UI and API build "Roll a Lease" transactions by default for RentRoll launches.
4. **Randomness keeper**: Switchboard On-Demand commit/reveal; settlement is permissionless.
5. **Rent weighting** in the existing reward distributor (section 5).
6. **Unlock-fee rebate (optional)**: leaving the vault is a second token transfer and pays the transfer fee again. The withheld-fee authority can rebate it, or the tenant can sell directly from the vault into the pool (one transfer, same as any sell).

Buys that bypass the vault (aggregators, direct pool calls) remain possible and stay unlocked. They do not earn rent. RentRoll is therefore an incentive, not an enforcement mechanism.

**Open avenue, not part of this design:** forcing a lease on 100% of buys would need a transfer hook (or an equivalent lock-on-buy primitive). Today's Raydium LaunchLab/CPMM do not accept mints with an active transfer hook, so this would require a change on Raydium's side.

## 3. Lease math (Roll a Lease)

Parameters (per launch, public): `min_lease`, size tiers `(size_ppm_k, max_lease_k)`, `window_slots`, `window_weight_pct`, `global_max_lease`, and an optional `lease_fee_bps` (a percentage of the quote amount; there is no fixed per-buy fee).

1. **Size** of a buy, in parts per million of total supply: `own = amount × 10^6 / supply`.
2. **Pool-wide rolling window**: the program keeps one running volume counter per pool (not per wallet). It resets once `window_slots` have passed since the window opened. Each buy adds to it.
   `effective = max(own, window_volume × window_weight_pct / 100)`
3. **Range**: `max_lease = interpolate(effective, tiers)`, clamped to `[min_lease, global_max_lease]`.
4. **Draw**: with `u` uniform in `[0, 1)`, derived from the revealed randomness and the tenant's secret (section 6),
   `lease = min_lease + (max_lease − min_lease) × u²`.
   The draw is skewed toward short leases (median at 25% of the range).
5. **Timeout**: if randomness is not revealed within ~150 slots, the position takes `max_lease`.
6. **Safety net**: every position is withdrawable after `buy_time + max_lease`, with no randomness or secret needed.

**Splitting across wallets.** The main defence is the rent formula (section 5). Rent is strictly linear in *size × time*, so splitting one buy across N wallets earns no extra rent. And because leasing is opt-in, a buyer who doesn't want a lease can simply buy without RentRoll.

**The pool-wide window is a secondary sizing aid, with limits.** It makes buys that land in the same short window, such as a burst or a bundle, share one size, so later slices draw from a longer lease range. Recommended `window_slots` ≈ 150 (about one minute). It does **not** link wallets: buys spread over minutes or hours are each sized on their own.

## 4. Sublease (P2P market for locked positions)

- A tenant lists a locked position at a fixed price in the quote asset; a buyer takes it whole.
- The position (amount, remaining lease) transfers to the buyer; the tokens never leave the vault and the pool price is unaffected.
- The buyer's `held_since` resets to the purchase time, so time-held rent restarts for the new tenant.
- Listing and cancelling are permissionless for the owner; a small market fee is configurable.

## 5. Rent math (Collect Rent): strictly linear

For each distribution period with reward amount `R` (quote asset):

```
w_i = a_i × t_i
rent_i = R × w_i / Σ_j w_j
```

- `a_i`: tokens locked in position `i` (net of transfer fee)
- `t_i`: seconds position `i` was held within the period

There are no multipliers, tiers or exponentials. Twice the tokens for the same time earns twice the rent, and so does the same tokens for twice the time. Unlocked or free-floating tokens have `w = 0` in RentRoll mode.

Two equivalent implementations:

- **Off-chain (native, simplest)**: the distributor reads position accounts and pays owners directly; the vault itself is excluded as a holder.
- **On-chain**: if the distributor pays the vault as a regular holder, the vault program streams the received quote tokens to positions with a standard reward-per-weight accumulator (`acc += R × 10^12 / Σw`), using `w = a` integrated over time.

## 6. Hidden lease durations

- At buy, the tenant commits `h = SHA-256(secret)`.
- The secret comes from a signature of a fixed message by the tenant's wallet, so the same wallet can always recompute it.
- After reveal, the program stores the randomness `r` but not the lease. The lease is derived from `SHA-256("rentroll-lease" ‖ r ‖ secret)` with the skewed draw of section 3, so only the tenant knows when the position unlocks.
- Each position's range `[min_lease, max_lease]` is public, because it depends on the buy size. Its exact end is not.
- To unlock, the tenant reveals `secret`; the program checks `h` and recomputes the lease.
- The safety net still applies at `buy_time + max_lease`, with no secret needed.
- For a Sublease, the seller either reveals the secret at listing or hands it privately to the chosen buyer.

## 7. Reference implementation

A compiled SBF proof-of-concept of the vault wrapper (Anchor 0.31) exists: CPIs to LaunchLab `buy_exact_in` and CPMM `swap_base_input` generated from Raydium's public IDLs, lease settlement, accumulator-based rent, sell-from-vault and withdraw. It is not deployed, not audited, and has not been tested against live pools.

## 8. Open questions

1. Do LaunchLab and CPMM accept an output token account owned by a PDA that is not the payer? The public IDLs only constrain the mint. To be confirmed on a mainnet fork.
2. Do the distributor's current eligibility rules include or exclude program-owned token accounts?
3. Do Stonk.fun's indexers count CPI-routed trades like direct ones (volume, charts, fee ledger)?
4. Which per-launch parameters (tiers, window, global maximum) does Stonk.fun want to expose to creators?
5. How is the unlock-fee rebate funded and capped?
