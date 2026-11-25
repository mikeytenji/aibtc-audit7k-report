# Pre-deploy audit — Juice savings + trading safes, auth-helpers-v3, Jing router v6-1

**Bounty:** AIBTC "Audit 7k" (`mwvrpx3a633f201e8caa`)
**Scope reviewed (new code only, against the stated audited bases):**

| Contract | Repo @ commit | Base |
|---|---|---|
| `juice-savings.clar` | Rapha-btc/pillar-wallets-xyz @ `321f103` | `juice-autostake-safe.clar` |
| `juice-trading.clar` | Rapha-btc/pillar-wallets-xyz @ `321f103` | `juice-savings` + `fakfun-wallet-v19.clar` wiring |
| `juice-safe-auth-helpers-v3.clar` | Rapha-btc/pillar-wallets-xyz @ `321f103` | helpers-v2 |
| `swap-router-sbtc-stx-jing-v6-1.clar` | Rapha-btc/jing-contracts-v3 @ `d2f6cad` | `swap-router-sbtc-stx-jing-v6.clar` (live) |

**Summary:** 2 Low findings, both in the Juice safes' new code. The headline redesigns are sound: the 2-key `execute-pending-*-now` logic cannot be bypassed, the pool is properly bound into the passkey signature, the v19 functions are wired in byte-identically, and the v6-1 Zest flip is a faithful, fully-guarded mirror of v6. Details of everything verified clean are listed per contract so the poster can see the coverage.

---

## Finding 1 — Low: `old-signer` is caller-supplied but not covered by the passkey signature (`update-stake-stx-juice`, `unstake`)

**Contract:** `juice-savings.clar`
**Functions:** `update-stake-stx-juice` (lines 1452–1521), `unstake` (lines 1523–1568)
**Builder:** `juice-safe-auth-helpers-v3.clar`, `build-update-stake-stx-juice-pool-hash` (lines 42–62); `juice-safe-auth-helpers-v1.clar`, `build-unstake-stx-juice-hash`

The v3 update-stake hash covers `auth-id`, `amount-increase`, `cycles-to-extend`, the **new** `pool`, and `signer-calldata` — but not `old-signer` (line 1454), which is passed to pox-5's `stake-update`. The gap is wider in `unstake`: its hash covers **only `auth-id`**, and the function passes the caller-supplied `old-signer` straight into `(contract-call? POX5 unstake old-signer)` with no `is-allowed-pool` check and no comparison against the safe's current position (`get-staker-info`). In the audited base both signer values were hard-coded constants (`JUICE-SIGNER`), so this unsigned-parameter exposure is introduced by this diff.

**Scenario.** `sig-auth` is relayable — `is-authorized (some …)` never checks `tx-sender`. The owner passkey-signs a 0/0 pool switch intending old = `JUICE-SIGNER` → new = `FASTPOOL-SIGNER`. Any relayer can submit the same signature with a different `old-signer` trait implementation. What executes is then decided solely by pox-5's `stake-update`/`unstake`: if pox-5 does not itself verify that the passed old manager is the manager of the stacker's current position, the operation runs against a manager pair the owner never signed for. The same applies to a generic signed "unstake" paired with an arbitrary `old-signer`.

**Why only Low.** The contract's own invariant treats the signer as state-determined — the new `restake` asserts `(is-eq (contract-of signer) (get signer info))` (line 1637) — so a wrong `old-signer` should revert inside pox-5 / the wrong manager, making today's realistic impact griefing and failed transactions. It becomes a fund-accounting bug only if pox-5 trusts the parameter.

**Fix (cheap).** Include `old-signer` in both hashes (unstake needs a new builder version), or assert `(contract-of old-signer)` equals the current position's signer before calling pox-5, exactly as `restake` already does.

## Finding 2 — Low: `onboard`'s new optional recovery silently defaults to the zero address, permanently disabling inactivity recovery

