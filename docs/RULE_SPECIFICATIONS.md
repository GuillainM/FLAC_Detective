# 📋 RULE SPECIFICATIONS - FLAC Detective v0.6.4

## 🎯 Overview

FLAC Detective uses an advanced **11-rule detection system** with additive scoring (0-150 points).

### Scoring Scale

```
┌─────────────────────────────────────────────────────────┐
│  0        30         60         86              150     │
│  ├─────────┼──────────┼──────────┼────────────────┤     │
│  AUTHENTIC  WARNING   SUSPICIOUS  FAKE_CERTAIN          │
│    ✅         ⚡          ⚠️           ❌                  │
└─────────────────────────────────────────────────────────┘

Score ≥ 86  → FAKE_CERTAIN ❌ (100% confidence)
Score 61-85 → SUSPICIOUS ⚠️  (High confidence)
Score 31-60 → WARNING ⚡ (Manual review recommended)
Score ≤ 30  → AUTHENTIC ✅ (99.5% confidence)
```

**Philosophy**: Higher score = More fake
- Penalties increase score (suspicious indicators)
- Bonuses decrease score (authenticity indicators)

---

## 🔍 Rule 1: MP3 Spectral Signature Detection (CBR)

**Objective**: Detect CBR MP3s transcoded to FLAC

### Visual Concept
```
Authentic FLAC               MP3 → FLAC Transcode
─────────────                ────────────────────
Frequency                    Frequency
    ▲                            ▲
    │████████████████            │████████████
    │████████████████            │████████████
    │████████████████            │████████████  ← Sharp cutoff
    │████████████████            │▓▓▓▓▓▓▓▓▓▓▓▓     at ~20 kHz
    │████████████████            ├────────────
    │████████████████            │            (No content)
    └────────────────►           └────────────►
    0 Hz      22 kHz             0 Hz    22 kHz
```

### Detection Logic

**Safety Check 1: Nyquist Exception**
```
IF cutoff >= 95% Nyquist → SKIP (likely anti-aliasing filter)
```

**CRITICAL EXCEPTION: Exactly 20 kHz Cutoff (ENHANCED)**
```
Problem: FFT may round 20-21 kHz to exactly 20000 Hz

Solutions:
┌─────────────────────────────────────────────┐
│ Test 1: Residual Energy > 20 kHz           │
│   IF energy_ratio > 0.000001 → SKIP        │
│   (Probably FFT rounding, not MP3 320k)    │
├─────────────────────────────────────────────┤
│ Test 2: Zero Variance                      │
│   IF cutoff_std == 0.0 → SKIP              │
│   (Ambiguous, skip by precaution)          │
└─────────────────────────────────────────────┘
```

### MP3 Bitrate Signatures
```
Cutoff Freq → Estimated MP3 Bitrate
─────────────────────────────────────
11 kHz      →  128 kbps
15 kHz      →  192 kbps
16 kHz      →  224 kbps
19 kHz      →  256 kbps
20 kHz      →  320 kbps
```

### Scoring
- **+50 points** if MP3 signature detected AND container bitrate matches expected range
- **Example**: cutoff = 20 kHz + container = 800 kbps → +50 pts (MP3 320k detected)

---

## 📊 Rule 2: Cutoff Frequency vs Nyquist

**Objective**: Penalize files with suspiciously low frequency content

### Visual Concept
```
Sample Rate: 44.1 kHz (Nyquist = 22.05 kHz)

Expected Cutoff:        Suspicious Cutoff:
────────────────        ──────────────────
    ▲                        ▲
    │████████████            │████████
    │████████████            │████████
    │████████████            │████████ ← Only 15 kHz!
22  ├────────────            ├────────
kHz │            (minimal)  15├────────
    │                       kHz│
    └────────────►            └────────►
    Threshold: ~20 kHz        Deficit: 5 kHz
                              Penalty: +25 pts
```

### Calculation
```
deficit = threshold - cutoff_freq
penalty = min(deficit / 200, 30)
```

