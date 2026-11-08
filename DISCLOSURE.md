# Security Disclosure: DFI-Inflation via ignored overflow in `SumAllTransfers()`

**Kontakt:** `security@defichain.com` (gemäß `SECURITY.md` in `DeFiCh/ain`)
**Klassifizierung:** Critical — Consensus-Bug, erzeugt DFI aus dem Nichts
**Betroffene Version:** v4.2.5 (`bd09486bb03e`), Mainnet / Testnet / Regtest
**Fix-Branch:** `heddes24/defichain:feature/fix-0001-sum-all-transfers` (`cd106b36c1`)
**Stand:** 06.11.2026

---

## 1. Kurzfassung

`SumAllTransfers()` summiert die Beträge aller Transfer-Zielkonten, **ignoriert
aber den Rückgabewert** von `AddBalances()`. Überläuft die Summe `int64_t`,
meldet `SafeAdd()` korrekt `Err("overflow")` — der Fehler wird jedoch
verworfen, und die Summe bleibt beim **ersten** Betrag stehen.

Das Quellkonto wird daraufhin nur um die **gekürzte** Summe belastet, während
jedes Zielkonto den **vollen** Betrag gutgeschrieben bekommt. Die Differenz ist
**neu erzeugtes DFI**.

| | |
|---|---|
| **Maximaler Schaden pro Transaktion** | `INT64_MAX` ≈ **92,2 Mrd. DFI** |
| **Kosten für den Angreifer** | **8,00000001 DFI** (≈ 0,01 €) |
| **Angriffsvektor** | `AccountToAccount` (`'B'`) und `AnyAccountsToAccounts` (`'a'`) |
| **Transaktion bauen** | roh (Hex + Signatur) — der RPC-Weg kann den Betrag nicht ausdrücken |
| **Status** | **Lücke ist offen** — jederzeit erneut ausnutzbar |

---

## 2. Beobachtungen auf dem Mainnet

Drei Transaktionen am **03.11.2026**, gefunden durch einen vollständigen Scan
des Blockbereichs 6.558.000 bis zum Tip (6.574.200):

| Block | Zeit (UTC) | TXID | Akteur |
|---|---|---|---|
| 6.564.035 | 00:13:18 | `c0248c396a0be9fe6122cab0c4f0c1ab4f8160bdde85893a9afd592732cfef0e` | `df1qqgvcp6…` → `df1qel8g4g…` |
| 6.565.374 | 12:10:36 | `b2518be0b1e17170c2fe6ff59600d740a74b133cef89ebe4761e900c1a76857a` | Selbstgutschriftung `df1qqtlz4u…` |
| 6.566.081 | 19:02:56 | `bc2e5a684592cc2f504268d23a418ca66cbb9900bf4dcb1b5c2eb3ec58b2e8ac` | `df1qe9eus7…` → `df1qqtlz4u…` |

Die ersten beiden Blöcke stammen von **verschiedenen Akteuren** — die Lücke
war also bereits bekannt bzw. gefunden worden.

**Kontostand des Empfängers `df1qqtlz4uw9w5s4pupwgucv4shl6atqw7xlz2wn07`:**

```
Account-Guthaben:  92.233.720.364,08003235 DFI   (≈ 92,2 Mrd.)
UTXO-Guthaben:              0,99643569 DFI
```

Der Betrag ist zum Zeitpunkt der Analyse **unbewegt**.

---

## 3. Root Cause

### 3.1 Die fehlerhafte Funktion

`src/dfi/balances.h:265`

```cpp
inline CBalances SumAllTransfers(const CAccounts &to) {
    CBalances sum;
    for (const auto &kv : to) {
        sum.AddBalances(kv.second.balances);   // ← Rückgabewert wird IGNORIERT
    }
    return sum;
}
```

`AddBalances` → `Add` → `SafeAdd` (`src/amount.h:96`) erkennt den Überlauf
**korrekt** und bricht **vor** der Mutation ab:

