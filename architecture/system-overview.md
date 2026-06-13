# System Architecture

Bayaan is a hybrid system. Two components work together, each doing what it does best.

---

## Architecture Diagram

```
Student recites a verse
        │
        ▼
┌──────────────────────────────────┐
│  Quran-fine-tuned Whisper         │   ← ASR + alignment
│  (Tarteel-style; Groq API or      │     transcribes audio
│   MaddoggProduction/whisper-lora) │     → word-level timestamps
└──────────────────────────────────┘
        │  transcript + word timings
        ▼
┌──────────────────────────────────┐
│  Tajweed Engine                   │   ← knows from TEXT what should
│  - expected post-Tajweed phonemes │     happen and WHERE each rule
│  - rule occurrence positions      │     occurs (no audio needed)
└──────────────────────────────────┘
        │
    ┌───┴───────────────────────────┐
    ▼                               ▼
┌────────────────────┐   ┌──────────────────────────┐
│ Class A            │   │ Class B                   │
│ Deterministic      │   │ Acoustic Classifiers      │
│ Rule Engine        │   │                           │
│                    │   │ wav2vec2-large-xlsr-53    │
│ Idgham, Iqlab,    │   │ -arabic                   │
│ Ikhfaa, Waqf       │   │ frozen backbone +         │
│                    │   │ 2-layer head → ONNX       │
│ Compare Whisper    │   │                           │
│ transcript vs      │   │ Binary per rule:          │
│ expected text.     │   │ → P(correct) for clip     │
│ Zero ML.           │   │                           │
│                    │   │ Rules: Ghunnah, Madd      │
│                    │   │ (Qalqalah: paper-only)    │
└────────────────────┘   └──────────────────────────┘
        └───────────┬───────────────┘
                    ▼
        ┌──────────────────────────┐
        │ Hybrid Decision Layer     │   ← merges into one report:
        │                           │     every violation carries its
        │ {rule, class, char,       │     Arabic character + position
        │  char_index, verse,       │     (this is what enables FGR)
        │  start_ms, end_ms,        │
        │  expected, detected,      │
        │  severity}                │
        └──────────────────────────┘
                    ▼
        ┌──────────────────────────┐
        │ LLM Feedback             │   ← Claude/Gemini, grounded via RAG
        │ (RAG-grounded)            │     in Tajweed rule sources
        └──────────────────────────┘
                    ▼
        ┌──────────────────────────┐
        │ TTS (ElevenLabs Arabic)   │   ← reads correction aloud
        └──────────────────────────┘
```

---

## Three Principles That Never Change

**1. Whisper is the only aligner.**  
No second forced-alignment model. Word-level timestamps are enough for everything we need. Both Class A and Class B benefit from word timing; sub-word alignment is not required for the MVP.

**2. The engine localizes; the classifier judges.**  
The position of every error comes from text + alignment (deterministic), never from the acoustic classifier. The classifier only answers "was this acoustic property correct, yes or no" at clip level. This is why Bayaan can produce character-mapped output — the text always knows where it is.

**3. Classifiers are narrow.**  
One small binary classifier per acoustic rule. They are not a giant statistical model that swallows audio and emits all errors — that is exactly what all 19 IQRA 2026 teams did, and it is what we are differentiating against.

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Android app | Kotlin Multiplatform + Compose |
| Backend | Ktor + Supabase (PostgreSQL) + Railway |
| Auth/sync | Firebase Auth + Firestore |
| ASR + alignment | Quran-fine-tuned Whisper (Tarteel-style), word-level timestamps |
| Tajweed engine | Pure Python — rules + expected-phoneme generator |
| Class B classifiers | `wav2vec2-large-xlsr-53-arabic` frozen backbone + 2-layer head → ONNX |
| Inference runtime | ONNX on backend (Python service). On-device = future work |
| LLM feedback | Claude Sonnet / Gemini 2.5 Flash + RAG |
| TTS | ElevenLabs Arabic (`eleven_multilingual_v2`) |
| Training | Kaggle GPU (free P100/T4) |

---

## Violation Report Format

Every violation produced by the Hybrid Decision Layer follows this structure:

```json
{
  "rule": "Ghunnah",
  "class": "B",
  "char": "ن",
  "char_index": 12,
  "verse": "1:3",
  "start_ms": 840,
  "end_ms": 1100,
  "expected": "2-beat nasalization on ن مشددة",
  "detected": "P(correct)=0.21",
  "severity": "major"
}
```

**`char_index`** is what enables Feedback Grounding Rate. It is always set — derived from the text by the Tajweed Engine, regardless of whether the violation is Class A or Class B.

---

## Feedback Grounding Rate (FGR)

The novel metric introduced in the paper:

```
FGR = |violations with char_index + expected description| / |total violations|
```

- Every IQRA 2026 system: **FGR = 0%** (they output phoneme sequences with no text mapping)
- Bayaan target: **FGR ≥ 90%**

FGR is not a replacement for F1 — it measures something different. F1 measures detection accuracy; FGR measures whether detected errors are actionable by a human learner.

---

## Latency Budget (Backend)

| Component | Target |
|-----------|--------|
| Upload + ffmpeg | < 100ms |
| Whisper ASR | < 200ms |
| Class A engine | < 10ms |
| Class B ONNX (2 rules) | < 150ms |
| LLM feedback | < 400ms |
| TTS | < 300ms |
| **Total** | **~1,160ms** |

Report measured median + p95 honestly. 800ms was the proposal aspiration; real latency over 20 trials is what the paper will cite.