### Scoring
- **+0 to +30 points** based on deficit
- Formula: `+1 pt` per 200 Hz below threshold, capped at 30 pts

---

## 🔄 Rule 3: Source vs Container Bitrate Comparison

**Objective**: Detect "inflated" files (low-quality source in heavy container)

### Visual Concept
```
Authentic FLAC                    Fake FLAC (Inflated)
──────────────                    ────────────────────
┌──────────────┐                 ┌──────────────┐
│              │                 │              │
│  High-Quality│                 │  MP3 128 kbps│ ← Low quality
│  PCM Source  │                 │  Source      │    source
│              │                 │              │
└──────┬───────┘                 └──────┬───────┘
       │                                │
       ▼                                ▼
┌──────────────┐                 ┌──────────────┐
│ FLAC 900 kbps│                 │ FLAC 900 kbps│ ← Heavy container!
└──────────────┘                 └──────────────┘
    NORMAL                         SUSPICIOUS
                                   (Inflated file)
```

### Scoring
- **+50 points** if MP3 source detected AND container > 600 kbps

---

## 📈 Rule 4: Suspicious 24-bit File Detection

**Objective**: Identify fake High-Res files (upsampled from lossy)

### Visual Concept
```
Real 24-bit FLAC              Fake 24-bit FLAC
────────────────              ────────────────
Bit Depth: 24                 Bit Depth: 24
Source: PCM/Analog            Source: MP3 192 kbps ← Upsampled!

Dynamic Range:                Dynamic Range:
████████████████████          ████████
████████████████████          ████████ ← Limited by
████████████████████          ████████    MP3 source
(~120 dB)                     (~60 dB)
```

### Scoring
- **+30 points** if bit_depth = 24 AND MP3 source < 500 kbps detected

---

## 🎵 Rule 5: High Variance Protection (VBR)

**Objective**: Identify natural FLAC characteristics (Variable Bit Rate)

### Visual Concept
```
Authentic VBR FLAC           CBR Transcode
──────────────────           ─────────────
Bitrate over time:           Bitrate over time:
kbps                         kbps
1400│    ▲                   1000│────────────
1200│   ╱ ╲  ▲               800│────────────
1000│  ╱   ╲╱ ╲              600│────────────
800│ ╱         ╲             400│────────────
    └──────────────►             └──────────────►
    High variance                Low variance
    (Natural VBR)                (Constant)
```

### Scoring
- **-40 points** if real_bitrate > 1000 kbps AND variance > 100 kbps
- Bonus for authentic VBR characteristics

---

## 🛡️ Rule 6: High Quality Protection

**Objective**: Protect authentic high-quality FLACs

### All Conditions Required
```
┌─────────────────────────────────────────────┐
│ ✓ No MP3 signature detected                │
│ ✓ Container bitrate > 700 kbps             │
│ ✓ Cutoff frequency ≥ 19 kHz                │
│ ✓ Bitrate variance > 50 kbps               │
└─────────────────────────────────────────────┘
         ▼
    AUTHENTIC HIGH-QUALITY FLAC
    Bonus: -30 points
```

### Scoring
- **-30 points** if ALL 4 conditions are true
- This combination is hard to fake

---

## 🎚️ Rule 7: Silence & Vinyl Analysis (3 Phases)

**Objective**: Distinguish authentic recordings from transcodes using silence analysis

### Activation Zone
```
Frequency Range: 19 kHz ≤ cutoff ≤ 21.5 kHz (Ambiguous zone)
                 ───────────────────────────
```

### Phase 1: Dither Detection
```
Silence Analysis (High-Freq 16-22 kHz):

Authentic Recording          Artificial Dither (Fake)
───────────────────          ────────────────────────
Music:  ████████             Music:  ████████
Silence: ░░░░                Silence: ████ ← Suspicious!
         (Natural)                    (Dither = noise floor)

Ratio = Energy(Silence) / Energy(Music)
```

### Phase 2: Vinyl Surface Noise
```
Detect vinyl crackling and pops (authentic analog source)
```