**Contracts:** `juice-savings.clar` → `onboard` (line 1672), `juice-trading.clar` → `onboard` (line 2516)

In the audited base, `recovery` was a required `principal`. Now `(default-to 'SP000000000000000000002Q6VF78 recovery-wallet)` applies when `none` is passed, and the guards only reject `recovery == current-contract` and `recovery == new-owner` — the zero address passes both, and `recovery-address` is set from it. `recover-inactive-wallet` requires `tx-sender == recovery-address`; `SP000…` can never originate a transaction, so inactivity recovery is dead for such a wallet.

**Scenario.** The deployer onboards a wallet passing `none` — an un-updated onboarding caller that previously *had* to supply a recovery principal can now omit it without any error. If the owner later loses both the admin key and the passkey, after `INACTIVITY-PERIOD` nobody can ever recover the wallet and its funds are stranded. The funds are not wrongly claimable (SP000 cannot sign, and no one else satisfies the check), and the owner can repair the recovery address while either key lives (`propose-recovery` + `confirm-recovery`), which is why this is Low — a silent deployer footgun rather than a theft path. (`onboard` being deployer-only is by design and not the issue; the issue is the silent dead default.)

**Fix.** Reject `SP000000000000000000002Q6VF78` as the recovery address in `onboard`, or keep `recovery-wallet` required.

---

## Verified clean

### juice-savings.clar

- **2-key rule in all 3 `execute-pending-*-now`** (stx 698–755, sbtc 853–911, withdrawal 1023–1093): the branch is driven by the stored `passkey-created` flag, and that flag cannot be forged — all three creation sites pass `(is-some sig-auth)`; creating with `sig-auth` requires a valid WebAuthn signature from a registered admin pubkey (`consume-signature` → `verify-signature` → `is-admin-pubkey`), creating without it requires `tx-sender ∈ admins`. Passkey-created ops confirm only via an admin `tx-sender`; admin-created ops require a passkey signature (`sig-auth = none` aborts). No same-key create+confirm path exists; the token-lock assert still applies before the branch.
- **Replay:** `consume-signature` (1267–1291) burns both the message-hash and `sha256(authenticator-data ‖ signature)`. The execute-now hash binds `{auth-id, op-id}`; amount/recipient are immutable map fields and the memo is checked against the stored payload.
- **`stake-stx-juice`** (1387–1450): the v3 hash covers amount, the pool (`(contract-of signer)`), and `signer-calldata`; `is-allowed-pool` asserted; `amount > 0` asserted.
- **0/0 rule in `update-stake-stx-juice`** (1498–1502): 0 increase + 0 cycles passes only when switching pools, and the new pool must be allow-listed.
- **`restake`** (1621–1662): keeper-only, and the new assert pins the signer to the position's current signer — the keeper cannot switch pools or move funds out.
- **Pool allow-list** (53–59): exactly the 3 vetted signer-managers, enforced on the pool that receives stake.
- **Error renumbering:** consistent, no stale codes; new `err-invalid-pool (u4023)` wired to its asserts.
- **Admin-map writers:** only `onboard`, wallet-transfer, and recovery flows write `admins`/`pubkey-to-admin` — no new path mints an admin.

### juice-trading.clar + juice-safe-auth-helpers-v3.clar

