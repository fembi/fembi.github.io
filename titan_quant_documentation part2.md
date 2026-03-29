# 📖 Titan Quant v11 — Deep Dive Part 2: Detail Implementasi

> Dokumen ini melanjutkan [Part 1](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_documentation.md) dengan penjelasan **kode-level**, variabel spesifik, dan logic flow per subsystem.

---

## A. DATA PIPELINE — Dari Raw Data ke Z-Score

### A.1 Fetch Hierarchy

Script menggunakan **3 level timeframe** untuk data:

```
Level 1: "D" (Daily)     → FRED data (RRP, DFII10, Credit, DGS20) — hanya tersedia harian
Level 2: tf_macro         → Default "D", preset bisa override ke "240" (H4)
Level 3: tf_fetch_*       → Hybrid routing: DXY/US02Y/US10Y bisa di-fetch dari "60" (H1)
                             pada preset "Aggressive Scalping 1m"
```

**Kode** ([L623-627](file:///d:/forex/titannew.pine#L623-L627)):
```pine
string tf_fetch_dxy = preset_select == "Aggressive Scalping 1m" ? "60" : tf_macro
string tf_fetch_us02y = preset_select == "Aggressive Scalping 1m" ? "60" : tf_macro
string tf_fetch_us10y = preset_select == "Aggressive Scalping 1m" ? "60" : tf_macro
```

### A.2 Helper Functions

**`f_get_macro_stats(sym, tf, len)`** ([L633-635](file:///d:/forex/titannew.pine#L633-L635)):
- Mengambil 4 nilai dalam 1 `request.security()` call: `[close, close[1], sma(close[1], len), stdev(close[1], len)]`
- `close[1]` digunakan sebagai model value (menghindari lookahead bias)
- `sma` dan `stdev` dihitung dari `close[1]` series untuk konsistensi

**`f_get_macro_stats_vol_close(sym, tf, len)`** ([L649-650](file:///d:/forex/titannew.pine#L649-L650)):
- Sama seperti di atas + volume data: `[close, close[1], volume, volume[1], sma(volume[1], len), stdev(volume[1], len)]`
- Hanya digunakan untuk SPY (perlu volume untuk sentiment proxy)

### A.3 Z-Score Computation Detail

**Step 1 — Safety Functions** ([L781-785](file:///d:/forex/titannew.pine#L781-L785)):
```pine
f_val(val) => na(val) ? 0.0 : val
std_safe(std) => na(std) or std == 0.0 ? 1.0 : math.max(0.01, std)  // ← KRITIS: clamp ke 0.01
z_from(val, mean, std) => (f_val(val) - f_val(mean)) / std_safe(std)
```

> [!CAUTION]
> Tanpa `std_safe`, StdDev mikro (mis. 0.0001) menghasilkan Z-score 140,000+. Ini menyebabkan score meledak dan signal false.

**Step 2 — Dual Z-Score** ([L788-791](file:///d:/forex/titannew.pine#L788-L791)):
```pine
f_calc_z_group(float v, float m, float s, float v_m, float v_i) =>
    z_live_val = z_from(not na(v_i) ? v_i : v, m, s)   // Prioritas: intraday → RT
    z_mod_val  = z_from(v_m, m, s)                       // Model: previous close
    [z_live_val, z_mod_val]
```

**Step 3 — Selection dengan Fallback** ([L906-911](file:///d:/forex/titannew.pine#L906-L911)):
```pine
f_choose_z(valid_live, z_live_val, valid_model, z_model_val, last_ref) =>
    float z_chosen = valid_live ? z_live_val :
                     (valid_model ? z_model_val : last_ref)
    z_chosen
```

**Step 4 — Persistent Store Update** ([L934-967](file:///d:/forex/titannew.pine#L934-L967)):
- Setiap Z-score yang valid disimpan ke `FallbackZ store`
- Jika `z_sel_tlthyg` masih NA tapi DGS20 tersedia, gunakan `-z_tltyield` sebagai fallback

### A.4 Yield Curve Special Processing

([L763-771](file:///d:/forex/titannew.pine#L763-L771)):
```
yc_rt = US10Y - US02Y                    // Raw spread
yc_roc = (yc_rt - yc_rt[1]) / |yc_rt[1] + 0.001|   // ROC normalisasi
Z-score dihitung dari ROC, bukan raw spread
```
> Alasan: Raw spread bisa -0.5 ke +2.0 — tidak comparable dengan Z-score aset lain. ROC membuat semua Z-score berada di skala yang sama.

---

## B. CORRELATION ENGINE — Ring-Buffer O(1) Detail

### B.1 MacroCorr Object Structure

([L975-986](file:///d:/forex/titannew.pine#L975-L986)):
```pine
type MacroCorr
    array<float> arr_g      // Gold values ring buffer
    array<float> arr_a      // Asset values ring buffer
    float last_corr = 0.0   // Cached correlation result
    float sum_g = 0.0       // Running sum of gold values
    float sum_a = 0.0       // Running sum of asset values
    float sum_g2 = 0.0      // Running sum of gold²
    float sum_a2 = 0.0      // Running sum of asset²
    float sum_ga = 0.0      // Running sum of gold×asset
    int head = 0            // Ring buffer head pointer
    int count = 0           // Current sample count
    int cap = 0             // Buffer capacity
```

### B.2 Update Algorithm

([L988-1054](file:///d:/forex/titannew.pine#L988-L1054)):

```
Pada setiap macro timeframe change:
1. Grow buffer jika capacity < lookback length
2. Jika count > len (lookback reduced): trim oldest via head pointer
3. Jika count < len: insert at (head + count) % cap
4. Jika count == len: replace oldest at head, advance head
5. Update rolling moments (O(1) — hanya add new, subtract old)
6. Recalculate Pearson correlation dari moments
7. Clamp result ke [-1.0, 1.0]
```

> [!NOTE]
> Korelasi HANYA di-recalculate saat `is_new_macro` (macro TF bar change), bukan setiap tick. Ini menghemat CPU dan mencegah over-fitting ke noise intraday.

### B.3 Minimum Sample Enforcement

([L620-621](file:///d:/forex/titannew.pine#L620-L621)):
```pine
sample_min_eff := sample_min_mode == "Manual" ? sample_min_manual :
                  (risk_tol == "Low" ? 10 : risk_tol == "Medium" ? 8 : 7)
```
- Low risk tolerance → butuh minimal 10 sample untuk korelasi valid
- High risk tolerance → cukup 7 sample

---

## C. MASTER RISK INDEX — Weighted Aggregation Detail

### C.1 Component Accumulation

([L1144-1148](file:///d:/forex/titannew.pine#L1144-L1148)):
```pine
f_risk_accum(z_val, w_base, is_fear, mult) =>
    w_eff = is_fear ? w_base * mult : w_base    // Fear assets di-boost saat vol tinggi
    num = not na(z_val) ? z_val * w_eff : 0.0   // Skip jika NA
    den = not na(z_val) ? math.abs(w_eff) : 0.0
    [num, den]
```

### C.2 Volatility Multiplier

([L1141](file:///d:/forex/titannew.pine#L1141)):
```pine
mult_vol = risk_calc_mode == "Dynamic Vol-Adj" ? (1.0 + 0.5 * vol_score) : 1.0
```

| vol_score | mult_vol | Efek |
|-----------|----------|------|
| 0.0 (calm) | 1.0 | Normal weights |
| 0.5 (avg) | 1.25 | Sedikit amplifikasi |
| 1.0 (high) | 1.50 | Fear assets 50% lebih berpengaruh |

### C.3 Flow Arah Sign per Komponen

| Komponen | Sign dalam formula | Logic |
|----------|-------------------|-------|
| VIX | `+z_vix` | VIX naik → Risk-Off → Gold bullish |
| Yield | `-z_yield` | Yield naik → bearish gold |
| Credit | `+z_credit` | Credit spread naik → stress → bullish gold |
| DXY | `-z_dxy` | Dollar kuat → bearish gold |
| USDJPY | `-z_usdjpy` | Yen menguat (USDJPY turun) → risk-off |
| TLT/HYG | `+z_tlthyg` | Flight to quality → bullish gold |
| XLP/XLY | `+z_xlpxly` | Rotation ke defensif → risk-off |
| RRP | `-z_rrp` | RRP turun → likuiditas naik |
| YC | `-z_yc` | Yield curve inverts → recession fear |
| US02Y | `-z_us02y` | 2Y turun → dovish → bullish gold |
| US10Y | `-z_us10y` | 10Y turun → bullish gold |
| SPY Vol | `+z_spyvol × ret_sign` | High vol + SPY down → risk-off |

---

## D. BAYESIAN ENGINE — Implementasi Lengkap

### D.1 Prior Calculation (Asymmetric Sigmoid)

([L662-665](file:///d:/forex/titannew.pine#L662-L665)):
```pine
float macro_adj = macro_score < 0 ? macro_score * 1.5 : macro_score  // ← Penalti negatif
float prior_bull = 1.0 / (1.0 + math.exp(-macro_adj))                // Sigmoid
float prior_bear = 1.0 - prior_bull
```

| macro_score | macro_adj | prior_bull | prior_bear |
|-------------|-----------|------------|------------|
| +2.0 | +2.0 | 88% | 12% |
| +1.0 | +1.0 | 73% | 27% |
| 0.0 | 0.0 | 50% | 50% |
| -1.0 | -1.5 | 18% | 82% |
| -2.0 | -3.0 | 5% | 95% |

> Asimetri ini sengaja karena Risk-Off forces pada Gold sangat violent — macro negatif di-amplifikasi 1.5×.

### D.2 Evidence Vector (3 Factors)

| Evidence | P(E|Bull) if true | P(E|Bull) if false | P(E|Bear) if true | P(E|Bear) if false |
|----------|-------------------|--------------------|--------------------|---------------------|
| Trend (price > HMA) | 0.75-0.95* | 0.05-0.25 | 0.25 | 0.75 |
| Momentum (RSI > 50) | 0.60 | 0.40 | 0.45 | 0.55 |
| Volume (bull vol) | 0.60 | 0.40 | — | — |
| Volume (bear vol) | — | — | 0.60 | 0.40 |

\* `p_trend_up_given_bull` configurable via input, preset M5 Pro = 0.88

### D.3 Posterior Calculation

```
post_bull = prior_bull × L_trend_bull × L_mom_bull × L_vol_bull
post_bear = prior_bear × L_trend_bear × L_mom_bear × L_vol_bear
prob_final = post_bull / (post_bull + post_bear) × 100
```

### D.4 Adaptive Sniper Threshold

([L3646-3657](file:///d:/forex/titannew.pine#L3646-L3657)):
```
dynamic_sniper_thresh = base_thresh
  + vol_score × (TF-scaled addition: M1=2.2, M5=2.8, M15=3.6)
  + (if vol_score > 0.85: +1.0)
  - density_relaxation (Aggressive=4.8, Adaptive=3.1)
  - quality_relaxation (A+=3.4, A=2.3, B=1.3)
  - structure_relaxation (ignition=1.0, power_div=1.4, divergence=0.8)

Final = clamp(result, floor, 85.0)
Floor: Aggressive=52%, Adaptive=55%, Classic=62%
```

---

## E. SIGNAL FILTER PIPELINE — 18-Layer Detail

### E.1 Institutional Confluence Score

([L3690-3715](file:///d:/forex/titannew.pine#L3690-L3715)):

Skor 0-18+ dihitung dari 9 dimensi:

| Dimensi | Max Score | Logic |
|---------|-----------|-------|
| Risk Regime | 2 | Risk-Off/On = 2, transisi = 1 |
| Trend | 2 | Aligned = 2, neutral = 1 |
| Bayes Conf | 2 | ≥75% = 2, ≥65% = 1 |
| MTF Valid | 2 | Valid = 2 |
| Quality | 2 | >0.6 = 2, >0.5 = 1 |
| Volume Candle | 1 | High vol + directional + non-exhaustion |
| Purity Sync | 2 | XAU basket aligned |
| Cluster Tier | 2 | Tier 3 = 2, Tier 2 = 1 |
| AO Pattern | 1 | Twin Peaks or Saucer |

**Minimum required** (adaptive):
- M1: 2-3 (further reduced by mode/quality/cluster)
- M5: 3-4
- M15+: 3-5

### E.2 Chop Market Filter

([L3719-3726](file:///d:/forex/titannew.pine#L3719-L3726)):
```pine
ema_spread = abs(EMA(9) - EMA(21))
trend_spread = ema_spread / max(ATR14, mintick)
chop_market = trend_spread < chop_gate AND vol_ratio < 0.95 AND abs(score_mom) < 0.20
```
- Chop gate: Aggressive=0.09, Adaptive=0.11, Classic=0.14
- Override: Cluster Tier ≥ 2, Power Div, Stop Vol, Sweep, Ignition

### E.3 Signal Rearm System (5 Trigger Types)

| Trigger | Condition | Cooldown |
|---------|-----------|----------|
| **Crossover** | `ta.crossover(score, th)` | — |
| **First Alignment** | `all_filters AND NOT all_filters[1]` | — |
| **Momentum Rearm** | `score_mom crossover 0 + all_filters` | `signal_rearm_cd` bars |
| **Continuation** | `score > th × continue_mult + mom > 0` | `signal_rearm_cd + 1` bars |
| **Periodic Pulse** | Quality + confluence + divergence support | `periodic_rearm_cd` bars |
| **Confluence Pulse** | High quality + strong anchor signal | `confluence_pulse_cd` bars |

### E.4 Anti-Whipsaw Guards

1. **Flip Risk Guard** ([L3846-3847](file:///d:/forex/titannew.pine#L3846-L3847)):
   - Jika baru SELL lalu mau BUY dalam `flip_lock_bars`: block jika score dan quality rendah

2. **Same-Side Echo** ([L3850-3852](file:///d:/forex/titannew.pine#L3850-L3852)):
   - Jika baru BUY lalu mau BUY lagi dalam `same_side_lock_bars`: block jika quality < 0.78 dan bukan Power Div

3. **Simultaneous Signal Resolution** ([L3855-3857](file:///d:/forex/titannew.pine#L3855-L3857)):
   ```pine
   if final_buy_signal and final_sell_signal
       final_buy_signal := score_live > 0   // Ikuti arah score
       final_sell_signal := score_live < 0
   ```

---

## F. POSITION EXIT LOGIC — Reversal-Aware Detail

### F.1 Reversal Context Detection

([L3873-3874](file:///d:/forex/titannew.pine#L3873-L3874)):
Entry di-tag sebagai "reversal" jika:
- APEX signal (counter-trend extreme reversal)
- Power Div saat score masih rendah (< th × 0.65)
- Divergence saat score belum cross threshold
- Bottom/Top rejection saat score masih lemah

### F.2 Timeframe-Adaptive Exit Parameters

| Parameter | M1 | M5 | M15 | H1 |
|-----------|----|----|-----|-----|
| Grace bars | 5 | 4 | 3 | 2 |
| Fail score | 0.28 | 0.22 | 0.17 | 0.13 |
| Profit lock (%) | 0.25 | 0.40 | 0.55 | 0.80 |
| Stop multiplier | 1.55 | 1.38 | 1.22 | 1.10 |
| Trail giveback (%) | 0.20 | 0.28 | 0.36 | 0.46 |

### F.3 Volatility Adaptation

([L3892-3901](file:///d:/forex/titannew.pine#L3892-L3901)):
- **High vol (>0.80)**: Grace +1 bar, fail score +0.01, stop mult +0.05, giveback +0.04
- **Low vol (<0.35)**: Fail score -0.01, profit lock +0.05 (tighten to avoid overstaying)

### F.4 Exit Decision Tree

```
EXIT LONG jika salah satu:
├── Opposite SELL signal (dan bukan dalam grace period)
├── Hard Exit:
│   ├── Reversal mode:
│   │   ├── PnL ≤ -hard_stop_pct (ATR-based)
│   │   └── Setelah grace: score < -fail_score ATAU trend flip confirmed
│   └── Trend mode (scalping preset):
│       ├── Emergency flow flip (score < -0.85×th + bayes flipped)
│       ├── Trend flip + PnL < -0.20%
│       └── Price fail (ATR stop)
├── Profit Protection:
│   └── Peak PnL ≥ lock AND (momentum decay OR top rejection OR giveback ≥ trail)
└── Momentum Decay Take:
    └── Mom decay + PnL > profit_lock threshold
```

---

## G. VPA & WHALE TRAP — Logic Detail

### G.1 Volume Delta (CVD)

([L2414-2422](file:///d:/forex/titannew.pine#L2414-L2422)):
```pine
buy_pressure  = volume × (close - low) / (high - low)    // Proporsi volume ke sisi beli
sell_pressure = volume × (high - close) / (high - low)    // Proporsi volume ke sisi jual
bar_delta     = buy_pressure - sell_pressure               // Net: positif = net buying
cum_delta     = ta.cum(bar_delta)                          // Kumulatif
```

**CVD Divergence** (10-bar lookback):
- `cvd_div_bull`: Price turun tapi CVD naik → hidden accumulation
- `cvd_div_bear`: Price naik tapi CVD turun → hidden distribution

### G.2 Whale Trap v10.0 Fusion Logic

([L2310-2311](file:///d:/forex/titannew.pine#L2310-L2311)):

**Bull Trap** (bearish signal — harga akan turun):
```
wick_upper > 60% range
AND close < (high - 70% range)          // Deep reflux
AND candle_range > ATR × 1.35           // Volatility outlier
AND high > highest_5                     // Break structure
AND (purity_div OR at_supply_OB OR kill_zone OR triple_trap)  // Confluence
AND (macro_sync_bull OR VIX_stress OR bear_exhaustion)        // Macro
AND NOT in_cooldown(5 bars)
```

**Sequential Detection (Triple Trap)** ([L2298-2299](file:///d:/forex/titannew.pine#L2298-L2299)):
```pine
trap_ratio = sma(bullTrap[1] or bearTrap[1] ? 1.0 : 0.0, 20)
is_triple_trap = (trap_ratio × 20) >= 1.5   // ≈ 2+ traps in last 20 bars
```

### G.3 Absorption Detection

([L1919-1920](file:///d:/forex/titannew.pine#L1919-L1920)):
```
Bottom Absorption:
  high_vol AND narrow_spread(< 0.45 ATR) AND low ≤ lowest_10 AND close > (low + 40% range)
  → Institutional buying at extreme low with controlled price movement

Top Absorption:
  high_vol AND narrow_spread AND high ≥ highest_10 AND close < (high - 40% range)
  → Institutional selling at extreme high
```

---

## H. PRICE PROJECTION — Quantum Model Detail

### H.1 Sensitivity × Correlation × Z-Score

([L2787-2796](file:///d:/forex/titannew.pine#L2787-L2796)):
```
Kontribusi DXY = z_dxy × sensitivity(-1.20) × |correlation(gold,dxy)| × gamma(z_dxy)
```

**Gamma Boost** ([L2777-2778](file:///d:/forex/titannew.pine#L2777-L2778)):
```
|z| > 2.5 → ×1.50 (extreme move, high-conviction response)
|z| > 1.8 → ×1.25
else      → ×1.00
```

### H.2 Adjustments Pipeline

```
raw_projection
  × norm_factor (jika > 5 assets aktif, normalize ke baseline 5)
  × dampening (fungsi jarak price dari SMA200, max 0.6)
  × vol_mult (1 + vol_score × 0.80)
  × proj_conf (quality × confidence_weight × mtf_conf × session_mult)
  × exhaustion_mult (0.75 jika momentum decay)
  × mtf_alignment_mult (0.65 jika MTF conflict)
  × momentum_boost (1.10 jika macro+price agree, 0.92 jika tidak)

Final = clamp(-3.5%, +3.5%)
```

### H.3 Target Smoothing & BB Anchor

([L2857-2866](file:///d:/forex/titannew.pine#L2857-L2866)):
```pine
raw_target = gold_c × (1 + proj_move / 100)
// BB Anchor: prevent overshooting beyond 3σ
if not is_risk_extreme
    raw_target = proj_move > 0 ? min(raw_target, bb_upper_3σ) : max(raw_target, bb_lower_3σ)
// Exponential smoothing (α=0.5)
smoothed_target = prev_smoothed + 0.5 × (raw_target - prev_smoothed)
```

---

## I. ORDER BLOCK — Quantum Confluence Detail

### I.1 Zone Creation Trigger

([L2534-2535](file:///d:/forex/titannew.pine#L2534-L2535)):
```
OB Bull = is_bottom_absorption AND close ≥ open AND valid_demand_tail
OB Bear = is_top_absorption AND close ≤ open AND valid_supply_tail
```

Tail validation: wick ≥ 40% range ATAU close di upper/lower 60%

### I.2 Quantum Confluence Check

([L2539-2562](file:///d:/forex/titannew.pine#L2539-L2562)):
Zone mendapat label "QUANTUM" jika Price Projection target jatuh di dalam zone:

```
tolerance = max(zone_height × 10%, ATR × 0.25)
is_quantum = proj_target ∈ [zone_bottom - tol, zone_top + tol]

Confidence scoring:
  + precision (seberapa dekat target ke zone midpoint): max 50 pts
  + freshness (zone age < 20 bars = 20pts, < 50 = 14pts): max 20 pts
  + alignment (projection + macro = 20pts, salah satu = 10pts): max 20 pts

Visual: Ultra (≥80) = gold glow, Strong (≥60) = gold border, Standard = muted gold
```

---

## J. PATTERN CLUSTER — Weighted Scoring Detail

### J.1 Signal Weights

| Signal | Bull Weight | Bear Weight |
|--------|-------------|-------------|
| Liquidity Sweep | 2.50 | 2.50 |
| Power Div | 2.00 | 2.00 |
| Order Block Zone | 2.00 | 2.00 |
| Stopping Volume | 1.50 | 1.50 |
| CVD Divergence | 1.50 | 1.50 |
| Engulfing | 1.50 | 1.50 |
| Hammer/Star | 1.50 | 1.50 |
| Directional Climax | 1.00 | 1.00 |
| Ignition (aligned) | 1.00 | 1.00 |
| Twin Peaks | 1.00 | 1.00 |
| Test Volume | 1.00 | 1.00 |
| No Demand/Supply | 0.75 | 0.75 |
| Doji | 0.50 | 0.50 |
| Near Zone (no OB) | 0.25 | 0.25 |

### J.2 Tier Classification

```
APEX (Tier 3):   Score ≥ 5.5 AND has anchor (Sweep/PowerDiv/StopVol)
POWER (Tier 2):  Score ≥ 3.5
COMBO (Tier 1):  Score ≥ 2.0
```

### J.3 Quality Score Boost per Tier

```
APEX:  +0.16 to quality_score
POWER: +0.10
COMBO: +0.06
```

---

## K. DIV+ SIGNAL — 11-Layer Institutional Logic

### K.1 Full Filter Chain

([L4238-4318](file:///d:/forex/titannew.pine#L4238-L4318)):

| # | Layer | BUY Logic | SELL Logic |
|---|-------|-----------|------------|
| 1 | Momentum Context | RSI pernah < 35 dalam 15 bar | RSI pernah > 65 dalam 15 bar |
| 2 | RSI Divergence | RSI naik dari low, < 80 | RSI turun dari high, > 20 |
| 3 | Candle Quality | Close di upper 60%+ range | Close di lower 40%- range |
| 4 | Price Trigger | Bullish candle + strong close | Bearish candle + strong close |
| 5 | Momentum Accel | ta.rising(RSI,2) AND RSI > 38 | ta.falling(RSI,2) AND RSI < 62 |
| 6 | Volume Confirm | Volume > SMA20 × mult | Volume > SMA20 × mult |
| 7 | Quality Gate | quality_score > 0.45/0.55 | quality_score > 0.45/0.55 |
| 8 | Trend Context | score < 0.30 OR trend_bear | score > -0.30 OR trend_bull |
| 9 | Regime Check | NOT Extreme Risk-On | NOT Extreme Risk-Off |
| 10 | Session | Optimal session OR (non-Asian + quality > 0.60) |
| 11 | Extreme Vol Gate | Lower wick reject + not impulse + AO/OB confluence + Bayes |

**Primary Path**: All 11 layers pass
**Fallback Path**: Layers 1-6,8-11 pass + (high_vol OR AO_recent OR power_div)

---

## L. REALTIME EXECUTION MODES

### L.1 Signal Timing

([L239-241](file:///d:/forex/titannew.pine#L239-L241)):

| Mode | Behavior | Use Case |
|------|----------|----------|
| **Balanced (No Spam)** | Signals on `barstate.isconfirmed` only | Default — stable |
| **Aggressive (Per Tick)** | Signals on first state change within bar | Ultra-fast scalping |

### L.2 Intrabar Anti-Spam

([L4013-4033](file:///d:/forex/titannew.pine#L4013-L4033)):
- 10 separate `*_fired` latches reset pada `barstate.isnew`
- Mencegah event yang sama muncul berkali-kali dalam 1 bar saat mode Aggressive

### L.3 Position-Aware Dashboard Override

([L3993-4003](file:///d:/forex/titannew.pine#L3993-L4003)):
```pine
if in_short_position
    pred_dir := trend_bear ? "STRONG SELL" : "SNIPER SELL"
    action_score_disp := -abs(action_score_proj)    // Force Action display negatif
else if in_long_position
    pred_dir := trend_bull ? "STRONG BUY" : "SNIPER BUY"
    action_score_disp := abs(action_score_proj)     // Force Action display positif
```
> Ini memastikan dashboard "Action" row selalu sinkron dengan posisi aktif, bukan score_live yang lambat berubah di macro TF.

---

## M. API BUDGET MANAGEMENT

### M.1 Call Counting

([L5029-5051](file:///d:/forex/titannew.pine#L5029-L5051)):

| Category | Count |
|----------|-------|
| Core (Gold chart/macro/analysis + XAU basket + SPY) | 7 |
| Primary macro (RRP, Yield, ETF, VIX, DXY, SPX) | 6 |
| Secondary (Credit, USDJPY, USDCHF) | 3 |
| Ratios (TLT+HYG, XLP+XLY → 2 calls each) | 4 |
| Additional (SPY vol, DGS20, US02Y, US10Y) | 4 |
| MTF Confluence | 0-4 |
| HTF Pattern Validation | 0-1 |
| **Total** | **~28-32 / 40 limit** |

### M.2 Optimisasi
- Intraday data di-reuse dari macro RT (menghindari separate call)
- MTF reuses `current_tf_trend` jika TF sama dengan chart
- HTF pattern validation hanya di-fetch jika ada pattern candidate (lazy fetch)
- FRED data di-hardcode ke "D" (tidak perlu separate intraday call)

---

> [!TIP]
> Untuk memahami interaksi antar subsistem, baca bagian **23. Flow Diagram** di [Part 1](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_documentation.md) terlebih dahulu, lalu gunakan referensi line number di dokumen ini untuk tracing kode spesifik.