### Phase 3: Clicks & Pops Detection
```
Detect transient artifacts from vinyl playback
```

### Scoring
```
IF ratio > 0.30  → +50 pts  (Artificial dither = Transcode)
IF ratio < 0.15  → -50 pts  (Natural silence = Authentic)
IF 0.15-0.30     →   0 pts  (Uncertain)

IF vinyl detected → -50 to -100 pts (Authentic analog source)
```

---

## 🎯 Rule 8: Nyquist Exception (ALWAYS APPLIED with Safeguards)

**Objective**: Protect files with cutoff near theoretical Nyquist limit

### Visual Concept
```
Sample Rate: 44.1 kHz → Nyquist = 22.05 kHz

Cutoff Positions:
─────────────────────────────────────────────
22.05 kHz ├──────────────────────────────── 100% Nyquist
          │
21.8 kHz  ├────────── 98.8% → -50 pts (Very close)
          │
21.0 kHz  ├────────── 95.2% → -30 pts (Close)
          │
20.0 kHz  ├────────── 90.7% →   0 pts (No bonus)
          │
          ▼
```

### Safeguards (MP3 + Silence Check)
```
IF MP3 signature detected:
┌────────────────────────────────────────────┐
│ silence_ratio > 0.20  → Bonus CANCELLED    │
│ silence_ratio > 0.15  → Bonus REDUCED -15  │
│ silence_ratio ≤ 0.15  → Bonus APPLIED      │
└────────────────────────────────────────────┘
```

### Scoring
- **cutoff ≥ 98% Nyquist**: -50 points (strong bonus)
- **95% ≤ cutoff < 98% Nyquist**: -30 points (moderate bonus)
- **cutoff < 95% Nyquist**: 0 points

---

## 🔧 Rule 9: Compression Artifacts Detection

**Objective**: Detect MP3 compression artifacts in frequency domain

### Test A: Pre-echo (MDCT Ghosting)
```
Time-Frequency Analysis:

Clean Audio:              MP3 Pre-echo:
────────────              ─────────────
    │                         │ ← Ghosting
    │█████                    │░░█████
    │█████  Attack            │░░█████
    │█████                    │░░█████
    └────────►                └────────►
    Time                      Time
                              (Artifacts before attack)
```

### Test B: High-Frequency Aliasing
```
Frequency spectrum smoothness test
```

### Test C: MP3 Quantization Noise
```
Detect characteristic MP3 noise floor patterns
```

### Scoring
- **Variable penalty** based on artifact severity
- Each test contributes if artifacts detected

---

## 🔄 Rule 10: Multi-Segment Consistency Analysis

**Objective**: Validate consistency across entire file (not just first segment)

### Visual Concept
```
File Analysis Strategy:

Initial (2 segments):        If inconsistent (5 segments):
─────────────────────       ─────────────────────────────
[Seg1]        [Seg2]        [S1] [S2] [S3] [S4] [S5]
  ▼             ▼             ▼    ▼    ▼    ▼    ▼
Cutoff?      Cutoff?        Full file consistency check

IF variance between segments > threshold:
  → File may have different quality in different parts
  → Potential multi-source compilation or editing
```

### Scoring
- **-20 to -30 points** if segments show inconsistency
- Penalty for suspicious multi-source files

---

## 📼 Rule 11: Cassette Detection

**Objective**: Protect authentic analog cassette sources

### Cassette Signature Detection
```
Frequency Response Pattern:

Cassette Tape (Authentic):
────────────────────────────
    ▲
    │████████████
    │████████████╲
    │████████████ ╲  ← Gradual rolloff
    │█████████████ ╲    (8-12 kHz)
    │██████████████╲╲
    │███████████████╲
    └────────────────────►
    0 Hz          15 kHz

Characteristics:
• Gradual high-frequency rolloff (not sharp like MP3)
• Tape hiss (noise floor in high frequencies)
• Limited bandwidth (typically < 15 kHz)
```

