# 0002 — `CBalances::Sum()` ignoriert denselben Overflow-Fehler

| | |
|---|---|
| **Schweregrad** | mittel (aktuell **kein** Exploit — toter Code, aber Landmine) |
| **Status** | offen |
| **Herkunft** | **Nebenbefund** aus [0001](0001-dfi-inflation-accounttoaccount.md) bzw. der [DFI-Inflation-Ermittlung](../exploits/2026-11-inflation/) |
| **Betroffen** | DeFiChain v4.2.5 (`bd09486bb03e`) |
| **Code-Stelle** | `src/dfi/balances.h:104` (`CBalances::Sum`) |
| **Auswirkung** | derzeit keine — die Funktion wird nirgends aufgerufen |
| **Aufwand** | ~6 Zeilen (mitfixen) bzw. 2 Zeilen (löschen) |

## Beschreibung

Beim Prüfen aller Aufrufstellen von `SumAllTransfers()` (für den Fix von
[0001](0001-dfi-inflation-accounttoaccount.md)) fiel auf, dass `CBalances`
**eine zweite Summenfunktion mit exakt demselben Fehlermuster** hat:

```cpp
// src/dfi/balances.h:104
static CBalances Sum(const std::vector<CTokenAmount> &tokens) {
    CBalances res;
    for (const auto &token : tokens) {
        res.Add(token);            // ← Rückgabewert wird ignoriert
    }
    return res;
}
```

`CBalances::Add()` meldet bei Überlauf `Err("overflow")` — der Rückgabewert
wird hier genauso weggeworfen wie in `SumAllTransfers()`. Das Verhalten ist
damit identisch mit dem Auslöser des [Exploits vom 03.11.2026](../exploits/2026-11-inflation/):

> Die Summe bleibt beim ersten Betrag stehen, während alle Beträge einzeln
> weiterverrechnet werden — die Differenz ist erzeugtes Geld.

## Warum es trotzdem dokumentiert werden muss

**Aktuell kein Exploit:** Ich habe alle Verwendung von `CBalances::Sum` und von
`CStatsTokenBalances` geprüft — **beide werden nirgends aufgerufen**. Es ist
toter Code im Header. Eine dritte ausnutzbare Lücke gibt es also nicht.

**Trotzdem relevant, aus zwei Gründen:**

1. **Landmine.** Die Funktion ist öffentlich (`static` in einem Header, also
   überall nutzbar) und sieht harmlos aus. Wer sie in einem Jahr erstmals
   verwendet, baut die Lücke des 03.11.2026 unbemerkt wieder ein — mit
   derselben Mechanik, nur an anderer Stelle.
2. **Symmetrie.** Ein Fix, der nur `SumAllTransfers()` härten, aber die
   gleichartige Schwesterfunktion stehen lässt, ist unvollständig. Die
   Fehlerklasse soll im Code nicht mehr vorkommen.

## Betroffene Nachbarn

| Funktion | Ort | Fehlermuster | Verwendung |
|---|---|---|---|
| `CBalances::Sum` | `balances.h:104` | ja (`res.Add(token)` ignoriert) | **keine** |
| `CStatsTokenBalances::Sum` | `balances.h:212` | ja (dito) | **keine** |
| `CStatsTokenBalances::Add` | `balances.h:160` | ja (`balances[id] += amount` ohne `SafeAdd`) | **keine** |
| `CBalances::Add` | `balances.h:18` | nein — prüft `current.Add(...)` ✅ | viele |

## Empfohlene Lösung

Zwei Varianten:

**A — mitfixen** (gleiche Fehler-Propagierung wie in 0001):

```diff
-static CBalances Sum(const std::vector<CTokenAmount> &tokens) {
-    CBalances res;
-    for (const auto &token : tokens) {
-        res.Add(token);
-    }
-    return res;
+static ResVal<CBalances> Sum(const std::vector<CTokenAmount> &tokens) {
+    CBalances res;
+    for (const auto &token : tokens) {
+        if (auto res2 = res.Add(token); !res2) {
+            return res2;
+        }
+    }
+    return {res, Res::Ok()};
 }
```

**B — löschen.** Da die Funktion (und `CStatsTokenBalances` insgesamt) nirgends
verwendet wird, wäre das Entfernen die einfachste und robusteste Lösung: Das
Fehlermuster existiert dann nicht mehr im Code.

Empfehlung: **B** — toten Code entfernen, der genau die Lücke enthält, die wir
schließen. Falls die Funktion künftig gebraucht wird, ist sie in der
Fehler-Propagierungs-Form neu zu schreiben (Muster aus 0001).

## Verifikation

| Schritt | Erwartung |
|---|---|
| `grep -rn "CBalances::Sum\|CStatsTokenBalances" src/` | keine Verwendung (bestätigt) |
| Build | grün |
| `make chain-test-py` | grün |

## Verweise

- Ursprung: [0001](0001-dfi-inflation-accounttoaccount.md) — gleiche
  Fehlerklasse, anderer Ort
- [FIX.md](../exploits/2026-11-inflation/FIX.md) §2.2 — Variantenvergleich
- [BUG-ANALYSE.md](../exploits/2026-11-inflation/BUG-ANALYSE.md) §3.1 — die
  Mechanik des verschluckten Fehlers
