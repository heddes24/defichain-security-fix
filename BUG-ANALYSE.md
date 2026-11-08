# Der DFI-Inflation-Bug — erklärt

**DeFiChain Mainnet · 03.11.2026 · C++ (`src/dfi/balances.h`)**

> **Eine Zeile Code, ein vergessener Rückgabewert, 92 Millionen DFI aus dem Nichts.**
>
> Betroffen: `AccountToAccount` (`'B'`) und `AnyAccountsToAccounts` (`'a'`)
> — beide erzeugen DFI, das nie existiert hat.
> Der Chain-Code wurde für dieses Dokument **nicht** verändert.

---

## Inhalt

1. [Die Idee hinter dem Bug](#1-die-idee-hinter-dem-bug)
2. [Der Code-Pfad](#2-der-code-pfad)
3. [Der Fehler im Detail](#3-der-fehler-im-detail)
4. [Was der Angreifer tun musste](#4-was-der-angreifer-tun-musste)
5. [Die drei Exploits auf dem Mainnet](#5-die-drei-exploits-auf-dem-mainnet)
6. [Warum die Prüfung nicht ausgelöst hat](#6-warum-die-prüfung-nicht-ausgelöst-hat)
7. [Der Fix](#7-der-fix)
8. [Der Regressionstest](#8-der-regressionstest)
9. [Auswirkung auf den Rollback](#9-auswirkung-auf-den-rollback)
10. [Was jetzt offen ist](#10-was-jetzt-offen-ist)

---

## 1. Die Idee hinter dem Bug

Stell dir eine Kasse vor, an der zwei Beträge übergeben werden:

```
Kasse bekommt:      8,00 DFI   +   92.233.720.360,55 DFI
Summe:                       92.233.720.368,55 DFI
```

Die Kasse hat aber nur **8,00 DFI** in der Kasse. Bei der Addition läuft der
Zähler über (die Summe ist größer als das, was ein 64-Bit-Zähler fassen kann)
— die Kasse merkt es, meldet „Überlauf“ und **bricht ab, ohne zu rechnen**.

Das Problem: die Kasse meldet den Überlauf zwar — aber **die Meldung wird
verworfen**. Die Kasse rechnet mit dem, was sie zuletzt hatte (**8,00 DFI**),
und der Kunde bekommt trotzdem die **vollen 92 Mrd. DFI** ausgehändigt.

```
  Quelle (Quellkonto)  -8,00 DFI   ← nur das wird abgezogen
  Empfänger A          +8,00 DFI
  Empfänger B   +92.233.720.360,55 DFI   ← aus dem Nichts
```

---

## 2. Der Code-Pfad

Der Weg einer `AccountToAccount`-Transaktion durch die Chain:

```
  tx vom Netz
     │
     ▼
  ProcessCustomTx()                       src/dfi/validation.cpp
     │
     ▼
  CAccountsConsensus::operator()(CAccountToAccountMessage)
                                          src/dfi/consensus/accounts.cpp:73
     │
     ├──► HasAuth(from)                  "darf der Absender das?"
     │
     ├──► SubBalanceDelShares(from, SumAllTransfers(to))   ← ABZUG
     │         │
     │         └──► SumAllTransfers()    src/dfi/balances.h:265  ★ HIER IST DER BUG
     │
     └──► AddBalancesSetShares(to)                       ← GUTSCHRIFT
```

Der entscheidende Punkt: **Abzug und Gutschrift verwenden unterschiedliche
Zahlen.**

- Beim **Abzug** wird die *Summe* der Transfers benutzt (also: wie viel
  insgesamt bewegt werden soll).
- Bei der **Gutschrift** wird jeder Empfänger **einzeln** mit seinem
  vollen Betrag bedient.

Wenn die Summe kleiner ist als die Summe der Einzelbeträge, entsteht die
Differenz — und die Differenz ist **neu geschaffenes Geld**.

---

## 3. Der Fehler im Detail

### 3.1 Die Summe wird abgeschnitten

`src/dfi/balances.h:265`

```cpp
inline CBalances SumAllTransfers(const CAccounts &to) {
    CBalances sum;
    for (const auto &kv : to) {
        sum.AddBalances(kv.second.balances);   // ★ Rückgabewert wird IGNORIERT
    }
    return sum;
}
```

`AddBalances` (`src/dfi/balances.h:83`) gibt bei einem Fehler `Err("overflow")`
zurück — aber `SumAllTransfers` schaut **nicht hin**.

### 3.2 Warum die Kasse abbricht, aber der Betrag stehen bleibt

`src/amount.h:147` → `CTokenAmount::Add` → `SafeAdd` (`src/amount.h:96`):

```cpp
inline ResVal<CAmount> SafeAdd(CAmount _a, CAmount _b) {
    if (_a < 0 || _b < 0) return Res::Err("negative amount");

    // convert to unsigned, because signed overflow is UB
    const auto a = static_cast<uint64_t>(_a);
    const auto b = static_cast<uint64_t>(_b);
    const auto sum = a + b;

    if ((sum - a) != b || static_cast<uint64_t>(std::numeric_limits<CAmount>::max()) < sum) {
        return Res::Err("overflow");            // ★ Überlauf wird erkannt
    }
    return {(CAmount) sum, Res::Ok()};
}
```

`SafeAdd` macht seine Arbeit **vorbildlich**: es benutzt `uint64_t`, damit
signed overflow (undefined behaviour) nicht passiert, und meldet den
Überlauf.

Und `CBalances::Add` bricht **vor** der Mutation ab:

```cpp
Res Add(CTokenAmount amount) {
    ...
    auto current = CTokenAmount{amount.nTokenId, balances[amount.nTokenId]};
    if (auto res = current.Add(amount.nValue); !res) {
        return res;                    // ★ balances bleibt UNVERÄNDERT
    }
    balances[amount.nTokenId] = current.nValue;
    return Res::Ok();
}
```

**Der Fehler ist also nicht im Addieren.** Er ist darin, dass niemand auf
den Fehler hört.

### 3.3 Die konkreten Zahlen

Aus Exploit #3 (`bc2e5a68…`, Block 6.566.081):

| | Betrag in satoshi | Betrag in DFI |
|---|---:|---:|
| Empfänger A | 800.000.001 | 8,00000001 |
| Empfänger B | 9.223.372.036.054.775.807 | 92.233.720.360,54775801 |
| **Summe** | **9.223.372.036.854.775.808** | **92.233.720.368,54775808** |
| `INT64_MAX` | 9.223.372.036.854.775.807 | 92.233.720.368,54775807 |

```
  Summe = INT64_MAX + 1   →  Überlauf!

  Add(Empfänger A):   0 + 800.000.001              = 800.000.001        ✓ passt
  Add(Empfänger B):   800.000.001 + 9.223.372.036.054.775.807
                    = 9.223.372.036.854.775.808  ✗ Überlauf → Abbruch
                      → balances bleibt bei 800.000.001 stehen
                      → Fehler wird ignoriert

  SumAllTransfers() liefert: 800.000.001   (= 8,00000001 DFI)
```

Der Abszug prüft also **8,00000001 DFI** — die vorhandenen. Die Gutschrift
zahlt **92.233.720.360,54775801 DFI** aus.

### 3.4 Warum die Adresse-Reihenfolge zählt

`CAccounts` ist `std::map<CScript, CBalances>` — die Iteration folgt den
**Script-Bytes**, nicht der Reihenfolge im Transaktions-Objekt.

Der Angreifer muss also die Adresse wählen, deren Script **byteweise
kleiner** ist, und ihr den **kleinen** Betrag geben. Nur dann bricht der
Überlauf beim *zweiten* Addieren ab, und die Summe bleibt beim kleinen
Betrag stehen.

Beobachtung am Mainnet — die Skript-Hashes sind byteweise sortiert:

```
Exploit #1:  00148b4c9ee6372a...  → 1 satoshi      (kleiner)
              0014cfce8aa2a9d6...  → INT64_MAX     (größer)

Exploit #3:  00140167258a996c...  → 8,00000001 DFI (kleiner)
              001402fe2af1c5752...  → INT64_MAX-8   (größer)
```

Das ist kein Zufall — der Angreifer hat die Adressen bewusst gewählt
(g grinding).

---

## 4. Was der Angreifer tun musste

| Schritt | Aufwand |
|---|---|
| Guthaben im Quellkonto | **8,00000001 DFI** (≈ 0,01 €) |
| Zwei Empfängeradressen | gratis (Hash-Grinding, Sekunden) |
| Transaktion bauen | **roh** — der RPC-Weg reicht nicht |
| Signieren | normale Wallet-Signatur (Schnorr) |

### Warum „roh“ nötig ist

Über `accounttoaccount` geht es **nicht**:

```
$ defi-cli accounttoaccount <from> '{"<addr>": "92233720360.54775807"}'
error code: -3
error message: <addr>: Invalid amount
```

`ParseFixedPoint` (`src/util/strencodings.cpp:457`) erlaubt höchstens
18 signifikante Stellen:

```cpp
// src/util/strencodings.cpp:532
if (exponent >= 18)
    return false; /* cannot represent values larger than or equal to 10^(18-decimals) */
```

Der Exploit-Betrag braucht 19 Stellen. Der Angreifer hat die Transaktion
also selbst als Hex gebaut und signiert — mit den üblichen Werkzeugen
für rohe Bitcoin-Transaktionen.

---

## 5. Die drei Exploits auf dem Mainnet

Vollständiger Scan des Blocks 6.558.000 bis zum Tip (6.574.200) — **genau
drei** Transaktionen mit dem Overflow-Muster:

| # | Block | Zeit (UTC) | TXID | Akteur → Empfänger |
|---|---|---|---|---|
| **#1** | 6.564.035 | 03.11. 00:13:18 | `c0248c39…` | `df1qqgvcp6…` → `df1qel8g4g…` |
| **#2** | 6.565.374 | 03.11. 12:10:36 | `b2518be0…` | `df1qqtlz4u…` → Selbstgutschriftung |
| **#3** | 6.566.081 | 03.11. 19:02:56 | `bc2e5a68…` | `df1qe9eus7…` → `df1qqtlz4u…` |

* **#1** liegt exakt auf dem Zeitpunkt, der im öffentlichen Telegram-Report
  genannt wird („am 03.11. um 01:13“, Ortszeit = 00:13 UTC) und stammt von
  einem **anderen Akteur** als #3.
* **#2** ist eine Selbstgutschriftung (`from` = `to[0]` = `to[1]`).
* Alle drei nutzen denselben Mechanismus und das Maximum aus `INT64_MAX`.

**Schadenshöhe:** maximal `INT64_MAX` ≈ **92,2 Mrd. DFI pro Transaktion**.
Der Telegram-Report nennt „rund 184 Mrd. DFI“ — das ist **zwei** mal
92,2 Mrd. Der dritte Exploit (12:10 UTC) ist darin vermutlich nicht
enthalten; die genaue Bilanzierung ist noch offen.

---

## 6. Warum die Prüfung nicht ausgelöst hat

Es **gibt** eine Balance-Prüfung — sie ist nur unschuldig:

`src/amount.h:163`

```cpp
Res Sub(CAmount amount) {
    if (amount < 0) return Res::Err("negative amount: %s", ...);
    if (this->nValue < amount) return Res::Err("amount %s is less than %s", ...);
    nValue -= amount;
    return Res::Ok();
}
```

Die Prüfung ist **korrekt**. Sie prüft nur den Betrag, der ihr übergeben
wird — und das ist der **abgeschnittene** Betrag (8 DFI statt 92 Mrd.).

> **Der Bug ist nicht in der Prüfung, sondern darin, dass die Prüfung nie
> die richtige Zahl sieht.**

Die Negativkontrolle im Reproduktionstest belegt das: Identische
Transaktion, aber mit Summe exakt `INT64_MAX` (kein Überlauf) → die Chain
lehnt korrekt ab:

```
AccountToAccountTx: amount 0.00000000 is less than 92233720360.54775807
```

---

## 7. Der Fix

### 7.1 Kernidee

`SumAllTransfers` darf Fehler nicht mehr verschlucken. Statt
`CBalances` muss sie `ResVal<CBalances>` zurückgeben.

```diff
--- a/src/dfi/balances.h
+++ b/src/dfi/balances.h
@@
-inline CBalances SumAllTransfers(const CAccounts &to) {
+inline ResVal<CBalances> SumAllTransfers(const CAccounts &to) {
     CBalances sum;
     for (const auto &kv : to) {
-        sum.AddBalances(kv.second.balances);
+        if (auto res = sum.AddBalances(kv.second.balances); !res) {
+            return res;                 // ★ Überlauf wird jetzt propagiert
+        }
     }
-    return sum;
+    return {sum, Res::Ok()};
 }
```

### 7.2 Die Aufrufstellen

`SumAllTransfers` wird **12-mal** aufgerufen. Konsensus-kritisch sind zwei:

| Datei:Zeile | Kontext | Konsensrelevant? |
|---|---|---|
| `src/dfi/consensus/accounts.cpp:80` | `AccountToAccount` | ★ **JA — die Lücke** |
| `src/dfi/consensus/accounts.cpp:96` | `AnyAccountsToAccounts` | ★ **JA — zweite Lücke** |
| `src/dfi/consensus/accounts.cpp:33` | `UtxosToAccount` | nein (Vergleich schlägt fehl) |
| `src/dfi/consensus/poolpairs.cpp:190` | `AddPoolLiquidity` | nein (zieht echte Beträge ab) |
| `src/dfi/mn_rpc.cpp:109` | Auswahl für Masternode-Transfers | nein |
| `src/dfi/rpc_accounts.cpp` (4×) | RPC-Pfade | nein |
| `src/dfi/rpc_customtx.cpp:196` | RPC-Ausgabe | nein |

### 7.3 Die zweite Lücke: `AnyAccountsToAccounts`

`src/dfi/consensus/accounts.cpp:87`

```cpp
const auto sumFrom = SumAllTransfers(obj.from);   // 8 DFI
const auto sumTo   = SumAllTransfers(obj.to);     // 8 DFI  (Überlauf ignoriert)
if (*sumFrom.val != *sumTo.val)
    return Res::Err("sum of inputs (from) != sum of outputs (to)");
// → beide abgeschnitten auf denselben Wert → Prüfung besteht!
SubBalancesDelShares(obj.from);      // zieht die echten 8 DFI ab
AddBalancesSetShares(obj.to);        // schreibt die vollen 92 Mrd. gut
```

Hier kontrolliert der Angreifer **beide** Seiten und kann die abgeschnittenen
Summen zur Übereinstimmung bringen. Dieselbe Lücke, anderer TX-Typ.

### 7.4 Empfehlung

Der Fix sollte **nicht** nur `SumAllTransfers` härten, sondern die
Overflow-Überprüfung an die Stelle legen, an der das Geld entsteht — also in
`AddBalancesSetShares` bzw. in die Gutschrift. Dann ist die Lücke auch dann
geschlossen, wenn künftig jemand eine neue Codepfad einführt, der
`SumAllTransfers` benutzt.

Zusätzlich empfohlen:

1. **`SafeAdd`/`SafeSub` konsequent verwenden.** `CTokenAmount::Add` nutzt
   `SafeAdd`, `Sub` prüft per Vergleich — beide OK, aber die Prüfungen
   sollten einheitlich über `SafeAdd`/`SafeSub` laufen.
2. **Betragsgrenzen erzwingen.** `MoneyRange()` (`src/amount.h:227`) begrenzt
   auf `MAX_MONEY` = 1,2 Mrd. DFI.

   **Wichtige Feinheit:** `MoneyRange` wird zwar an mehreren Stellen
   geprüft, aber immer nur für **UTXO-Werte** — in `GetValueOut` /
   `GetValuesOut` (`src/primitives/transaction.cpp:106,119`) und in der
   Gebühren-/Input-Prüfung (`src/consensus/tx_verify.cpp:219,240`).
   Die **Account-Balances** werden nirgends gegen `MAX_MONEY` geprüft.
   Deshalb konnte ein Konto auf 92 Mrd. DFI wachsen, ohne dass eine
   Obergrenze griff.

   ⇒ Der Node sollte einen **Account-Saldo** über `MAX_MONEY` ablehnen.
   Dann wäre der Exploit auch ohne den Overflow-Bug unmöglich — und die
   Regel wäre zusätzlich eine Art Gummizugnetz für künftige Bug-Klassen
   derselben Art.

   Der Kommentar an dieser Stelle ist bemerkenswert — er beschreibt genau
   die Bug-Klasse, um die es hier geht (`src/amount.h:218-226`):

   > *„As this sanity check is used by consensus-critical validation code,
   > the exact value of the MAX_MONEY constant is consensus critical; in
   > unusual circumstances like a(nother) overflow bug that allowed for the
   > creation of coins out of thin air modification could lead to a fork.“*

   ⇒ Die DeFiChain-Entwickler kannten die Gefahr bereits. Der `MAX_MONEY`-Check
   greift beim Account-Pfad aber **nicht**, weil er dort nicht aufgerufen wird.
3. **Statische Analyse.** `Res`/`ResVal` sind so gebaut, dass man Fehler
   nicht versehentlich ignorieren kann — aber `SumAllTransfers` hat genau das
   getan. Ein Lint (analog `test/lint/lint-*.sh`) könnte nach
   `Res`-Rückgabewerten suchen, die unbenutzt bleiben.

---

## 8. Der Regressionstest

Der Test existiert bereits und läuft gegen den **unveränderten** Chain-Stand:

```bash
./scripts/cluster-up.sh 20
python3 docs/exploits/2026-11-inflation/repro/reproduce_exploit.py \
    --url "http://defi:defi@127.0.0.1:19654"
```

Erwartete Ausgabe **nach** dem Fix:

```
[*] Summe          : 9223372036854775808 sat  > INT64_MAX -> Overflow
[*] TX akzeptiert: <txid>
  Konto des grossen Empfaengers : 92,233,720,360.54776001 DFI
```

Erwartete Ausgabe **vor** dem Fix (heute, Chain unverändert):

```
[!] sendrawtransaction abgelehnt: AccountToAccountTx: ... overflow
  Konto des grossen Empfaengers : 0,00000000 DFI
```

Für die Upstream-Übernahme sollte daraus ein Test im vorhandenen Harness
werden (`chain/test/functional/`, Muster: `feature_*.py`):

```python
class AccountOverflowTest(DefiTestFramework):
    def set_test_params(self):
        self.num_nodes = 1
        self.setup_clean_chain = True

    def run_test(self):
        # 1. Quellkonto mit 8.00000001 DFI fuellen
        # 2. AccountToAccount mit Summe = INT64_MAX + 1 bauen
        # 3. erwarten: sendrawtransaction schlägt fehl (overflow)
        # 4. Erwartung: Guthaben des Ziels unverändert
```

Der Testfall ist in `docs/exploits/2026-11-inflation/repro/README.md`
vollständig beschrieben (inkl. der drei Serialisierungs-Fallen:
SegWit-Marker, `CTxOut.nTokenId`, Script-Byte-Sortierung).

---

## 9. Auswirkung auf den Rollback

Die Prüfung `CAccountsConsensus` läuft **sowohl** im Mempool-Pfad
(`CheckTx`) **als auch** in der Block-Validierung (`ConnectBlock`):

```
Mempool:  CheckTx      → ProcessCustomTx → CAccountsConsensus
Block:    ConnectBlock → ProcessCustomTx → CAccountsConsensus
```

⇒ Ein Fix an dieser Stelle wirkt in **beiden** Pfaden.

Wird die Chain mit der gepatchten Software neu indexiert (`-reindex`),
werden die Exploit-Blöcke beim Revalidieren **invalid** — die Node
reorganisiert auf den Stand vor dem Exploit. Das entspricht der im
Telegram-Report erwähnten „anderen Einstellung“, die die beiden
Exploit-Blöcke ungültig machen soll.

**Wichtig:** Das ist eine **Konsens-Änderung**. Alle Masternodes müssen
gemeinsam umgestellt werden — ein einzelner Node, der die neue Software
fährt, würde die Blöcke ablehnen und sich vom Netz isolieren.

---

## 10. Was jetzt offen ist

- [ ] Bilanzierung: wo ist das Geld aus Exploit #1/#2 hin? Der Empfänger
      `df1qqtlz4u…` hält 92,2 Mrd. DFI; laut Telegram sind 184 Mrd. im Umlauf.
- [ ] Abgleich mit dem im Telegram-Report erwähnten Patch-Entwurf (ungeprüft).
- [ ] Fix als Patch formalisieren (12 Call-Sites) — **Chain-Code bleibt bis
      dahin unverändert**.
- [ ] Den Fix in einen `test/functional/`-Test gießen und upstream einreichen.
- [ ] Prüfen, ob `lib/ain-dftx` (Rust) dieselbe Logik dupliziert.
- [ ] Rollback-Verfahren mit den Masternodes abstimmen.
- [ ] Disclosure / Koordination mit den Börsen (Bitfine hat den Handel auf
      „HALT“).

---

## Anhang: Referenzen

| Was | Wo |
|---|---|
| Bug | `src/dfi/balances.h:265` (`SumAllTransfers`) |
| Aufrufstelle 1 | `src/dfi/consensus/accounts.cpp:80` |
| Aufrufstelle 2 | `src/dfi/consensus/accounts.cpp:96` |
| Overflow-Erkennung (korrekt) | `src/amount.h:96` (`SafeAdd`) |
| Balance-Prüfung (korrekt) | `src/amount.h:163` (`CTokenAmount::Sub`) |
| RPC-Betragsgrenze | `src/util/strencodings.cpp:457` (`ParseFixedPoint`) |
| Betragsobergrenze | `src/amount.h:226` (`MAX_MONEY`) || Reproduktionstest | `docs/exploits/2026-11-inflation/repro/reproduce_exploit.py` |
| Mainnet-Scanner | `docs/exploits/2026-11-inflation/repro/find_exploit.py` |
| Rohdaten (Block/TX/Dumps) | `docs/exploits/2026-11-inflation/patches/data/` |
| Vollständiges Notebook | `docs/exploits/2026-11-inflation/NOTEBOOK.md` |
| Chain-Stand | `bd09486bb0` (v4.2.5) |
