# Fix-Vorschlag: DFI-Inflation (`SumAllTransfers`)

**Status:** implementiert und in `dev` integriert.
**Nur Dokumentation.** Der Fix liegt auf `feature/fix-0001-sum-all-transfers`
(`cd106b36c1`) und ist in den Chain-Branch `dev` (`4475d31aaa`) gemergt
(Chain-Version 4.2.6). Der Workspace pinnt diesen Stand in
`submodules/chain`. Für die Mainnet-Forensik bleibt der ungepatchte
Upstream-Stand `bd09486bb0` maßgeblich (Binary `v4.2.5.0`).

**Ziel:** So einfach wie möglich, so robust wie möglich.

---

## 1. Kurzfassung

| | |
|---|---|
| **Empfehlung** | `SumAllTransfers()` gibt Fehler zurück statt sie zu schlucken (`CBalances` → `ResVal<CBalances>`) |
| **Umfang** | 1 Hilfsfunktion + 12 Aufrufstellen, davon **2 konsensuskritisch** |
| **Warum robust** | Die Funktion kann keinen falschen Wert mehr liefern. **Wichtig:** Der Compiler zwingt die Anpassung der Aufrufstellen *nicht* — `ResVal` hat einen impliziten `operator T()`, siehe [Issue 0006](../issues/0006-resval-implicit-conversion-hides-errors.md). Die 12 Stellen müssen über `grep` + Review abgedeckt werden. |
| **Wirkt auf** | `AccountToAccount` (`'B'`) **und** `AnyAccountsToAccounts` (`'a'`) — beide Lücken |
| **Rollback** | Die betroffenen Blöcke werden beim Revalidieren (`-reindex`) ungültig → Rollback möglich |
| **Zusatz** | 5 Zeilen: Account-Salden gegen `MAX_MONEY` begrenzen (Schutz gegen dieselbe Fehlerklasse) |
| **Status** | implementiert und verifiziert (Branch `feature/fix-0001-sum-all-transfers`) |

---

## 2. Warum gerade diese Lösung

### 2.1 Die drei Kandidaten

| | Kandidat | Zeilen | Robustheit |
|---|---|---|---|
| **A** | **Fehler propagieren** (`ResVal<CBalances>`) | ~40 | **hoch** — die Funktion kann keinen falschen Wert mehr liefern |
| B | Summe sättigen (bei Überlauf `INT64_MAX` zurückgeben) | ~8 | **mittel** — funktioniert, aber die Funktion liefert weiterhin einen *falschen* Wert |
| C | Nur den Gutschriftpfad prüfen (`MAX_MONEY` pro Konto) | ~10 | **mittel** — stoppt den realen Exploit, aber umgehbar mit 100+ Empfängern (siehe 2.3) |

**Empfehlung: A als Fix, C als zusätzliche Härtung.**

> **Korrektur (06.11.2026):** Die ursprüngliche Fassung dieses Dokuments
> argumentierte, der Compiler zwinge die Anpassung aller 12 Aufrufstellen.
> Das ist **falsch**: `ResVal<T>` besitzt einen impliziten
> `operator T()` (`src/dfi/res.h:72`), deshalb kompilieren die alten
> Schreibweisen unverändert weiter — im Debug-Build mit `assert`, im
> Release-Build mit undefiniertem Verhalten. Verifiziert: alle fünf
> betroffenen Übersetzungseinheiten kompilieren ohne Anpassung.
> → [Issue 0006](../issues/0006-resval-implicit-conversion-hides-errors.md)

### 2.2 Warum nicht B (die „kleinste“ Variante)

B ist die kleinste mögliche Änderung: `SumAllTransfers` liefert bei Überlauf
stattdessen `INT64_MAX` zurück. Der Abzug schlägt dann fehl, die Transaktion
wird abgelehnt — die Lücke ist zu.

Aber:

1. **Die Funktion liefert weiterhin etwas Falsches.** Ein Wert, der „groß
   genug zum Scheitern“ ist, ist keine Summe. Das ist eine Landmine für
   den nächsten Aufrufer.
2. **Die Fehlermeldung irreführt.** Der Nutzer bekommt
   *„Not enough balance“* statt *„overflow“* — debugging wird unnötig
   schwer.
3. **Die Robustheit ist nur scheinbar.** Der Compiler zwingt hier nichts
   nach; wer die Funktion später in einem Konsenspfad benutzt, baut die
   Lücke unbemerkt wieder ein.

