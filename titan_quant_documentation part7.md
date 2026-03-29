# 📖 Titan Quant v11 — Part 7: Code Map & Remaining Subsystems

> Bagian pelengkap akhir — memetakan **setiap baris kode** dan mendokumentasikan subsistem yang belum tercakup di Part 1-6.

---

## 📑 Daftar Isi

1. [Complete Code Section Map](#1-complete-code-section-map)
2. [f_build_signal_filters() — Complete Breakdown](#2-f_build_signal_filters--complete-breakdown)
3. [Bayesian Sniper Relaxation Tiers](#3-bayesian-sniper-relaxation-tiers)
4. [Signal Rearm — Adaptive Cooldown System](#4-signal-rearm--adaptive-cooldown-system)
5. [DIV+ Signal Emission Pipeline](#5-div-signal-emission-pipeline)
6. [AO Standalone Signal Filters](#6-ao-standalone-signal-filters)
7. [Complete Chart Label Catalog](#7-complete-chart-label-catalog)

---

## 1. Complete Code Section Map

### Line-by-Line Architecture

| Line Range | Section | Subsystem |
|-----------|---------|-----------|
| 1-30 | Header | Script metadata, version, indicator declaration |
| 31-230 | Inputs (Core) | Base threshold, model settings, risk weights, display toggles |
| 231-250 | Inputs (Mode) | Realtime mode, signal timing, Bayesian toggles |
| 251-350 | Inputs (SMC) | SMC parameters, FVG settings, pattern settings |
| 351-450 | Inputs (Advanced) | Session config, AO params, projection, DIV+ tuning |
| 451-570 | Preset System | 3 presets: parameter override blocks |
| 571-630 | Session Setup | Session windows, timezone, kill zone detection, sample minimum |
| 631-660 | Data Fetch Helpers | `f_get_macro_stats`, `f_get_macro_stats_vol_close` |
| 661-680 | Bayesian Function | `f_get_bayesian_prob` — posterior calculation |
| 681-780 | Data Fetching | 14+ `request.security()` calls — all macro assets |
| 781-800 | Z-Score Functions | `f_val`, `std_safe`, `z_from`, `f_calc_z_group` |
| 801-900 | Z-Score Computation | Dual Z-score calculation for all 14 assets |
| 901-970 | Z-Score Selection | `f_choose_z`, FallbackZ store update |
| 971-1055 | Correlation Engine | `MacroCorr` type, ring-buffer update algorithm |
| 1056-1100 | Score Functions | `f_calc_score_model`, `f_calc_fallback`, `f_get_total_delta` |
| 1101-1140 | Volatility & Threshold | ATR, vol_score, base_th, th_buy/th_sell |
| 1141-1172 | Risk Index | `f_risk_accum`, `f_calculate_master_risk_index` |
| 1173-1217 | Regime Classification | Auto-adaptive thresholds, 7-tier regime, risk_state |
| 1218-1275 | Macro Vote | `f_macro_vote` — 26-factor (13 off + 13 on) |
| 1276-1320 | Score Computation | Weighted score + delta + hyper-response smoothing |
| 1321-1415 | Trend & Dashboard Prep | HMA trend, pred_dir, scenario_index, conf_score, status eval |
| 1416-1460 | MTF Confluence | `f_get_mtf_trend`, 4-TF grid, scoring |
| 1461-1500 | Intrinsic Basket | XAUEUR/XAUJPY fetch, intrinsic_roc calculation |
| 1501-1600 | Quality Score Functions | `f_status_eval_advanced`, setup classifier |
| 1601-1700 | Pattern Detection (Setup) | Candlestick pattern infrastructure, BB, fractal prep |
| 1700-1770 | Hammer/Star | Detection + 7-component strength scoring |
| 1770-1830 | Engulfing/Sweep | Detection + strength scoring + grade |
| 1831-1940 | VPA 3.0 Events | Session, Volume Z, Ignition, Climax, Churn, Absorption |
| 1940-1975 | Structural Classifier | Breakout/pullback detection, setup_type |
| 1975-2095 | RSI Divergence | Pivot detection, 4 divergence types, strength scoring, memory |
| 2096-2260 | AO Divergence v2.0 | VWAO, 6 upgrades, patterns, Power Div fusion |
| 2260-2320 | Whale Trap v10.0 | 6-layer fusion detection |
| 2322-2460 | Quality Score | Component calculation, boosters, VPA-Bayesian feedback |
| 2460-2480 | Momentum Decay | score_mom, velocity detection |
| 2482-2700 | Order Blocks | OB creation, confluence scoring, quantum check, visualization |
| 2700-2900 | Price Projection | 9-factor model, adjustments, BB anchor, MAE tracking |
| 2900-3050 | Risk Management | Stop loss, TP1/2/3, position sizing |
| 3050-3170 | Cluster Detection | Weighted scoring, tier classification, tooltip builder |
| 3170-3240 | Pattern Labels | `f_render_pattern_labels` — visual output |
| 3248-3520 | SMC Engine | Fractal pivots, BOS/CHoCH, FVG v2.0, Breaker Blocks |
| 3520-3580 | SMC Visualization | Lines, boxes, labels, cleanup functions |
| 3580-3640 | Regime Gates | regime_valid_buy/sell, counter-trend logic, signal context |
| 3641-3800 | **Signal Filters** | `f_build_signal_filters()` — the 160-line monster |
| 3800-3862 | Signal Rearm | 6 trigger types, cooldown, anti-whipsaw, final signal |
| 3863-3900 | Position Setup | Entry vars, reversal context, TF-adaptive exit params |
| 3900-3990 | Exit Logic | 6 exit conditions, reversal-aware, scalping mode |
| 3990-4050 | Dashboard Override | Position-aware pred_dir, intrabar anti-spam latches |
| 4050-4200 | Raw Divergence Labels | Early warning labels, hidden divergence |
| 4200-4340 | DIV+ Signal | 11-layer institutional logic, one-shot filter |
| 4340-4500 | AO/Power Labels | Labels: POWER BUY, AO DIV, Hidden, VPA plotchar |
| 4500-4600 | Signal Labels | ▲ BUY / SELL ▼ main labels, rejected ✕ labels |
| 4600-4830 | Dashboard Render | `f_render_standard_panel`, all dashboard rows |
| 4830-5030 | Dashboard (cont.) | Scanner, scenario, MTF grid, API counter, OB/Projection rows |
| 5030-5070 | Status Line & Plots | TP/SL plots, status line, API comment |

---

## 2. f_build_signal_filters() — Complete Breakdown

([L3641-3797](file:///d:/forex/titannew.pine#L3641-L3797)):

### 2.1 Internal Computation Blocks

```
BLOCK 1: Dynamic Sniper Threshold (L3642-3657)
├── base = bayes_sniper_thresh
├── + vol_add (TF-scaled: M1=2.2, M5=2.8, M15=3.6, H1+=4.4)
├── + vol_spike bonus (+1.0 if vol_score > 0.85)
├── - density relaxation (Aggressive=4.8, Adaptive=3.1)
├── - quality relaxation (A+=3.4, A=2.3, B=1.3)
├── - structure relaxation (ignition+1.0, power_div+1.4, div+0.8)
└── clamp(floor, 85.0)

BLOCK 2: Trend Alignment (L3659-3670)
├── counter_gate = TF-based (M1=0.22, M5=0.25, M15+=0.30)
├── - mode relaxation (Agg=0.06, Adapt=0.04)
├── - quality bonus (>0.75: 0.04, >0.65: 0.02)
├── - reversal support bonus (0.02)
├── floor = 0.14
└── aligned = trend_bull OR (not bear + score > gate) OR (reversal + score > rev_gate)

BLOCK 3: Volatility Regime (L3680-3687)
├── ratio = ATR5 / ATR20
├── range: vol_low to vol_high (per mode)
└── override: ratio > 0.52 + quality > gate + has structure signal

BLOCK 4: Institutional Confluence (L3689-3715)
├── 9 dimensions × max 2 pts each = max 18
└── See Part 2, Section E.1 for detail

BLOCK 5: Chop Market Filter (L3719-3726)
├── chop = spread < gate + vol_ratio < 0.95 + mom < 0.20
└── override: cluster≥2, power_div, stop_vol, sweep, ignition

BLOCK 6: Quality Gate (L3732-3736)
├── Primary: quality_signal_valid (>= qual_min)
├── Alt 1: quality >= floor + (power_div OR ignition OR confluence≥5)
└── Alt 2: quality >= 0.48 + confluence≥6 + strong anchor

BLOCK 7: Five Signal Paths (L3738-3789)
├── Path 1 (Bypass): quality + regime + trend + (ignition|power_div) + bayes≥65
├── Path 2 (Fast): regime + trend + score > 1.5×th + vol_guard + !exhaustion
├── Path 3 (APEX): counter_regime + prev_extreme + mom_flip + rejection + vol_guard
├── Path 4 (Standard): ALL filters pass
└── Path 5 (Deep Reversal): regime + rejection + vol + (confluence≥3 | cluster) + divergence

BLOCK 8: Final Assembly (L3791-3794)
├── all_filters = (std OR bypass OR fast OR apex OR deep) AND chop_allow
└── all_filters = all_filters AND optimal_session
```

### 2.2 Path Priority 

Jika multiple paths TRUE, signal tetap fire sekali. Paths are **OR-combined** — sinyal yang paling mudah path-nya menang.

| Path | Difficulty | Use Case |
|------|-----------|----------|
| Standard | Hardest (all 18) | Normal trading — highest accuracy |
| Bypass | Medium | Strong momentum event (ignition/power div) |
| Fast | Low | Extreme score (>1.5× threshold) |
| APEX | Medium | Counter-trend reversal |
| Deep Reversal | Medium | Strong rejection at extreme |

---

## 3. Bayesian Sniper Relaxation Tiers

([L3781-3782](file:///d:/forex/titannew.pine#L3781-L3782)):

Bayesian sniper BUKAN binary pass/fail — ia memiliki **4 relaxation tiers**:

| Tier | Bayes ≥ | Confluence ≥ | Quality > | Extra Requirement |
|------|---------|-------------|-----------|-------------------|
| **1 (Standard)** | `thresh` | — | — | — |
| **2 (Relaxed)** | `thresh - 3` | 5 | 0.60 | — |
| **3 (Institutional)** | `thresh - 6` | 6 | `relaxed_quality` | Power Div / Ignition / Div / AO |
| **4 (Deep Confluence)** | `thresh - 8` | 7 | `relaxed_quality` | Cluster≥2 / Stop Vol / Sweep |

**`relaxed_quality`**:
- Aggressive: `max(sniper_floor, 0.58)`
- Adaptive: `max(sniper_floor, 0.62)`
- Classic: `0.72`

**Contoh** (threshold = 65%):
```
Tier 1: Bayes ≥ 65% → Pass
Tier 2: Bayes ≥ 62% + confluence ≥ 5 + quality > 0.60 → Pass
Tier 3: Bayes ≥ 59% + confluence ≥ 6 + quality > 0.62 + Power Div → Pass
Tier 4: Bayes ≥ 57% + confluence ≥ 7 + quality > 0.62 + Cluster≥2 → Pass
```

---

## 4. Signal Rearm — Adaptive Cooldown System

### 4.1 Base Cooldown per TF

([L3803-3818](file:///d:/forex/titannew.pine#L3803-L3818)):

| Timeframe | Base CD | After Aggressive | After Adaptive | After Quality+Conf | After Cluster | Final Min |
|-----------|---------|-------------------|----------------|--------------------|---------------|-----------|
| M1 | 3 bars | 1 | 2 | -1 if quality>0.80 | -1 if cluster≥2 | 1 |
| M5 | 4 bars | 2 | 3 | -1 | -1 | 1 |
| M15 | 5 bars | 3 | 4 | -1 | -1 | 1 |
| H1+ | 6 bars | 4 | 5 | -1 | -1 | 1 |

### 4.2 Six Trigger Types

| # | Trigger | Condition | Cooldown Required |
|---|---------|-----------|-------------------|
| 1 | **Crossover** | `ta.crossover(score, th)` | None — immediate |
| 2 | **First Alignment** | `all_filters AND NOT all_filters[1]` | None — immediate |
| 3 | **Momentum Rearm** | `score_mom > 0` + `score_mom_cross_up` + all_filters | `signal_rearm_cd` |
| 4 | **Continuation** | `score > th × continue_mult` + `mom > 0` + all_filters | `signal_rearm_cd + 1` |
| 5 | **Periodic Pulse** | Quality ≥ periodic_gate + confluence ≥ periodic_min + sniper/div/AO | `periodic_rearm_cd` |
| 6 | **Confluence Pulse** | Quality ≥ periodic_gate + confluence ≥ conf_min + anchor signal | `confluence_pulse_cd` |

### 4.3 Continue Multiplier

([L3821-3825](file:///d:/forex/titannew.pine#L3821-L3825)):
```
continue_mult = TF-based: M1=1.05, M5=1.08, M15+=1.10
  - Adaptive: -0.02
  - Aggressive: -0.04

Efek: Score harus LEBIH TINGGI dari threshold × mult untuk continuation rearm.
M1 Aggressive: score > th × 1.01 (sangat mudah)
M15 Classic: score > th × 1.10 (butuh momentum kuat)
```

---

## 5. DIV+ Signal Emission Pipeline

### 5.1 Complete Flow

([L4221-4340](file:///d:/forex/titannew.pine#L4221-L4340)):

```
1. CHECK ALL 11 LAYERS (primary path)
   ├── Layer 1-11 ALL pass → div_sell_primary = true
   └── Layer 1-6,8-11 pass + (high_vol|AO|power) → div_sell_fallback = true

2. COMBINE: div_sell_logic = primary OR fallback

3. ONE-SHOT FILTER:
   ├── Cooldown: (bar_index - last_divplus_sell_bar) > divplus_cd_bars
   ├── Dedup: last_signaled_bear_div_id != last_div_bear_bar (new event only)
   └── If both pass → div_sell_accurate = true

4. UPDATE TRACKING:
   ├── last_signaled_bear_div_id = last_div_bear_bar
   └── last_divplus_sell_bar = bar_index

5. RENDER LABEL:
   └── "⚡DIV+SELL" at (high + ATR×0.8), textcolor=red, tooltip=""
```

### 5.2 Extreme Volatility Gate (Layer 11)

([L4297-4302](file:///d:/forex/titannew.pine#L4297-L4302)):

Aktif HANYA saat `rev_extreme_vol_env = true`:
```
Requirement (SELL):
  ├── Upper wick rejection (high - max(open,close) > range × 0.40)
  ├── NOT bull impulse (body > 70% range + close near high)
  ├── AO/Power/StopVol/Sweep/OB confluence
  └── Bayes bear ≥ dynamic floor

Jika SEMUA terpenuhi → gate passed → signal allowed even in extreme vol
```

### 5.3 Cooldown Timing

```
Normal: divplus_cd_bars = div_cd_norm (default ~5)
Extreme Vol: divplus_cd_bars = div_cd_ext (longer to avoid noise)
```

---

## 6. AO Standalone Signal Filters

### 6.1 TF-Adaptive Filter Profile

([L4376-4412](file:///d:/forex/titannew.pine#L4376-L4412)):

| Parameter | M1 | M5 | M15 | H1+ |
|-----------|----|----|-----|-----|
| `ao_vol_mult_req` | 1.10 | 0.95 | 0.90 | 1.00 |
| `ao_quality_floor` | max(0.50, input) | max(0.40, input-0.07) | max(0.42, input-0.05) | input |
| `ao_strength_floor` | 48 | 34 | 32 | 38 |
| `ao_rsi_buy_min` | 28 | 28 | 30 | 30 |
| `ao_rsi_buy_max` | 68 | 68 | 65 | 65 |

### 6.2 Two Paths

**Primary**: `volume > sma × mult` + RSI in range + quality ≥ floor + strength ≥ floor + regime + session + extreme_gate

**Fallback**: Requires `ao_conf` (Power Div / Stop Vol / Sweep / OB / CVD / Absorption) + `ao_strong` (strength ≥ floor+16 or power ≥ 70) — relaxed volume × 0.85, quality - 0.08

### 6.3 One-Shot Tracking

```pine
// Track divergence event ID to avoid duplicate signals from same event
if ao_reg_bull_recent AND ao_f_buy AND ao_last_sig_bull_id != ao_last_reg_bull_bar
    ao_standalone_buy := true
    ao_last_sig_bull_id := ao_last_reg_bull_bar    // Mark this event as signaled
```

---

## 7. Complete Chart Label Catalog

### 7.1 All Label Types & Conditions

| Label Text | Direction | Size | Trigger | Tooltip Content |
|-----------|-----------|------|---------|-----------------|
| `▲ BUY` | Below bar | Normal | `buy_entry_signal` | Score, regime, quality grade, Bayes, context |
| `SELL ▼` | Above bar | Normal | `sell_entry_signal` | Score, regime, quality grade, Bayes, context |
| `⚡DIV+BUY` | Below (offset) | Normal | `div_buy_accurate` | — |
| `⚡DIV+SELL` | Above (offset) | Normal | `div_sell_accurate` | — |
| `🟢⚡ POWER BUY` | Below (offset) | Normal | `power_buy_signal` | PWR score, AO str, RSI str, Q grade, Bayes |
| `🔴⚡ POWER SELL` | Above (offset) | Normal | `power_sell_signal` | PWR score, AO str, RSI str, Q grade, Bayes |
| `🟩 AO↑ BUY` | Below (offset) | Small | `ao_standalone_buy` | AO strength, Q grade, Bayes%, AO value |
| `🟥 AO↓ SELL` | Above (offset) | Small | `ao_standalone_sell` | AO strength, Q grade, Bayes%, AO value |
| `💚 AO↑` | Below (tiny offset) | Tiny | `ao_hid_buy_plot` | "Hidden Bull - Continuation" |
| `🧡 AO↓` | Above (tiny offset) | Tiny | `ao_hid_sell_plot` | "Hidden Bear - Continuation" |
| `HID BUY` | Below bar | Tiny | `hid_buy_plot` | "Hidden Bullish Divergence" |
| `HID SELL` | Above bar | Tiny | `hid_sell_plot` | "Hidden Bearish Divergence" |
| `⚡ APEX n.n ⬆` | Below | Normal | `cluster_tier_bull == 3` | Cluster score, strength, signal list |
| `🔗 n.n ⬆ G` | Below | Small | `cluster_tier_bull == 2` | Cluster detail |
| `💎 ⬆ G` | Below | Small | `cluster_tier_bull == 1` | Pattern + strength |
| `⬆` | Below | Tiny | Standalone pattern | Pattern type + strength |
| `↑ G` / `↓ G` | Standalone | Tiny | Hammer/Star/Engulfing (no cluster) | Strength/100 |
| `○` | Above | Tiny | `pat_doji_sig` | "Doji Indecision" |
| `✕` | Near close | Tiny | `cond_bias_buy/sell` + filter fail | "BLOCKED: [reason list]" |
| `BOS` / `CHoCH` | At level | Tiny | Structure break detected | Major/Minor tag + FVG quality |
| `☠️ Trap` | Above/below | Tiny | `bull_trap/bear_trap` (plotchar) | — |
| `❇️` / `🛑` | Below/above | Tiny | `absorption` (plotchar) | — |

### 7.2 Main Signal Tooltip Content

```
▲ BUY tooltip:
  "▲ [Type] BUY | Score: X.XX
   Regime: [regime_txt] | [signal_context]
   Quality: [grade] ([score]) | Bayes: [bull]% / [bear]%
   Proj: [pred_dir] [action_score_disp]"
   
Where [Type] = "SNIPER" (if bayes pass) or "SIGNAL"
```

### 7.3 Rejected Signal (✕) Tooltip

```
"BLOCKED:
  [list of failing filters]
  Q:[grade] Bayes:[value] Score:[value]"

Possible blockers:
  "QUALITY" — quality_score < qual_min
  "REGIME" — regime_valid_* = false
  "SNIPER" — Bayes confidence below threshold
  "TREND" — HMA not aligned
  "INTRINSIC" — XAU basket opposing
  "VOL_REGIME" — ATR ratio out of range
  "EXHAUSTION" — high vol + momentum decay
  "CHOP" — market too choppy
  "SESSION" — outside active session
  "CONFLUENCE" — inst score below minimum
```

---

> [!IMPORTANT]
> **Dokumentasi Titan Quant v11 — SELESAI**
>
> | Part | Sections | Cakupan |
> |------|----------|---------|
> | [Part 1](file:///d:/forex/titan_quant_documentation%20part1.md) | 23 | Arsitektur, fitur, flow diagram |
> | [Part 2](file:///d:/forex/titan_quant_documentation%20part2.md) | 13 | Detail implementasi kode (A-M) |
> | [Part 3](file:///d:/forex/titan_quant_documentation%20part3.md) | 5 | Scenario walkthrough, tuning, troubleshooting |
> | [Part 4](file:///d:/forex/titan_quant_documentation%20part4.md) | 9 | Input catalog (70+ params), preset, dependency map |
> | [Part 5](file:///d:/forex/titan_quant_documentation%20part5.md) | 8 | Formula matematika, edge cases |
> | [Part 6](file:///d:/forex/titan_quant_documentation%20part6.md) | 6 | SMC, quality walkthrough, glossary |
> | **Part 7** | 7 | Code map, signal filters, labels, rearm system |
> | **Total** | **71** | **100% coverage of 5,070 lines** |
