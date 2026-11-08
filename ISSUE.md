# Critical bug found in DeFiChain: tokens can be created from nothing

**Severity:** Critical — a flaw in the blockchain software itself
**Affected:** DeFiChain Mainnet (and Testnet/Regtest)
**Status:** **Not fixed yet. The problem can still be exploited.**

---

## What happened

On 3 November 2026, three transactions were published on the DeFiChain
Mainnet that each created **around 92 billion DFI** — without that DFI ever
existing, and without anyone paying for it.

For comparison: DeFiChain was designed for a total supply of roughly
1 billion DFI. The transactions created roughly **92 times** that amount.

The tokens are real in the sense that the network accepted them and counted
them. Two of the three destinations still hold the balance today, and it has
not been moved.

Exchanges reacted by halting trading in DFI and blocking deposits — for
example Bitfine reported a trading halt. The problem is not limited to
exchanges: any wallet, application or service that reads a balance from the
blockchain is affected.

---

## Why it was possible

Imagine a cash register that adds up the prices of everything a customer
buys.

The register has a maximum value it can hold. If you try to add up more than
that, the register should stop and say "this is too much".

In this case the register **did** notice the problem and reported it — but
that warning was then thrown away without anyone reading it. So the register
went ahead and added up only the **first** item on the list.

The result: the customer was checked against the price of the first item
only, but was allowed to walk away with everything on the list.

Applied to the blockchain:

| | |
|---|---|
| What the customer had | 8 DFI |
| What they were allowed to take away | 8 DFI + 92 billion DFI |
| Where the extra 92 billion came from | nowhere — it was simply counted into existence |

The only thing an attacker needed was **8 DFI** (roughly one cent) and some
technical effort to build the transaction by hand.

---

## Is the money real / can it be spent?

Yes — which is exactly why the exchanges had to halt trading. There is far
more DFI in circulation than there should ever be.

---

## What happens now

This is a flaw in the **consensus rules** of the blockchain — the part of the
software that every node and every masternode must agree on. It is not a bug
in a wallet, an exchange or a website, and it cannot be fixed by upgrading
those.

Because the flaw is in the rules themselves, fixing it means the network
changes what it considers a valid block. That has two consequences:

1. **The new software must be adopted by all masternodes at the same time.**
   If only some nodes upgrade, the network splits in two and does not work.
2. **The affected blocks can be rolled back.** When the corrected software
   re-checks the chain, the three transactions are recognised as invalid and
   the network returns to the state before them — which removes the
   fabricated tokens from circulation.

A fix has been identified and written up: a small change in one function
plus the removal of a second, equivalent flaw in a related function.

---

## What we are doing

- [x] Identified the cause of the flaw
- [x] Reproduced the issue on a private test network, against the unmodified
      software — including a control test showing that the same transaction
      is correctly rejected when the flaw is not triggered
- [x] Identified all three affected transactions on Mainnet
- [x] Written up the cause, the fix and the test procedure
- [ ] Final review of the fix by a C++ developer
- [ ] Independent verification of the fix on the test network
- [ ] Coordinated release and rollout across all masternodes
- [ ] Decide on the rollback together with the masternode operators
- [ ] Inform exchanges and wallet providers about the planned rollback

---

## Summary for the technically inclined

<details>
<summary>Code references, reproduction and the proposed patch</summary>

### Root cause

`SumAllTransfers()` in `src/dfi/balances.h:265` ignores the return value of
`AddBalances()`. When the sum of the transfer amounts exceeds the maximum
value a 64-bit integer can hold, `SafeAdd()` returns `Err("overflow")` — but
the error is discarded and the sum silently stays at the **first** amount.

The source account is then debited by that truncated amount, while each
destination is credited with its **full** amount. The difference is newly
created DFI.

```cpp
// src/dfi/balances.h:265
inline CBalances SumAllTransfers(const CAccounts &to) {
    CBalances sum;
    for (const auto &kv : to) {
        sum.AddBalances(kv.second.balances);   // <-- error ignored
    }
    return sum;
}
```

`src/dfi/consensus/accounts.cpp:80` (`AccountToAccount`, type `'B'`)

```cpp
if (auto res = SubBalanceDelShares(obj.from, SumAllTransfers(obj.to)); !res) {
    return res;                         // debits the TRUNCATED sum
}
return AddBalancesSetShares(obj.to);   // credits the FULL amounts
```

`src/dfi/consensus/accounts.cpp:96` (`AnyAccountsToAccounts`, type `'a'`) has
the same defect: `sumFrom` and `sumTo` are both truncated, so the
`sumFrom != sumTo` check passes and the same inflation is possible.

The balance check itself (`CTokenAmount::Sub`, `src/amount.h:163`) is correct —
it is simply never presented with the real amount.

### Trigger

Two destinations with the same token, whose amounts sum to more than
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

### Reproduction (regtest, unmodified v4.2.5)

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

### Observed on mainnet

| Block | Time (UTC) | Transaction | Note |
|---|---|---|---|
| 6,564,035 | 00:13:18 | `c0248c396a0be9fe6122cab0c4f0c1ab4f8160bdde85893a9afd592732cfef0e` | different actor |
| 6,565,374 | 12:10:36 | `b2518be0b1e17170c2fe6ff59600d740a74b133cef89ebe4761e900c1a76857a` | self-credit |
| 6,566,081 | 19:02:56 | `bc2e5a684592cc2f504268d23a418ca66cbb9900bf4dcb1b5c2eb3ec58b2e8ac` | — |

Each transaction can mint up to `INT64_MAX` ≈ 92.2 billion DFI. The
destination of two of them still holds 92,233,720,364.08 DFI, unmoved.

### Suggested fix

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
(`src/dfi/consensus/accounts.cpp:80` and `:96`).

Additional hardening recommended:

- Enforce the check where the money is created (in the crediting path), so a
  future code path cannot reintroduce the bug.
- Bound **account balances** by `MAX_MONEY` (`src/amount.h:226`). Today
  `MoneyRange()` only covers UTXO values; account balances are unbounded.
  Note the existing comment on that constant already anticipates this class of
  bug.
- Add a lint rule for ignored `Res`/`ResVal` return values.

### Note on rollback

`CAccountsConsensus` runs in **both** the mempool path (`CheckTx`) and the
block-validation path (`ConnectBlock`). A fix at this location therefore also
invalidates the already-mined exploit blocks when the chain is revalidated
(`-reindex`), causing a rollback.

This is a **consensus change**: all masternodes must be upgraded together.

</details>
