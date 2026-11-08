# Security Policy

## Melding einer Lücke

Sicherheitslücken im DeFiChain-Node bitte an `security@defichain.com`
melden (laut `SECURITY.md` in `DeFiCh/ain`).

## Bekannte, offene Lücke in v4.2.5

`SumAllTransfers()` in `src/dfi/balances.h` ignoriert den Overflow-Fehler
von `AddBalances()`. Dadurch kann DFI aus dem Nichts erzeugt werden
(Kosten: 8,00000001 DFI pro Transaktion, Ertrag bis ~92,2 Mrd. DFI).

Details: [`0001-dfi-inflation-accounttoaccount.md`](0001-dfi-inflation-accounttoaccount.md)
Fix: [`fix-proposed.diff`](fix-proposed.diff)

Betroffene Versionen: v4.2.5 (`bd09486bb03e`) — Mainnet, Testnet, Regtest.
