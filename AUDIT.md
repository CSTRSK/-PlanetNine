# Sicherheits-Audit: Planet9-Coin Smart Contract

**Audit-Datum:** 2026-08-14
**Contract:** `contract.sol` (BSC, SafeMoon-Fork)
**Compiler:** Solidity ^0.6.12
**Audit-Tool:** Slither 0.11.6 (102 Detektoren)
**Auditor:** CSTRSK-Hermes (automatisierte statische Analyse)

---

## Zusammenfassung

| Metrik | Wert |
|--------|------|
| Contracts analysiert | 10 |
| Gesamt-Findings | 45 |
| **High** | **1** (Reentrancy) |
| Medium | 3 |
| Low | 11 |
| Informational | 27 |
| Optimization | 3 |

**Gesamt-Risiko: MITTEL-HOCH** — Ein High-Severity-Finding (Reentrancy) im
Kern-Transfer-Pfad. Der Contract ist ein klassischer SafeMoon-Fork mit
9%-Gebührenmodell (Liquidity + Reflexionen) und wurde 2022 auf BscScan
verifiziert.

---

## Findings

### 🔴 HIGH — Reentrancy in `_transfer` (Zeile 1012-1056)

**Check:** `reentrancy-eth`
**Impact/Confidence:** High/Medium

**Beschreibung:**
`PlanetNine._transfer(address,address,uint256)` macht externe Calls via
`swapAndLiquify(contractTokenBalance)` (Zeile 1043), das wiederum
`uniswapV2Router.addLiquidityETH{value: ethAmount}` (Zeile 1104-1111) aufruft.
Bei jedem Token-Transfer wird ein Teil der Gebühr in ETH gewandelt und
Liquidität hinzugefügt. Der State wird dabei nicht konsistent vor externen
Calls aktualisiert.

**Risiko:**
Ein Angreifer kann während des externen Calls (Uniswap-Router) reentrant in
den Transfer-Pfad eintreten, bevor der State vollständig aktualisiert ist.
Bei Fee-on-Transfer-Token mit Swap-in-Transfer-Muster kann dies zu
Wertverlust oder Manipulation führen.

**Empfehlung:**
- Checks-Effects-Interactions-Muster einhalten: Alle State-Änderungen VOR
  den externen Calls
- `nonReentrant`-Modifier (OpenZeppelin ReentrancyGuard) auf `_transfer`
- Swap/Liquidity-Anteil aus dem Transfer-Pfad in eine separate
  `swapAndLiquify()`-Funktion mit Guard auslagern

---

### 🟠 MEDIUM — Ignorierter Rückgabewert `addLiquidityETH` (Zeile 1104-1111)

**Check:** `unused-return`
**Impact/Confidence:** Medium/Medium

**Beschreibung:**
`PlanetNine.addLiquidity(uint256,uint256)` ignoriert den Rückgabewert von
`uniswapV2Router.addLiquidityETH`. Wenn die Liquiditäts-Transaktion fehlschlägt,
wird der Fehler stillschweigend geschluckt — Token und ETH können im Contract
"stecken bleiben".

**Empfehlung:**
Rückgabewerte prüfen und bei Fehlschlag revertieren (oder Token/ETH
zurückerstatten).

---

### 🟡 LOW — Timestamp-Abhängigkeit

**Check:** `timestamp`
**Impact/Confidence:** Low/Medium

Mehrere Stellen nutzen `block.timestamp` für Zeitlimits. Miner können den
Timestamp in einem kleinen Fenster manipulieren.

---

### 🟡 LOW — Reentrancy-Events (3×)

**Check:** `reentrancy-events`
**Impact/Confidence:** Low/Medium

Events werden nach externen Calls emittiert — Reihenfolge verletzt das
Checks-Effects-Interactions-Muster für Event-Konsistenz.

---

### ⚪ INFORMATIONAL (Auswahl)

- `assembly` (2×) — Inline-Assembly ohne dokumentierte Notwendigkeit
- `low-level-calls` (2×) — direkte `.call`-Nutzung
- `function-init-state` (3×) — Funktionen überschreiben Init-State
- `solc-version` — veralteter Compiler (0.6.12, 2020)
- `dead-code` — ungenutzter Code
- `costly-loop` — ineffiziente Loops (Gas)
- `shadowing-local` (2×) — lokale Variablen überschatten

---

## Methodik

1. Contract-Quellcode aus dem Repository `CSTRSK/-PlanetNine` entnommen
2. Kompilierung mit exakter Compiler-Version (solc 0.6.12)
3. Statische Analyse mit Slither 0.11.6 (102 Detektoren)
4. Findings nach Impact klassifiziert (High/Medium/Low/Informational)

**Limitationen:**
- Reine statische Analyse — keine dynamische Ausführung oder
  Fuzzing durchgeführt
- Manuelle Prüfung der Reentrancy-Ausnutzbarkeit im Kontext des
  Fee-Modells empfohlen
- Kein Exploit-Code im Rahmen dieses Audits erstellt oder ausgeführt

---

## Empfehlung

Der Contract wird als **nicht mehr für Produktion geeignet** eingestuft
(veralteter Compiler, bekannte Fork-Basis, High-Reentrancy-Finding).
Für Weiterentwicklung: Neuaufbau mit modernem Standard (OpenZeppelin
ERC20, ReentrancyGuard, solc >= 0.8 mit SafeMath-integriert).

---

*Audit erstellt mit bsc-bounty-scanner (Slither 0.11.6) — nur statische
Erkennung, kein Exploit-Code. Für Bug-Bounty-Einreichungen: Scope vorher prüfen.*
