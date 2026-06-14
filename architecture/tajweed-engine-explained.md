# Tajweed Engine — How It Works

A plain-language reference for the Tajweed rule engine at the core of Bayaan's hybrid architecture.

---

## What the Engine Is

The Tajweed engine is a **deterministic Python module** — not a trained model, not a download. It encodes centuries-old Tajweed scholarship (Matn Al-Jazariyyah, Tuhfat Al-Atfal) as `if/else` logic that scans Arabic text character by character.

**Inputs:**
1. Arabic verse text (from Quran database, with diacritics — harakat, sukoon, shadda)
2. Whisper ASR transcript + word-level timestamps (from `POST /audio/transcribe`)

**Output:** A structured list of every position where a Tajweed rule applies, what correct behavior looks like, and whether the student violated it.

```json
{
  "char": "ن",
  "char_index": 12,
  "rule": "Idgham",
  "class": "A",
  "expected": "ن fully merges into following ي — no distinct ن sound",
  "detected": "distinct ن found in Whisper transcript at this position",
  "violation": true,
  "start_ms": 840,
  "end_ms": 1100,
  "severity": "major"
}
```

---

## Tajweed Rules Reference

### Nun Sakinah / Tanween Family

**Trigger:** ن with sukoon (ن ساكنة) OR any tanween (ـً ـٍ ـٌ)

The engine finds every such position, then looks at the immediately following letter:

| Rule | Trigger letters | What correct sounds like | Violation signal in transcript |
|------|----------------|--------------------------|-------------------------------|
| **Izhar** | ء ه ع غ خ ح (throat letters) | ن pronounced fully and clearly | N/A — clear ن is correct here |
| **Idgham** | ي ر م ل و ن | ن disappears — merges into next letter | Distinct ن heard where it should merge |
| **Iqlab** | م only | ن transforms to م-sound with nasalization | ن sound instead of م-sound |
| **Ikhfa** | 15 remaining letters | ن hidden — nasal halfway state, not full ن | Clear full ن where hiding was expected |

These 4 rules are **Class A** — fully detectable by comparing the expected behavior against the Whisper transcript. No audio waveform analysis needed.

### Ghunnah

Nasalization held for ~2 counts (~500ms) on ن مشددة and م مشددة (both letters with shadda).

- Position: text-determinable (look for shadda on ن or م)
- Correctness: acoustic-only (duration + nasality quality)
- Class: **B** → handled by wav2vec2 classifier

### Madd (Vowel Prolongation)

Long vowel extension. Three durations depending on context:
- 2 counts: natural Madd (no following hamza or sukoon)
- 4 counts: connected Madd (hamza in same word)
- 6 counts: obligatory Madd (specific contexts)

- Position: text-determinable (Madd letters + context)
- Correctness: acoustic-only (measured duration)
- Class: **B** → handled by wav2vec2 classifier

### Qalqalah

Vibrating/echoing plosive release on قطبجد at sukoon or end of word.

- Position: text-determinable (the 5 letters + sukoon condition)
- Correctness: acoustic-only (vibration quality in waveform)
- Class: **B** → wav2vec2 classifier (also the focus of TajweedAI, NeurIPS 2025)

### Waqf (Stopping)

Stop markers embedded in Quranic text — obligatory (وقف), prohibited (صلى), permissible (ج).

- Position AND correctness: text-determinable (markers in text + silence in timestamps)
- Class: **A** → rule engine only

---

## The Class A / Class B Partition

Every Tajweed rule has two components:
1. **Where does the rule apply?** — always derivable from text
2. **Was it performed correctly?** — sometimes text-comparable, sometimes requires waveform

| Class | Detection method | Examples |
|-------|-----------------|---------|
| **A** | Engine compares expected behavior vs Whisper transcript | Idgham, Iqlab, Ikhfa, Waqf |
| **B** | wav2vec2 XLSR-53 binary classifier on extracted audio clip | Ghunnah, Madd, Qalqalah |

**Every IQRA 2026 team (19 systems, best F1 = 0.72) treated all rules identically** — one end-to-end model sees audio, outputs phoneme-level errors. Using ML for Class A (where deterministic detection is exact) adds unnecessary noise. Using a rule engine for Class B (where the acoustic property defines the violation) produces meaningless results.

Bayaan uses **the cheapest sufficient tool per class.** This is not an optimization — it is architecturally correct.

---

## How the Engine Enables FGR

Because the engine derives rule positions from Arabic text characters, every violation — even Class B violations that route through the acoustic classifier — carries `char_index`: the exact position in the Arabic string.

IQRA 2026 baseline approach:
```
audio → [end-to-end model] → phoneme error flags
                                     ↑
                          no Arabic character here
```

Bayaan's approach:
```
Arabic text → [engine] → char_index 12 = "Idgham violation"
                ↓
          char_index 12 sent to classifier clip extractor (for Class B)
                ↓
          violation output always carries char_index
```

**Feedback Grounding Rate (FGR)** = percentage of flagged violations that are successfully mapped to their Arabic character position. IQRA 2026 teams: FGR = 0%. Bayaan target: ≥ 90%.

---

## ASR and Temporal Alignment

**ASR (Automatic Speech Recognition):** Converts audio to text. Whisper is the ASR model used in Bayaan.

**Temporal alignment:** The ASR output includes *when* each word appears in the audio:
```json
[
  {"word": "بِسْمِ",      "start": 0.00, "end": 0.45},
  {"word": "اللَّهِ",    "start": 0.48, "end": 0.90},
  {"word": "الرَّحْمَٰنِ", "start": 0.92, "end": 1.60}
]
```

Whisper's `verbose_json` mode with `timestamp_granularities=["word"]` produces this. For Class B rules, timestamps let the engine extract the precise audio clip (e.g., 840ms–1100ms) and send only that to the classifier.

---

## wav2vec2 XLSR-53 — Role in Bayaan

wav2vec2 is a **speech encoder**, not an ASR model. No text comes out — acoustic feature vectors come out.

In Bayaan, a binary classification head is trained on top of the frozen wav2vec2 backbone:

```
audio clip (float32, 16kHz) → [wav2vec2 XLSR-53 frozen] → feature vector (1024-dim) → [linear head] → P(correct)
```

- P(correct) > threshold → rule was performed correctly
- P(correct) < threshold → violation flagged (with char_index inherited from engine)

**Why XLSR-53 (Arabic-pretrained):** The model was pre-trained on 53 languages including Arabic. Arabic phoneme characteristics (nasality, Ghunnah quality, throat sounds) are already in the weights. Head-only fine-tuning on Bayaan's small QDAT dataset is sufficient — no need to teach Arabic acoustics from scratch.

Full backbone is frozen during training. Only the classification head (a few linear layers) learns.

---

## Implementation Notes

- Arabic diacritics are Unicode combining characters — handle with `unicodedata.normalize` before indexing
- `TajweedEngine.expected(verse_text)` → map of char_index → expected rule behavior (runs before audio is recorded)
- `RuleEngine.analyze(verse_text, transcript, word_timings)` → violation list (runs after Whisper returns)
- Class B clips are extracted using Whisper word timestamps, padded to 12 seconds (192,000 samples at 16kHz) before passing to wav2vec2
- ONNX export: wav2vec2 head exported for backend inference — input `[1, 192000]`, output single logit
