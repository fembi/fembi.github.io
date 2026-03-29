# 📖 Titan Quant v11 — Part 6: SMC Engine, Quality Walkthrough & Glossary

> Bagian terakhir dari dokumentasi Titan Quant v11, melengkapi coverage seluruh 5,070 baris kode.

---

## 📑 Daftar Isi

1. [SMC Structure Engine — Complete Internals](#1-smc-structure-engine--complete-internals)
2. [Quality Score — Complete Computation Walkthrough](#2-quality-score--complete-computation-walkthrough)
3. [Power Divergence v2.0 — Weighted Fusion Detail](#3-power-divergence-v20--weighted-fusion-detail)
4. [VPA ↔ Bayesian Feedback Loop](#4-vpa--bayesian-feedback-loop)
5. [Whale Trap v10.0 — Complete Code-Level Breakdown](#5-whale-trap-v100--complete-code-level-breakdown)
6. [Glossary — Semua Singkatan & Istilah](#6-glossary--semua-singkatan--istilah)

---

## 1. SMC Structure Engine — Complete Internals

### 1.1 Fractal Pivot Detection

([L3444-3465](file:///d:/forex/titannew.pine#L3444-L3465)):

**Dual-Scale Pivots:**

| Scale | Function | Lookback | Purpose |
|-------|----------|----------|---------|
| Minor | `ta.pivothigh(high, smc_len, smc_len)` | 5 bars ea. side | Intraday structure |
| Major | `ta.pivothigh(high, smc_len_major, smc_len_major)` | 20 bars ea. side | Swing structure |

**4 Persistent Variables** mengikuti pivot terakhir:
```pine
var float last_high_pivot = na    // Minor high
var float last_low_pivot  = na    // Minor low
var float last_major_high = na    // Major high (20-bar)
var float last_major_low  = na    // Major low (20-bar)
```

Saat pivot baru terdeteksi, index disimpan dengan offset:
```pine
if not na(ph)
    last_high_pivot := ph
    last_high_idx := bar_index - smc_len    // Pivot terjadi 5 bar lalu
```

### 1.2 Break Classification Algorithm

([L3325-3394](file:///d:/forex/titannew.pine#L3325-L3394)):

**Priority**: Major break diperiksa LEBIH DULU dari minor break.

```
BULLISH BREAK CHECK:
├── Jika close > last_major_high:
│   ├── is_major_break = true
│   ├── Jika trend_dir sudah == 1 → BOS (continuation)
│   └── Jika trend_dir == -1 → CHoCH (trend reversal) → set trend_dir = 1
│   └── Clear major_high (consumed)
├── ELSE jika close > last_high_pivot:
│   ├── is_major_break = false
│   ├── Sama logic: BOS jika same dir, CHoCH jika opposite
│   └── Clear minor_high (consumed)
└── Return all results

BEARISH BREAK CHECK: (mirror logic)
```

**Key Design**: Setelah level di-break, ia di-clear ke `na` sehingga level yang sama tidak terdeteksi ulang.

### 1.3 SMC Filter Pipeline

([L3396-3418](file:///d:/forex/titannew.pine#L3396-L3418)):

| Filter | BOS Bull | CHoCH Bull | BOS Bear | CHoCH Bear |
|--------|----------|------------|----------|------------|
| **Macro (score_live)** | `score > 0` | `score > -0.5` | `score < 0.5` | `score < 1.2` |
| **RSI Momentum** | `RSI > 50` | `RSI > 40` | `RSI < 60` | `RSI < 70` |
| **Bypass Mode** | All passed | All passed | All passed | All passed |

> [!NOTE]
> CHoCH filters lebih permissive dari BOS karena CHoCH menandakan trend reversal — perlu mendeteksi lebih awal sebelum score catch up.

### 1.4 FVG v2.0 — 3-Tier Quality Grading

([L3267-3323](file:///d:/forex/titannew.pine#L3267-L3323)):

**Core Detection (Bull FVG):**
```
gap_low  = high[2]      // Top of left candle
gap_high = low[0]       // Bottom of right candle
FVG exists jika: gap_high > gap_low (gap between them)
```

**3 Filters:**

| # | Filter | Requirement |
|---|--------|------------|
| 1 | Displacement | Middle candle body > 45% of range |
| 2 | Direction | Middle candle closes in FVG direction |
| 3 | Volume | Middle candle volume > SMA(20) |

**Quality Tiers:**

| Tier | Name | Extra Requirements | Visual |
|------|------|-------------------|--------|
| 1 | Valid | Passes 3 base filters | Green (85% transparency) |
| 2 | Strong | + Direction aligned + gap > 1.5× ATR×min | Lime (80% transparency) |
| 3 | Premium | + Volume premium (>1.5× avg) + gap > 2× ATR×min | Gold ★ (75% transparency) |

**CE (Consequent Encroachment):**
```
fvg_ce = gap_low + (gap_size / 2.0)    // Midpoint for precision entry
```
- CE line ditampilkan hanya untuk Strong/Premium FVG
- Trader menggunakan CE sebagai entry point yang lebih tepat dari zone boundary

### 1.5 Breaker Blocks

Dibuat saat **Major CHoCH** terjadi:
```
if choch_bull AND show_breaker:
    Buat box dari broken level ke bar_index+15 (bearish block yang menjadi demand)
if choch_bear AND show_breaker:
    Buat box dari broken level ke bar_index+15 (bullish block yang menjadi supply)
```

### 1.6 Visualization Cleanup

([L3520-3525](file:///d:/forex/titannew.pine#L3520-L3525)):
```pine
f_clean_lines(arr) => if array.size(arr) > 50: delete oldest
f_clean_boxes(arr) => if array.size(arr) > 40: delete oldest
```
Mencegah Pine Script limit 500 labels/boxes/lines.

---

## 2. Quality Score — Complete Computation Walkthrough

### 2.1 Step-by-Step Formula

**Step 1: Raw Components (5 dimensions)**

([L1959-1972](file:///d:/forex/titannew.pine#L1959-L1972)):
```
quality_corr    = avg(abs(corr_dxy), abs(corr_yield), ..., abs(corr_usdjpy)) / 7
quality_agree   = status_agree / 4.0
quality_mag     = min(1.0, abs(scenario_index))
quality_vol     = 1.0 - vol_score
quality_vol_div = SPY volume Z / 2.0  (clamped 0-1, default 0.5 if NA)
```

**Step 2: Weight Normalization**

([L2350-2356](file:///d:/forex/titannew.pine#L2350-L2356)):
```
total_weight = w_corr + w_agree + w_mag + w_vol + w_vol_div + 1.5  (elite shared weight)

norm_corr    = w_corr / total_weight
norm_agree   = w_agree / total_weight
norm_mag     = w_mag / total_weight
norm_vol     = w_vol / total_weight
norm_vol_div = w_vol_div / total_weight
norm_elite   = 1.5 / total_weight     (shared by intrinsic + bayes + mtf)
```

**Step 3: Base Weighted Sum**

([L2359](file:///d:/forex/titannew.pine#L2359)):
```
quality_base = corr × norm_corr + agree × norm_agree + mag × norm_mag
             + vol × norm_vol + vol_div × norm_vol_div
```

**Step 4: Elite Components**

([L2386](file:///d:/forex/titannew.pine#L2386)):
```
quality_base += (intrinsic_sync × 0.4 + bayes_conf × 0.4 + mtf_conf × 0.2) × norm_elite
```

**Step 5: Multipliers**

([L2388](file:///d:/forex/titannew.pine#L2388)):
```
quality_base × session_mult × setup_mult
```

| Setup | Multiplier |
|-------|-----------|
| Whale Trap | ×1.25 |
| Ignition | ×1.20 |
| Power Div | ×1.30 |
| Absorption | ×1.15 |
| AO Divergence | ×1.10 |
| Twin Peaks | ×1.10 |
| Pullback | ×1.05 |
| Default | ×1.00 |

**Step 6: Add All Boosters**

([L2390](file:///d:/forex/titannew.pine#L2390)):
```
quality += div_booster          // RSI divergence: +0.15
         + pat_booster          // Candlestick: 0-0.20 (strength-scaled)
         + vol_booster          // VPA event: -0.10 to +0.20
         + ao_div_booster       // AO divergence: 0-0.12
         + power_div_booster    // Power Div: 0.08-0.20
         + ao_pattern_booster   // Twin Peaks/Saucer: 0.08-0.12
         + ao_velocity_booster  // AO velocity: -0.04 to +0.06
```

**Step 7: VPA Bayesian Feedback**

([L2452-2456](file:///d:/forex/titannew.pine#L2452-L2456)):
```
Jika VPA event (Stopping Vol, Absorption, CVD Div) boost Bayesian:
  bayes_conf × 1.4 → delta = (new_bayes - old_bayes) × 0.4 × norm_elite × session × setup
  quality += delta   (range biasanya +0.01 to +0.05)
```

**Step 8: Final Clamp**

```
quality_score = clamp(quality, 0.0, 1.0)
```

### 2.2 Contoh Numerik

```
Misalkan:
  corr_avg=0.5, agree=3, scenario=0.8, vol_score=0.4, vol_div=0.6
  w_corr=1.5, w_agree=1.0, w_mag=1.0, w_vol=0.5, w_vol_div=0.5
  
  total_weight = 1.5+1.0+1.0+0.5+0.5+1.5 = 6.0
  
  base = 0.5×(1.5/6) + 0.75×(1/6) + 0.8×(1/6) + 0.6×(0.5/6) + 0.6×(0.5/6)
       = 0.125 + 0.125 + 0.133 + 0.050 + 0.050 = 0.483
  
  elite = (1.0×0.4 + 0.78×0.4 + 0.75×0.2) × (1.5/6.0)
        = (0.40 + 0.31 + 0.15) × 0.25 = 0.215
  
  subtotal = 0.483 + 0.215 = 0.698
  
  × session(1.25 kill zone) × setup(1.30 power div) = 0.698 × 1.625 = 1.134 → capped 1.0
  
  + boosters: RSI div(0.15) + AO div(0.08) + power div(0.14) + vel(0.06) = 0.43
  
  TOTAL = 1.0 + 0.43 = 1.43 → clamped 1.0
  
  Grade: A+ ✅
```

---

## 3. Power Divergence v2.0 — Weighted Fusion Detail

### 3.1 Activation Condition

([L2233-2234](file:///d:/forex/titannew.pine#L2233-L2234)):
```pine
power_div_bull = enable_power_div AND bull_div_recent AND ao_reg_bull_recent
power_div_bear = enable_power_div AND bear_div_recent AND ao_reg_bear_recent
```

**Kedua divergence HARUS aktif secara bersamaan** (dalam memory window masing-masing):
- RSI div: 60-bar window
- AO div: 40-50 bar window (adaptive)

### 3.2 Combined Score

([L2241-2252](file:///d:/forex/titannew.pine#L2241-L2252)):
```
power_combined = AO_strength × 0.60 + RSI_strength × 0.40 + velocity_bonus

velocity_bonus:
  ao_mom_ignite (bull) / ao_mom_collapse (bear) → +10 pts
  else → +0
```

**AO mendapat bobot 60%** karena volume-confirmed dan lebih reliable dalam M5 scalping.

### 3.3 Grade Classification

| Grade | Score Range | Quality Booster |
|-------|-----------|----------------|
| **ELITE** | ≥ 80 | +0.20 |
| **STRONG** | ≥ 60 | +0.16 |
| **STANDARD** | ≥ 40 | +0.12 |
| **WEAK** | < 40 | +0.08 |

### 3.4 Downstream Effects

Power Div mempengaruhi 7 subsystem:

| Subsystem | Effect |
|-----------|--------|
| Quality Score | +0.08 to +0.20 booster |
| Quality Multiplier | ×1.30 setup mult |
| Signal Filter | Bypass path allowed (Path 2) |
| Chop Filter | Override (ignore chop) |
| Confluence Score | +2 via cluster tier |
| Exit Logic | Reversal context tag |
| Same-side echo | Override (allowed even if recent same-signal) |

---

## 4. VPA ↔ Bayesian Feedback Loop

### 4.1 Mekanisme

Ini adalah **feedback loop positif** yang memperkuat institutional conviction:

```
VPA Detection → Bayesian Boost → Quality Boost → Signal More Likely
```

### 4.2 Implementation

([L2441-2456](file:///d:/forex/titannew.pine#L2441-L2456)):

```
Step 1: Save pre-VPA Bayesian value
  b_conf_norm_pre = bayes_conf / 100

Step 2: VPA Boost (×1.4)
  if stop_vol_bull OR absorption_bottom OR cvd_div_bull:
    bayes_conf_bull = min(99, bayes_conf_bull × 1.4)
    bayes_conf_bear = 100 - bayes_conf_bull

Step 3: Calculate delta
  b_conf_norm_post = new_bayes_conf / 100
  delta = (post - pre) × 0.4 × norm_elite × session_mult × setup_mult

Step 4: Apply to quality
  quality_score += delta
  quality_signal_valid = quality_score >= qual_min   // Re-check gate
```

### 4.3 Dampak Numerik

```
Contoh: bayes_conf_bull = 68%
  After VPA boost: 68 × 1.4 = 95.2% (capped at 99)
  Delta: (0.95 - 0.68) × 0.4 × 0.25 × 1.25 × 1.15 = 0.039

  Quality 0.55 → 0.59 (bisa push dari Grade B ke A)
```

> [!WARNING]
> Loop ini bisa membuat kualitas "melonjak" saat VPA event terjadi pada bar yang sudah kuat secara macro. Ini by design — institutional footprint di confluence point memang harus meningkatkan confidence secara agresif.

---

## 5. Whale Trap v10.0 — Complete Code-Level Breakdown

### 5.1 Baseline Detection (Pre-v10)

([L1900-1913](file:///d:/forex/titannew.pine#L1900-L1913)):
```pine
// Wick analysis
is_upper_v4_wick = (high - max(open, close)) > (candle_range * 0.40)
is_v4_deep_bull = close < (high - candle_range * 0.60)

// Baseline formula
bull_trap = is_upper_v4_wick AND is_v4_deep_bull
            AND high > highest_5 AND close < highest_5
            AND (vol_z < 0.8 OR volume_eff < 0.85)
```

> Baseline mendeteksi "fakeout" di mana volume RENDAH — menandakan retail-driven breakout tanpa institutional backing.

### 5.2 v10.0 Fusion Upgrade (6 Layers)

([L2272-2319](file:///d:/forex/titannew.pine#L2272-L2319)):

**Layer A — Structural Boundaries:**
```
high > highest_5        // Price breaks recent structure
```

**Layer B — Volatility Dynamics:**
```
wick_u > candle_range × 0.60       // Strong rejection wick
AND close < (high - candle_range × 0.70)   // Deep close reversal
AND candle_range > ATR × 1.35      // Statistically significant range
```

**Layer C — Purity Divergence:**
```
high > highest_5 AND xau_intrinsic_roc < 0.0005
// Harga Gold breakout TAPI XAU basket (EUR+JPY) TIDAK ikut
// = Breakout hanya karena USD movement, bukan true Gold demand
```

**Layer D — Session Edge:**
```
in_kz_london OR in_kz_ny
// Whale traps paling sering terjadi di Kill Zone
```

**Layer E — Sequential Stop-Runs:**
```
trap_ratio = SMA(recent_traps, 20)
is_triple_trap = trap_ratio × 20 >= 1.5
// 2+ traps in last 20 bars = institutional stop-hunt campaign
```

**Layer F — Macro Fusion:**
```
m_sync_bull = z_DXY > 1.0 OR z_Yield > 1.0  // Macro pressure
vbx_stress = abs(z_VIX) > 1.5               // VIX extreme
has_bear_exhaustion = bear_div OR ao_bear OR ao_collapse
```

### 5.3 Final Fusion Formula

```pine
logic_v10_bull =
    (Layer B: wick 60% + deep reflux + vol outlier)
    AND (Layer A: high > highest_5)
    AND (Layer C OR D OR E: purity_div OR at_supply_OB OR kill_zone OR triple_trap)
    AND (Layer F: macro_sync OR vix_stress OR exhaustion)
    AND NOT in_cooldown(5 bars)
```

**Setiap "AND group" memerlukan minimal 1 dari beberapa opsi** — ini adalah fusi bukan filter ketat. Sistem butuh **structural evidence (B) + confluence (C/D/E) + macro context (F)**.

### 5.4 Cooldown

```pine
var int last_trap_v10 = 0
bool in_v10_cooldown = (bar_index - last_trap_v10) < 5   // 5-bar cooldown

if bull_trap or bear_trap
    last_trap_v10 := bar_index
```

> Cooldown mencegah deteksi berulang pada volatile price action setelah trap.

---

## 6. Glossary — Semua Singkatan & Istilah

### 6.1 Akronim Umum

| Akronim | Kepanjangan | Penjelasan |
|---------|------------|-----------|
| AO | Awesome Oscillator | Indikator momentum Bill Williams (SMA5-SMA34 dari HL2) |
| ATR | Average True Range | Ukuran volatilitas rata-rata |
| BB | Bollinger Bands | Band atas/bawah berdasarkan SMA ± nσ |
| BOS | Break of Structure | Harga break level structure searah trend |
| CE | Consequent Encroachment | Midpoint FVG (50% retracement) |
| CHoCH | Change of Character | Harga break level structure melawan trend |
| CVD | Cumulative Volume Delta | Akumulasi selisih buy vs sell pressure |
| DIV+ | Divergence Plus | Sinyal divergence institusional 11-layer filter |
| DXY | Dollar Index | Indeks kekuatan USD vs 6 major currencies |
| EMA | Exponential Moving Average | Moving average eksponensial |
| FVG | Fair Value Gap | Area yang belum "diisi" harga antara 3 candle |
| HMA | Hull Moving Average | MA dengan lag rendah (Alan Hull) |
| MTF | Multi-Timeframe | Analisis beberapa timeframe sekaligus |
| OB | Order Block | Zona supply/demand institusional |
| ROC | Rate of Change | Persentase perubahan dari N bar lalu |
| RSI | Relative Strength Index | Indikator momentum 0-100 |
| SL | Stop Loss | Level exit kerugian |
| SMA | Simple Moving Average | Rata-rata bergerak sederhana |
| SMC | Smart Money Concepts | Teori order flow institusional |
| TP | Take Profit | Level exit keuntungan |
| VPA | Volume Price Analysis | Analisis hubungan volume-harga |
| VWAO | Volume-Weighted AO | AO yang di-weight dengan volume |
| Z | Z-Score | Standar deviasi dari mean |

### 6.2 Istilah Teknikal Titan Quant

| Istilah | Penjelasan |
|---------|-----------|
| **Absorption** | Institusi membeli/menjual di extreme tanpa menggerakkan harga jauh — narrows spread + high volume |
| **APEX** | Cluster tier tertinggi (≥5.5 score) dengan anchor signal |
| **Bayesian Sniper** | Filter probabilistik menggunakan Naive Bayes untuk confidence directional |
| **Chop Market** | Kondisi sideways di mana EMA spread rendah, tidak ada trend |
| **Churn** | High volume tapi harga tidak bergerak — pertarungan bull vs bear |
| **Circuit Breaker** | Sistem counter yang melacak data NA per aset, block jika terlalu banyak |
| **Climax** | Ultra-high volume + rejection wick — potensi exhaustion |
| **Counter-Trend Gate** | Score minimum untuk sinyal yang melawan arah HMA trend |
| **Deep Reflux** | Close di bagian jauh dari wick arah — menandakan rejection kuat |
| **FallbackZ** | Persistent store menyimpan Z-score terakhir yang valid per aset |
| **Fear Asset** | Aset yang diberi boost saat volatilitas tinggi (VIX, DXY, Yields) |
| **Flip Risk Guard** | Mencegah sinyal terlalu cepat berubah arah (buy→sell→buy) |
| **Grace Period** | Jendela bar di mana exit filter tidak aktif (untuk reversal entry) |
| **Hyper-Response** | EMA smoothing dengan alpha adaptif terhadap volatilitas |
| **Ignition** | Breakout institusional: vol tinggi + body kuat + momentum expansion |
| **Institutional Confluence** | Skor 0-18 dari 9 dimensi yang menentukan kekuatan sinyal |
| **Intrinsic Strength** | Kekuatan Gold murni dihitung dari XAUEUR+XAUJPY (tanpa pengaruh USD) |
| **Kill Zone** | Overlap session London-NY — volatilitas dan likuiditas tertinggi |
| **Macro TF** | Timeframe untuk data makro (default H4/Daily) |
| **Momentum Decay** | Score masih kuat tapi velocity melambat — potensi reversal |
| **No Demand/Supply** | Low-volume pullback dalam trend — koreksi sehat |
| **Power Div** | RSI + AO divergence bersamaan — confluence tertinggi |
| **Purity Divergence** | Harga USD breakout tapi XAU basket tidak ikut — fakeout indicator |
| **Quantum Confluence** | Order Block yang align dengan Price Projection target |
| **Regime** | Klasifikasi pasar (Risk-On/Off/Extreme) berdasarkan macro index |
| **Ring Buffer** | Struktur data circular O(1) untuk correlation computation |
| **Same-Side Echo** | Sinyal buy setelah buy baru (atau sell setelah sell) — biasanya di-block |
| **Scenario Index** | Score sederhana dari 5 macro terpenting untuk dashboard display |
| **Signal Rearm** | Mekanisme yang memungkinkan sinyal baru muncul setelah cooldown |
| **Sniper** | Sinyal yang melewati Bayesian confidence threshold — high conviction |
| **Stopping Volume** | Ultra-high volume di extreme yang diikuti reversal close |
| **Test Volume** | Low volume saat harga kembali ke OB zone — konfirmasi zone masih valid |
| **Triple Trap** | 2+ whale traps dalam 20 bar — institutional stop-hunt campaign |
| **Velocity** | Rate of change score — leading indicator untuk signal direction |
| **Vol Score** | `log(vol_ratio + 1) / log(3)` — normalized volatility measure |
| **Whale Trap** | Fakeout breakout yang dilakukan oleh institusi untuk menjebak retail |

### 6.3 Simbol Dashboard

| Simbol | Arti |
|--------|------|
| ⚔️ | Titan Quant header |
| 🎯 | Sniper Ready — sinyal imminent |
| ⚡ | Velocity spike / power event |
| 📉 | Regime block / dead volatility |
| ⚠️ | Counter-trend / extreme warning |
| 🧱 | Exhaustion |
| ⌛ | Qualifying / neutral |
| 🛡️ | Position active |
| 🚀 | Pure Bull intrinsic |
| 💎 | Institutional accumulation |
| 💀 | Pure Bear intrinsic |
| ☠️ | Whale trap |
| ❇️ | Bottom absorption |
| 🛑 | Top absorption |
| ⭐ | Order block star rating |
| 🟢🔴⚪ | MTF bullish / bearish / neutral |
| ✕ | Rejected signal (filters blocked) |
| ▲ / ▼ | Buy / Sell confirmed signal |

---

> [!IMPORTANT]
> **Dokumentasi Titan Quant v11 selesai — 6 bagian, 66 sections total, covering 100% dari 5,070 baris kode.**
>
> | Part | File | Sections | Topik |
> |------|------|----------|-------|
> | 1 | [Part 1](file:///d:/forex/titan_quant_documentation%20part1.md) | 23 | Arsitektur & fitur overview |
> | 2 | [Part 2](file:///d:/forex/titan_quant_documentation%20part2.md) | 13 | Detail implementasi kode |
> | 3 | [Part 3](file:///d:/forex/titan_quant_documentation%20part3.md) | 5 | Scenario, tuning & troubleshooting |
> | 4 | [Part 4](file:///d:/forex/titan_quant_documentation%20part4.md) | 9 | Input catalog & dependency map |
> | 5 | [Part 5](file:///d:/forex/titan_quant_documentation%20part5.md) | 8 | Formula & edge cases |
> | 6 | **Part 6** | 6 | SMC, quality walkthrough & glossary |
