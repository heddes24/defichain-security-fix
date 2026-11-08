# Critical: DFI inflation via ignored overflow in `SumAllTransfers`

**Severity:** Critical — consensus bug, creates money out of thin air
**Affected:** DeFiChain v4.2.5 (`bd09486bb0`), mainnet + testnet + regtest
**Component:** Account transfers (custom transaction types `'B'` and `'a'`)
**Impact:** Unlimited DFI creation — up to `INT64_MAX` (~92.2 billion DFI) per transaction
**Cost to attacker:** 8.00000001 DFI
**Status:** Reproduced locally against the unmodified v4.2.5 binary

---

## Summary

`SumAllTransfers()` sums the amounts of all transfer destinations, but ignores
the return value of `AddBalances()`. When the sum overflows `int64_t`,
`SafeAdd()` correctly reports `Err("overflow")` — but the error is discarded and
the sum silently stays at the **first** amount.

The source account is then debited by that truncated amount, while each
destination is credited with its **full** amount. The difference is newly
created DFI.

## Root cause

`src/dfi/balances.h:265`

```cpp
inline CBalances SumAllTransfers(const CAccounts &to) {
    CBalances sum;
    for (const auto &kv : to) {
        sum.AddBalances(kv.second.balances);   // <-- return value ignored
    }
    return sum;
}
```

`src/dfi/consensus/accounts.cpp:80` (`AccountToAccount`, type `'B'`)

```cpp
if (auto res = SubBalanceDelShares(obj.from, SumAllTransfers(obj.to)); !res) {
    return res;                    // debits the TRUNCATED sum
}
return AddBalancesSetShares(obj.to);   // credits the FULL amounts
```

`src/dfi/consensus/accounts.cpp:96` (`AnyAccountsToAccounts`, type `'a'`) has the
same defect: `sumFrom` and `sumTo` are both truncated, so the
`sumFrom != sumTo` check passes and the same inflation is possible.

The balance check itself (`CTokenAmount::Sub`, `src/amount.h:163`) is correct —
it is simply never presented with the real amount.

## Trigger

Two destinations with the **same token**, whose amounts sum to more than
`INT64_MAX`:

| | satoshi | DFI |
|---|---:|---:|
| destination A | 800,000,001 | 8.00000001 |
| destination B | 9,223,372,036,054,775,807 | 92,233,720,360.54775801 |
| **sum** | **9,223,372,036,854,775,808** | **92,233,720,368.54775808** |

`INT64_MAX` = 9,223,372,036,854,775,807 — the sum is exactly `INT64_MAX + 1`.

Two details are required:

1. **`CAccounts` is `std::map<CScript, CBalances>`**, so iteration follows the
   *script bytes*. The destination with the **smaller** script must carry the
   **smaller** amount, otherwise the truncated sum is not small enough to pass
   the balance check. Addresses must therefore be ground.
2. The transaction must be built **raw**. `ParseFixedPoint`
   (`src/util/strencodings.cpp:457`) rejects amounts with more than 18
   significant digits, so the RPC path (`accounttoaccount`) cannot express the
   required value.

## Reproduction

Against an unmodified v4.2.5 node (regtest):

```
source account:            8.00000001 DFI
destination A:       +     8.00000001 DFI
destination B:       +92,233,720,360.54776001 DFI   <-- created from nothing
```

The transaction is accepted and the destination account balance becomes
92,233,720,360.54776001 DFI.

**Negative control:** with the identical setup but a sum of exactly
`INT64_MAX` (no overflow), the transaction is correctly rejected:

```
AccountToAccountTx: amount 0.00000000 is less than 92233720360.54775807
```

This confirms that the overflow — not the transaction itself — is what opens
the hole.

A reproduction script is available in the incident workspace
(`docs/exploits/2026-11-inflation/repro/reproduce_exploit.py`).

## Observed on mainnet

Three transactions were found between block 6,558,000 and the chain tip
(6,574,200), all on 2026-11-03:

| Block | Time (UTC) | Transaction | Note |
|---|---|---|---|
| 6,564,035 | 00:13:18 | `c0248c396a0be9fe6122cab0c4f0c1ab4f8160bdde85893a9afd592732cfef0e` | different actor |
| 6,565.374 | 12:10:36 | `b2518be0b1e17170c2fe6ff59600d740a74b133cef89ebe4761e900c1a76857a` | self-credit |
| 6,566,081 | 19:02:56 | `bc2e5a684592cc2f504268d23a418ca66cbb9900bf4dcb1b5c2eb3ec58b2e8ac` | — |

Each transaction can mint up to `INT64_MAX` ≈ 92.2 billion DFI. The
destination of two of them still holds 92,233,720,364.08 DFI, unmoved.

**The issue is open — it can be exploited again at any time.**

## Suggested fix

Propagate the error instead of discarding it:

```diff
-inline CBalances SumAllTransfers(const CAccounts &to) {
+inline ResVal<CBalances> SumAllTransfers(const CAccounts &to) {
     CBalances sum;
     for (const auto &kv : to) {
-        sum.AddBalances(kv.second.balances);
+        if (auto res = sum.AddBalances(kv.second.balances); !res) {
+            return res;
+        }
     }
-    return sum;
+    return {sum, Res::Ok()};
 }
```

All 12 call sites need to be updated; two of them are consensus-critical
(`accounts.cpp:80` and `accounts.cpp:96`).

Additional hardening recommended:

- Enforce the check where the money is created (in the crediting path), so a
  future code path cannot reintroduce the bug.
- Bound **account balances** by `MAX_MONEY` (`src/amount.h:226`). Today
  `MoneyRange()` only covers UTXO values; account balances are unbounded.
  Note the existing comment on that constant already anticipates this class of
  bug.
- Add a lint rule for ignored `Res`/`ResVal` return values.

## Note on rollback

`CAccountsConsensus` runs in **both** the mempool path (`CheckTx`) and the
block-validation path (`ConnectBlock`). A fix at this location therefore also
invalidates the already-mined exploit blocks when the chain is revalidated
(`-reindex`), causing a rollback.

This is a **consensus change**: all masternodes must be upgraded together.
