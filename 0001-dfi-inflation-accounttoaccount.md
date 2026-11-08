# 0001 — DFI-Inflation: `SumAllTransfers()` ignoriert den Overflow-Fehler

| | |
|---|---|
| **Schweregrad** | **kritisch** |
| **Status** | **implementiert und verifiziert** (Branch `feature/fix-0001-sum-all-transfers`, 3 Commits) — noch nicht gepusht, kein Rollback durchgeführt |
| **Herkunft** | [DFI-Inflation 03.11.2026](../exploits/2026-11-inflation/) |
| **Betroffen** | DeFiChain v4.2.5 (`bd09486bb03e`), Mainnet + Testnet + Regtest |
| **Code-Stelle** | `src/dfi/balances.h:265` (Wurzel), `src/dfi/consensus/accounts.cpp:80` und `:96` (Ausnutzung) |
| **Auswirkung** | Beliebig viele DFI aus dem Nichts — bis `INT64_MAX` (≈ 92,2 Mrd. DFI) pro Transaktion |
| **Aufwand** | ~1 Tag Implementierung + Test |

## Beschreibung

`SumAllTransfers()` summiert die Beträge aller Transfer-Zielkonten, **ignoriert
aber den Rückgabewert** von `AddBalances()`. Überläuft die Summe `int64_t`, meldet
`SafeAdd()` korrekt `Err("overflow")` — der Fehler wird aber verworfen, und
die Summe bleibt beim **ersten** Betrag stehen.

Das Quellkonto wird daraufhin nur um die **gekürzte** Summe belastet, während
jedes Zielkonto den **vollen** Betrag gutgeschrieben bekommt. Die Differenz
ist neu erzeugtes DFI.

```cpp
// src/dfi/balances.h:265
inline CBalances SumAllTransfers(const CAccounts &to) {
    CBalances sum;
    for (const auto &kv : to) {
        sum.AddBalances(kv.second.balances);   // ← Rückgabewert wird ignoriert
    }
    return sum;
}
```

## Ausnutzung

Zwei Empfänger **mit derselben Token-ID**, deren Beträge zusammen mehr als
`INT64_MAX` ergeben:

| | satoshi | DFI |
|---|---:|---:|
| Empfänger A | 800.000.001 | 8.00000001 |
| Empfänger B | 9.223.372.036.054.775.807 | 92.233.720.360,54775801 |
| **Summe** | **9.223.372.036.854.775.808** | **92.233.720.368,54775808** |

`INT64_MAX` = 9.223.372.036.854.775.807 — die Summe ist exakt `INT64_MAX + 1`.

Zwei Randbedingungen:

1. `CAccounts` ist `std::map<CScript, CBalances>` — die Iteration folgt den
   **Script-Bytes**. Der Empfänger mit dem **kleineren** Script muss den
   **kleineren** Betrag bekommen, damit die gekürzte Summe klein bleibt.
   Die Adressen sind daher zu „grounden“ (Hash-Brute-Force).
2. Die Transaktion muss **roh** gebaut werden. `ParseFixedPoint`
   (`src/util/strencodings.cpp:457`) erlaubt höchstens 18 signifikante
   Stellen; der Exploit-Betrag braucht 19. Über den RPC-Weg
   (`accounttoaccount`) ist der Angriff deshalb **nicht** möglich.

**Angriffskosten:** 8,00000001 DFI (~0,01 €).

## Betroffene Transaktionstypen

| Typ | Zeichen | Konsensstelle | Ausnutzbar |
|---|---|---|---|
| `AccountToAccount` | `'B'` (0x42) | `accounts.cpp:80` | **ja** — dokumentiert |
| `AnyAccountsToAccounts` | `'a'` (0x61) | `accounts.cpp:96` | **ja** — im Report nicht erwähnt, bisher nicht ausgenutzt |

Nicht ausnutzbar (geprüft):

| Typ | Grund |
|---|---|
| `UtxosToAccount` (`'U'`) | Der Vergleich `burnt != mustBeBurnt` schlägt bei gekürzter Summe fehl → TX wird abgelehnt |
| `AccountToUtxos` (`'b'`) | Nutzt `SumAllTransfers` nicht |
| `AddPoolLiquidity` (`'l'`) | Zieht die echten Beträge ab (`kv.second`), nicht die Summe |
| `RemovePoolLiquidity` (`'r'`) | Nutzt `SumAllTransfers` nicht |

## Beleg

**Auf dem Mainnet gefunden** — drei Transaktionen am 03.11.2026:

| Block | Zeit (UTC) | TXID |
|---|---|---|
| 6.564.035 | 00:13:18 | `c0248c396a0be9fe6122cab0c4f0c1ab4f8160bdde85893a9afd592732cfef0e` |
| 6.565.374 | 12:10:36 | `b2518be0b1e17170c2fe6ff59600d740a74b133cef89ebe4761e900c1a76857a` |
| 6.566.081 | 19:02:56 | `bc2e5a684592cc2f504268d23a418ca66cbb9900bf4dcb1b5c2eb3ec58b2e8ac` |

