# A. FINALIZED ARCHITECTURE SPECIFICATION

All ten locked decisions are incorporated below. This supersedes Sections 2–4 and 8 of the prior Source Audit document; Sections 1 (Source Audit Table), 5 (Ambiguities — mostly resolved now), 6 (Repaint Risks), and 7 (Technical Risks) remain the reference baseline.

**Namespacing convention (governs everything below):**

| Prefix | Source | Scope |
|---|---|---|
| `me_` | Master Engine synthesis | Context, Displacement, Gate chain, Signal Engine, Genealogy, Debug |
| `liq_` | Master Engine synthesis (canonical wrapper) | Normalized Liquidity Object |
| `crt_` | CRT Pro+ | CRT model, CRT-internal reclaim/"manipOpen" logic, CRT SMT sub-signal |
| `cisd_` | CISD Pro+ | Master CISD Engine (authoritative) |
| `ifvg_` | IFVG Pro+ | IFVG Engine (isolated) |
| `uni_` | Unicorn Model | Unicorn Engine (isolated) |
| `ts_` | ICT Turtle Soup Model | Turtle Soup alt-model (isolated), MSS source |

**Layering (top to bottom, matches locked decision #1/#2/#10):**

```
me_Context1W / me_Context1D   (synthesis, price-structure based)
        │
liq_LiquidityObject            (canonical, CRT-oriented priority, source-tagged)
        │
crt_Sweep (confirmed HTF sweep — THE 4H Master Engine sweep)
        │
crt_Model (Model 1 OB + Model 2 reclaim, both tagged, both retained)
        │
crt_Reclaim ("manipOpen"-based — NOT the Master CISD Engine)
        │
cisd_Engine (AUTHORITATIVE)  ‖  ts_MSS (Turtle Soup pivot-break, corrected states)
        │                          │
        └──────────┬───────────────┘
                    ▼
        me_StructureConfirmed (CISD/MSS identity preserved)
                    │
        me_Displacement (research filter, documented formulation, not source-derived)
                    │
        ┌───────────┼─────────────────┐
        ▼           ▼                 ▼
   me_FVGRef    ifvg_Engine       uni_Engine        (each independently traceable POI)
        │           │                 │
        └───────────┴─────────────────┘
                    ▼
        crt_SMT_HTF + crt_SMT_Pivot → me_SMT_Aggregate (BULLISH/BEARISH/NEUTRAL/CONTRADICTORY)
                    │
        me_Regime (session filter — configurable, not mandatory)
                    │
        me_Execution30M → me_SignalEngine → ENTRY

PARALLEL, NON-WIRED BRANCH:
        ts_TurtleSoupModel (fully isolated alt setup, own sweep/MSS/D-Purge/retest/SD-targets)
```

Isolated engines (`ifvg_`, `uni_`, `ts_`) feed the POI layer / alt-model layer as **independently tagged sources**, never algorithmically merged into `crt_Sweep`, `cisd_Engine`, or `me_FVGRef`. This is now a hard architectural constraint, not a pending question.

---

# B. CANONICAL DATA OBJECTS / UDT DESIGN

### B.1 Master Engine synthesis types

```pinescript
type me_LiquidityObject
    string source          // "CRT_HTF" | "UNICORN_A" | "UNICORN_B" | "UNICORN_C" | "UNICORN_D" | "IFVG_HTF" | "TURTLESOUP_PIVOT"
    string liqType         // "BSL" | "SSL" | "INTERNAL" | "EXTERNAL" | "MAJOR" | "MINOR"
    string tf              // resolved timeframe string
    float  price
    int    createdBar
    string status          // "CREATED" | "ACTIVE" | "SWEPT" | "CONSUMED" | "INVALIDATED" | "EXPIRED"
    int    sweptBar
    bool   isMasterPrimary // true only for the CRT-oriented 4H liquidity used by the causal gate chain

type me_SweepEvent
    string source          // "CRT" (Master primary) | "UNICORN" | "IFVG" | "TURTLESOUP" (diagnostic-only, non-primary)
    string direction       // "BULLISH_SWEEP" | "BEARISH_SWEEP" | "NO_SWEEP"
    string tf
    int    barIndex
    float  sweepExtreme
    float  sweptLevel
    bool   isMasterPrimary

type me_StructureEvent
    string kind            // "CISD_BULL" | "CISD_BEAR" | "MSS_BULL" | "MSS_BEAR" | "NONE"
    string sourceEngine    // "cisd_Engine" | "ts_MSS"
    int    barIndex
    float  level
    bool   normalizedToStructureConfirmed  // true once folded into me_StructureConfirmed

type me_DisplacementEvent
    string state           // "NONE" | "WEAK" | "VALID" | "STRONG"
    float  measuredValue    // e.g., body-size/ATR ratio (see §G7 formulation)
    float  thresholdUsed
    int    barIndex

type me_SMTSubSignal
    string kind            // "HTF_DIVERGENCE" | "PIVOT_DIVERGENCE"
    string state           // "BULLISH" | "BEARISH" | "NONE"
    int    barIndex
    string pair             // "XAGUSD"

type me_SMTAggregate
    string state           // "BULLISH_SMT" | "BEARISH_SMT" | "NEUTRAL" | "CONTRADICTORY"
    me_SMTSubSignal htfSignal
    me_SMTSubSignal pivotSignal

type me_ContextState
    string tf              // "1W" | "1D"
    string state           // "BULLISH" | "BEARISH" | "NEUTRAL" | "TRANSITION"
    string basis            // human-readable structural justification (HH/HL, LH/LL, etc. — see §J)

type me_SetupGenealogy       // spec §29 — mandatory traceability record
    string id
    string direction         // "LONG" | "SHORT"
    me_ContextState ctx1W
    me_ContextState ctx1D
    me_LiquidityObject liq4H
    me_SweepEvent sweep4H
    string crtModelUsed       // "MODEL_1_OB" | "MODEL_2_RECLAIM" | "BOTH"
    string reclaimState       // "WAIT_RECLAIM" | "RECLAIM_CONFIRMED" | "RECLAIM_FAILED"
    me_StructureEvent structure1H
    me_DisplacementEvent displacement
    string poiType            // "FVG" | "IFVG" | "UNICORN"
    string poiRef             // pointer/id to source-specific POI object
    me_SMTAggregate smt
    bool   regimePermits
    string execution30M        // "PENDING" | "CONFIRMED"
    string finalState          // "WAIT" | "LONG_CANDIDATE" | "SHORT_CANDIDATE" | "LONG_ENTRY" | "SHORT_ENTRY" | "INVALIDATED" | "EXPIRED"
    float  entry, sl, tp
```

### B.2 Source-namespaced types (renamed only — internal fields/logic untouched)

| Original | Renamed | Source file |
|---|---|---|
| `SwingFVG` | `ifvg_SwingFVG` | IFVG Pro+ |
| `Trade` | `ifvg_Trade` | IFVG Pro+ |
| `SwingPoint` (IFVG) | `ifvg_SwingPoint` | IFVG Pro+ |
| `SwingSet` | `ifvg_SwingSet` | IFVG Pro+ |
| `HTFLevel` | `uni_HTFLevel` | Unicorn Model |
| `Setup` | `uni_Setup` | Unicorn Model |
| `SwingPoint` (Turtle Soup) | `ts_SwingPoint` | ICT Turtle Soup |
| `TSModel` | `ts_Model` | ICT Turtle Soup |
| `Candle` | `crt_Candle` | CRT Pro+ |
| `CandleSettings` | `crt_CandleSettings` | CRT Pro+ |
| `Settings` | `crt_Settings` | CRT Pro+ |
| `CandleSet` | `crt_CandleSet` | CRT Pro+ |
| `SMT` | `crt_SMT` | CRT Pro+ |
| `CRTModel` | `crt_Model` | CRT Pro+ |
| `FVG` | `cisd_FVG` | CISD Pro+ |
| `cisdSetup` | `cisd_Setup` | CISD Pro+ |
| `kz` / `kz_helper` | `cisd_kz` / `cisd_kz_helper` | CISD Pro+ |

**Rule:** internal field names, internal function bodies, and internal thresholds inside every namespaced type are byte-for-byte preserved from source. Only the type identifier and any function name that would otherwise collide are renamed (Technical Risk #1/#2 from the prior audit).

---

# C. EVENT TYPES AND EVENT LIFECYCLES

### C.1 `me_LiquidityObject` lifecycle
`CREATED → ACTIVE → SWEPT → {CONSUMED | INVALIDATED | EXPIRED}`
- `isMasterPrimary = true` only for CRT-oriented 4H levels feeding the causal gate chain (Decision #1).
- Unicorn/IFVG/Turtle Soup liquidity objects populate the same canonical struct with `isMasterPrimary = false`, remain queryable in Debug Mode and Genealogy, but never gate the Signal Engine.

### C.2 `me_SweepEvent` lifecycle
`NO_SWEEP → {BULLISH_SWEEP | BEARISH_SWEEP}` — single-shot per liquidity object (spec §9 "do not repeatedly signal the same liquidity event").
- **Primary path:** `source = "CRT"`, computed only from CRT's confirmed-HTF-candle breach+close-back method (Decision #2). This is the only sweep event with `isMasterPrimary = true`.
- **Diagnostic paths:** Unicorn (intrabar touch), IFVG (HTF-aggregated swing breach via `time(htf)` boundary trick), Turtle Soup (pivot breach+close-back) each emit their own `me_SweepEvent` with `isMasterPrimary = false`, using their own unmodified detection methods, for Genealogy/Debug only.

### C.3 `cisd_Engine` lifecycle (authoritative — Decision #3)
Preserves CISD Pro+'s native `market_bias` flip machine verbatim:
`market_bias == 0 (UNSET) → market_bias = ±1 (TRACKING) → {bias_bullish | bias_bearish} (CISD_BULL/CISD_BEAR fires, line drawn) → market_bias flips, re-tracks`
- Session/HTF-FVG confluence gating (`filter_session`, `filter_fvg`, `confluence_ok`) is **stripped out of this engine's core signal emission** per Decision #3 and relocated: HTF-FVG confluence becomes an input into `me_FVGRef` / POI layer; session confluence becomes an input into `me_Regime`. The raw `CISD_BULL / CISD_BEAR / NONE` signal from `cisd_Engine` is unconditional on those filters — this is the modification explicitly authorized by the Source Audit (§1, CISD Pro+ row) and now executed.

### C.4 `ts_MSS` lifecycle (source: Turtle Soup `f_checkMSS`)
Preserves pivot-break detection and D-Purge extend-once logic verbatim: `NONE → PRE_INVALIDATION (>75% of i_mssMaxBars) → {MSS_CONFIRMED | INVALIDATED (timeout or double-breach)}`.

### C.5 `ts_Model` lifecycle — **CORRECTED per Decision #4**

Original source reused the string `"Formation"` for both the pre-sweep idle state and the post-entry-confirmed state — a documented defect (Source Audit §5, item 5). Master Engine synthesis introduces an explicit corrected enumeration used **only in the Master Engine's own state tracking layer**; the underlying Turtle Soup detection algorithm (sweep detection, MSS detection, inner-sweep retest math, SD-target math) is untouched.

| Corrected Master state | Original source condition it replaces | Original string reused (defect) |
|---|---|---|
| `FORMATION` | No sweep yet / model just created, `mssConfirmed = false` | `"Formation"` |
| `AWAITING_RETEST` | `mssConfirmed = true`, zone found, waiting for inner sweep | `"Awaiting Retest"` (this one was already distinct — no defect here) |
| `ENTRY_CONFIRMED` | `entryConfirmed = true` (post `f_checkInnerSweep` hit) | `"Formation"` ← **defect: collided with pre-sweep idle state** |
| `RESOLVED_SUCCESS` | SD-target 1.0 hit | `"Success"` |
| `INVALIDATED` | Global invalidation / timeout / double-breach failure | `"Invalidation"` |

**Documentation requirement (per instruction):** this correction is recorded here and must be repeated as an inline code comment at the point of translation (`// SOURCE DEFECT CORRECTED: original sets m.status="Formation" post-entry, colliding with pre-sweep idle state; Master Engine tracks ENTRY_CONFIRMED as a distinct state — see Engineering Doc §C.5`). The underlying boolean flags (`entryConfirmed`, `mssConfirmed`, `isInvalidated`, `isTargetHit`) that the source actually uses for branching logic are **not** touched — only the display/filter-facing `status` string's semantic mapping is corrected.

### C.6 `uni_Setup` lifecycle (source: Unicorn Model, unmodified)
`PENDING_SWEEP → BREAKER_FORMING (opposite-candle run ≥2) → CONFIRMED ("+Unicorn" if FVG-overlap found / "+Breaker" if not, per `unicornMode`) → {INVALIDATED | TARGET_HIT}`

### C.7 `ifvg_Trade` lifecycle (source: IFVG Pro+, unmodified)
`CANDIDATE (SwingFVG pushed) → IFVG_CONFIRMED (opposite close trigger) → {SL_HIT | BE_HIT}` — with `hideLosing` **forced OFF** for the research/debug build (per prior audit resolution #4, now finalized: this input is hardcoded `false` in the Master Engine build regardless of user toggle, to satisfy spec §17/§40.3; a source-faithful "display mode" may optionally re-expose the toggle for chart declutter only, never for the backtest/debug data layer).

### C.8 `me_SMTAggregate` lifecycle (Decision #5)
Two independently persisted sub-signals (`crt_SMT_HTF`, `crt_SMT_Pivot`, both unmodified CRT Pro+ algorithms) feed a pure aggregation function evaluated at signal-check time only (not stateful itself):

```
htf=BULL,  pivot=BULL  → BULLISH_SMT
htf=BEAR,  pivot=BEAR  → BEARISH_SMT
htf=NONE,  pivot=NONE  → NEUTRAL
htf=BULL,  pivot=BEAR  → CONTRADICTORY
htf=BEAR,  pivot=BULL  → CONTRADICTORY
htf=BULL,  pivot=NONE  → BULLISH_SMT (single-sub-signal alignment, weaker — flagged in genealogy)
htf=NONE,  pivot=BULL  → BULLISH_SMT (weaker — flagged in genealogy)
(mirror for BEAR)
```
This aggregation rule (single-signal = same-direction but "weaker") is a Master Engine synthesis choice, not present in source — flagged for empirical calibration in §J. SMT never independently triggers `me_SignalEngine`; it only annotates `me_SetupGenealogy.smt` and optionally gates via `me_Config.smtContradictoryBlocks` (boolean, spec §19).

### C.9 `me_DisplacementEvent`
Not a persistent multi-bar lifecycle — evaluated once at the moment `me_StructureConfirmed` fires, produces a single `{NONE, WEAK, VALID, STRONG}` classification attached to that setup's genealogy record (see §G item 7 for the documented initial formulation).

### C.10 `me_ContextState` (1W/1D)
Re-evaluated once per new 1W/1D bar close (via `ta.change(time("1W"))` / `ta.change(time("1D"))` boundary detection, non-repainting), persists (`var`) until next boundary. See §J for the exact structural rule to be formalized before coding (explicitly deferred per Decision #6 — "formalize before coding" is honored below as a concrete proposal you must approve, not an open question I'm punting on).

---

# D. MTF DATA FLOW

```
                         ┌─────────────────────────────┐
                         │   XAUUSD chart feed (30M)     │  ← primary bars, all "current" logic runs here
                         └───────────────┬───────────────┘
                                         │
        ┌────────────────────────────────┼────────────────────────────────────┐
        ▼                                 ▼                                     ▼
request.security(1W)             request.security(1D)               request.security(4H)
me_Context1W_fn()                 me_Context1D_fn()                  crt_4H_fn()
(confirmed-bar only)               (confirmed-bar only)                 (Liquidity+Sweep+CRT Model
                                                                          +CRT reclaim, confirmed-bar)
        │                                 │                                     │
        └────────────────────────────────┴─────────────────────────────────────┘
                                         │
                                         ▼
                          request.security(1H) — cisd_1H_fn() + ts_MSS_1H_fn()
                          (bundled: CISD leg-extreme state machine + Turtle-Soup-style
                           pivot MSS, both confirmed-bar-safe, mirrors CISD Pro+'s own
                           proven `detectFVG()`-inside-security() pattern)
                                         │
                                         ▼
                  POI layer — runs at chart TF (30M) directly, no security() needed:
                     me_FVGRef (decoupled HTF-FVG reference, own request.security if HTF≠chart TF)
                     ifvg_Engine (own internal HTF alignment via `time(htf)` trick — NO security() call)
                     uni_Engine (4 independent request.security calls, Source A/B/C/D)
                                         │
                                         ▼
        request.security(XAGUSD, 4H)          request.security(XAGUSD, chart TF)
        crt_SMT_HTF_fn()                        crt_SMT_Pivot_fn()
                                         │
                                         ▼
                              me_Regime (session, pure chart-TF `time()` calls, no security())
                                         │
                                         ▼
                              me_Execution30M / me_SignalEngine

PARALLEL, UNWIRED:
        request.security(ts_midTF), request.security(ts_htfTF)  →  ts_TurtleSoupModel
        (self-contained, own fractal alignment, feeds Genealogy/Debug only as an alt-model tag)
```

**Non-repaint contract enforced at every HTF boundary (Decision #11):**
- Every `request.security(...)` call in the bundled functions above uses `barmerge.gaps_off, barmerge.lookahead_off` and, where the source's own logic requires it (CISD Pro+'s `detectFVG`), an internal `barstate.isconfirmed` gate — this is the **reference pattern** flagged as good practice in the prior audit (Repaint Risk table, item #6) and is now mandated for every HTF-fetching function in the Master Engine, including the ones ported from CRT and Unicorn that did **not** originally have this gate (CRT's SMT divergence block, Unicorn's `f_getHTFData()`).
- **This is a real modification to CRT's and Unicorn's source behavior**, required by Decision #11 (hard non-repaint requirement) overriding raw source fidelity where the two directly conflict. Flagged explicitly per the Development Protocol: CRT's `bearD_htf`/`bullD_htf` computation and Unicorn's `isSwingHigh`/`isSwingLow` computation will be wrapped with confirmed-bar gating before being trusted by the Master SMT/Liquidity engines. Their *unwrapped, source-faithful* intrabar versions remain available and displayed only in a clearly labeled "Live/Unconfirmed" debug sub-panel, never used for `me_SetupGenealogy` finalization.

---

# E. REQUEST.SECURITY BUDGET PLAN

| # | Call | Timeframe | Bundled contents | Notes |
|---|---|---|---|---|
| 1 | `me_Context1W_fn()` | 1W | 1W structural context tuple | 1 call |
| 2 | `me_Context1D_fn()` | 1D | 1D structural context tuple | 1 call |
| 3 | `crt_4H_fn()` | 4H | Liquidity registration + confirmed sweep + CRT Model 1 (OB) + Model 2 (reclaim/manipOpen) — all same-source, same-TF, bundled into one tuple return | 1 call (was implicitly ~2-3 separate concerns in source; consolidated per Decision #10) |
| 4 | `cisd_ts_1H_fn()` | 1H | CISD Pro+ leg-extreme state machine + Turtle-Soup-style MSS pivot detection, bundled | 1 call |
| 5 | `me_FVGRef_fn()` | 1H (or 4H — pick one canonical HTF for the *reference* layer only; source-specific FVG detectors below are unaffected) | Decoupled HTF-FVG reference (formerly CISD Pro+'s `detectFVG`) | 1 call |
| 6 | `crt_SMT_HTF_fn()` | 4H | XAGUSD close/open/high/low for HTF divergence | 1 call |
| 7 | `crt_SMT_Pivot_fn()` | chart TF (30M) | XAGUSD `high[3]`/`low[3]` for pivot divergence | 1 call |
| 8 | `uni_Source_A_fn()` | user-configured (Auto or Manual) | Unicorn liquidity Source A | 1 call |
| 9 | `uni_Source_B_fn()` | " | Unicorn liquidity Source B | 1 call |
| 10 | `uni_Source_C_fn()` | " | Unicorn liquidity Source C | 1 call |
| 11 | `uni_Source_D_fn()` | " | Unicorn liquidity Source D | 1 call |
| 12 | `ts_midTF_fn()` | Turtle Soup's own fractal-mid TF | 24-value OHLC tuple (alt-model only) | 1 call |
| 13 | `ts_htfTF_fn()` | Turtle Soup's own fractal-HTF TF | 25-value OHLC tuple (alt-model only) | 1 call |
| — | `ifvg_Engine` | n/a | Uses `time(htf) != time(htf)[1]` boundary trick directly on chart-native `high`/`low` — **zero `request.security` calls** | Confirmed by source inspection — no call needed |

**Total: 13 `request.security` calls.** TradingView's per-script ceiling for `request.*` calls is commonly documented at 40 for Pine v6; 13 leaves substantial headroom for the Debug Engine, Backtest Engine, and any later addition (e.g., a second SMT confirmation pair) without approaching the ceiling. **This number should be re-verified against current TradingView documentation at implementation time**, since platform limits are occasionally revised.

---

# F. OBJECT / LOOP / MEMORY BUDGET PLAN

### F.1 Object ceilings (hard Pine limits — script-wide, not per-module)
`max_lines_count = 500`, `max_boxes_count = 500`, `max_labels_count = 500`, `max_boxes_count = 500`, `max_polylines_count = 100` (only CISD Pro+ declares this; harmless to keep at default if unused elsewhere).

Since these are **shared across all namespaced modules**, source-default history counts must be re-budgeted downward — this is a Master Engine performance policy (spec §35), not a change to detection logic:

| Module | Source default history/count input | Proposed Master Engine cap | Rationale |
|---|---|---|---|
| `crt_Model` | `max_display_inp = 4` HTF candles, `history_lookback = 2` models | Keep as-is | Already conservative |
| `ts_Model` (alt model) | `i_maxModels = 2`, `i_htfCount = 6` candles ×2 panels | Keep as-is | Already conservative |
| `uni_Setup` | `historyCount` input range **1–100** | **Clamp input max to 10** | Source allows configuring up to 100 setups × (levelLine+levelLabel+breakerBox+fvgBox+setupLabel+2×targetLines+2×targetLabels) ≈ 9 objects each → 900 objects alone, exceeding the shared 500 ceiling by itself |
| `ifvg_Trade` | `setupHistory` input range **0–50** (0 = unlimited) | **Clamp input max to 15, disallow 0/unlimited** | "Unlimited" directly risks silent object recycling/corruption once other modules are also drawing |
| `cisd_Setup` | `cisd_maxLines = 5`, `maxFVGs = 5` | Keep as-is | Already conservative |

This clamping is a **documented Master Engine synthesis modification**, distinct from source-derived detection logic, per spec §39. It changes only the *display/retention* ceiling, never the underlying detection or signal logic.

### F.2 Loop budget (per-bar execution cost)

| Loop | Source | Worst-case iterations | Risk if combined |
|---|---|---|---|
| `getCurrentLegExtreme` / `getPreviousLegExtreme` | CISD Pro+ | 500 bars each | Runs every bar on chart TF (not HTF-throttled) — highest-frequency heavy loop in the corpus |
| `f_findExtremeBar` | Unicorn | 1000 bars | Runs on every new HTF level registration, not every chart bar — lower frequency, larger single cost |
| `f_registerLevel` breach-check | Unicorn | up to 999 bars | Same trigger frequency as above |
| `f_findUpCandlesSeries` / `f_findDownCandlesSeries` | Unicorn | 40+20 nested | Runs while a sweep is pending, every bar until breaker resolves |
| `f_findBullishFVG` / `f_findBearishFVG` | Unicorn | up to 100 | Runs once breaker candidate exists |
| `ict_ScanExtreme` | IFVG Pro+ | span-bounded (bounded by swing-to-confirmation distance) | Runs once per IFVG confirmation, moderate frequency |
| Turtle Soup candle-series/zone scans | Turtle Soup | ≤40 bars | Runs per model per bar while unconfirmed |

**Finding:** CISD Pro+'s 500-bar scan running unconditionally *every chart bar* (not gated to only run on HTF boundary changes, since CISD operates on whatever TF it's applied to — here, the 1H synthetic feed inside `request.security`) is the single largest sustained per-bar cost once merged with the other four modules' loops on a multi-year 30M backtest. This is flagged formally in **§J** as requiring empirical performance testing (measured compile/execution time) rather than a pre-emptive shortening of the lookback — shortening it would be a source-fidelity violation (Decision #3 requires CISD Pro+ remain authoritative/unmodified), so this must be *tested*, not *guessed*, before any tightening is considered.

### F.3 `max_bars_back`
Single script-wide directive: **`max_bars_back = 5000`** (the effective ceiling already required independently by IFVG Pro+, Turtle Soup, and CRT Pro+; CISD Pro+'s smaller internal 500-bar scan is an algorithmic loop bound unrelated to this compiler directive and is unaffected by using the larger global value).

---

# G. MASTER STATE MACHINE

```
                              ┌───────────────────────┐
                              │ me_Context1W / 1D       │
                              │ (BULLISH/BEARISH/       │
                              │  NEUTRAL/TRANSITION)    │
                              └───────────┬─────────────┘
                                          ▼
                              CONTEXT_VALID? ──NO──► WAIT
                                          │YES
                                          ▼
                              ┌───────────────────────┐
                              │ liq_LiquidityObject     │
                              │ (CRT-oriented, primary) │
                              └───────────┬─────────────┘
                                          ▼
                              LIQUIDITY_IDENTIFIED? ──NO──► WAIT
                                          │YES
                                          ▼
                              ┌───────────────────────┐
                              │ crt_Sweep (confirmed)   │
                              └───────────┬─────────────┘
                              NO_SWEEP ────┘──► WAIT (re-evaluate)
                                          │SWEPT
                                          ▼
                              ┌───────────────────────┐
                              │ crt_Reclaim             │
                              │ (WAIT_RECLAIM state)    │
                              └───────────┬─────────────┘
                              RECLAIM_FAILED ──► INVALIDATED
                                          │RECLAIM_CONFIRMED
                                          ▼
                     ┌────────────────────┴────────────────────┐
                     ▼                                          ▼
           ┌──────────────────┐                     ┌──────────────────┐
           │ cisd_Engine        │                     │ ts_MSS             │
           │ CISD_BULL/BEAR      │                     │ MSS_BULL/BEAR       │
           └─────────┬───────────┘                     └─────────┬────────────┘
                     └──────────────────┬──────────────────────┘
                                        ▼ (first to confirm wins; BOTH logged if both fire)
                              me_StructureConfirmed
                                        │
                                        ▼
                              ┌───────────────────────┐
                              │ me_Displacement         │
                              │ NONE/WEAK/VALID/STRONG  │
                              └───────────┬─────────────┘
                    NONE/WEAK ────────────┘──► WAIT (blocks progression, no auto-invalidate)
                                        │VALID/STRONG
                                        ▼
                     ┌──────────────────┴──────────────────────┐
                     ▼                    ▼                     ▼
              me_FVGRef            ifvg_Engine            uni_Engine
              (reference POI)      (independent POI)      (independent POI)
                     └──────────────────┬──────────────────────┘
                                        ▼ (POI_IDENTIFIED — any one qualifies, type recorded)
                              ┌───────────────────────┐
                              │ crt_SMT_HTF             │
                              │ crt_SMT_Pivot            │
                              │  → me_SMTAggregate       │
                              └───────────┬─────────────┘
                    CONTRADICTORY ────────┘──► (blocks IF me_Config.smtContradictoryBlocks, else continues flagged)
                                        │
                                        ▼
                              ┌───────────────────────┐
                              │ me_Regime (session)     │
                              └───────────┬─────────────┘
                              NOT_PERMITTED ──► WAIT (configurable, not mandatory — Decision #8)
                                        │PERMITTED
                                        ▼
                              ┌───────────────────────┐
                              │ me_Execution30M         │
                              └───────────┬─────────────┘
                                        ▼
                              EXECUTION_READY
                                        │
                                        ▼
                         LONG_ENTRY / SHORT_ENTRY ──► me_SetupGenealogy finalized
                                        │
                              (TP → EXPIRED_SUCCESS) or (SL → INVALIDATED)

ISOLATED, UNWIRED PARALLEL BRANCH:
  ts_TurtleSoupModel: FORMATION → SWEEP → MSS_CONFIRMED → AWAITING_RETEST →
                       ENTRY_CONFIRMED → {RESOLVED_SUCCESS | INVALIDATED}
  (states corrected per §C.5; never feeds the chain above)
```

---

# H. MODULE DEPENDENCY GRAPH

```
me_Context1W ─┐
              ├──► me_GateChain (context gate only)
me_Context1D ─┘

liq_LiquidityObject ──depends on──► crt_4H_fn() [primary]
                      ‖ independently populated by: uni_Source_A..D, ifvg internal HTF, ts pivot arrays
                        (no cross-dependency between these — parallel population, shared struct only)

crt_Sweep ──depends on──► liq_LiquidityObject (primary tag only)
crt_Model ──depends on──► crt_Sweep
crt_Reclaim ──depends on──► crt_Model

cisd_Engine ──depends on──► crt_Reclaim (RECLAIM_CONFIRMED gate) [Master gate-chain wiring only;
                            cisd_Engine's own internal algorithm has ZERO dependency on CRT —
                            it is purely self-contained per source]
ts_MSS ──depends on──► crt_Reclaim (same gate-chain wiring; internal algorithm self-contained)

me_StructureConfirmed ──depends on──► cisd_Engine OR ts_MSS (first-wins, both-logged)

me_Displacement ──depends on──► me_StructureConfirmed (trigger point only; own calc is self-contained ATR/body-based, no source dependency)

me_FVGRef ──depends on──► me_Displacement (gate-chain wiring only; own detection self-contained)
ifvg_Engine ──depends on──► NOTHING upstream (fully self-contained, own internal sweep/HTF logic) — wired into gate chain only at the POI-check step
uni_Engine ──depends on──► NOTHING upstream (fully self-contained) — same wiring pattern

crt_SMT_HTF, crt_SMT_Pivot ──depends on──► NOTHING upstream (independent XAGUSD fetches) — feed me_SMTAggregate, consumed at POI→Regime transition only

me_Regime ──depends on──► NOTHING upstream (pure chart-TF time() checks) — consumed at SMT→Execution transition only

me_Execution30M ──depends on──► me_Regime (gate) + chart-native 30M bars

me_SignalEngine ──depends on──► me_Execution30M + full upstream chain (assembles me_SetupGenealogy)

ts_TurtleSoupModel ──depends on──► NOTHING from the above graph. Fully isolated.
                                     Reads only its own request.security(midTF/htfTF) fetches
                                     and chart-native OHLC. Feeds Debug/Genealogy as an
                                     annotation-only alt-model tag, never the gate chain.
```

**Key property:** every source-derived engine (`cisd_`, `crt_`, `ifvg_`, `uni_`, `ts_`) is internally self-contained — none of them call into another source module's detection function. The **only** cross-module dependencies are Master-Engine-synthesized gate transitions (context→liquidity→sweep→reclaim→structure→displacement→POI→SMT→regime→execution), which is precisely what spec §3 ("hierarchical gated system, not a simple confluence score") requires.

---

# I. EXACT IMPLEMENTATION ORDER

1. **Namespacing pass** — rename all UDTs/functions per §B.2 table; compile-check for zero collisions before any logic changes. No behavior change at this step — pure mechanical renaming, diffable against source.
2. **`me_LiquidityObject` + `me_SweepEvent` canonical wrapper** — build the shared struct (§B.1), write adapter functions that populate it from each source's native output (CRT primary, Unicorn ×4, IFVG, Turtle Soup) without altering any source detection algorithm.
3. **`crt_4H_fn()` bundling** — consolidate CRT's Liquidity+Sweep+Model1+Model2+Reclaim into the single request.security-wrapped function per §D/§E, with the confirmed-bar gating hardening required by Decision #11.
4. **`cisd_ts_1H_fn()` bundling** — port CISD Pro+'s state machine unmodified except for the confluence-stripping specified in §C.3; port Turtle Soup's `f_checkMSS` unmodified; wrap both in the single 1H `request.security` call.
5. **`me_StructureConfirmed` merge logic** — first-wins/both-logged normalization (spec §14), with `me_SetupGenealogy` fields populated.
6. **`me_Displacement`** — implement the documented initial formulation (§J) as a pure function triggered at `me_StructureConfirmed`.
7. **POI layer** — port `me_FVGRef` (decoupled reference), `ifvg_Engine` (with `hideLosing` forced off), `uni_Engine` (with `historyCount` clamped) as three parallel, independently-taggable modules feeding the same `poiType`/`poiRef` genealogy fields.
8. **SMT layer** — port `crt_SMT_HTF` and `crt_SMT_Pivot` unmodified as independent sub-signals; implement `me_SMTAggregate` per §C.8.
9. **`me_Context1W` / `me_Context1D`** — implement the structural rule formalized in §J, confirmed-bar gated.
10. **`me_Regime`** — implement the exact session windows from Decision #8, configurable/non-mandatory per spec.
11. **`me_Execution30M` + `me_SignalEngine`** — assemble the full gate chain per §G, implement duplicate-event protection (spec §30) and the WAIT/CANDIDATE/ENTRY/INVALIDATED/EXPIRED signal states (spec §24).
12. **`ts_TurtleSoupModel`** — port as fully isolated alt-model, applying the state correction from §C.5.
13. **Debug Mode** — implement the compact diagnostic dump (spec §31) reading directly from `me_SetupGenealogy`.
14. **Object/loop budget enforcement** — apply the clamps from §F.1, verify against the shared 500-object ceilings under stress (max settings across all modules simultaneously).
15. **Full repaint audit** (spec §34) — Historical vs. Realtime vs. Reloaded-chart, specifically re-testing every item flagged in the prior Repaint Risk table (§6 of the Source Audit) now that the confirmed-bar hardening from Decision #11 has been applied.
16. **Alert Engine** — wire alerts to the same confirmed states as chart signals (spec §33 — no more-permissive alert logic).
17. **Backtest/Validation Engine** — spec §36–37, in/out-of-sample split, only after steps 1–16 are stable.

---

# J. REMAINING ITEMS THAT MUST BE EMPIRICALLY CALIBRATED

Per spec §38 Tier 2/Tier 3 policy — these are **not** architecture ambiguities, they are deliberately deferred numeric/behavioral parameters requiring historical-data testing before finalization:

1. **1W/1D structural context rule (Decision #6 requires formalization before coding).** Proposed initial formulation for your approval: classify each closed 1W/1D bar via a simple higher-high/higher-low vs. lower-high/lower-low swing comparison against the prior N=2 confirmed swing points on that timeframe (reusing the same pivot-detection pattern already proven in Turtle Soup's `ta.pivothigh/low`, applied to the 1W/1D security feed) → `BULLISH` (HH+HL), `BEARISH` (LH+LL), `NEUTRAL` (no clear pivot yet), `TRANSITION` (mixed, e.g. HH+LL). This is a proposal, not yet locked — please confirm or amend before Step 9 of §I.
2. **Displacement formulation.** Proposed initial, documented (non-arbitrary) formulation: `displacement = (confirmation-candle body size) / ATR(14, same TF as structure event)`. Thresholds: `<0.5×ATR = NONE`, `0.5–1.0× = WEAK`, `1.0–2.0× = VALID`, `>2.0× = STRONG`. These specific multipliers are placeholders explicitly flagged for empirical calibration per spec §15/§38 — not to be treated as final.
3. **SMT single-sub-signal aggregation weighting** (§C.8) — whether a lone HTF or lone pivot divergence should count as full `BULLISH_SMT`/`BEARISH_SMT` or a distinct weaker tier needs backtested comparison.
4. **SMT matching window** — how many bars apart the HTF and pivot sub-signals may occur and still be considered "aligned" for aggregation (spec §38 Tier 2 example) — no source defines this; must be tuned.
5. **POI expiry / setup expiry windows** for `me_FVGRef`, `ifvg_Engine`, `uni_Engine` cross-timeframe staleness — source files each have their own (IFVG's `graceCandles` 6/15, Turtle Soup's `i_mssMaxBars`, Unicorn has none) — Master Engine needs one policy for when a POI becomes too old to gate a 30M execution.
6. **Swing sensitivity parameters** — Turtle Soup's `i_swingLen`(15)/`i_mssLeft`/`i_mssRight`(3/3), CISD's implicit consecutive-candle sensitivity, Unicorn's breaker-candle-count(≥2) — none of these were tuned for XAUUSD specifically; all Tier 2 per spec §38.
7. **CISD 500-bar / Unicorn 1000-bar loop performance impact when merged** (flagged in §F.2) — requires actual compile/execution timing on a real multi-year 30M XAUUSD dataset before deciding whether any lookback bound needs tightening. **This is the one item from the original Source Audit ambiguities that remains genuinely unresolved by the locked architecture decisions** — it is a runtime performance question, not a design question, and cannot be answered without running the merged script.
8. **Regime-permits-vs-blocks calibration** — spec §22 says session filter is configurable/non-mandatory; the actual threshold for when a non-killzone setup should still be allowed vs. suppressed is a Tier 2 parameter requiring win-rate comparison in/out of session.
9. **Duplicate-event protection window** (spec §30) — how much price/time separation constitutes a "genuinely new qualifying event" vs. a re-trigger of the same liquidity/structure/POI combination — needs an explicit distance/time threshold, currently undefined by any source.

None of the above blocks proceeding to implementation per your locked decisions — they are backtest-phase calibration targets, consistent with spec §38's Tier 2 classification, and will be surfaced again as explicit inputs (not hardcoded magic numbers) per spec §40.4.
