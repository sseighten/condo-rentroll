# RentRoll — a launch mode for Stonk.fun reward coins

**Designed by the CONDO team (@CONDOonSol). Public specification, November 2026.**

RentRoll is a token-level rule for Stonk.fun reward coins:

- **Every incoming transfer is locked.** Any buy, from any route, and any wallet-to-wallet send creates a lock on the amount received. Locked tokens cannot be sold or sent until the lock ends.
- **Each lock length is random and secret.** It is drawn from public randomness and a secret committed in advance by the holder, so only the holder can compute when it ends. Its possible range is public; its exact end is not.
- **Bigger buys get wider ranges.** The maximum lock scales with the buy's share of supply, measured against a pool-wide rolling window.
- **The reward tax follows time held.** The reward Stonk.fun already distributes in the quote asset is split by *balance × time held*.
- **Locks can be transferred.** A holder can sell a locked position to another wallet, which inherits the remaining lock.

The rule is enforced by a Token-2022 transfer hook, so it applies on every route: the Stonk.fun UI, an aggregator, a direct pool swap, or a plain transfer.

**Status:** design only. RentRoll needs transfer-hook support from Raydium LaunchLab and Raydium CPMM, which does not exist today (section 2). No program is deployed.

---

## 1. Prior art

Per-buy locks enforced through transfer hooks already exist on Solana (for example Lockd, Hooked and Hooker in 2026), as do fixed per-buy locks such as $HUNDRED, and randomized vesting was proposed by ChainSafe in 2024. To our knowledge, RentRoll's combination is new: a random lock on every incoming transfer, scaled by size, with an end known only to the holder, plus a time-weighted reward split and transferable locked positions.

## 2. What it needs from Raydium and aggregators

Stonk.fun launches on Raydium LaunchLab and graduates to Raydium CPMM. Today:

- LaunchLab's Token-2022 path rejects TransferHook on the base mint.
- CPMM's mint allow-list rejects active transfer hooks, and its transfer helpers do not forward the extra accounts a hook needs.

RentRoll requires:

1. **CPMM**: support for admin-enshrined hook programs, with hook extra accounts forwarded in swaps, deposits and withdrawals. This matches the design proposed in raydium-cp-swap PR #81 (a `support_hook` PDA checked at pool creation, extras passed through `remaining_accounts`).
2. **LaunchLab**: a TransferHook option on `initialize_with_token_2022`, limited to enshrined hook programs, with extras forwarded in buy, sell and migration.
3. **Graduation**: automatic `support_mint` and `support_hook` registration for RentRoll mints when they migrate to CPMM.
4. **Deterministic, documented vault addresses** for LaunchLab and CPMM pools, so the hook can exempt exact accounts (section 5).
5. **Aggregators**: Jupiter routing for enshrined-hook mints. Until then, RentRoll tokens trade on Stonk.fun and Raydium only.

## 3. Mint

- Token-2022, 1B supply, 6 decimals.
- Extensions: TransferFeeConfig (the existing reward tax), TransferHook (the RentRoll program), MetadataPointer and TokenMetadata.
- Mint authority and freeze authority set to none.
- The hook program is immutable after audit, or held by a multisig behind a public timelock. Per-launch parameters are frozen at creation.

## 4. Accounts

| Account | Seeds | Purpose |
|---|---|---|
| `Config` | `["config", mint]` | Lock parameters, exempt vault accounts, rolling-window counter |
| `LockPage` | `["page", mint, owner[0]]` | Lock entries for every wallet whose owner pubkey starts with that byte (256 pages per mint) |
| `SecretRoot` | `["secret", mint, owner]` | Optional. The holder's Merkle root of per-lock commitments |
| `Listing` | `["listing", mint, owner, entry]` | Optional. A locked position offered for sale |

A transfer hook has no payer and cannot create accounts. `LockPage` accounts are therefore created and grown by a permissionless instruction that any payer can call. The Stonk.fun buy flow prepends it when a page is missing or near capacity, and a keeper keeps headroom. A transfer whose page is missing or full fails, so no transfer can skip its lock. Expired entries are evicted to reuse space.

