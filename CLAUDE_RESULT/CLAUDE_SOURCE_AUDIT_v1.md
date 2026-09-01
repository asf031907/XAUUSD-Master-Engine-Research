# 1. SOURCE AUDIT TABLE

| Source | Original Role | Master Engine Role | Preserve As-Is? | Modification Required? | Risk | Implementation Note |
|---|---|---|---|---|---|---|
| **CISD Pro+** | Standalone chart-TF structure-shift detector using consecutive same-direction candle **open-price leg extremes** (not wicks, not pivots) + `market_bias` flip state machine. Confirmation = close crossing the stored `cisd_level`. Has independent HTF-FVG and session-H/L confluence gates. | CISD Engine (Module 10) | Partially | Yes — must strip session/FVG confluence gating from the *detection* logic (that belongs in POI/Regime engines) and expose raw `CISD_BULL/CISD_BEAR/NONE` + level | **HIGH** — this is the *only* dedicated CISD source, and its definition (open-price leg extreme, not wick/pivot) is idiosyncratic. Do not conflate with generic "MSS" definitions. | This is the authoritative CISD definition per spec §13. CRT's internal "CISD" logic (below) is a *different* algorithm and must NOT be merged with this one. |
| **CRT Pro+** | HTF range-box model (Model 1: sweep + OB; Model 2: sweep + reclaim-through-`manipOpen`). Includes its own SMT engine (2 independent divergence detectors) and its own sweep/D-Purge logic. | CRT Engine (Module 08) + partial source for SMT Engine (Module 16) | Partially | Yes — the two Models (1 vs 2) are mutually exclusive UX toggles in the source; Master Engine needs both concepts reconciled (OB from Model 1, reclaim-level from Model 2) or an explicit decision on which to inherit. | **HIGH** — CRT's "CISD" label (`f_cisdLevel`, `manipOpen`) is a *different algorithm* from CISD Pro+ (body-extreme-of-consecutive-candles-after-sweep vs. leg-extreme-open-price). Naming collision risk is severe. | Map CRT's `manipOpen`/reclaim logic to **CRT Engine's own reclaim/manipulation state** (spec §11, not §13's CISD Engine). Do not let this "CISD" name bleed into Module 10. |
| **IFVG Pro+** | Inversion-FVG detector: normal FVG invalidated by an opposite close becomes a POI. Includes independent HTF-sweep detector, BE/RR trade lifecycle, and a `hideLosing` toggle (**default `true`**). | IFVG Engine (Module 14) | Mostly | Yes — **`hideLosing` must default OFF / be structurally excluded** from the research engine per spec §17 & §40.3. This is a direct conflict as-shipped. | **CRITICAL** — shipped default violates the Master Engine's anti-survivorship-bias rule outright. | Flag explicitly to user (done below, §5). Preserve `Series` vs `Single` mode and `requireBEIntact` exactly as coded — these are non-trivial, source-specific rules. |
| **Unicorn Model** | Sweep (of one of 4 configurable HTF liquidity sources) → breaker block (≥2 opposite candles) → optional FVG-overlap requirement → RR-multiple targets. No CISD/MSS concept exists in this file at all. | Unicorn Engine (Module 15) | Yes | Minor — only naming/namespacing for merge | **MEDIUM** — sweep detection here is *level-touch* based (intrabar, not close-confirmed), distinct from every other file's sweep definition. | Do not let Unicorn's sweep infra get silently unified with CRT's or Turtle Soup's sweep infra — spec §18 already warns against reducing Unicorn to "FVG + Breaker." |
| **ICT Turtle Soup Model** | Chart-TF pivot sweep → MSS (via separate smaller pivots) → zone confluence (FVG/iFVG) → **inner-sweep retest** → entry → SD-multiple targets. Has its own D-Purge (extend-once) and MSS engine, distinct from CRT's D-Purge (simultaneous high+low sweep in one HTF candle). | Turtle Soup Engine (Module 21, "alternative setup model" per spec) + partial source for MSS Engine (Module 11) | Mostly | Minor | **MEDIUM** — contains a source-level bug/ambiguity: `m.status` is set back to `"Formation"` after entry confirmation, reusing the same string as the pre-sweep default state (see §5 below). | Do not silently "fix" the status overload — flag it and ask whether to preserve the literal (broken) behavior or treat it as an acknowledged defect to correct in synthesis. |