- **Signed-field matrix (helpers-v3):** domain is sound for all builders (SIP-018 prefix, per-operation topic, `chain-id`, `wallet: contract-caller`). `build-stake-stx-juice-pool-hash`, `build-place-ladder-hash`, `build-withdraw-ladder-hash`, `build-claim-ladder-hash` sign every outcome-changing field (pair, side, totals, every rung principal + amount). In `build-jing-swap-hash` the unsigned fields (`jing-amount`, `fallback`, `amm-amounts`, `amm-mins`, `zest-route`, `update`) are bounded by the signed ones — see next point. The only unsigned principal is `old-signer` (Finding 1).
- **`jing-swap` allowances** (2088–2228): per action, the `as-contract?` allowance is exactly the sold asset at exactly the signed `amount`; the router principal is a hardcoded constant per action; the bought asset returns to the safe. Verified against the v6-1 router: it asserts `amount == jing-amount + dlmm + xyk + velar + zest`, so the unsigned split cannot exceed the signed amount, and it ends with `(>= out min-stx-out)` on the caller's actual balance gain — a hostile route/split can only revert or still deliver ≥ the signed `min-out`.
- **Ladders:** `place-ladder` dispatchers are hardcoded constants per pair and the allowance is the signed `total` only; `claim-ladder` grants **no** asset allowances (nothing can leave the safe); `withdraw-ladder`'s `with-all-assets-unsafe` is intentional per the brief, and its amounts/rungs/side are all signed regardless.
- **v19 wiring:** all 14 copied forms are byte-identical to `fakfun-wallet-v19.clar`; the host auth core (`is-authorized`, `is-admin-calling`, `pay-gas-accounted`, `consume-signature`) is unchanged; `verify-signature` differs only in the RP-ID hash — trading correctly accepts only `juiceofbtc.com`. No allowance-map or auth-principal drift.
- All 43 functions shared with juice-savings are byte-identical except the expected mechanical diffs (`confirm-transfer-wallet`, `recover-inactive-wallet`, `onboard`: keeper removal, `register-wallet` self-reference).

### swap-router-sbtc-stx-jing-v6-1.clar

- **Flip complete:** exactly 4 Zest swap calls (`leg-zest` 299/311, `zest-swap` 1440/1451), all passing `SBTC ZEST_WSTX`; sell-sBTC = `swap-x-for-y` with max-price/fee-rate-x/ticks up, sell-STX = `swap-y-for-x` with min-price/fee-rate-y/ticks down. No `ZEST_WSTX SBTC` or stale v6 references remain; dx/dy result mapping flipped consistently, and `leg-zest`'s `scale-min` consumes the correctly-mapped in/out.
- **Bounds correct, verified numerically:** sell-sBTC `max-price = floor(floor(1e28/limit) × (1e8−fee)/1e8)`; sell-STX `min-price = ceil(ceil(1e28/limit) × 1e8/(1e8−fee))`. Worked in exact integer arithmetic over limit ∈ {1, 3e12, 7e12, 1e13, 1e21} × fee ∈ {0, 1e6, 2.5e6}: both bounds equal or tighten the true bound — a leg can underfill but can never execute past the caller's limit. Both branches double-round toward a stricter bound, same policy as v6.
- **Edge cases guarded:** `zest-stage-pool` now sanity-checks `limit > 0 ∧ mid > 0 ∧ limit ≤ ZEST_MAX_LIMIT ∧ mid ≤ ZEST_MAX_LIMIT` before calling `zest-bound` (the `> 0` checks are new and are exactly what the inversion requires); division by `(ZEST_FEE_ONE − fee)` guarded by `(< fee ZEST_FEE_ONE)`; worst-case intermediate `1e28 × 1e8 = 1e36 < 2^128`. Degenerate bounds collapse to `ZEST_NONE` (no swap) rather than reverting the outer swap.
- **Guards intact:** `ZEST_MAX_LIMIT` retained; fee side and tick-walk direction flipped correctly in `zest-stage-pool`/`zest-ticks`; header comments match the code.

## Informational note (inherited, out of scope)

The split path (`leg-zest`) takes `price` and `ticks` from the front-end route unchecked on-chain, so the router cannot validate the FE's price orientation — identical to v6's trust model, and the poster declared FE route builders out of scope. Noted only for completeness; the v6-1 header documents the correct new convention for the FE.

---

*Method: full diffs against the stated audited bases, full reads of every touched function and of the auth core (`is-authorized`, `consume-signature`, `verify-signature`), byte-level comparison of the copied v19 forms, and exact-integer verification of the v6-1 Zest bound math. Findings 1–2 were re-verified line-by-line in the pinned sources above.*
