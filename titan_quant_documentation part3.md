# 📖 Titan Quant v11 — Part 3: Scenario, Tuning & Troubleshooting

> Lanjutan dari [Part 1](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_documentation.md) (Arsitektur & Fitur) dan [Part 2](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_deep_dive_part2.md) (Detail Implementasi)

---

## 📑 Daftar Isi

1. [Scenario Walkthroughs](#1-scenario-walkthroughs)
2. [Parameter Tuning Guide](#2-parameter-tuning-guide)
3. [Troubleshooting](#3-troubleshooting)
4. [Maintenance Checklist](#4-maintenance-checklist)
5. [Variable Cross-Reference Index](#5-variable-cross-reference-index)

---

## 1. Scenario Walkthroughs

### 1.1 Scenario: Sniper BUY Signal pada M5

**Kondisi Pasar:**
- VIX naik ke 22 (fear moderate), DXY melemah, US10Y turun
- Harga Gold pullback ke demand zone

**Pipeline yang dilalui:**

```
STEP 1 — DATA FETCH
├── VIX Z > +0.5 (fear), DXY Z < -0.4 (dollar lemah)
├── US10Y Z < -0.3 (yield turun)
└── Circuit Breaker: semua data valid (counter < 3)

STEP 2 — RISK INDEX
├── risk_off_index ≈ +0.65 (kontribusi positif: VIX, -DXY, -US10Y)
├── Regime: "Risk-Off" (>= th_low)
└── risk_state = 1

STEP 3 — SCORE COMPUTATION
├── Correlation weights: DXY w=-0.85, VIX w=+0.72, US10Y w=-0.68
├── Weighted score = Σ(z × corr × risk_w) / Σ|w| ≈ +0.38
├── Responsiveness delta (live > model) += +0.05
├── Smoothed: score_live ≈ +0.40
└── th_buy = 0.25 (VIX>18 → lowered), th_buy passed ✅

STEP 4 — BAYESIAN
├── Prior: macro_adj = +0.40, prior_bull = 60%
├── Evidence: HMA trend up ✅, RSI 55 ✅, Volume above avg ✅
├── Posterior: bayes_conf_bull ≈ 78%
└── dynamic_sniper_thresh ≈ 63% → 78% > 63% → Sniper PASS ✅

STEP 5 — TECHNICAL SIGNALS
├── RSI regular bullish divergence detected (strength 62)
├── AO regular bullish divergence (strength 54)
├── Power Div active (combined 59 = STANDARD)
├── Hammer candle at demand zone (+15 OB proximity bonus)
└── Cluster tier = 2 (POWER: sweep 2.5 + hammer 1.5 + OB 2.0 = 6.0 ≥ 3.5)

STEP 6 — QUALITY
├── Base: correlation(0.18) + agreement(0.25) + magnitude(0.15)
├── + vol_div(0.08) + intrinsic(0.10) + bayes(0.12) + mtf(0.08) = 0.52
├── Boosters: RSI div(+0.15) + AO div(+0.08) + Power Div(+0.12) + Cluster(+0.10) = +0.45
├── Raw: 0.52 + 0.45 = 0.97 → clamped to 1.0
├── × Session: Kill Zone(1.25) → 1.0 (already capped)
├── × Setup: Power Div(1.30) → 1.0 (capped)
└── FINAL: quality_score = 1.0, Grade = "A+" ✅

STEP 7 — SIGNAL FILTERS (18 checks)
├── [1] Quality gate: 1.0 > qual_min ✅
├── [2] Regime valid: risk_state=1, score>0.08 ✅
├── [3] Sniper: 78% > 63% ✅
├── [4] Intrinsic: XAU basket aligned ✅
├── [5] Trend aligned: HMA bullish ✅
├── [6] Inst confluence: regime(2)+trend(2)+bayes(2)+mtf(2)+quality(2)+purity(2)+cluster(2) = 14 ≥ 3 ✅
├── [7] Vol regime: ATR5/ATR20 = 1.1 (in range 0.80-2.50) ✅
├── [8] Vol guard: ratio > 0.4 ✅
├── [9] Session: London/NY overlap ✅
├── [10] Exhaustion: false (no momentum decay) ✅
├── [11] Chop: trend_spread > 0.14 ✅
├── [12] Whale trap: false ✅
├── [13] Counter-acc: OK ✅
├── [14] Flip risk: no recent sell ✅
├── [15] Same-side echo: no recent buy ✅
├── [16] Rearm cooldown: passed ✅
├── [17] Path: Standard path PASS (all filters) ✅
└── [18] ALL_FILTERS = true

STEP 8 — FINAL SIGNAL
├── cond_buy: score crossover th_buy ✅
├── all_filters_buy: true ✅
├── final_buy_signal = true
├── buy_entry_signal = true (not in position)
└── LABEL: "▲ BUY" di bawah bar

STEP 9 — POSITION TRACKING
├── in_long_position = true
├── position_entry_price = close
├── position_is_reversal = false (trend-follow)
└── Dashboard: Scanner → "IN LONG +0.00% [0 bars]"
```

---

### 1.2 Scenario: Sniper SELL Signal (Counter-Trend)

**Kondisi Pasar:**
- Regime masih Risk-Off (gold bullish), TAPI harga sudah overshoot
- Top absorption + RSI > 72 + AO bearish divergence

```
CRITICAL DIFFERENCE — Counter-Trend Entry Path:

STEP 2 — REGIME
├── risk_off_index ≈ +0.45, Regime: "Risk-Off"
├── Normal regime_valid_sell = false (risk-off blocks sell)
├── TAPI: counter_sell_ok aktif karena:
│   ├── score_live < -0.22 (harga sudah reversal dari puncak)
│   └── EMA(20) < EMA(50) (price downtrend confirmed)
└── regime_valid_sell = true via counter-trend path ✅

STEP 5 — TECHNICAL: APEX REVERSAL PATH
├── Trigger: score_live[1] > th_buy (sebelumnya bullish)
├── Momentum flip: score_mom < -0.15
├── Top rejection: is_top_absorption = true
├── vol_guard_valid = true
└── apex_sell = true ✅ (Path 4 aktif — APEX REVERSAL)

STEP 8 — FINAL SIGNAL
├── Path resolution: apex_sell passes all_filters_sell
├── final_sell_signal = true
├── position_is_reversal = true (tagged for wider exit windows)
└── LABEL: "SELL ▼" + tooltip "Counter-Trend (Risk-Off)"
```

> [!NOTE]
> Counter-trend signals masuk via Path 4 (APEX) atau Path 5 (Deep Reversal). Mereka memerlukan **rejection evidence** yang lebih kuat karena melawan macro bias.

---

### 1.3 Scenario: Whale Trap Detection

**Kondisi Pasar:**
- Harga breakout agresif melewati recent high, lalu immediate reversal close di bawah

```
LAYER A: Structural Boundaries
├── high > highest(5) → price breaks previous highs ✅
└── Sweep: harga melampaui swing level lalu kembali

LAYER B: Volatility Dynamics
├── candle_range > ATR × 1.35 → significant range ✅
├── Upper wick > 60% of range → rejection ✅
└── close < (high - 70% range) → deep reflux ✅

LAYER C: Purity Divergence
├── XAU/EUR dan XAU/JPY TIDAK breakout → harga USD-driven saja
└── xau_intrinsic_roc < -0.005 → purity divergence ✅

LAYER D: Session Edge
└── in_london AND in_ny → Kill Zone overlap ✅

LAYER E: Sequential Stop-Runs
└── trap_ratio × 20 < 1.5 → belum triple trap (optional)

LAYER F: Macro Fusion
├── DXY Z > +1.0 ATAU Yield Z > +0.8 → macro pressure ✅
└── is_mom_decay = true → momentum exhaustion ✅

RESULT: bull_trap = true → ☠️ label + quality_score × 1.25
```

---

### 1.4 Scenario: Regime Shift (Risk-On → Risk-Off)

```
BAR N-3: risk_off_index = -0.40 (Risk-On)
├── VIX = 14, DXY strong, yields stable
├── Regime = Risk-On, sells allowed, buys blocked

BAR N-2: Breaking news — geopolitical tension
├── VIX Z jumps +1.8, DXY Z drops -0.9
├── risk_off_index recalculates → +0.55
├── REGIME SHIFT: Risk-On → Risk-Off (transition bar)
├── risk_state = 1, neutral_eligible = true
└── Dashboard: "⚠️ EXTREME RISK-OFF"

BAR N-1: Market follows through
├── score_live smoothing catches up: score → +0.35
├── Bayesian prior flips: prior_bull → 72% (from 38%)
├── MTF starts aligning (M15, H1 turn bullish)
└── is_trade_ready_buy = true → Dashboard: "🎯 SNIPER READY"

BAR N: Confirmed buy signal
├── Score crosses th_buy
├── All 18 filters pass (new regime supports buy)
├── final_buy_signal = true
└── Signal context: "⚡ Extreme Risk-Off BUY"
```

> [!IMPORTANT]
> Regime shift memerlukan **2-3 bar lag** karena:
> 1. EMA smoothing pada score_live (alpha ≈ 0.15-0.25)
> 2. Bayesian posterior menggunakan smoothed score sebagai prior
> 3. MTF memerlukan macro TF bar close untuk update

---

## 2. Parameter Tuning Guide

### 2.1 Preset Comparison Matrix

| Parameter | M5 Pro | Agg Scalping 1m | Day Trading 5m |
|-----------|--------|-----------------|----------------|
| `tf_macro` | "240" | "240" | "240" |
| `tf_analysis` | "5" | "1" | "15" |
| `model_len` | 35 | 22 | 35 |
| `lookback_corr` | 18 | 12 | 18 |
| `smoothing_alpha` | 0.22 | 0.38 | 0.22 |
| `responsiveness_gain` | 2.80 | 4.50 | 2.80 |
| `base_threshold` | 0.30 | 0.18 | 0.30 |
| `final_qual_min` | 0.38 | 0.15 | 0.38 |
| `bayes_sniper_thresh` | 65.0 | 58.0 | 65.0 |
| `hma_len` | 40 | 28 | 40 |
| `cb_threshold` | 3 | 2 | 3 |
| `sniper_density_mode` | "Adaptive" | "Aggressive" | "Adaptive" |

### 2.2 Tuning: Lebih Banyak Signal

Jika signal terlalu jarang, adjust bertahap (dari paling aman ke paling agresif):

| Step | Action | Parameter | Efek |
|------|--------|-----------|------|
| 1 | Turunkan quality floor | `sniper_quality_floor_in`: 0.55 → 0.45 | +20% signals |
| 2 | Mode density | `sniper_density_mode`: Classic → Adaptive | +30% signals |
| 3 | Turunkan Bayes threshold | `bayes_sniper_thresh`: 65 → 60 | +15% signals |
| 4 | Kurangi confluence minimum | `inst_min_*`: auto-reduced by mode | +10% signals |
| 5 | Extreme: Aggressive mode | `sniper_density_mode`: Aggressive | +50% signals |

> [!WARNING]
> Setiap step yang meningkatkan frekuensi **menurunkan akurasi**. Monitor win rate setelah setiap adjustment selama minimal 50 signals.

### 2.3 Tuning: Lebih Akurat (Fewer False Signals)

| Step | Action | Parameter | Efek |
|------|--------|-----------|------|
| 1 | Naikkan quality floor | `final_qual_min`: 0.38 → 0.55 | -40% signals |
| 2 | Naikkan Bayes threshold | `bayes_sniper_thresh`: 65 → 72 | -20% signals |
| 3 | Mode density Classic | `sniper_density_mode`: Classic | Strictest filtering |
| 4 | Require session filter | `enable_session_filter`: true | Kill Zone only |
| 5 | Naikkan base threshold | `base_threshold`: 0.30 → 0.40 | -25% signals |

### 2.4 Tuning: Timeframe-Specific Tips

**M1 (Aggressive Scalping):**
- Smoothing alpha tinggi (0.35-0.45) — score harus reaktif
- HMA short (24-32) — trend detection cepat
- Quality floor rendah (0.15-0.25) — mengikuti price action cepat
- Circuit breaker threshold rendah (2) — data harus fresh

**M5 (Standard Scalping):**
- Balanced smoothing (0.18-0.25)
- HMA medium (36-45)
- Quality floor moderate (0.35-0.45)
- Bayesian tetap strict (65-68%)

**M15+ (Day Trading/Swing):**
- Smoothing alpha rendah (0.10-0.18) — smoother score
- HMA panjang (50-65)
- Quality floor tinggi (0.50-0.60)
- Bayesian bisa lebih ketat (68-75%)

### 2.5 Risk Weight Tuning

Jika ingin menekankan/menurunkan pengaruh suatu macro:

| Macro | Default Weight | Increase if... | Decrease if... |
|-------|---------------|----------------|----------------|
| DXY | 1.80 | USD sangat volatile | USD range-bound |
| VIX | 1.65 | Market stress tinggi | VIX crushed (<13) |
| Yield | 1.30 | Fed pivot expected | Yield stabil |
| Credit | 0.90 | Credit crisis concern | Normal spreads |
| US02Y | 0.60 | Rate cut anticipation | Data noisy |

> [!TIP]
> Gunakan korelasi display (`corr_*`) di debug mode untuk melihat bobot yang paling efektif saat ini. Jika korelasi Gold-DXY menurun, pertimbangkan menurunkan `w_risk_dxy`.

---

## 3. Troubleshooting

### 3.1 Tidak Ada Signal (Chart Kosong)

```
DIAGNOSIS FLOWCHART:
│
├── Check Dashboard "Status" row:
│   ├── "📉 REGIME BLOCK" → Score dan regime tidak aligned
│   │   └── FIX: Enable counter-trend mode (preset Aggressive/Day Trading)
│   │        atau tunggu regime shift
│   │
│   ├── "⚠️ COUNTER TREND" → Trend HMA melawan signal direction
│   │   └── FIX: Pertimbangkan turunkan `trend_counter_gate_loc` 
│   │        atau switch ke Adaptive mode
│   │
│   ├── "📉 DEAD VOLATILITY" → ATR sangat rendah
│   │   └── FIX: Tunggu market aktif, atau rendahkan `vol_low_gate`
│   │        (Aggressive mode sudah 0.60)
│   │
│   ├── "🧱 EXHAUSTION" → High volume + momentum decay
│   │   └── FIX: Normal — sistem hindari masuk saat puncak volatilitas
│   │        Tunggu 2-3 bar
│   │
│   └── "⌛ QUALIFYING" → Filters sedang building
│       └── FIX: Quality/confluence belum cukup — normal behavior
│
├── Check "Quality" row:
│   ├── Grade "D" (<0.40) → Data macro mungkin stale
│   │   └── FIX: Check API counter. Jika near 40/40, beberapa
│   │        data mungkin NA → disable unused features (MTF/projection)
│   │
│   └── Grade "C" (0.40-0.55) → Conditional quality
│       └── FIX: Turunkan `final_qual_min` jika terlalu strict
│
├── Check "Bayes" row:
│   └── Both <50% → No directional conviction
│       └── FIX: Market indecisive — no action needed (correct behavior)
│
└── Check "API" row:
    └── 40/40 → AT LIMIT
        └── FIX: Disable MTF confluence atau price projection untuk free up calls
```

### 3.2 Signal Terlalu Banyak (Noise/Whipsaw)

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| Signals setiap 1-2 bar | Aggressive mode + low quality floor | Switch ke Adaptive/Classic |
| Buy lalu Sell lalu Buy | `flip_lock_bars` terlalu rendah | Naikkan ke 2-3 |
| Signal di sesi Asia | Session filter disabled | Enable `enable_session_filter` |
| False divergence signals | DIV+ filters terlalu relaxed | Naikkan `div_quality_norm` |
| Repeated same-direction | `same_side_lock_bars` rendah | Naikkan ke 3-4 |

### 3.3 Dashboard Stale / Tidak Update

**Penyebab**: Dashboard hanya render pada `barstate.islast` (bar terakhir).

| Issue | Cause | Fix |
|-------|-------|-----|
| Action masih "NEUTRAL" padahal ada signal | Position-override belum trigger | Tunggu 1 bar, atau check `in_long/short_position` |
| Scanner masih kosong | Tidak ada posisi aktif | Normal — akan terisi saat signal fire |
| Score tidak bergerak | Macro TF belum close bar baru | Normal untuk H4 macro — score hanya update per 4H bar via `is_new_macro` |
| Regime tidak berubah | risk_off_index smoothed | Didesain lambat — hindari whipsaw regime. Butuh 2-3 macro bars |

### 3.4 API Limit Issues

**Total limit**: 40 `request.security()` calls.

**Jika error "request.security calls exceeded":**

| Fitur | API Calls | Safe to Disable? |
|-------|-----------|-----------------|
| MTF Confluence | 4 calls | ✅ Aman — signal utama tetap jalan |
| Price Projection | 0 extra* | — Reuses existing data |
| HTF Pattern Validation | 1 call | ✅ Hanya untuk pattern candle |
| XLP/XLY Ratio | 2 calls | ⚠️ Minor impact ke risk index |
| TLT/HYG Ratio | 2 calls | ⚠️ Minor impact (DGS20 sebagai fallback) |
| SPY Volume | 1 call | ✅ Hanya untuk vol divergence |

\* Price Projection tidak menambah API call karena reuses Z-scores yang sudah di-fetch.

### 3.5 Score Terjebak di Satu Nilai

**Penyebab umum:**
1. **Macro TF terlalu besar** — Jika `tf_macro = "D"`, score hanya berubah signifikan per hari
   - Fix: Gunakan preset yang set `tf_macro = "240"` untuk update per 4 jam
2. **Smoothing alpha terlalu rendah** — Score di-smooth agresif
   - Fix: Naikkan `smoothing_alpha` (0.15 → 0.25)
3. **Semua Z-scores near zero** — Market flat
   - Fix: Normal behavior — market tidak trending

### 3.6 Signal Muncul Terlambat (Lag)

| Sumber Lag | Besaran | Solusi |
|-----------|---------|--------|
| `barstate.isconfirmed` | 1 bar | Switch ke Aggressive mode |
| EMA smoothing | 2-4 bars | Naikkan `smoothing_alpha` |
| Macro TF data | Sampai 1 macro bar | Gunakan H1 hybrid fetch (preset Aggressive) |
| Bayesian smoothing | 1-2 bars | Inherent — tradeoff vs stability |
| HMA trend detection | HMA_len/2 bars | Kurangi `hma_len` (40 → 28) |

---

## 4. Maintenance Checklist

### 4.1 Saat Menambah Fitur Baru

- [ ] Check total line count — Pine Script v6 limit ≈ 5,500 lines
- [ ] Jika mendekati limit, **offload ke helper function** (pattern: `f_functionname()`)
- [ ] Check API call count — harus ≤ 40
- [ ] Tambahkan variabel baru ke `CircuitTracker` jika menggunakan data eksternal baru
- [ ] Update `FallbackZ` store jika menambah Z-score baru
- [ ] Test pada semua 3 preset
- [ ] Verify `barstate.isconfirmed` vs aggressive mode behavior

### 4.2 Saat Mengubah Signal Logic

- [ ] Test pada M1, M5, M15 secara terpisah (behavior berbeda per TF)
- [ ] Verify BUY dan SELL muncul seimbang (asymmetry check)
- [ ] Monitor rejected signals (`✕`) — rasio rejected/accepted yang sehat: 2:1 sampai 5:1
- [ ] Check exit logic — reversal-tagged entries harus punya grace period
- [ ] Validate dashboard "Status" row menampilkan blocker yang benar

### 4.3 Pine Script v6 Gotchas

| Issue | Symptom | Prevention |
|-------|---------|-----------|
| "Main function too long" | Compilation error | Pindahkan logic ke `f_*()` functions |
| Variable scope leak | Unexpected `na` values | Gunakan `:=` bukan `=` untuk persistent vars |
| Array growth | Memory/performance | Gunakan ring-buffer pattern |
| Label/Box limit | 500 per type | Implement cleanup functions (`f_clean_*`) |
| Mutable var in function | Compilation error | Pass as parameter, return modified value |

### 4.4 Data Availability Monitoring

FRED data (RRP, DFII10, Credit, DGS20) hanya update **sekali per hari** pada ~15:15 EST.
- Jika trading di luar jam tersebut, data menggunakan previous close
- Circuit Breaker melindungi dari stale data (counter system)
- Jika FRED endpoint berubah, Z-score akan fallback ke `FallbackZ` store

---

## 5. Variable Cross-Reference Index

### 5.1 Core Engine Variables

| Variable | Type | Line | Description |
|----------|------|------|-------------|
| `score_live` | float | [~1350](file:///d:/forex/titannew.pine#L1350) | Final smoothed macro score |
| `score_mom` | float | [~1355](file:///d:/forex/titannew.pine#L1355) | Score momentum (velocity) |
| `risk_off_index` | float | [~1200](file:///d:/forex/titannew.pine#L1200) | Master risk metric |
| `risk_state` | int | [~1260](file:///d:/forex/titannew.pine#L1260) | -2 to +2 regime state |
| `risk_state_extreme` | int | [~1265](file:///d:/forex/titannew.pine#L1265) | ±2 for extreme regimes |
| `quality_score` | float | [~1650](file:///d:/forex/titannew.pine#L1650) | Signal quality 0.0-1.0 |
| `bayes_conf_bull` | float | [~1540](file:///d:/forex/titannew.pine#L1540) | Bayesian bull probability |
| `bayes_conf_bear` | float | [~1541](file:///d:/forex/titannew.pine#L1541) | Bayesian bear probability |

### 5.2 Signal Variables

| Variable | Type | Line Range | Description |
|----------|------|------------|-------------|
| `final_buy_signal` | bool | [L3853](file:///d:/forex/titannew.pine#L3853) | Final buy trigger |
| `final_sell_signal` | bool | [L3854](file:///d:/forex/titannew.pine#L3854) | Final sell trigger |
| `buy_entry_signal` | bool | [L3907](file:///d:/forex/titannew.pine#L3907) | Position-aware buy |
| `sell_entry_signal` | bool | [L3908](file:///d:/forex/titannew.pine#L3908) | Position-aware sell |
| `all_filters_buy` | bool | [L3791](file:///d:/forex/titannew.pine#L3791) | All 18 filters pass |
| `all_filters_sell` | bool | [L3792](file:///d:/forex/titannew.pine#L3792) | All 18 filters pass |
| `regime_valid_buy` | bool | [L3593](file:///d:/forex/titannew.pine#L3593) | Regime allows buy |
| `regime_valid_sell` | bool | [L3608](file:///d:/forex/titannew.pine#L3608) | Regime allows sell |
| `trend_aligned_buy` | bool | [L3798](file:///d:/forex/titannew.pine#L3798) | HMA trend aligned |
| `trend_aligned_sell` | bool | [L3798](file:///d:/forex/titannew.pine#L3798) | HMA trend aligned |

### 5.3 Position Tracking

| Variable | Type | Line | Description |
|----------|------|------|-------------|
| `in_long_position` | bool | [L3864](file:///d:/forex/titannew.pine#L3864) | Currently in long |
| `in_short_position` | bool | [L3865](file:///d:/forex/titannew.pine#L3865) | Currently in short |
| `position_entry_bar` | int | [L3866](file:///d:/forex/titannew.pine#L3866) | Entry bar index |
| `position_entry_price` | float | [L3867](file:///d:/forex/titannew.pine#L3867) | Entry price |
| `position_is_reversal` | bool | [L3868](file:///d:/forex/titannew.pine#L3868) | Reversal entry flag |
| `position_peak_pnl` | float | [L3870](file:///d:/forex/titannew.pine#L3870) | Peak PnL for trailing |
| `exit_long` | bool | [L3976](file:///d:/forex/titannew.pine#L3976) | Exit long trigger |
| `exit_short` | bool | [L3977](file:///d:/forex/titannew.pine#L3977) | Exit short trigger |

### 5.4 Technical Analysis Flags

| Variable | Description | Line Range |
|----------|-------------|------------|
| `is_bull_div` / `is_bear_div` | RSI regular divergence | [~1760-1770](file:///d:/forex/titannew.pine#L1760) |
| `ao_reg_bull_recent` / `ao_reg_bear_recent` | AO regular divergence | [~2100-2110](file:///d:/forex/titannew.pine#L2100) |
| `power_div_bull` / `power_div_bear` | RSI+AO combined | [~2160-2170](file:///d:/forex/titannew.pine#L2160) |
| `is_whale_trap` | Whale trap active | [~2340](file:///d:/forex/titannew.pine#L2340) |
| `is_bottom_absorption` / `is_top_absorption` | Institutional absorption | [~1920](file:///d:/forex/titannew.pine#L1920) |
| `is_ignition` / `is_climax` / `is_churn` | Volume Price Analysis | [~1900-1910](file:///d:/forex/titannew.pine#L1900) |
| `bos_bull` / `choch_bull` | SMC structure breaks | [L3468-3471](file:///d:/forex/titannew.pine#L3468) |
| `cluster_tier_bull` / `cluster_tier_bear` | Candlestick cluster grade | [~3150-3160](file:///d:/forex/titannew.pine#L3150) |
| `div_buy_accurate` / `div_sell_accurate` | DIV+ confirmed signals | [L4322-4323](file:///d:/forex/titannew.pine#L4322) |

### 5.5 Dashboard Display Variables

| Variable | Description | Line |
|----------|-------------|------|
| `pred_dir` | Action direction string | [~1400](file:///d:/forex/titannew.pine#L1400) |
| `trade_status_txt` | Status row text | [L4549](file:///d:/forex/titannew.pine#L4549) |
| `setup_type` | Current setup classifier | [L4350](file:///d:/forex/titannew.pine#L4350) |
| `quality_grade` | Quality letter grade | [~1670](file:///d:/forex/titannew.pine#L1670) |
| `risk_reg_txt` | Regime display text | [~1280](file:///d:/forex/titannew.pine#L1280) |

### 5.6 Key Functions Reference

| Function | Purpose | Lines |
|----------|---------|-------|
| `f_get_macro_stats` | Fetch macro data | [L633-635](file:///d:/forex/titannew.pine#L633) |
| `f_calc_z_group` | Compute dual Z-scores | [L788-791](file:///d:/forex/titannew.pine#L788) |
| `f_choose_z` | Select best Z with fallback | [L906-911](file:///d:/forex/titannew.pine#L906) |
| `f_calculate_master_risk_index` | Risk index aggregation | [~L1140](file:///d:/forex/titannew.pine#L1140) |
| `f_calc_score_model` | Score computation | [~L1300](file:///d:/forex/titannew.pine#L1300) |
| `f_status_eval_advanced` | Quality score calc | [~L1600](file:///d:/forex/titannew.pine#L1600) |
| `f_get_institutional_setup` | Setup classifier | [~L1680](file:///d:/forex/titannew.pine#L1680) |
| `f_build_signal_filters` | 18-layer signal filter | [L3641](file:///d:/forex/titannew.pine#L3641) |
| `f_eval_smc_break_state` | BOS/CHoCH detection | [L3325](file:///d:/forex/titannew.pine#L3325) |
| `f_detect_fvg` | FVG quality grading | [L3267](file:///d:/forex/titannew.pine#L3267) |
| `f_create_ob_zone` | Order Block creation | [~L2530](file:///d:/forex/titannew.pine#L2530) |
| `f_run_projection_step` | Quantum price projection | [~L2780](file:///d:/forex/titannew.pine#L2780) |
| `f_render_standard_panel` | Dashboard main render | [L4598](file:///d:/forex/titannew.pine#L4598) |
| `f_get_dashboard_display` | Scanner row builder | [~L4828](file:///d:/forex/titannew.pine#L4828) |
| `f_signal_context` | Regime context labeling | [L3615](file:///d:/forex/titannew.pine#L3615) |
| `f_cluster_tip` | Cluster tooltip builder | [~L3170](file:///d:/forex/titannew.pine#L3170) |

---

> [!TIP]
> Dokumen lengkap terdiri dari 3 bagian:
> - **Part 1**: Arsitektur & fitur overview — [buka](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_documentation.md)
> - **Part 2**: Detail implementasi kode — [buka](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_deep_dive_part2.md)
> - **Part 3**: Scenario, tuning & troubleshooting — dokumen ini
