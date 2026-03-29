# 📖 Titan Quant v11 — Part 5: Formula-Level Deep Dive

> Lanjutan dari [Part 1-4](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_documentation.md). Bagian ini membahas **formula matematika** dan **algoritma detail** setiap subsistem.

---

## 📑 Daftar Isi

1. [RSI Divergence Algorithm](#1-rsi-divergence-algorithm)
2. [AO Divergence v2.0 Internals](#2-ao-divergence-v20-internals)
3. [Candlestick Strength Scoring Formulas](#3-candlestick-strength-scoring-formulas)
4. [Macro Vote — 26 Factor Detail](#4-macro-vote--26-factor-detail)
5. [Score Model Computation Chain](#5-score-model-computation-chain)
6. [VPA Event Detection Formulas](#6-vpa-event-detection-formulas)
7. [Order Block Lifecycle](#7-order-block-lifecycle)
8. [Edge Cases & Known Limitations](#8-edge-cases--known-limitations)

---

## 1. RSI Divergence Algorithm

### 1.1 Pivot Detection

([L1980-1983](file:///d:/forex/titannew.pine#L1980-L1983)):
```pine
p_r = 2, p_l = 2                          // 2 bar lookahead, 2 bar lookback
rsi_pl = ta.pivotlow(rsi_v, p_l, p_r)     // RSI pivot low
rsi_ph = ta.pivothigh(rsi_v, p_l, p_r)    // RSI pivot high
```

> Pivot terdeteksi pada offset `p_r` (2 bar lalu). Ini berarti saat RSI pivot muncul, harga sudah bergerak 2 bar dari titik pivot.

### 1.2 Divergence Matching Logic

([L2000-2072](file:///d:/forex/titannew.pine#L2000-L2072)):

**Regular Bullish** (Reversal signal):
```
Kondisi: Price membuat Lower Low, RSI membuat Higher Low
→ Harga turun tapi momentum sudah mulai naik = potensi reversal naik

curr_p_price_l < last_p_price_l     // Price: lower low
AND curr_p_rsi_l > last_p_rsi_l     // RSI: higher low
AND curr_p_rsi_l < 65               // Strict tier (< 80 for relaxed)
AND bars_diff < 100                 // Within 100 bars
```

**Hidden Bullish** (Continuation signal):
```
Kondisi: Price membuat Higher Low, RSI membuat Lower Low
→ Trend naik masih kuat meskipun RSI dip = momentum continuation

curr_p_price_l > last_p_price_l     // Price: higher low
AND curr_p_rsi_l < last_p_rsi_l     // RSI: lower low
```

**Regular Bearish** (Reversal signal):
```
curr_p_price_h > last_p_price_h     // Price: higher high
AND curr_p_rsi_h < last_p_rsi_h     // RSI: lower high
AND curr_p_rsi_h > 35               // Strict tier (> 20 for relaxed)
```

**Hidden Bearish** (Continuation signal):
```
curr_p_price_h < last_p_price_h     // Price: lower high
AND curr_p_rsi_h > last_p_rsi_h     // RSI: higher high
```

### 1.3 RSI Strength Scoring (5-Component, Max 100)

([L2022-2028](file:///d:/forex/titannew.pine#L2022-L2028)):

| # | Component | Formula | Max | Penjelasan |
|---|-----------|---------|-----|-----------|
| 1 | RSI Magnitude | `min(30, rsi_gap / 5 × 10)` | 30 | Berapa besar perbedaan RSI antar pivot |
| 2 | RSI Zone | `<35→25, <45→18, <55→10, else→5` | 25 | Semakin oversold/overbought, semakin kuat |
| 3 | Distance | `10-50 bars→15, ≥5→10, else→5` | 15 | Jarak ideal antar pivot (10-50 bar) |
| 4 | Tier Quality | `Strict→20, Relaxed→10` | 20 | Divergence strict lebih bernilai |
| 5 | Candle Confirm | `Close > Open→10, else→3` | 10 | Konfirmasi arah candle |

**Contoh perhitungan:**
```
RSI pivot low sebelumnya: 32, sekarang: 40 (gap = 8)
rs1 = min(30, 8/5 × 10) = min(30, 16) = 16
rs2 = 40 < 45 → 18
rs3 = bars_diff = 25 (ideal range) → 15
rs4 = Strict (RSI < 65) → 20
rs5 = Close > Open → 10
TOTAL = 16 + 18 + 15 + 20 + 10 = 79/100 (Kuat)
```

### 1.4 Memory Window

([L2074-2093](file:///d:/forex/titannew.pine#L2074-L2093)):
```pine
bull_div_recent = (bar_index - last_div_bull_bar) <= 60   // 60 bar window
bear_div_recent = (bar_index - last_div_bear_bar) <= 60
```
- Divergence "stays active" selama 60 bar setelah terdeteksi
- Ini memungkinkan sinyal DIV+ atau Sniper terbentuk bahkan setelah pivot sudah lewat beberapa bar

---

## 2. AO Divergence v2.0 Internals

### 2.1 VWAO (Volume-Weighted Awesome Oscillator)

([L2100-2130](file:///d:/forex/titannew.pine#L2100-L2130)):
```pine
// Adaptive compression based on volatility
ao_compress = max(0.65, min(1.0, 1.0 - (atr_cur / atr_avg - 1.0) * 0.5))

// Dynamic periods
ao_fast_eff = round(ao_fast_len * ao_compress)    // e.g., 5 × 0.85 = 4
ao_slow_eff = round(ao_slow_len * ao_compress)    // e.g., 34 × 0.85 = 29

// Volume-Weighted Median Price
vwao_src = ta.vwma(hl2, 2)                        // hl2 = (high+low)/2

// AO = Fast SMA - Slow SMA of VWMA(hl2)
ao_val = ta.sma(vwao_src, ao_fast_eff) - ta.sma(vwao_src, ao_slow_eff)
```

**Kelebihan VWAO vs standard AO:**
- Volume weighting mengurangi noise dari bar bervolume rendah
- Adaptive period compression mempersingkat lookback saat volatilitas tinggi

### 2.2 Six Accuracy Upgrades

| # | Upgrade | Implementation | Dampak |
|---|---------|---------------|--------|
| 1 | **Min Divergence Gap** | `abs(ao_pivot_now - ao_pivot_prev) > ATR × 0.001` | Menghindari micro-divergence yang tidak bermakna |
| 2 | **Candle Confirmation** | `close > open` (bull) atau `close < open` (bear) requirement | Mengurangi false positive |
| 3 | **AO Zone Filter** | AO harus di bawah nol untuk bull, di atas nol untuk bear | Memastikan divergence di zona yang benar |
| 4 | **Adaptive Pivot** | `ao_pivot_len` = TF-based: M1=3, M5=3, M15=4, H1=5 | Noise reduction per timeframe |
| 5 | **Strength Scoring** | 6-component scoring (max 100) — lihat tabel di bawah | Grading untuk filter & display |
| 6 | **Tight Standalone Filters** | Vol, RSI, quality, strength gate requirements | Standalone AO signals berkualitas |

### 2.3 AO Strength Scoring (6-Component, Max 100)

| # | Component | Max | Bull Logic |
|---|-----------|-----|-----------|
| 1 | AO Magnitude | 30 | `min(30, ao_gap / ao_range × 10)` |
| 2 | AO Zone Depth | 25 | AO deep below zero → 25, moderate → 18, etc. |
| 3 | Pivot Distance | 15 | 8-40 bars optimal → 15 |
| 4 | Candle Confirm | 10 | Close > Open → 10 |
| 5 | Volume Z | 10 | `vol_z > 2.0→10, >1.0→6, >0→3` |
| 6 | Velocity Bonus | 10 | `ao_velocity > 0→10` (momentum turning) |

### 2.4 AO Patterns

**Twin Peaks** (Bill Williams):
```
Bullish: Dua valley berurutan di AO < 0
         Valley kedua lebih tinggi dari pertama
         AO berubah dari merah ke hijau di antara keduanya
```

**Saucer** (Bill Williams):
```
Bullish: AO > 0, tiga bar berurutan
         ao[2] > ao[1] (turun)
         ao[1] < ao[0] (naik kembali) — bentuk "piring"
```

### 2.5 AO Velocity & Acceleration

```pine
ao_velocity = ta.roc(ao_val, 3)       // Rate of change 3 bar
ao_accel = ao_velocity - ao_velocity[1]  // Acceleration (velocity of velocity)
```

Digunakan untuk:
- Quality score booster: high velocity menambah +0.06
- Signal rearm trigger: velocity crossing zero
- Power Div strength bonus

---

## 3. Candlestick Strength Scoring Formulas

### 3.1 Hammer/Shooting Star (7-Component, Max 115 → capped 100)

([L1706-1725](file:///d:/forex/titannew.pine#L1706-L1725)):

| # | Component | Formula (Bull Hammer) | Max |
|---|-----------|----------------------|-----|
| 1 | Wick Ratio Quality | `min(25, (lower_wick/body) / 4 × 25)` | 25 |
| 2 | ATR-Relative Size | `min(20, (range/ATR) × 12)` | 20 |
| 3 | Volume Z-Score | `>2.0→20, >1.0→12, >0→7, else→3` | 20 |
| 4 | Context Depth | `min(15, abs(close-baseline)/ATR × 12)` atau 5 | 15 |
| 5 | MTF Alignment | `htf_bull ? 10 : 3` | 10 |
| 6 | BB Proximity | `low ≤ BB-1σ→10, ≤ BB-0.5σ→5, else→0` | 10 |
| 7 | OB Proximity | `at_ob_demand→15, near_zone→7, else→0` | 15 |

**Max teoritis**: 115 → **Capped di 100** via `math.min(100, total)`

### 3.2 Engulfing (7-Component, Max 120 → capped 100)

([L1768-1787](file:///d:/forex/titannew.pine#L1768-L1787)):

| # | Component | Formula (Bull Engulfing) | Max |
|---|-----------|-------------------------|-----|
| 1 | Body Dominance | `min(25, (body/prev_range) × 20)` | 25 |
| 2 | ATR-Relative | `min(20, (range/ATR) × 12)` | 20 |
| 3 | Volume Z | `>2.0→20, >1.0→12, >0→7, else→3` | 20 |
| 4 | Context Depth | Sama dengan Hammer | 15 |
| 5 | MTF | `htf_bull ? 10 : 3` | 10 |
| 6 | Power Move | `body > 1.5× prev_range → 10` | 10 |
| 7 | OB Proximity | `at_ob→15, near→7, else→0` | 15 |

### 3.3 Liquidity Sweep (7-Component, Max 115 → capped 100)

([L1801-1825](file:///d:/forex/titannew.pine#L1801-L1825)):

| # | Component | Formula (Bull Sweep) | Max |
|---|-----------|---------------------|-----|
| 1 | Sweep Depth | `min(25, abs(swing - low) / ATR × 30)` | 25 |
| 2 | ATR-Relative | `min(20, (range/ATR) × 12)` | 20 |
| 3 | Volume Z | `>2.0→20, >1.0→12, else→7` | 20 |
| 4 | Close Direction | `close > open → 15, else → 5` | 15 |
| 5 | MTF | `htf_bull ? 10 : 3` | 10 |
| 6 | BB Proximity | `low ≤ BB-1σ→10, else→5` | 10 |
| 7 | OB Proximity | OB bonus | 15 |

### 3.4 Grade System

([L1828](file:///d:/forex/titannew.pine#L1828)):
```pine
f_pat_grade(strength) =>
    strength > 80 ? "S" : strength > 60 ? "A" : strength > 40 ? "B" : strength > 20 ? "C" : "D"
```

Grade ditampilkan pada label chart dan tooltip.

---

## 4. Macro Vote — 26 Factor Detail

### 4.1 Fungsi `f_macro_vote`

([L1222-1259](file:///d:/forex/titannew.pine#L1222-L1259)):

#### Risk-Off (Gold Bullish) — 13 Factors

| # | Factor | Condition | Threshold |
|---|--------|-----------|-----------|
| 1 | Real Yield ↓ | `z_yield < -0.3` | Z < -0.3 |
| 2 | VIX Elevated | `19 ≤ VIX ≤ 40` | Range 19-40 |
| 3 | SPX Down | `spx_roc < 0` | Negative ROC |
| 4 | DXY Weak | `z_dxy < -0.2` | Z < -0.2 |
| 5 | Credit Spread ↑ | `z_credit > 0.3` | Z > +0.3 |
| 6 | Yen Strong | `z_usdjpy < -0.2` | Z < -0.2 |
| 7 | CHF Strong | `z_usdchf < -0.2` | Z < -0.2 |
| 8 | TLT > HYG | `z_tlthyg > 0.2` | Z > +0.2 |
| 9 | XLP > XLY | `z_xlpxly > 0.2` | Z > +0.2 |
| 10 | US10Y ↓ | `z_us10y < -0.3` | Z < -0.3 |
| 11 | US02Y ↓ | `z_us02y < -0.3` | Z < -0.3 |
| 12 | YC Inverted/Steepen | `z_yc < -0.2` OR (`z_yc > 0.1` AND `z_us02y < -0.3`) | Composite |
| 13 | GLD Inflow | `z_etf > 0.2` | Z > +0.2 |

#### Risk-On (Gold Bearish) — 13 Factors

| # | Factor | Condition | Special Logic |
|---|--------|-----------|---------------|
| 1 | Real Yield ↑ | `z_yield > 0.2` | — |
| 2 | VIX Low/Panic | `VIX < 19 OR VIX > 40` | VIX > 40 = forced liquidation |
| 3 | SPX Up | `spx_roc > 0` | — |
| 4 | DXY Strong | `z_dxy > 0.2` | — |
| 5 | Credit Tight | `z_credit < -0.1` | — |
| 6 | Yen Weak | `z_usdjpy > 0.2` | — |
| 7 | CHF Weak | `z_usdchf > 0.2` | — |
| 8 | HYG > TLT | `z_tlthyg < -0.2` | — |
| 9 | XLY > XLP | `z_xlpxly < -0.2` | — |
| 10 | US10Y ↑ | `z_us10y > 0.2 AND z_yield > -0.1` | ⚠️ Exception: only bearish if Real Yield stable |
| 11 | US02Y ↑ | `z_us02y > 0.3` | — |
| 12 | YC Normalizing | `z_yc > 0.2` | — |
| 13 | GLD Outflow | `z_etf < -0.2` | — |

> [!IMPORTANT]
> **Factor #10 Exception** (US10Y): Jika US10Y naik TAPI Real Yield turun (z_yield < -0.1), ini berarti **inflasi meroket**, yang sebenarnya **BULLISH Gold**. Exception ini mencegah false bearish vote.

### 4.2 Confirmation Threshold

([L1264-1271](file:///d:/forex/titannew.pine#L1264-L1271)):
```pine
macro_off = vote_off >= 3    // Minimal 3/13 = confirmed Risk-Off
macro_on  = vote_on  >= 3    // Minimal 3/13 = confirmed Risk-On

// Final: Require BOTH risk_state alignment + vote confirmation
// Fallback: If vote >= 1, still accept if risk_state already aligned
is_risk_off = (risk_state == 1 AND macro_off) OR (risk_state == 1 AND vote_off >= 1)
is_risk_on  = (risk_state == -1 AND macro_on) OR (risk_state == -1 AND vote_on >= 1)
```

---

## 5. Score Model Computation Chain

### 5.1 Complete Pipeline (5 Stages)

**Stage 1: Weighted Score Model** ([L1277-1283](file:///d:/forex/titannew.pine#L1277-L1283)):
```
score_raw = Σ(z_i × w_corr_i × w_risk_i) / Σ|w_corr_i × w_risk_i|

Dimana:
  z_i      = Z-score terpilih untuk aset i
  w_corr_i = Korelasi Gold-vs-Aset (dari correlation engine, range -1 to +1)
  w_risk_i = Risk weight dari input (e.g., DXY=1.80, VIX=1.65)
```

**Stage 2: Responsiveness Delta** ([L1294-1303](file:///d:/forex/titannew.pine#L1294-L1303)):
```
delta_i = z_live_i - z_model_i         // Seberapa jauh current dari baseline
weighted_delta = Σ(delta_i × sign_i × w_risk_i) / Σ|w_risk_i|

score = score_raw + resp_gain × weighted_delta
```
> Delta memberikan "real-time bonus/penalty" di atas model baseline

**Stage 3: Fallback Guard** ([L1285-1290](file:///d:/forex/titannew.pine#L1285-L1290)):
```
Jika active_count < 2 (kurang dari 2 aset aktif):
  score = simple_average(all_valid_z_scores)   // Fallback sederhana
```

**Stage 4: Hyper-Response Smoothing** ([L1306-1316](file:///d:/forex/titannew.pine#L1306-L1316)):
```
alpha_target = smooth_a × (0.5 + 0.5 × vol_score)   // Vol-aware alpha
alpha_live = clamp(alpha_prev ± 0.30, 0.10, 1.0)     // Step-limited change
score_live = score_prev + alpha × (score_raw - score_prev)   // EMA
```

**Stage 5: Velocity Projection** ([L1331-1333](file:///d:/forex/titannew.pine#L1331-L1333)):
```
mom_action = score_live - score_live[1]
action_score_proj = score_live + (mom_action × 2.5)   // Project 2-3 bars
```

### 5.2 Asymmetric Threshold Calculation

([L1108-1118](file:///d:/forex/titannew.pine#L1108-L1118)):
```
base_th = model_thresh × (1.0 + 0.15 × vol_score)
base_th = clamp(base_th, model_thresh×0.5, model_thresh×1.3)

th_buy  = VIX > 18 ? base_th × 0.70 : base_th × 0.85
th_sell = VIX < 17 ? base_th × 0.90 : base_th × 1.10

Contoh (model_thresh=0.30, vol_score=0.5):
  base_th = 0.30 × 1.075 = 0.32
  VIX = 25 → th_buy = 0.32 × 0.70 = 0.225  (mudah trigger buy)
            th_sell = 0.32 × 1.10 = 0.354  (sulit trigger sell)
```

---

## 6. VPA Event Detection Formulas

### 6.1 Volume Z-Score

([L1861-1867](file:///d:/forex/titannew.pine#L1861-L1867)):
```pine
vol_z_score = (volume - SMA(volume,20)) / StdDev(volume,20)

is_ultra_vol = vol_z_score > 3.0    // 3σ event (~0.3% probability)
is_high_vol  = vol_z_score > 2.0    // 2σ event (~2.3% probability)
```

### 6.2 Event Detection

| Event | Condition | Code Line |
|-------|-----------|-----------|
| **Ignition** | `vol_z > 2.5 AND body > 70%range AND breakout_elite AND range > 1.2×avg` | [L1881](file:///d:/forex/titannew.pine#L1881) |
| **Climax** | `ultra_vol AND body < 45%range` (rejection) | [L1885](file:///d:/forex/titannew.pine#L1885) |
| **Churn** | `high_vol AND range < 0.9×avg AND body < 50%range` | [L1891](file:///d:/forex/titannew.pine#L1891) |
| **Absorption** | `high_vol AND narrow_spread(<0.45ATR) AND extreme_location AND recovery_close` | [L1919](file:///d:/forex/titannew.pine#L1919) |
| **Stopping Vol** | `ultra_vol AND trend_extreme AND reversal_close` | L2380+ |
| **No Demand** | `low_vol AND narrow_bar AND uptrend` | L2390+ |
| **Test Vol** | `low_vol AND at_OB_zone` | L2400+ |

### 6.3 Vol Booster Impact Chain

([L1923-1933](file:///d:/forex/titannew.pine#L1923-L1933)):
```
Ignition      → +0.20 (strong momentum confirmation)
Absorption    → +0.15 (institutional activity)
Climax        → +0.05 (cautionary — could be exhaustion)
Standard Vol  → +0.05 (generic volume surge)
Churn         → -0.10 (PENALTY — wasted energy)
```

---

## 7. Order Block Lifecycle

### 7.1 Creation

```
Trigger: Absorption event (bottom/top) + directional close + valid tail
Storage: Array of objects (max 3 per side, FIFO)
Data: [top, bottom, bar_index, confluence_score, star_rating]
```

### 7.2 Confluence Scoring (On Creation)

7 factors, total max 100 → converted to 1-5 stars:

| Factor | Max Points | Logic |
|--------|-----------|-------|
| Volume | 20 | `ultra_vol→20, high_vol→15, above_avg→10` |
| Macro Score | 15 | `abs(score) > 0.3→15, >0.15→10` |
| MTF | 15 | `mtf_valid→15, partial→8` |
| Session | 15 | `kill_zone→15, london/ny→10, asian→3` |
| Body % | 15 | `body > 80%range→15, >60%→10` |
| RSI | 10 | `oversold/overbought→10, extended→6` |
| Bayes | 10 | `bayes > 75%→10, >65%→6` |

Star rating: `score/20` → 1-5 stars ⭐

### 7.3 Monitoring (Per Bar)

```
For each active zone:
  1. Check mitigation: close below demand / above supply → REMOVE
  2. Check expiry: (bar_index - creation_bar) > 200 → REMOVE
  3. Check proximity: current price near zone → set at_ob_demand/supply = true
  4. Check quantum: projected_target within zone ± tolerance → upgrade visual
```

### 7.4 Cluster Detection

```
If new zone overlaps with existing zone by ≥ 40%:
  → Merge (average boundaries, sum confluence scores)
  → Display "Cluster" label
```

### 7.5 Disposal

```
Reasons for removal:
  1. Mitigation (price close through zone) → zone "consumed"
  2. Expiry (>200 bars old) → zone too old to be relevant
  3. FIFO overflow (>3 zones) → oldest removed when new created
  4. Box cleanup (>40 boxes) → oldest visual deleted
```

---

## 8. Edge Cases & Known Limitations

### 8.1 Data Availability

| Issue | Impact | Mitigation |
|-------|--------|-----------|
| FRED data delayed (weekends/holidays) | Z-scores stale 2-3 days | FallbackZ store + Circuit Breaker |
| VIX unavailable (market closed) | Missing fear component | Risk index recalculates with reduced denominator |
| TradingView API rate limit | `request.security` calls fail | Hardcoded limit tracking (API counter in dashboard) |
| XAU basket (XAUEUR/XAUJPY) NA | Intrinsic strength unavailable | `intrinsic_neutral = true` → no intrinsic filter |

### 8.2 Pine Script Constraints

| Constraint | Limit | Current Usage | Impact |
|-----------|-------|---------------|--------|
| Script body length | ~5,500 lines | ~5,070 | ⚠️ Near limit — modularize new features |
| `request.security()` | 40 calls | 28-32 | ✅ Safe, 8-12 free |
| Labels per script | 500 | Dynamic | Auto-cleaned via cooldown logic |
| Boxes per script | 500 | ~80 active | Auto-cleaned via `f_clean_boxes()` |
| Lines per script | 500 | ~50 active | Auto-cleaned via `f_clean_lines()` |
| Table cells | 4×50 = 200 | Using most | Dashboard row tracking prevents overflow |

### 8.3 Signal Edge Cases

| Edge Case | Behavior | Why |
|-----------|----------|-----|
| **BUY and SELL on same bar** | Score direction wins | [L3855-3857](file:///d:/forex/titannew.pine#L3855) — `score > 0 → buy, else sell` |
| **Signal pada bar pertama** | Suppressed | `barstate.isfirst` guards on smoothing vars |
| **Extreme Z-score (>5σ)** | Risk index clamped to ±3.0 | [L1170](file:///d:/forex/titannew.pine#L1170) — prevents score blowout |
| **Gap up/down (limit move)** | VPA may fire incorrectly | `candle_range` check against ATR provides guard |
| **Realtime repaint** | Confirmed mode: no repaint | `barstate.isconfirmed` ensures final value |
| **Aggressive mode repaint** | Signal may appear/disappear | By design — `*_fired` latches prevent duplicates |
| **Midnight timezone rollover** | Session flags briefly incorrect | UTC-based sessions avoid most issues |
| **Score = exactly 0** | Regime = Neutral, no signal | `score > 0.08` / `< -0.08` required for regime validity |

### 8.4 Performance Considerations

| Concern | Solution in Script |
|---------|-------------------|
| Correlation computation cost | Ring-buffer O(1) + only on `is_new_macro` |
| Table rendering cost | Only on `barstate.islast` |
| Array growth (OB, FVG, lines) | FIFO caps + `f_clean_*` functions |
| Multiple `ta.highest/lowest` | Pre-computed into variables, reused |
| Long if-else chains | Offloaded to `f_*()` functions |

### 8.5 Instrumentasi untuk Debugging

| Tool | Cara Akses |
|------|-----------|
| Score status line | Status line bawah chart (selalu aktif) |
| Risk index status line | Status line bawah chart (selalu aktif) |
| Dashboard regime row | Panel: lihat regime + extreme warning |
| Dashboard status row | Panel: menunjukkan blocker aktif |
| Rejected labels (✕) | Chart: `cond_bias_buy/sell` true tapi filter gagal |
| API counter | Dashboard row terakhir |
| SMC bypass mode | `smc_bypass_filters = true` → raw detection tanpa filter |
| Quality score display | Dashboard: grade + numeric score |
| Tooltip hover | Hover label di chart → detail breakdown |

---

> [!TIP]
> **Dokumentasi lengkap Titan Quant v11 — 5 bagian:**
> - [Part 1](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_documentation.md) — Arsitektur & fitur (23 sections)
> - [Part 2](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_deep_dive_part2.md) — Detail implementasi kode (13 sections)
> - [Part 3](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_part3_practical.md) — Scenario, tuning & troubleshooting (5 sections)
> - [Part 4](file:///C:/Users/fembi/.gemini/antigravity/brain/3c407812-46ba-4401-a03f-1ebddc94faeb/titan_quant_part4_reference.md) — Input catalog & dependency map (9 sections)
> - **Part 5** — Formula-level deep dive (dokumen ini, 8 sections)