### 2.3 Warum C (nur den Gutschriftpfad prüfen) kein Ersatz ist

Der naheliegende Einwand: wenn man beim Gutschreiben prüft, entsteht das Geld
ja dort — also dort prüfen.

Diese Prüfung hätte den **real dokumentierten Exploit** tatsächlich
verhindert: der große Betrag (92,2 Mrd. DFI) liegt über `MAX_MONEY`
(1,2 Mrd. DFI), ein Kontosaldo über dieser Grenze wäre abgelehnt worden.

**Aber sie ist umgehbar.** Der Angreifer verteilt die Auszahlung auf viele
Empfänger, dann bleibt jeder Einzelkontostand unter der Grenze:

| Empfänger | je Empfänger | Summe | erzeugte DFI | Kontoprüfung |
|---:|---:|---:|---:|---|
| 2 | 46.116.860.184,27 DFI | > `INT64_MAX` | 46,1 Mrd. | greift |
| **2** (der reale Fall) | **92.233.720.360,55 DFI** | > `INT64_MAX` | **92,2 Mrd.** | **greift** |
| 100 | 922.337.203,69 DFI | > `INT64_MAX` | 91,3 Mrd. | **greift nicht** |
| 1000 | 92.233.720,37 DFI | > `INT64_MAX` | 92,1 Mrd. | **greift nicht** |

⇒ C wäre eine **wirksame zusätzliche Härtung** gegen den konkreten
Exploit — aber sie prüft die Summe nicht. Ein Angreifer mit 100 Empfängern
erzeugt weiterhin ~91 Mrd. DFI.

**⇒ C ist die zweite Schranke, nicht die erste.** Die erste Schranke muss
die Summe prüfen — und das ist genau Fix A.

---

## 3. Der Fix

### 3.1 Die Hilfsfunktion

`src/dfi/balances.h:265`

```diff
--- a/src/dfi/balances.h
+++ b/src/dfi/balances.h
@@ -262,12 +262,16 @@
     }
 };
 
-inline CBalances SumAllTransfers(const CAccounts &to) {
+// Summiert die Betraege aller Zielkonten.
+//
+// ACHTUNG: Der Rueckgabewert von AddBalances muss geprueft werden! Bei einem
+// Ueberlauf meldet SafeAdd "overflow" und Add bricht VOR der Mutation ab -
+// die Summe bliebe dann beim ersten Betrag stehen. Ohne diese Pruefung
+// wuerde die Quelle zu wenig belastet, waehrend die Zielkonten die vollen
+// Betraege gutgeschrieben bekommen (DFI-Inflation, 03.11.2026).
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

**Kein Include nötig:** `balances.h` includet `amount.h`, das wiederum
`dfi/res.h` includiert — `Res`/`ResVal` sind dort also bereits verfügbar.

**Kein Rust-Fix nötig:** in `lib/ain-dftx` gibt es keine Summenlogik; die
Konsenslogik liegt ausschließlich im C++-Teil.

### 3.2 Die Aufrufstellen

Die Änderung des Rückgabetyps zwingt den Compiler, **alle** Aufrufer
anzupassen. Es gibt drei Muster:

**Muster 1 — Wert weiterreichen (Konsenspfade, 3 Stellen)**

`src/dfi/consensus/accounts.cpp:33` (`UtxosToAccount`)

```diff
-    const auto mustBeBurnt = SumAllTransfers(obj.to);
-    if (*burnt.val != mustBeBurnt) {
+    const auto mustBeBurnt = SumAllTransfers(obj.to);
+    if (!mustBeBurnt) {
+        return mustBeBurnt;
+    }
+    if (*burnt.val != *mustBeBurnt.val) {
         return Res::Err(
-            "transfer tokens mismatch burnt tokens: (%s) != (%s)", mustBeBurnt.ToString(), burnt.val->ToString());
+            "transfer tokens mismatch burnt tokens: (%s) != (%s)", mustBeBurnt.val->ToString(), burnt.val->ToString());
     }