The hook resolves accounts through its `ExtraAccountMetaList`: `Config`, the source owner's page and the destination owner's page (seeded from the first byte of each token account's `owner` field), and the destination owner's `SecretRoot`.

## 5. The hook

Token-2022 calls the hook after balances are updated. The hook first checks that it is called by Token-2022 during a real transfer, then applies two rules.

**Outgoing (source is not exempt):** the source account's balance after the transfer must be at least the sum of its active locks. A lock is active until it is unlocked (section 7) or until `start + max_lock`, whichever comes first.

**Incoming (destination is not exempt):** create a lock entry for the **net** amount received (the amount minus the transfer fee):

```
entry = { amount_net, start_slot, start_time, min_lock, max_lock,
          secret_index (or none), status: Pending }
```

**Exempt accounts:** the exact vault token accounts of the official pool, never a pool authority. Every CPMM pool shares one authority PDA, so exempting the authority would open any unofficial pool. The LaunchLab vault is recorded at creation. The CPMM vault is recorded once at graduation by a permissionless instruction that verifies the pool created by the migration. Stonk.fun's fee-distribution account is also exempt.

**Unofficial pools cannot bypass the rule:** tokens deposited into any other pool's vault are locked like any wallet's, and every buyer that pool pays receives a new lock.

**Entry limit:** each wallet holds at most 8 active entries. When all 8 are in use, a new amount goes into an overflow entry that unlocks at the latest `start + max_lock` of everything merged into it. Overflow never shortens a lock; it only removes the randomness for that amount.

## 6. Lock length

Parameters per launch (public, frozen at creation): `min_lock`, size tiers `(share_ppm_k, max_lock_k)`, `global_max_lock` (at most 21 days), `window_slots`, `window_weight_pct`, `settle_delay` (slots).

1. **Size**: `own = amount_net × 10^6 / supply`.
2. **Pool-wide window**: one volume counter per mint, reset when `window_slots` have passed since the window opened. `effective = max(own, window_volume × window_weight_pct / 100)`. It catches bursts and bundles landing in the same short window. It does not link wallets.
3. **Range**: `max_lock = interpolate(effective, tiers)`, clamped to `[min_lock, global_max_lock]`.
4. **Randomness**, without an oracle: `r = SlotHash(start_slot + settle_delay)` (or of the first non-skipped slot after it), read from the SlotHashes sysvar by a permissionless `settle` instruction. If nobody settles before that slot hash leaves the sysvar (about 512 slots), the entry takes `max_lock`. Holding back a settlement can only lengthen a lock.
5. **Draw**: `u = H(r ‖ key) / 2^256`, then `lock = min_lock + (max_lock − min_lock) × u²`, skewed toward short locks (median at 25% of the range).
   - With a secret: `key = s_k`, the holder's per-lock secret (section 7).
   - Without one: `key = mint ‖ owner ‖ entry_id`, and `settle` stores the end time publicly.
6. **Until settled**, an entry counts as locked for its full `max_lock`.
7. **Safety net**: every entry is free at `start + max_lock`, with no randomness, secret or keeper needed.

A slot leader could bias one slot hash by skipping its slot. It cannot choose a lock length, and the bias only matters to a holder who is that leader.

## 7. Secret lock lengths

1. **Master secret**: the wallet signs a fixed, domain-separated message (`RentRoll lock secret for <mint>. Sign only on stonkfun.xyz.`). `master = SHA-256(signature)`. Ed25519 signatures are deterministic, so the same wallet can always recompute it. Wallets that cannot do this keep a random master secret in a local, exportable backup.
2. **Per-lock secrets**: `s_k = H(master ‖ k)` and commitments `c_k = H(s_k)` for `k = 0 … 1023`.
3. **Commitment**: the wallet registers `R = MerkleRoot(c_0 … c_1023)` in its `SecretRoot` **before** buying. Each new lock entry uses the next unused index `k`, tracked in `SecretRoot`. When the indexes run out, new locks are public until the wallet registers a new root.
4. **Why the holder cannot cheat**: `c_k` is fixed before the transfer and `r` comes from a later slot, so the holder cannot pick a secret that gives a short lock.
5. **Unlock**: the holder calls `unlock(entry, s_k, proof)`. The program checks `H(s_k)` against `R`, recomputes the length, checks that it has elapsed, and marks the entry free. Only that lock's length becomes public, and only once it is already over.
6. **No root registered**: the lock is public (section 6, step 5).

What others can see: each entry's amount, start and range. What they cannot see: when a secret lock ends, until its holder moves the tokens.

## 8. Reward split

Stonk.fun's distributor already pays the reward tax to holders in the quote asset. In RentRoll mode it weights each holder by **balance × time held** over the distribution period:

```
w_i = Σ balance_i(t) × Δt
reward_i = R × w_i / Σ_j w_j
```

Exempt vaults and program-owned accounts are excluded. The weighting is linear: no multipliers, tiers or exponentials. Splitting a buy across many wallets earns nothing extra.

## 9. Selling a locked position

1. The holder lists an entry at a price in the quote asset and approves the RentRoll program as delegate for that amount.
2. A buyer calls `buy_listing`. The program marks the entry in the seller's page for this buyer and amount, then transfers the tokens as delegate. The hook finds the mark and **moves the entry** (amount, start, range, randomness) to the buyer instead of creating a new lock.
3. For a secret lock, the seller either reveals `s_k` in the listing (the end becomes public) or sends it privately to a named buyer, who checks it against the seller's commitment.
4. Tokens never touch a pool, so the price does not move.

Native wallet "Send" buttons do not pass hook accounts. Transfers and listings go through the Stonk.fun token page.

## 10. What Stonk.fun runs

1. A "RentRoll" platform config: reward launch with the RentRoll hook.
2. UI flows: enable secret locks (sign, register root), buy (prepend page instruction when needed), unlock, send, list and buy positions.
3. A keeper: settles entries, keeps page headroom, records the CPMM vault at graduation.
4. Distributor weighting by balance × time held, with exclusions.

## 11. Open questions

1. Will Raydium enshrine an audited RentRoll hook in LaunchLab and CPMM (section 2)?
2. Does Stonk.fun's distributor currently pay program-owned accounts?
3. Compute cost of the hook on LaunchLab and CPMM swaps, within limits shared with Raydium's own accounts.
4. Exact deterministic seeds of LaunchLab and CPMM vault accounts.
5. Which wallets sign the secret message deterministically (Phantom, Solflare, Backpack, Ledger, MPC wallets).
6. Page sizing and rent per launch.
7. Whether withdrawing withheld transfer fees invokes the hook on Stonk.fun's current flow.
