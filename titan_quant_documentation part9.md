# 📋 Titan Quant v11 — Part 9: Quick Reference Card

> **Cheat sheet operasional** — referensi cepat untuk trading dan maintenance sehari-hari.
> Untuk detail teknis lengkap, rujuk [Part 1-8](file:///d:/forex/titan_quant_documentation%20part1.md).

---

## 1. Signal Hierarchy & Priority

### 🎯 Ranking Sinyal (Tertinggi → Terendah)

| # | Signal | Label | Akurasi | Lot % | Kapan Muncul |
|---|--------|-------|---------|-------|--------------|
| 1 | **POWER DIV** | 🟢⚡ POWER BUY/SELL | ★★★★★ | 100% | RSI + AO divergence bersamaan |
| 2 | **DIV+** | ⚡DIV+BUY/SELL | ★★★★☆ | 90% | 11-layer institutional divergence |
| 3 | **Sniper Signal** | ▲ SNIPER BUY/SELL | ★★★★☆ | 80% | All 18 filters pass + Bayesian ≥ thresh |
| 4 | **APEX Cluster** | ⚡ APEX | ★★★★☆ | 80% | Cluster score ≥ 5.5 + anchor signal |
| 5 | **AO Standalone** | 🟩 AO↑ BUY/SELL | ★★★☆☆ | 60% | AO divergence dengan volume+quality |
| 6 | **Standard Signal** | ▲ BUY / SELL ▼ | ★★★☆☆ | 50% | Filters pass tanpa Bayesian sniper |
| 7 | **Raw Divergence** | 💎 / 🌌 / 💣 / 💥 | ★★☆☆☆ | 0% | Early warning — JANGAN trade langsung |
| 8 | **Hidden Div** | HID BUY/SELL | ★★☆☆☆ | 30% | Continuation — hanya jika sudah in-position |

---

## 2. Entry Decision Flowchart

```
SINYAL MUNCUL DI CHART
    │
    ├── Cek Dashboard: "Trade Status"
    │   ├── "🎯 SNIPER READY" → ✅ PROCEED
    │   ├── "⌛ QUALIFYING" → ⚠️ Wait for next bar confirmation
    │   ├── "📉 REGIME BLOCK" → ❌ SKIP (macro opposing)
    │   ├── "⚠️ COUNTER-TREND" → Hanya ambil jika POWER/DIV+
    │   └── "🧱 EXHAUSTION" → ❌ SKIP (momentum habis)
    │
    ├── Cek Dashboard: "Quality"
    │   ├── A+ / A → ✅ Full size
    │   ├── B → ⚠️ Half size
    │   └── C / D → ❌ SKIP (atau minimal size)
    │
    ├── Cek Dashboard: "Bayesian"
    │   ├── Bull ≥ 70% (untuk BUY) → ✅ Confident
    │   ├── 60-70% → ⚠️ Moderate — reduce size
    │   └── < 60% → ❌ Low confidence
    │
    ├── Cek Dashboard: "MTF"
    │   ├── 🟢🟢🟢🟢 (4/4 aligned) → ✅ Strong confluence
    │   ├── 🟢🟢🟢⚪ (3/4) → ✅ Good
    │   ├── 🟢🟢🔴⚪ (mixed) → ⚠️ Caution
    │   └── 🔴🔴🔴🔴 (all against) → ❌ SKIP
    │
    └── EXECUTE TRADE
        ├── Entry: Market order at signal bar close
        ├── SL: Ditampilkan di dashboard "Money Mgmt"
        ├── TP1: Conservative scalp (ambil 50% posisi)
        ├── TP2: Projected target (ambil 30%)
        └── TP3: Runner (trail sisanya)
```

---

## 3. Exit Decision Flowchart

```
POSISI AKTIF
    │
    ├── Dashboard Scanner: "IN LONG +X.XX%"
    │
    ├── PnL NEGATIF?
    │   ├── Lebih dari -SL% → 🔴 EXIT SEKARANG (hard stop)
    │   ├── Score berubah arah → Tunggu 2 bar konfirmasi
    │   └── Masih dalam grace period → HOLD (reversal entry)
    │
    ├── PnL POSITIF?
    │   ├── Di atas profit lock level?
    │   │   ├── Momentum decay muncul → 🟡 EXIT (lock profit)
    │   │   ├── Rejection pattern muncul → 🟡 EXIT
    │   │   └── Giveback > trail threshold → 🟡 EXIT
    │   └── Di bawah profit lock → HOLD
    │
    ├── SINYAL OPPOSITE MUNCUL?
    │   ├── Reversal entry still in grace → HOLD (kecuali losing)
    │   └── No grace → EXIT + consider new position
    │
    └── TREND FLIP (2 bar)?
        ├── In profit → EXIT
        └── In loss + reversal → HOLD (grace protects)
```

---

## 4. Quick Parameter Tuning

### Skenario Umum & Solusi

| Masalah | Parameter | Ubah Ke | Efek |
|---------|-----------|---------|------|
| **Sinyal terlalu jarang** | Sniper Density Mode | "Adaptive Frequent" | ↑ frekuensi ~2× |
| **Sinyal masih jarang** | Sniper Density Mode | "Aggressive Scalper" | ↑ frekuensi ~4× |
| **Terlalu banyak false signal** | Quality Min Score | ↑ 0.50 → 0.60 | Filter ketat |
| **Miss entry di volatile market** | Vol-Adaptive Conf | ON | Lower Bayesian thresh saat vol↑ |
| **Dashboard terlalu penuh** | Show Scenario Impact | OFF | Kurangi rows |
| **Ingin sinyal pada M1** | Preset | "Aggressive Scalping 1m" | Full M1 optimization |
| **Sinyal counter-trend terlalu banyak** | Counter-Trend Gate | ↑ | Butuh score lebih tinggi |
| **OB zone terlalu banyak** | Max Active Zones | 2 | Kurangi visual clutter |
| **Projection terlalu volatile** | Confidence Weight | ↓ 0.6 → 0.4 | Smoothen projection |
| **MTF blocking signals** | MTF Min Agreement | 1 | Relax dari 2 ke 1 |

---

## 5. Dashboard Status Decoder

### Action Row

| Text | Warna | Arti | Tindakan |
|------|-------|------|---------|
| STRONG BUY | 🟢 Hijau | Score + trend aligned bullish | Entry BUY jika filter pass |
| SNIPER BUY | 🟢 Hijau | Score bullish, trend netral | Entry BUY dengan caution |
| PULLBACK BUY | 🟡 Dim green | Score slightly bullish | Tunggu konfirmasi |
| NEUTRAL | ⚪ Abu | No directional bias | Jangan trade |
| PULLBACK SELL | 🟡 Dim red | Score slightly bearish | Tunggu konfirmasi |
| SNIPER SELL | 🔴 Merah | Score bearish, trend netral | Entry SELL dengan caution |
| STRONG SELL | 🔴 Merah | Score + trend aligned bearish | Entry SELL jika filter pass |

### Regime Row

| Text | Warna | Gold Outlook |
|------|-------|-------------|
| Extreme Risk-Off | 🟢 Bright lime | **Very Bullish** — maximum buy bias |
| Strong Risk-Off | 🟢 Green | **Bullish** — buy preferred |
| Risk-Off | 🟢 Light green | **Slightly Bullish** |
| Neutral | ⚪ Gray | **No bias** — trade both ways |
| Risk-On | 🔴 Light red | **Slightly Bearish** |
| Strong Risk-On | 🔴 Red | **Bearish** — sell preferred |
| Extreme Risk-On | 🔴 Bright red | **Very Bearish** — maximum sell bias |

### Intrinsic Row

| Text | Arti | Signal Reliability |
|------|------|-------------------|
| 🚀 PURE BULL | XAU basket + USD price both bullish | ★★★★★ Highest |
| 💎 INST ACCUM | XAU basket bullish, USD price bearish | ★★★★☆ Accumulation phase |
| ⚠️ FAKE PUMP | XAU basket bearish, USD price bullish | ★☆☆☆☆ Likely to reverse |
| 💀 PURE BEAR | Both bearish | ★★★★★ Highest (for sells) |
| ⚪ NEUTRAL | No clear basket signal | ★★★☆☆ Proceed with caution |

---

## 6. Regime-Aware Trading Rules

| Regime | BUY | SELL | Recommended Signals |
|--------|-----|------|-------------------|
| **Extreme Risk-Off** | ✅ Full size | ⚠️ Counter-trend only | All buy signals valid |
| **Strong Risk-Off** | ✅ Full size | ⚠️ Needs score < -0.22 | POWER, DIV+, Sniper |
| **Risk-Off** | ✅ Normal | ⚠️ Cautious | Any passing signal |
| **Neutral** | ⚠️ Quality > 0.67 | ⚠️ Quality > 0.67 | Only high-quality |
| **Risk-On** | ⚠️ Counter-trend only | ✅ Normal | Any passing signal |
| **Strong Risk-On** | ⚠️ Needs score > 0.22 | ✅ Full size | POWER, DIV+, Sniper |
| **Extreme Risk-On** | ⚠️ Very cautious | ✅ Full size | All sell signals valid |

---

## 7. Troubleshooting Quick-Fix

| Symptom | Likely Cause | Quick Fix |
|---------|-------------|-----------|
| **No signals for hours** | Quality too high or regime blocking | Lower Quality Min → 0.40, check regime |
| **"Script error" on load** | Too many API calls | Disable MTF Confluence → saves 4 calls |
| **Dashboard shows "Calculating..."** | FRED data unavailable (weekend) | Normal — wait for market open |
| **Signals appear then disappear** | Realtime Aggressive mode | Change to "Confirmed Bar" mode |
| **Same BUY repeated 3× in 5 bars** | Aggressive Scalper mode | Lower to "Adaptive Frequent" |
| **Score stuck at 0** | All data sources NA | Check Circuit Breaker in console |
| **"FAKE PUMP" but price rising** | USD-driven move, not gold demand | Wait for intrinsic to resolve |
| **Projection MAE too high** | Volatile news event | Reset by changing horizon → change back |
| **OB zones not appearing** | Absorption not detected | Lower vol threshold or check VPA settings |
| **Rejected ✕ labels everywhere** | Filters very strict | Hover tooltip → shows which filter blocks |

---

## 8. Key Macro Events Calendar

### Trading Around Events

| Event | Impact on System | Recommended Action |
|-------|-----------------|-------------------|
| **FOMC** | VIX spike → threshold shift, score volatility | Widen SL, wait for post-statement candle |
| **NFP** | Extreme vol → projection unreliable for 15min | Avoid signals during first 15 bars post-release |
| **CPI** | Yield + DXY moves → instant regime shift | Watch regime row — may flip rapidly |
| **Fed Speak** | Minor — gradual score drift | No special action needed |
| **Weekend Gap** | FRED data stale, Z-scores frozen | FallbackZ handles — monitor Monday Asian session |
| **Tokyo Fix (9:55 AM JST)** | Gold liquidity spike | Good time for Kill Zone signals |
| **London Fix (3PM GMT)** | Strongest gold session | Highest signal reliability |
| **NY AM** | Overlap with London | Maximum liquidity, best signals |

---

## 9. Documentation Quick Navigator

### Cari Berdasarkan Topik

| Ingin Tahu Tentang... | Baca | Section |
|----------------------|------|---------|
| Cara kerja keseluruhan | [Part 1](file:///d:/forex/titan_quant_documentation%20part1.md) | §1 + §23 |
| Data macro apa saja | [Part 1](file:///d:/forex/titan_quant_documentation%20part1.md) | §2 |
| Bagaimana score dihitung | [Part 5](file:///d:/forex/titan_quant_documentation%20part5.md) | §5 |
| Formula divergence | [Part 5](file:///d:/forex/titan_quant_documentation%20part5.md) | §1 + §2 |
| Candlestick scoring | [Part 5](file:///d:/forex/titan_quant_documentation%20part5.md) | §3 |
| 26 macro vote factors | [Part 5](file:///d:/forex/titan_quant_documentation%20part5.md) | §4 |
| Signal filter detail | [Part 7](file:///d:/forex/titan_quant_documentation%20part7.md) | §2 |
| Bayesian 4-tier logic | [Part 7](file:///d:/forex/titan_quant_documentation%20part7.md) | §3 |
| Exit logic rules | [Part 8](file:///d:/forex/titan_quant_documentation%20part8.md) | §1 |
| SL/TP calculation | [Part 8](file:///d:/forex/titan_quant_documentation%20part8.md) | §3 |
| Projection engine | [Part 8](file:///d:/forex/titan_quant_documentation%20part8.md) | §2 |
| Parameter tuning | [Part 3](file:///d:/forex/titan_quant_documentation%20part3.md) | §2 |
| Scenario walkthrough | [Part 3](file:///d:/forex/titan_quant_documentation%20part3.md) | §1 |
| All input params | [Part 4](file:///d:/forex/titan_quant_documentation%20part4.md) | §1 |
| Preset differences | [Part 4](file:///d:/forex/titan_quant_documentation%20part4.md) | §2 |
| SMC (BOS/CHoCH) | [Part 6](file:///d:/forex/titan_quant_documentation%20part6.md) | §1 |
| Whale Trap v10 | [Part 6](file:///d:/forex/titan_quant_documentation%20part6.md) | §5 |
| Glossary / istilah | [Part 6](file:///d:/forex/titan_quant_documentation%20part6.md) | §6 |
| Code line map | [Part 7](file:///d:/forex/titan_quant_documentation%20part7.md) | §1 |
| Dashboard rows | [Part 8](file:///d:/forex/titan_quant_documentation%20part8.md) | §5 |
| Edge cases | [Part 5](file:///d:/forex/titan_quant_documentation%20part5.md) | §8 |

---

> [!TIP]
> **Print** dokumen ini dan taruh di samping monitor saat trading. Rujuk Part teknis hanya saat perlu deep-dive.