```cpp
const auto sum = a + b;
if ((sum - a) != b || static_cast<uint64_t>(std::numeric_limits<CAmount>::max()) < sum) {
    return Res::Err("overflow");
}
```

Der Fehler wird nur nicht weitergereicht.

### 3.2 Die Ausnutzung

`src/dfi/consensus/accounts.cpp:80` (`AccountToAccount`, Typ `'B'`)

```cpp
if (auto res = SubBalanceDelShares(obj.from, SumAllTransfers(obj.to)); !res) {
    return res;                         // ← belastet die Quelle mit der GEKÜRZTEN Summe
}
return AddBalancesSetShares(obj.to);    // ← schreibt die VOLLEN Beträge gut
```

`src/dfi/consensus/accounts.cpp:96` (`AnyAccountsToAccounts`, Typ `'a'`) — gleiche
Fehlerklasse, beide Seiten sind kontrollierbar:

```cpp
const auto sumFrom = SumAllTransfers(obj.from);   // gekürzt
const auto sumTo   = SumAllTransfers(obj.to);     // gekürzt
if (sumFrom != sumTo) return Res::Err(...);        // beide gekürzt → gleich → Prüfung besteht
```

Die Balance-Prüfung selbst (`CTokenAmount::Sub`, `src/amount.h:163`) ist
**korrekt** — sie sieht nur nie die richtige Zahl.

### 3.3 Die konkreten Zahlen (Exploit #3)

| | satoshi | DFI |
|---|---:|---:|
| Empfänger A | 800.000.001 | 8,00000001 |
| Empfänger B | 9.223.372.036.054.775.807 | 92.233.720.360,54775801 |
| **Summe** | **9.223.372.036.854.775.808** | **92.233.720.368,54775808** |

`INT64_MAX` = 9.223.372.036.854.775.807 → die Summe ist exakt `INT64_MAX + 1`.

Der erste Add (8 DFI) passt, der zweite löst den Überlauf aus → die Summe
bleibt bei 8 DFI stehen, die Quelle braucht also nur **8,00000001 DFI**.

### 3.4 Zwei Randbedingungen, die der Angreifer erfüllen muss

1. **Adressreihenfolge:** `CAccounts` ist `std::map<CScript, CBalances>` — die
   Iteration folgt den **Script-Bytes**. Der Empfänger mit dem *kleineren*
   Script muss den *kleineren* Betrag bekommen, sonst ist die gekürzte Summe
   zu groß. Die Adressen sind daher zu „grounden“.
2. **Roh bauen:** `ParseFixedPoint` (`src/util/strencodings.cpp:457`) erlaubt
   höchstens 18 signifikante Stellen, der Exploit-Betrag braucht 19. Über
   `accounttoaccount` ist der Angriff deshalb **nicht** möglich.

---

## 4. Der Fix

Branch `heddes24/defichain:feature/fix-0001-sum-all-transfers` (6 Commits, `cd106b36c1`).

**Grundidee:** `SumAllTransfers()` darf Fehler nicht mehr verschlucken.

```diff
--- a/src/dfi/balances.h
+++ b/src/dfi/balances.h
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

Alle **12** Aufrufstellen werden angepasst, davon 2 konsensuskritisch
(`accounts.cpp:80` und `:96`). Die vollständigen Diffs mit Begründung je
Stelle liegen im Branch bei.

**Hinweis für den Review:** `ResVal<T>` besitzt einen impliziten
`operator T()` (`src/dfi/res.h:72`). Die alten Schreibweisen kompilieren
deshalb **weiterhin** — die Anpassung der 12 Stellen muss also per Review
erfolgen, nicht per Compiler. Im Release-Build würde ein nicht behandelter
Fehler zu undefiniertem Verhalten führen (Dereferenzierung eines leeren
`std::optional`). Diesen Punkt haben wir im Branch in den Commit-Messages
explizit dokumentiert.

---

## 5. Verifikation

### 5.1 Reproduktion (gegen den unveränderten Stand)

```
source account:            8.00000001 DFI
destination A:       +     8.00000001 DFI
destination B:       +92,233,720,360.54776001 DFI   <-- erzeugt
```

Die Transaktion wird **akzeptiert**, das Zielkonto wächst auf 92,2 Mrd. DFI.

**Negativkontrolle** (Summe exakt `INT64_MAX`, kein Überlauf) → die Chain lehnt
korrekt ab: `AccountToAccountTx: amount 0.00000000 is less than
92233720360.54775807`. Damit ist belegt, dass nicht die Transaktion, sondern
genau der Überlauf die Lücke ist.

### 5.2 Verhalten mit dem Fix

```
[+] EXPLOIT ABGELEHNT (overflow) - der Fix greift
  Konto des grossen Empfaengers : 0.00000000 DFI
  Konto des kleinen Empfaengers : 0.00000000 DFI
  Guthaben des Angreifers       : 8.00000001 DFI