```

`src/dfi/consensus/accounts.cpp:80` (`AccountToAccount`, **die Lücke**)

```diff
-    // transfer
-    if (auto res = SubBalanceDelShares(obj.from, SumAllTransfers(obj.to)); !res) {
+    // transfer
+    auto mustTransfer = SumAllTransfers(obj.to);
+    if (!mustTransfer) {
+        return mustTransfer;
+    }
+    if (auto res = SubBalanceDelShares(obj.from, *mustTransfer.val); !res) {
         return res;
     }
```

`src/dfi/consensus/accounts.cpp:96` (`AnyAccountsToAccounts`, **die zweite Lücke**)

```diff
-    const auto sumFrom = SumAllTransfers(obj.from);
-    const auto sumTo = SumAllTransfers(obj.to);
-
-    if (sumFrom != sumTo) {
+    const auto sumFrom = SumAllTransfers(obj.from);
+    if (!sumFrom) {
+        return sumFrom;
+    }
+    const auto sumTo = SumAllTransfers(obj.to);
+    if (!sumTo) {
+        return sumTo;
+    }
+
+    if (*sumFrom.val != *sumTo.val) {
         return Res::Err("sum of inputs (from) != sum of outputs (to)");
     }
```

`src/dfi/consensus/poolpairs.cpp:190` (`AddPoolLiquidity`)

```diff
-    CBalances sumTx = SumAllTransfers(obj.from);
+    auto sumTxRes = SumAllTransfers(obj.from);
+    if (!sumTxRes) {
+        return sumTxRes;
+    }
+    CBalances sumTx = *sumTxRes.val;
```

**Muster 2 — Fehler als RPC-Fehler melden (4 Stellen in `rpc_accounts.cpp`)**

```diff
-    const auto toBurn = SumAllTransfers(msg.to);
-    if (toBurn.balances.empty()) {
+    auto toBurnRes = SumAllTransfers(msg.to);
+    if (!toBurnRes) {
+        throw JSONRPCError(RPC_INVALID_PARAMETER, toBurnRes.msg);
+    }
+    const auto toBurn = *toBurnRes.val;
+    if (toBurn.balances.empty()) {
         throw JSONRPCError(RPC_INVALID_PARAMETER, "zero amounts");
     }
```

Für die übrigen drei Stellen in `rpc_accounts.cpp` (Zeilen 989, 1102, 2129)
ist das Muster identisch:

```diff
-    if (SumAllTransfers(msg.to).balances.empty()) {
+    auto sumRes = SumAllTransfers(msg.to);
+    if (!sumRes) {
+        throw JSONRPCError(RPC_INVALID_PARAMETER, sumRes.msg);
+    }
+    if (sumRes.val->balances.empty()) {
         throw JSONRPCError(RPC_INVALID_PARAMETER, "zero amounts");
     }
```

**Muster 3 — Wert ignorieren bzw. ausgeben (2 Stellen)**

`src/dfi/mn_rpc.cpp:109`

```diff
-    const auto selectedBalancesSum = SumAllTransfers(selectedAccountsBalances);
-    if (selectedBalancesSum != targetBalances) {
+    auto selectedBalancesSumRes = SumAllTransfers(selectedAccountsBalances);
+    if (!selectedBalancesSumRes) {
+        return {};
+    }
+    const auto selectedBalancesSum = *selectedBalancesSumRes.val;
+    if (selectedBalancesSum != targetBalances) {
         // we have not enough tokens balance to transfer
         return {};
     }
```

`src/dfi/rpc_customtx.cpp:196`

```diff
-        CBalances sumTx = SumAllTransfers(obj.from);
-        if (sumTx.balances.size() == 2) {
+        auto sumTxRes = SumAllTransfers(obj.from);
+        if (!sumTxRes) {
+            rpcInfo.pushKV("error", sumTxRes.msg);
+            return;
+        }
+        const CBalances sumTx = *sumTxRes.val;
+        if (sumTx.balances.size() == 2) {
```

### 3.3 Was der Fix bewirkt

```
vorher:  Summe = 800.000.001 (gekürzt)   → Abzug 8 DFI     → Auszahlung 92 Mrd.
nachher: SafeAdd meldet "overflow"       → Transaktion wird abgelehnt
```

Der Fehlertext ist aussagekräftig (`overflow`), die Transaktion wird in
Mempool **und** Block-Validierung abgelehnt.

---

## 4. Zusätzliche Härtung (5 Zeilen, unabhängig wirksam)

Auch mit Fix A bleibt `MAX_MONEY` (`src/amount.h:226`) die Obergrenze für
*erlaubte* Geldmengen — bisher wird sie aber nur auf **UTXO-Werte**
angewendet, nicht auf **Account-Salden**.

```diff
--- a/src/dfi/accounts.cpp
+++ b/src/dfi/accounts.cpp
@@
 Res CAccountsView::AddBalance(const CScript &owner, CTokenAmount amount) {
     ...
 }
```

Konkret in `CAccountsView::AddBalance` (bzw. `AddBalances`) prüfen, dass der
resultierende Saldo `MoneyRange()` nicht verlässt:

```cpp
if (!MoneyRange(newBalance)) {
    return Res::Err("account balance out of range");
}
```

Das ist eine **unabhängige** Schranke: Jeder Pfad, der über kurz oder lang
Guthaben erzeugt, würde daran scheitern. Der Kommentar an `MAX_MONEY`
beschreibt genau diese Absicht — er greift nur bisher nicht, weil er im
Account-Pfad nicht aufgerufen wird:

> *„…the exact value of the MAX_MONEY constant is consensus critical; in
> unusual circumstances like a(nother) overflow bug that allowed for the
> creation of coins out of thin air modification could lead to a fork.“*
> — `src/amount.h:218-226`

**Grenzen dieser Härtung:** Sie greift **nur pro Konto**. Der Angriff mit
100 Empfängern mit je 0,92 Mrd. DFI würde sie passieren (jeder Saldo bleibt
unter 1,2 Mrd. DFI) und trotzdem ~91 Mrd. DFI erzeugen. Sie ersetzt Fix A
also **nicht** — sie ist die zweite Schranke.

---

## 5. Regressionstest

### 5.1 Sofort verfügbar (ohne Code-Änderung)

`docs/exploits/2026-11-inflation/repro/reproduce_exploit.py` prüft die
Transaktion. Erwartetes Verhalten:

| | vorher | nach Fix A |
|---|---|---|
| Exploit-TX (Summe > `INT64_MAX`) | **akzeptiert**, 92 Mrd. erzeugt | **abgelehnt** (`overflow`) |
| Negativkontrolle (Summe = `INT64_MAX`) | abgelehnt (`Not enough balance`) | abgelehnt (`Not enough balance`) |

Die Negativkontrolle ist wichtig: Sie darf sich **nicht** ändern. Sonst hätte
der Fix die Semantik kaputtgemacht.

### 5.2 Regressionstest in der Chain-Testsuite — erledigt

Der Test liegt jetzt **in der Chain** und läuft bei jedem `make chain-test-py`
und in der CI mit:

```
chain/test/functional/feature_account_overflow.py
```

Er ist in `test_runner.py` in der `BASE_SCRIPTS`-Liste registriert und deckt
den Angriffsvektor vollständig ab. Zwei Details waren dafür nötig:

1. **Die Transaktion muss roh gebaut werden** — der RPC-Weg kann den Betrag
   nicht ausdrücken (`ParseFixedPoint` erlaubt max. 18 signifikante Stellen).
2. **Die Empfänger sind nach Script-Bytes zu sortieren** — `CAccounts` ist
   `std::map<CScript, CBalances>`.

Assertions:

```python
# 1. Exploit-TX muss abgelehnt werden
assert_raises_rpc_error(-26, "overflow", node.sendrawtransaction, signed_hex)

# 2. Kein Guthaben entstanden
assert_equal(self.account_balance(node, big_addr), 0)
assert_equal(self.account_balance(node, small_addr), 0)

# 3. Negativkontrolle: ohne Overflow weiterhin abgelehnt
assert_raises_rpc_error(-26, "is less than", node.sendrawtransaction, signed_control["hex"])
```

**Die Gegenprobe (beide Richtungen):**

| Code | Test | Ergebnis |
|---|---|---|
| ungepatcht (`bd09486bb03e`) | `feature_account_overflow.py` | **✖ Failed** — `AssertionError: No exception raised` (die TX wird akzeptiert!) |
| gepatcht (Branch) | `feature_account_overflow.py` | **✓ Passed** (2 s) |

Beleg: `patches/data/testruns/gegenprobe-gepatcht-ungepatcht.log`

Ohne diese Gegenprobe wäre der Test wertlos gewesen — er besteht auch gegen
den ungepatchten Code, weil das *Binary* im Build die alte Version war.

### 5.3 Unit-Test (optional, billig)

`src/test/amount_tests.cpp` ist das passende Zuhause für einen Test, der
`SafeAdd`/`SafeSub` und die neue Signatur von `SumAllTransfers` direkt prüft:

```cpp
BOOST_AUTO_TEST_CASE(sum_all_transfers_overflow)
{
    CAccounts to;
    to[script1] = CBalances{TAmounts{{DCT_ID{0}, 800000001}}};
    to[script2] = CBalances{TAmounts{{DCT_ID{0}, INT64_MAX - 800000000}}};

    const auto sum = SumAllTransfers(to);
    BOOST_CHECK(!sum);              // Fehler wird gemeldet
    BOOST_CHECK(sum.msg.find("overflow") != std::string::npos);
}
```

---

## 6. Reihenfolge und Risiko

| Schritt | Risiko | Aufwand |
|---|---|---|
| 1. Fix A umsetzen (1 Datei + 12 Call-Sites) | **niedrig** — rein mechanisch, Compiler hilft | ~1 Tag |
| 2. Bestehende Tests laufen lassen (`make chain-test`) | niedrig | ~1 h |
| 3. Regressionstest ergänzen (Functional + Unit) | niedrig | ~0,5 Tag |
| 4. §4-Härtung (`MAX_MONEY` für Account-Salden) | **mittel** — touched die Balance-Buchhaltung | ~0,5 Tag |
| 5. Unabhängige Review durch einen C++-Entwickler | — | — |
| 6. Rollback-Entscheidung mit den Masternodes | — | — |

**Schritt 4 ist optional** — Fix A schließt die Lücke vollständig. §4 ist
Schadensbegrenzung für die Zukunft.

### Zur Konsens-Kompatibilität

Fix A ändert, **welche Transaktionen gültig sind**. Konsequenzen:

- Die drei Exploit-Transaktionen werden ungültig → bei `-reindex` werden die
  Blöcke 6.564.035 / 6.565.374 / 6.566.081 ungültig → **Rollback**.
- **Alle Masternodes müssen gleichzeitig umgestellt werden.** Ein einzelner
  Node mit neuer Software würde die Blöcke ablehnen und sich vom Netz
  isolieren.
- Für Testnet und Regtest ist die Umstellung unkritisch.

---

## 7. Prüfliste vor dem Release

- [ ] `SumAllTransfers` propagiert den Fehler (`ResVal<CBalances>`)
- [ ] alle 12 Aufrufstellen angepasst (per `grep` verifiziert, **nicht** vom Compiler erzwungen)
- [ ] Fehlertext ist aussagekräftig (`overflow`)
- [ ] `AccountToAccount` (`'B'`) lehnt ab
- [ ] `AnyAccountsToAccounts` (`'a'`) lehnt ab
- [ ] Negativkontrolle (Summe = `INT64_MAX`) verhält sich **unverändert**
- [ ] bestehende Functional-Tests laufen durch (`make chain-test-py`)
- [ ] bestehende Unit-Tests laufen durch (`make chain-test-cpp`, `make chain-test-rs`)
- [ ] `createmasternode` / Pool-Operationen funktionieren weiterhin
      (diese Pfade nutzen `SumAllTransfers` ebenfalls)
- [ ] Regressionstest vorhanden und rot ohne / grün mit Fix
- [ ] unabhängige Review erfolgt
- [ ] Rollback-Ablauf mit den Masternode-Betreibern abgestimmt

---

## 8. Referenzen

| Was | Wo |
|---|---|
| Bug | `src/dfi/balances.h:265` |
| Lücke 1 (`'B'`) | `src/dfi/consensus/accounts.cpp:80` |
| Lücke 2 (`'a'`) | `src/dfi/consensus/accounts.cpp:96` |
| Overflow-Erkennung (korrekt) | `src/amount.h:96` (`SafeAdd`) |
| Balance-Prüfung (korrekt) | `src/amount.h:163` (`CTokenAmount::Sub`) |
| Betragsobergrenze | `src/amount.h:226` (`MAX_MONEY`) |
| Betragsgrenze im RPC | `src/util/strencodings.cpp:457` (`ParseFixedPoint`) |
| Reproduktionstest | `docs/exploits/2026-11-inflation/repro/reproduce_exploit.py` |
| Ausführliche Analyse | `docs/exploits/2026-11-inflation/BUG-ANALYSE.md` |