**Lokal reproduziert** gegen den unveränderten Chain-Stand:

```
source account:            8.00000001 DFI
destination A:       +     8.00000001 DFI
destination B:       +92,233,720,360.54776001 DFI   <-- erzeugt
```

**Negativkontrolle** (Summe exakt `INT64_MAX`, kein Überlauf) → die Chain lehnt
korrekt ab: `amount 0.00000000 is less than 92233720360.54775807`.

Details: [BUG-ANALYSE.md](../exploits/2026-11-inflation/BUG-ANALYSE.md),
[NOTEBOOK.md](../exploits/2026-11-inflation/NOTEBOOK.md),
[Bug-Report (EN)](../exploits/2026-11-inflation/BUG-REPORT.md),
[Issue-Text (EN)](../exploits/2026-11-inflation/ISSUE.md).

## Empfohlene Lösung

`SumAllTransfers()` Fehler propagieren lassen statt sie zu verschlucken:

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

Alle 12 Aufrufstellen sind anzupassen (2 davon konsensuskritisch). Der
**Wichtig:** Der Compiler zwingt die Vollständigkeit **nicht** — `ResVal` hat
einen impliziten `operator T()`, deshalb kompilieren die alten Schreibweisen
unverändert weiter (Debug: `assert`, Release: undefiniertes Verhalten).
Die 12 Stellen müssen per `grep` und Review abgedeckt werden →
[Issue 0006](0006-resval-implicit-conversion-hides-errors.md).

Vollständiger Diff mit Begründung je Stelle:
[FIX.md](../exploits/2026-11-inflation/FIX.md) ·
Patch-Datei: [`fix-proposed.diff`](../exploits/2026-11-inflation/fix-proposed.diff)
(gegen `bd09486bb03e` erzeugt, C++-Kompilierung verifiziert; **noch nicht
angewendet**)

## Verifikation

| Schritt | Erwartung | Ergebnis |
|---|---|---|
| C++-Kompilierung | baut fehlerfrei | ✅ `defid` gelinkt, 0 Fehler |
| Reproduktionstest: Exploit-TX | wird mit `overflow` abgelehnt | ✅ `AccountToAccountTx: overflow` |
| Reproduktionstest: Kontostände | kein Guthaben erzeugt | ✅ Empfänger 0,00000000 DFI (vorher 92 Mrd.) |
| Reproduktionstest: Quellkonto | unverändert | ✅ 8,00000001 DFI |
| Reproduktionstest: Negativkontrolle | verhält sich **unverändert** | ✅ weiterhin abgelehnt (`amount 8.00000001 is less than …`) |
| **Regressionstest** | schlägt ohne Fix fehl, besteht mit Fix | ✅ `feature_account_overflow.py` — **ungepatcht: `AssertionError: No exception raised`** (die TX wird akzeptiert!), **gepatcht: grün** |
| Functional Tests | grün | ✅ **26/26** bestanden (u. a. `feature_any_accounts_to_accounts`, `feature_poolpair_liquidity`, `feature_sendtokenstoaddress`, `feature_accounts_validation`) |
| Test mit Regressionstest | grün | ✅ **12/12** bestanden |
| `make chain-test-cpp` / `-rs` | grün | ⬜ noch nicht gelaufen (Rust-Bau in dieser Umgebung nicht möglich, siehe [Issue 0006](0006-resval-implicit-conversion-hides-errors.md)) |

Belege: `../exploits/2026-11-inflation/patches/data/repro-run-fixed.log`,
`../exploits/2026-11-inflation/patches/data/testruns/`
(`gegenprobe-gepatcht-ungepatcht.log` = die Gegenprobe in beide Richtungen)

## Konsens- und Rollback-Hinweis

`CAccountsConsensus` läuft im Mempool- **und** im Block-Validierungspfad. Ein
Fix an dieser Stelle invalidiert die bereits geminteten Exploit-Blöcke bei
`-reindex` → Rollback der drei Blöcke möglich.

Das ist eine **Konsens-Änderung**: alle Masternodes müssen gleichzeitig
umgestellt werden.

## Verwandte Issues

- [0002](0002-cbalances-sum-ignores-overflow.md) — dieselbe Fehlerklasse an
  anderer Stelle (`CBalances::Sum`, toter Code)
- [0003](0003-account-balance-not-bounded-by-max-money.md) — fehlende
  Betragsobergrenze für Account-Salden (zweite Schranke)
- [0006](0006-resval-implicit-conversion-hides-errors.md) — warum der Compiler
  die Anpassung der Aufrufstellen **nicht** erzwingt