```

Die Negativkontrolle verhält sich **unverändert** (weiterhin abgelehnt) — der
Fix ändert die Semantik gültiger Transaktionen nicht.

### 5.3 Regressionstest

`chain/test/functional/feature_account_overflow.py`, registriert in
`test_runner.py` — läuft damit bei jedem `make chain-test-py` und in der CI.

Der Test wurde **in beide Richtungen** geprüft:

| Code | Ergebnis |
|---|---|
| ungepatcht (`bd09486bb03e`) | **✖ Failed** — `AssertionError: No exception raised` (die TX wird akzeptiert) |
| gepatcht (`cd106b36c1`) | **✓ Passed** (2 s) |

### 5.4 Testsuite

- 26/26 Functional Tests grün (u. a. `feature_any_accounts_to_accounts`,
  `feature_poolpair_liquidity`, `feature_sendtokenstoaddress`,
  `feature_accounts_validation`)
- 12/12 Functional Tests mit dem neuen Regressionstest grün
- C++-Build fehlerfrei

---

## 6. Auswirkung auf den Rollback

`CAccountsConsensus` läuft im **Mempool-Pfad** (`CheckTx`) **und** in der
**Block-Validierung** (`ConnectBlock`). Ein Fix an dieser Stelle invalidiert
daher auch die bereits geminteten Exploit-Blöcke, wenn die Chain neu
indexiert wird (`-reindex`) → die Node reorganisiert auf den Stand vor dem
Exploit, die gefälschten DFI verschwinden aus dem Umlauf.

Das ist eine **Konsens-Änderung**: alle Masternodes müssen gemeinsam umgestellt
werden. Ein einzelner Node mit neuer Software würde die Blöcke ablehnen und
sich vom Netz isolieren.

---

## 7. Vorschlag zum weiteren Vorgehen

1. **Private Meldung** dieses Briefings an `security@defichain.com`.
2. **Verteilung** des Fixes an alle Masternodes (Release mit Changelog).
3. **Rollback** der Blöcke 6.564.035 / 6.565.374 / 6.566.081 per `-reindex`.
4. **Bilanzierung** der beim Empfänger verbliebenen 92,2 Mrd. DFI
   (Untersuchung läuft; bislang unbewegt).
5. **Öffentliche Veröffentlichung** des Fixes, sobald die Kette gesichert ist.

---

## 8. Offene Fragen an das Security-Team

1. Sollen die Meldung und der Fix vorab **nur** an die Masternode-Betreiber
   gehen, oder ist ein öffentlicher PR gewünscht?
2. Gibt es einen etablierten Ablauf für die abgestimmte Masternode-Umstellung
   (`-reindex`) bei einer bestehenden Chain?
3. Ist der Zeitpunkt der Veröffentlichung (03.11.2026, 00:13 / 12:10 / 19:02
   UTC) relevant für die Priorisierung?

---

## Kontakt

Sebastian Behnisch — `info@behnisch.online`
Fork: `github.com/heddes24/defichain` (Branch `feature/fix-0001-sum-all-transfers`)
