# DeFiChain — Fix für die DFI-Inflation (`SumAllTransfers`)

Dieses Repository enthält den vollständigen Fix-Vorschlag für den
Consensus-Bug, der am **03.11.2026** auf dem DeFiChain-Mainnet ausgenutzt
wurde.

Es ist bewusst **kein Fork**. Es hängt an keinem Fork-Netzwerk, an keiner
Organisation und an keiner Upstream-Bestandsgarantie. Wenn `DeFiCh/ain`
oder irgendein Fork davon verschwindet, bleibt dieses Repository lesbar,
prüfbar und anwendbar.

---

## Was passiert ist

`SumAllTransfers()` summiert die Beträge aller Transfer-Zielkonten,
**ignoriert aber den Rückgabewert** von `AddBalances()`.

```cpp
// src/dfi/balances.h (vor dem Fix)
inline CBalances SumAllTransfers(const CAccounts &to) {
    CBalances sum;
    for (const auto &kv : to) {
        sum.AddBalances(kv.second.balances);   // Rückgabewert verworfen!
    }
    return sum;
}
```

Überläuft die Summe `int64_t`, meldet `SafeAdd()` korrekt `Err("overflow")`.
Der Fehler wird verworfen, und die Summe bleibt beim **ersten** Betrag
stehen. Das Quellkonto wird daraufhin nur um die **gekürzte** Summe belastet,
während jedes Zielkonto den **vollen** Betrag gutgeschrieben bekommt. Die
Differenz ist neu erzeugtes DFI.

| | |
|---|---|
| Maximaler Schaden pro Transaktion | `INT64_MAX` ≈ **92,2 Mrd. DFI** |
| Kosten für den Angreifer | **8,00000001 DFI** (≈ 0,01 €) |
| Angriffsvektor | `AccountToAccount` (`'B'`) und `AnyAccountsToAccounts` (`'a'`) |
| Transaktion bauen | roh (Hex + Signatur) — der RPC-Weg kann den Betrag nicht ausdrücken |
| Betroffene Version | v4.2.5 (`bd09486bb03e`), Mainnet / Testnet / Regtest |

---

## Inhalt dieses Repositories

| Datei | Zweck |
|---|---|
| [`fix-proposed.diff`](fix-proposed.diff) | **der Patch** — Diff gegen den Upstream-Stand `bd09486bb03e`, anwendbar mit `git apply` |
| [`FIX.md`](FIX.md) | Fix-Vorschlag (DE): Variantenvergleich, fertige Diffs für alle 12 Call-Sites, Härtung, Regressionstest, Rollback-Risiken |
| [`BUG-ANALYSE.md`](BUG-ANALYSE.md) | Deutsche Analyse mit Code-Weg, Zahlen und Fix-Entwurf |
| [`BUG-REPORT.md`](BUG-REPORT.md) | Ausführlicher englischer Report |
| [`DISCLOSURE.md`](DISCLOSURE.md) | Offenlegungsstand, Klassifizierung, betroffene Versionen |
| [`ISSUE.md`](ISSUE.md) | Kurztext für einen Issue-Tracker (EN) |
| [`0001-dfi-inflation-accounttoaccount.md`](0001-dfi-inflation-accounttoaccount.md) | Issue 0001 (Kern) |
| [`0002-cbalances-sum-ignores-overflow.md`](0002-cbalances-sum-ignores-overflow.md) | Issue 0002 (dieselbe Fehlerklasse, toter Code) |

---

## Anwendung

```bash
git clone https://github.com/DeFiCh/ain.git      # Upstream-Stand bd09486bb03e
cd ain
git checkout bd09486bb03e                       # v4.2.5
git apply /pfad/zu/fix-proposed.diff
```

Betroffene Dateien:

- `src/dfi/balances.h` — `SumAllTransfers()` (Kern)
- `src/dfi/balances.h` — `CBalances::Sum()` (toter Code, Issue 0002)
- 12 Call-Sites, die den Rückgabewert prüfen müssen

Der vollständige Diff-Umfang und die Begründung je Call-Site steht in
[`FIX.md`](FIX.md).

---

## Status

| | |
|---|---|
| Root Cause | gefunden, lokal reproduziert |
| Fix | vorgeschlagen, als Diff vorliegend |
| Upstream-PR | offen (Head: `feature/fix-0001-sum-all-transfers`, Base: `DeFiCh:master`) |
| Lücke im Upstream | **offen** — sofern nicht upstream übernommen, jederzeit erneut ausnutzbar |

---

## Lizenz

Dieses Repository steht unter der **MIT-Lizenz**, wie das Upstream-Projekt
`DeFiCh/ain`. Der beigefügte Patch ist ein Derivat von Upstream-Code und
steht unter dessen Lizenz.