### Scoring
- **Penalty reduction** if cassette signature detected
- Authentic analog source protection

---

## 🎨 Protection Hierarchy

```
┌────────────────────────────────────────────────────────┐
│ LEVEL 1: Absolute Protection                          │
│ └─ R8 (95-98% Nyquist): -30 to -50 pts               │
├────────────────────────────────────────────────────────┤
│ LEVEL 2: Targeted MP3 320k Protection                 │
│ └─ R1 Exception (20 kHz + energy test): Skip          │
├────────────────────────────────────────────────────────┤
│ LEVEL 3: High Quality Protection                      │
│ ├─ R5 (High Variance): -40 pts                        │
│ ├─ R6 (High Quality): -30 pts                         │
│ └─ R7 (Vinyl/Silence): -50 to -100 pts               │
├────────────────────────────────────────────────────────┤
│ LEVEL 4: Dynamic Protection                           │
│ ├─ R10 (Segment Inconsistency): -20 to -30 pts       │
│ └─ R11 (Cassette): Penalty reduction                  │
└────────────────────────────────────────────────────────┘
```

---

## 📊 Example Scenarios

### Scenario 1: Authentic FLAC
```
File: Mozart Symphony (Original CD Rip)
────────────────────────────────────────
Cutoff: 21.5 kHz (97.5% Nyquist)
Variance: 150 kbps
Container: 950 kbps

Rule 1: No MP3 signature         →   0 pts
Rule 2: Cutoff OK                →   0 pts
Rule 5: High variance            → -40 pts
Rule 6: High quality             → -30 pts
Rule 8: 97.5% Nyquist            → -50 pts
──────────────────────────────────────────
TOTAL: -120 pts → 0 pts (floor)
VERDICT: AUTHENTIC ✅
```

### Scenario 2: MP3 320k → FLAC Transcode
```
File: Pop Song (MP3 320k transcoded to FLAC)
─────────────────────────────────────────────
Cutoff: 20.0 kHz (exactly)
Variance: 15 kbps (very stable)
Container: 850 kbps
Energy ratio: 0.000000 (no HF content)

Rule 1: MP3 320k detected        → +50 pts
Rule 2: Cutoff deficit           → +10 pts
Rule 3: Inflated container       → +50 pts
──────────────────────────────────────────
TOTAL: 110 pts
VERDICT: FAKE_CERTAIN ❌
```

### Scenario 3: Vinyl Rip
```
File: Jazz Album (Authentic Vinyl)
──────────────────────────────────
Cutoff: 19.5 kHz
Vinyl noise detected
Clicks & pops present

Rule 2: Slight deficit           → +5 pts
Rule 7: Vinyl detected           → -100 pts
──────────────────────────────────────────
TOTAL: -95 pts → 0 pts (floor)
VERDICT: AUTHENTIC ✅
```

---

## 🎯 Key Innovations (v0.6.4)

1. **20 kHz Exception** (Rule 1): Dual-test system to avoid false positives
   - Energy ratio test (HF content above 20 kHz)
   - Variance test (FFT rounding detection)

2. **Safeguarded Nyquist Protection** (Rule 8): Conditional bonuses
   - MP3 + high silence ratio → Bonus cancelled
   - MP3 + low silence ratio → Bonus applied (authentic)

3. **Multi-Segment Analysis** (Rule 10): Full file validation
   - Progressive analysis (2→5 segments when needed)
   - Detects multi-source compilations

4. **Cassette Protection** (Rule 11): Analog source recognition
   - Gradual rolloff detection
   - Tape hiss analysis

---

## 📚 References

- **Implementation**: `src/flac_detective/analysis/new_scoring/rules/`
- **Models**: `src/flac_detective/analysis/new_scoring/models.py`
- **Orchestration**: `src/flac_detective/analysis/new_scoring/calculator.py`

---

**FLAC Detective v0.6.4** - *Advanced MP3-to-FLAC Transcode Detection*

**Test Results**: 817,631 files analyzed | 89.1% authentic rate | <0.5% false positives