---

# 2. SOURCE → MASTER ENGINE MAPPING

| Master Engine Module | Primary Source(s) | Secondary/Contributing Source(s) | Notes |
|---|---|---|---|
| 04 1W Context Engine | *(none)* | — | No source defines macro context. Pure synthesis — needs explicit rule set. |
| 05 1D Context Engine | *(none)* | — | Same as above. |
| 06 Liquidity Engine | Unicorn (`HTFLevel` registration, 4 sources) | CRT (HTF range H/L), Turtle Soup (pivot arrays), IFVG (HTF swing points) | **Four incompatible liquidity representations** exist. Must pick one canonical liquidity object model — see Ambiguity #1. |
| 07 Sweep Engine | Turtle Soup (pivot breach+close-back) | CRT (HTF-candle breach+close-back), Unicorn (level touch, intrabar), IFVG (HTF swing breach) | Each source defines "sweep" differently (see Ambiguity #2) — **not a drop-in unification.** |
| 08 CRT Engine | CRT Pro+ | — | Preserve Model 1 (OB) and Model 2 (reclaim) as source-faithful sub-behaviors. |
| 09 Reclaim Engine | CRT Pro+ (`manipOpen` reclaim) | Turtle Soup (inner-sweep retest is conceptually a reclaim-style wait state) | These are NOT the same mechanism — Turtle Soup's retest targets a zone; CRT's reclaim targets a level. |
| 10 CISD Engine | CISD Pro+ | — | Authoritative source. CRT's internal "CISD" naming is excluded from this module (routed to Module 08/09 instead). |
| 11 MSS Engine | Turtle Soup (`f_checkMSS`, pivot-based) | — | Only real dedicated MSS implementation in the corpus. |
| 12 Displacement Engine | *(none)* | — | **No source implements this.** Pure synthesis — flagged as unresolved. |
| 13 FVG Engine | IFVG (base FVG detection embedded before inversion), Unicorn (`f_findBullishFVG`/`f_findBearishFVG`), Turtle Soup (`f_findFVG`) | CISD Pro+ (`detectFVG`, HTF only) | **Four separate FVG detection implementations**, each with different mitigation/gap-size rules. Must reconcile without erasing distinctions (spec §16). |
| 14 IFVG Engine | IFVG Pro+ | — | Source-faithful; note `hideLosing` conflict. |
| 15 Unicorn Engine | Unicorn Model | — | Source-faithful. |
| 16 SMT Engine | CRT Pro+ (`getSMTPair`, HTF-divergence + chart-pivot-divergence) | — | Only SMT implementation in the corpus. It bundles **two distinct SMT triggers** into one array — must decide whether Master Engine treats them as one signal or two. |
| 17 Fractal/Structural Reference Engine | *(none, dedicated)* | Turtle Soup's swing-pivot arrays, Unicorn's swing detection | No source builds a "fractal reference" layer distinct from pivots already used contextually — synthesis required. |
| 18 Regime/Session Engine | Turtle Soup (killzone shading + `allowNewModel` gate) | CISD Pro+ (session H/L), IFVG (alert sessions), Unicorn (time filter) | Session window values **disagree slightly** across files vs. the Master Engine spec (see Ambiguity #4). |
| 21 Turtle Soup (alt model) | ICT Turtle Soup Model | — | Preserve as an isolated, non-causal-chain alternative model per spec §21. |

---

# 3. MASTER ENGINE ARCHITECTURE DIAGRAM

```
                         ┌────────────────────────┐
                         │   1W CONTEXT ENGINE      │  (Synthesis — no source)
                         └───────────┬─────────────┘
                                     ▼
                         ┌────────────────────────┐
                         │   1D CONTEXT ENGINE      │  (Synthesis — no source)
                         └───────────┬─────────────┘
                                     ▼
      ┌─────────────────────────────────────────────────────────┐
      │                 4H LIQUIDITY ENGINE                       │
      │  Source candidates: Unicorn HTFLevel / CRT range H-L /     │
      │  Turtle Soup pivots / IFVG HTF swings  (UNRESOLVED — pick  │
      │  canonical model, see Ambiguity #1)                        │
      └───────────────────────────┬────────────────────────────┘
                                   ▼
      ┌─────────────────────────────────────────────────────────┐
      │                   SWEEP ENGINE (4H)                       │
      │  4 incompatible sweep definitions in source — must select  │
      │  or explicitly parameterize per timeframe (Ambiguity #2)   │
      └───────────────────────────┬────────────────────────────┘
                                   ▼
      ┌─────────────────────────────────────────────────────────┐
      │  CRT ENGINE (source: CRT Pro+)                             │
      │   ├─ Model 1: Sweep → Order Block                          │
      │   └─ Model 2: Sweep → manipOpen (reclaim level)             │
      └───────────────────────────┬────────────────────────────┘
                                   ▼
      ┌─────────────────────────────────────────────────────────┐
      │  RECLAIM ENGINE                                            │
      │   CRT manipOpen-reclaim  (source-derived, CRT-specific)     │
      └───────────────────────────┬────────────────────────────┘
                                   ▼
      ┌─────────────────────────────────────────────────────────┐
      │                    1H STRUCTURE LAYER                      │
      │  ┌─────────────────────┐   ┌─────────────────────────┐   │
      │  │  CISD ENGINE          │   │   MSS ENGINE              │   │
      │  │  source: CISD Pro+    │   │   source: Turtle Soup     │   │
      │  │  (leg-extreme/open)   │   │   (pivot break)           │   │
      │  └──────────┬────────────┘   └───────────┬───────────────┘   │
      │             └───────────┬────────────────┘                   │
      │                         ▼                                    │
      │              STRUCTURE_CONFIRMED (normalized, but with        │
      │              CISD/MSS identity preserved for diagnostics)     │
      └───────────────────────────┬────────────────────────────┘
                                   ▼
      ┌─────────────────────────────────────────────────────────┐
      │              DISPLACEMENT ENGINE (Synthesis — no source)   │
      └───────────────────────────┬────────────────────────────┘
                                   ▼
      ┌─────────────────────────────────────────────────────────┐
      │                    POI LAYER (1H/30M)                      │
      │  ┌───────────┐  ┌───────────┐  ┌────────────────────┐    │
      │  │FVG Engine  │  │IFVG Engine │  │  Unicorn Engine     │    │
      │  │(4 variants)│  │(IFVG Pro+) │  │  (Unicorn Model)    │    │
      │  └───────────┘  └───────────┘  └────────────────────┘    │
      └───────────────────────────┬────────────────────────────┘
                                   ▼
      ┌─────────────────────────────────────────────────────────┐
      │   SMT ENGINE (XAUUSD ↔ XAGUSD)  — source: CRT Pro+         │
      │   Two bundled trigger types — VALIDATION ONLY, not entry    │
      └───────────────────────────┬────────────────────────────┘
                                   ▼
      ┌─────────────────────────────────────────────────────────┐
      │   REGIME / SESSION ENGINE (source: Turtle Soup primary,     │
      │   cross-checked against CISD/IFVG/Unicorn session inputs)   │
      └───────────────────────────┬────────────────────────────┘
                                   ▼
      ┌─────────────────────────────────────────────────────────┐
      │                30M EXECUTION ENGINE                        │
      └───────────────────────────┬────────────────────────────┘
                                   ▼
      ┌─────────────────────────────────────────────────────────┐
      │                    SIGNAL ENGINE → ENTRY                   │
      └─────────────────────────────────────────────────────────┘

   PARALLEL / NON-CAUSAL-CHAIN BRANCH:
      ┌─────────────────────────────────────────────────────────┐
      │  TURTLE SOUP MODEL (alternative setup, isolated)            │
      │  source: ICT Turtle Soup Model — own sweep/MSS/D-Purge/      │
      │  inner-sweep-retest/SD-target lifecycle, fed by its own      │
      │  swing-pivot infra, NOT merged into the main gate chain.      │
      └─────────────────────────────────────────────────────────┘
```

---

# 4. STATE MACHINE

**Per-setup lifecycle (Master Engine synthesis, gated per spec §27):**

```
CONTEXT_UNVALIDATED
   │ (1W/1D bias check)
   ▼
CONTEXT_VALID
   │ (Liquidity Engine identifies qualified level)
   ▼
LIQUIDITY_IDENTIFIED
   │ (Sweep Engine: source-specific breach+reaction test — timeframe dependent)
   ▼
NO_SWEEP ──────────────► (loop / re-evaluate)
   │
   ▼ (sweep confirmed)
SWEPT
   │ (Reclaim Engine: WAIT_RECLAIM)
   ▼
RECLAIM_FAILED ─────────► INVALIDATED
   │
   ▼ (RECLAIM_CONFIRMED)
STRUCTURE_PENDING
   │ (CISD Engine OR MSS Engine — independently tracked, first to confirm wins,
   │  both logged if both fire)
   ▼
STRUCTURE_CONFIRMED (CISD_BULL/BEAR or MSS_BULL/BEAR — identity preserved)
   │ (Displacement Engine — NONE/WEAK/VALID/STRONG)
   ▼
DISPLACEMENT_CHECK
   │  NONE/WEAK → WAIT (no auto-invalidate, but blocks progression per gate policy)
   ▼ (VALID/STRONG)
POI_SEARCH (FVG / IFVG / Unicorn — independently identified, not merged)
   │
   ▼ (POI found)
POI_IDENTIFIED
   │ (SMT Engine — VALIDATION only)
   ▼
SMT_CHECKED (ALIGNED / NEUTRAL / CONTRADICTORY — contradictory may block per config)
   │
   ▼
REGIME_CHECK (session/killzone permission)
   │
   ▼ (30M confirmation)
EXECUTION_READY
   │
   ▼
LONG_ENTRY / SHORT_ENTRY
   │
   ▼
(TP hit → EXPIRED_SUCCESS) or (SL hit → INVALIDATED)
```

**Note:** Two of the source models (Unicorn, IFVG) implement their **own internal, self-contained state machines** with a different shape (no separate reclaim-wait; confirmation is instantaneous on same-bar breaker/inversion break). These do **not** map cleanly onto the causal chain above and must run as parallel POI-generation sub-machines, not be forced into it — consistent with spec §18's "do not reduce Unicorn" warning.

---

# 5. UNRESOLVED AMBIGUITIES

These require your decision before implementation. I am not guessing at any of them.

1. **Liquidity object model conflict.** Four sources build "liquidity levels" four different ways (Unicorn: 4 configurable HTF sources w/ OHLC+Swing; CRT: HTF range H/L only; Turtle Soup: chart-TF pivot arrays; IFVG: HTF-aggregated swing highs/lows via 3-bar midpoint check). The Master Engine's 4H Liquidity Engine (spec §9) needs ONE canonical liquidity object. **Which source's liquidity model is authoritative for the 4H Liquidity Engine, or do all four coexist as independently-tagged liquidity types?**

2. **Sweep definition conflict.** No two source files define "sweep" identically:
   - CRT: HTF-candle close-confirmed (or realtime breach+close-back)
   - Unicorn: intrabar level touch (no close confirmation)
   - Turtle Soup: pivot breach + close back inside, chart TF
   - IFVG: HTF-aggregated swing breach, chart TF
   The spec's Sweep Engine (§10) implies ONE sweep concept feeding the causal chain. **Does the 4H Sweep Engine use CRT's HTF-candle-close method (since CRT is the designated 4H module), while Unicorn/IFVG/Turtle Soup's sweep methods are preserved only within their own isolated engines?** This is my working assumption but needs your confirmation — I have not implemented it.

3. **CISD naming collision.** CISD Pro+'s CISD (open-price leg-extreme) and CRT Pro+'s internal "CISD" (`manipOpen`, consecutive-candle body extreme after sweep) are genuinely different algorithms sharing the same name in source. Per spec §13/§39, source-derived logic must remain traceable and not silently merged. **Confirmed reading: CRT's internal "CISD" routes to the CRT/Reclaim Engine (§11/§9), NOT the dedicated CISD Engine (§13), which is exclusively CISD Pro+'s algorithm. Please confirm this is the intended interpretation** — the alternative is that you intend CRT's version to be the CISD Engine's source, which would mean discarding CISD Pro+.

4. **IFVG `hideLosing` default.** Shipped default is `true`, which flips visibility of invalidated setups. This is a direct violation of spec §17/§40.3 ("Historical losing setups must NEVER be hidden"). **Confirming: this input must be forced OFF (or removed entirely) in the research/debug engine, correct?** I will not silently change source behavior without this confirmation, per the Development Protocol.

5. **Turtle Soup status-string overload.** After entry confirmation (`f_checkInnerSweep`), the source sets `m.status := "Formation"` — the identical string used for the pre-sweep, no-model-yet state. This appears to be a source-level defect (not an intentional design choice), since it collapses two semantically distinct states into one label, breaking `f_shouldShow`'s "Formation Only" filter logic. **Do you want this literal (arguably buggy) behavior preserved as-is per source fidelity, or should it be flagged as a defect and corrected (with the correction documented separately from source-derived logic, per spec §39)?**

6. **SMT dual-trigger bundling.** CRT's SMT engine pushes to the same `smtArray` from two structurally different detectors: (a) HTF-close-based divergence (`bearD_htf`/`bullD_htf`) and (b) chart-TF pivot-based divergence (`chartPivDir` w/ `ta.pivothigh/low(3,3)`). Spec §19 wants SMT states (`BULLISH_SMT`/`BEARISH_SMT`/`NEUTRAL`/`CONTRADICTORY`) as a single validation signal. **Should these two triggers be merged into one SMT state (first/either fires), or exposed as two independently-traceable SMT sub-signals** (consistent with the "traceability" principle elsewhere in the spec)?

7. **No source defines Displacement, 1W Context, 1D Context, or the Fractal/Structural Reference Engine.** These are pure Master Engine synthesis per spec §15/§7/§8/§20. Per the Development Protocol I will not invent these silently — I need your parameters (e.g., is 1D/1W context based on close-vs-prior-close, HH/HL structure, EMA slope, or something else you have in mind? Is Displacement ATR-relative body size, FVG-creation-force, or something else?).

8. **Session window discrepancies.** Master spec (§22) defines: Asia 19:00–22:00, London 02:00–05:00, NY 07:00–10:00, London Close 10:00–12:00 (America/New_York). Turtle Soup matches this exactly. IFVG's NY session defaults to **07:00–09:00** (not 07:00–10:00), and its window is separately labeled "NY" without the "AM" distinction Turtle Soup uses for `i_nyAmSess` (0700-1000) vs `i_nyPmSess` (1330-1600). **Confirm the Master Engine Regime Engine uses the spec's exact windows, and source files' own default session values are irrelevant / superseded**, not blended.

9. **Reconciling four independent FVG implementations** (IFVG's own embedded FVG check, Unicorn's `f_findBullishFVG`/`f_findBearishFVG` with ATR-size filtering + mitigation-by-touch, Turtle Soup's `f_findFVG` with no ATR filter, CISD Pro+'s `detectFVG` which is HTF-only). The Master Engine FVG Engine (§16) needs a canonical FVG detector distinct from IFVG's inversion logic. **Which of these is the base FVG Engine's source, or does each POI sub-engine keep its own embedded FVG detector** (which would mean "FVG Engine" as a standalone module doesn't really exist independently)?

10. **CRT Model 1 vs Model 2 coexistence.** These are mutually-exclusive UI toggles in source (`modelMode`), producing structurally different outputs (OB line only vs. reclaim/CISD-style level). **Does the Master Engine CRT module need to run both models simultaneously (as independently-tagged CRT sub-types), or does the Master Engine standardize on one (and if so, which)?**

---

# 6. REPAINT / LOOKAHEAD RISKS

| # | Location | Risk Description | Severity |
|---|---|---|---|
| 1 | CRT Pro+, SMT block (`bearD_htf`/`bullD_htf` computed from live `request.security` HTF OHLC without an explicit `[1]`-confirmed offset on the *current* forming HTF bar) | The divergence flags are recalculated every chart bar including intrabar of an unclosed HTF period. The `newPeriod_smt` gate uses `bearD_htf[1]`, which should reference the just-closed period's final value — but this needs explicit non-repaint audit (confirmed-bar boundary verification), not an assumption. | **HIGH — must audit before porting** |
| 2 | CRT Pro+, Model 1 realtime sweep/OB detection (`rtSweptHigh`, `rtSweptLow`, `rtObLine`) | Explicitly intrabar — no `barstate.isconfirmed` gate. Signal state changes tick-by-tick until the HTF candle closes. If the Master Engine's Execution/Signal Engine consumes this without gating, historical vs. realtime behavior will diverge (violates spec §34's hard requirement). | **HIGH** |
| 3 | Unicorn Model, `f_getHTFData()` (`isSwingHigh`/`isSwingLow` computed inside `request.security`, then `ta.valuewhen`) | Swing point flags are evaluated on the *current* (potentially still-forming) HTF bar's `high`/`close` inside the security expression. If the referenced HTF bar hasn't closed, this is a provisional/repainting swing designation. | **HIGH** |
| 4 | IFVG Pro+, `ict_TrackLine` pivot-origin detection (`ta.lowestbars(3)==-1` / `ta.highestbars(3)==-1`) | This is a same-bar 3-window check (not an offset pivot with right-side confirmation bars), meaning the "pivot" label attaches with only 1-bar lag rather than a confirmed N-bar pivot. Functionally low risk (it's consistent bar-to-bar) but must be explicitly documented as a lagging-approximation, not a true confirmed pivot, to avoid confusion with Turtle Soup's proper `ta.pivothigh/low`. | **MEDIUM** |
| 5 | IFVG Pro+ / Unicorn / CRT — all box/line objects updated every bar including live/intrabar bars | Visual objects reposition intrabar before candle close. Not a signal-integrity repaint per se, but must be separated from Debug Mode's decision-log (spec §31) which must reflect only confirmed-bar state. | **MEDIUM (visual only, if isolated correctly)** |
| 6 | CISD Pro+, `detectFVG()` gated by `barstate.isconfirmed` inside the function passed to `request.security` | This is actually a *good* pattern (confirms HTF bar before FVG registration) — flagged here only to confirm it as the standard to replicate elsewhere, since it's inconsistent with how Unicorn/CRT handle their own HTF fetches. | **LOW / reference-pattern only** |
| 7 | Turtle Soup, HTF candle panel projection scaling (`i_htfRangeLen`, `liveRange`) | Uses `ta.sma` of live chart range for visual scaling only — does not touch signal logic. No repaint risk to decision-making, purely cosmetic. | **NONE (confirmed cosmetic)** |
| 8 | Cross-cutting: All 5 source files run their full detection logic on every bar without a unified "only re-evaluate on confirmed bar close" wrapper | When merged into one script covering 1W/1D/4H/1H/30M simultaneously, inconsistent confirmation-gating between modules could produce a Master Engine signal that is internally inconsistent (e.g., 4H sweep still forming while 1H structure is already confirmed against a still-moving 4H reference). This must be resolved architecturally, not per-module. | **HIGH — architectural, not cosmetic** |

**Audit obligation carried forward:** Per spec §34, before any code is declared complete I will explicitly re-audit **Historical vs. Realtime vs. Reloaded-chart** behavior for every flagged item above once implemented.

---

# 7. PINE SCRIPT TECHNICAL RISKS

1. **Type name collisions on merge.** Source files independently declare `Trade` (IFVG), `Setup` (Unicorn), `TSModel` (Turtle Soup), `CRTModel`/`Candle`/`SMT`/`Settings`/`CandleSettings`/`CandleSet` (CRT), `FVG`/`cisdSetup`/`kz`/`kz_helper` (CISD). Pine v6 requires unique UDT names per script — **every type must be namespaced** (e.g., `ifvg_Trade`, `uni_Setup`, `ts_Model`) before merge. This is mechanical but must be done exhaustively and consistently or compilation fails.

2. **Function name collisions.** `f_lineStyle`, `f_labelSize`, `f_tablePos`/`f_dashPos`, `autoHTF`/`getAutoHTFs`/`getHTFPeriod` (5 different auto-TF schemes across 4 files) all need namespacing. Behaviorally distinct functions with the same name across sources must not be accidentally deduplicated into one — that would silently alter source behavior (violates spec §2/§39).

3. **`request.security` call budget.** Rough count if all 5 modules are ported with their existing security call patterns: CRT (5 calls: htf1 monitor context, SMT pair, SMT chart pivot, 24-value HTF OHLC tuple, +1 for `time_close`), Unicorn (4 calls, one per liquidity source), Turtle Soup (2 calls, 24–25 values each), CISD Pro+ (2 calls: FVG detection + HTF close). Plus the Master Engine's own explicit 1W/1D/4H/1H layer fetches and the XAGUSD SMT fetch. Pine's per-script `request.*` limit and per-call series-count limits must be checked against this combined load — **this is a real risk of hitting compiler limits**, not a hypothetical one, and needs explicit budgeting in Phase 5.

4. **Object count ceilings.** Each source file independently declares `max_lines_count=500`, `max_boxes_count=500`, `max_labels_count=500`. When merged, these are **script-wide limits, not per-module** — five modules each assuming they get their own 500-object budget will collectively exceed practical limits and cause silent object recycling/visual corruption. Requires a shared object-budget policy (likely aggressive pruning/history caps per spec §35).

5. **Heavy per-bar loops.** Unicorn's `f_findExtremeBar` (up to 1000-bar scan) and `f_registerLevel`'s breach-check loop (up to 999 bars), CISD Pro+'s `getCurrentLegExtreme`/`getPreviousLegExtreme` (500-bar scans), Turtle Soup's multiple 40-bar candle-series scans — combined and multiplied across 5 modules × multiple timeframes, this risks exceeding Pine's per-bar execution time limits ("Script requesting too many securities" / general script timeout errors) at 30M resolution over years of XAUUSD history.

6. **`max_bars_back` conflicts.** Different files declare different `max_bars_back` (IFVG: 5000, CISD: 500, Turtle Soup: 5000, Unicorn: uses explicit `max_bars_back(high,1000)` calls, CRT: 5000). A merged script needs one coherent, deliberately-chosen value — arbitrarily taking the max risks memory/performance issues; taking the min risks breaking a module's lookback logic (e.g., CISD Pro+'s 500-bar leg scan).

7. **Duplicate/conflicting auto-timeframe mapping tables.** Five different schemes (noted in Ambiguity-adjacent finding above) become dead weight once the Master Engine's explicit fixed hierarchy (1W/1D/4H/1H/30M) supersedes them — but if not explicitly removed, they add unused complexity and potential confusion in debug mode.

8. **String-based ticker matching for SMT/contract-size logic** (CRT's `getSMTPair()`, Unicorn's `getContractSize()`) is broker/symbol-suffix fragile (e.g., `XAUUSD` vs `OANDA:XAUUSD` vs `FOREXCOM:XAUUSD`) — since the Master Engine is XAUUSD-specific and explicitly pairs with XAGUSD, this entire string-matching apparatus can likely be replaced with a hardcoded pair, but that is a **modification**, not a preservation, and must be flagged as Master-Engine-Synthesis when it happens, not silently folded into "source fidelity."

---

# 8. PROPOSED IMPLEMENTATION PLAN

I'm not proceeding past this point without your review, per instructions. Proposed sequence once you resolve the ambiguities in §5:

1. **Ambiguity resolution pass** — you answer items 1–10 above. No implementation starts before this.
2. **Namespacing pass** — produce the full type/function rename map (mechanical, but must be complete and reviewed) before any logic is touched.
3. **Security-call budget design** — lay out exactly which timeframes get fetched, how many `request.security` calls total, consolidated tuple design to stay under Pine limits (addresses Technical Risk #3).
4. **Canonical Liquidity + Sweep object design** — resolve Ambiguities #1/#2 into one shared data structure that each engine (CRT, Unicorn, Turtle Soup, IFVG) can tag/reference without losing source identity.
5. **Port CISD Engine and MSS Engine independently**, verify they remain non-collapsing per spec §14.
6. **Port CRT Engine (both models, tagged)**, Reclaim Engine.
7. **Port FVG/IFVG/Unicorn POI engines**, each retaining its own detection method (per Ambiguity #9 resolution).
8. **Port SMT Engine**, resolve dual-trigger question (#6).
9. **Build Displacement, 1W/1D Context, Fractal Reference, Regime engines** as clearly-labeled Master Engine Synthesis (not source-derived) once you provide the missing parameters (#7).
10. **Build the hierarchical gate chain, Signal Engine, Debug Mode, Duplicate Event Protection.**
11. **Compile/debug pass**, then the full Historical-vs-Realtime-vs-Reload repaint audit (§34) against every item flagged in Section 6 above.
12. **Backtest/validation mode**, in/out-of-sample split, per spec §36–37.

Awaiting your resolution of Section 5 before Phase 1 of implementation begins.
