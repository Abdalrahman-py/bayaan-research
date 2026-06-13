# Implementation Roadmap

**Timeline:** June 14 – late September 2026 (~15 weeks)  
**Principle:** One system. The app is the artifact; the paper is the measurement and write-up.

---

## Phase Map

```
Phase 0 → Phase 1 → Phase 2 + 3 (parallel) → Phase 4
 Setup     Data           Whisper              Class B
  1wk       1wk          Engine + Class A     Classifiers
                           1.5wk + 2wk         2wk
                                │
                                ▼
                       Phase 5 ∥ Phase 6
                     Hybrid Layer  LLM Feedback
                         1wk each
                                │
                                ▼
                       Phase 7 ∥ Phase 8
                      Backend    Android UI
                       2wk        (ongoing)
                                │
                                ▼
                 Phase 9 → Phase 10 → Phase 11 → Phase 12
                  Eval     User Study   Paper     Defense
                  1wk        1wk        3wk        1-2wk
```

---

## Weekly Tracker

| Week | Dates | Phase | Milestone |
|------|-------|-------|-----------|
| 1 | Jun 14–20 | 0 | Environment + datasets ready; QuranMB.v2 locked |
| 2 | Jun 21–27 | 1 | `labels.csv` from QDAT |
| 3–4 | Jun 28 – Jul 11 | 2 + 3 | Whisper alignment + engine + Class A (30 tests pass) |
| 4–5 | Jul 9–22 | 4 | Ghunnah + Madd ONNX, val F1 ≥ 0.75 |
| 6 | Jul 23–29 | 5 + 6 | Hybrid layer + RAG feedback |
| 6–7 | Jul 23 – Aug 5 | 7 | `/audio/analyze` done, latency measured |
| 8 | Aug 6–13 | 9 | QuranMB.v2 results |
| 9 | Aug 13–20 | 10 | User study (5–10 participants) |
| 11–13 | Aug 24 – Sep 13 | 11 | Paper written |
| 14+ | Sep 14+ | 12 | Defense + paper submission |

---

## Phase 0 — Environment & Data Acquisition (Jun 14–20)

### Goals
- `ml/.venv` activates; torch, transformers, onnxruntime import successfully
- QDAT downloaded from `obadx/qdat` (16kHz mono version)
- EveryAyah clips downloaded (20–28 reciters)
- QuranMB.v2 downloaded and **locked** (do not open until Phase 9)
- Iqra_train downloaded (needed only if Qalqalah is in scope)

### Key check
```bash
cd /home/jade/bayaan/ml
source .venv/bin/activate
python -c "import torch, transformers, onnxruntime; print('ok')"
```

---

## Phase 1 — Data Preparation (Jun 21–27)

QDAT is already labeled. Phase 1 is converting it to the format `dataset.py` expects — not manual labeling.

### Target format
`data/processed/labels.csv` with columns: `filename, rule, label, split`

### Label semantics (settled and locked)
- `label = 1` → correct recitation
- `label = 0` → incorrect recitation
- Classifier outputs P(correct)
- Violation flagged when P(correct) < threshold

### Class imbalance fix
Ghunnah is ~4:1 correct-to-incorrect. Use `pos_weight` in BCEWithLogitsLoss:
```python
neg = (train_labels == 0).sum()
pos = (train_labels == 1).sum()
criterion = nn.BCEWithLogitsLoss(pos_weight=torch.tensor([neg / pos], device=device))
```

### Success criteria
- [ ] `labels.csv` exists with columns: `filename, rule, label, split`
- [ ] Ghunnah: ≥ 1,200 rows; Madd: ≥ 1,400 rows
- [ ] Stratified train/val/test split per rule
- [ ] A handful of `.wav` files spot-checked: 16kHz mono, plays correctly

---

## Phase 2 — Whisper Alignment + Tajweed Engine (Jun 28 – Jul 8)

The intelligence of the system. No ML training — ASR + deterministic logic.

### 2.1 Whisper transcription + alignment
- POST `/audio/transcribe` endpoint
- Calls Groq Whisper with `response_format=verbose_json, timestamp_granularities=["word"]`
- Returns: `{ "words": [{"word":"بِسْمِ","start":0.0,"end":0.45}, ...] }`

Test on a recording of Al-Fatihah and confirm timings are correct before building anything on top.

### 2.2 Expected-phoneme / rule-position generator
`TajweedEngine.expected(verse_text)` returns per-character:
```python
{"char": "ن", "char_index": 12, "expected_phoneme": "...", "rules_here": ["Idgham"]}
```
This is what lets every later violation carry `char_index` — the foundation of FGR.

### Success criteria
- [ ] `/audio/transcribe` returns word timestamps for Al-Fatihah
- [ ] `TajweedEngine.expected("بِسْمِ اللَّهِ الرَّحْمَٰنِ الرَّحِيمِ")` returns correct rule positions
- [ ] Every Ghunnah/Madd/Idgham position in Al-Fatihah verified manually
- [ ] ASR latency < 400ms per ayah

---

## Phase 3 — Class A Deterministic Rule Engine (Jun 28 – Jul 11, parallel with Phase 2)

### Detection by transcript comparison
`RuleEngine.analyze(verse_text, transcript, word_timings)` → list of violations

Each violation:
```python
{
  "rule": "Idgham", "class": "A",
  "char": "ن", "char_index": 12, "verse": "1:7",
  "start_ms": 840, "end_ms": 1100,
  "expected": "ن assimilates into following ي",
  "detected": "ن pronounced distinctly",
  "severity": "major"
}
```

### Rules to implement

| Rule | Trigger | Violation signal |
|------|---------|----------------|
| Idgham | ن ساكنة/تنوين before يرملون | distinct ن where it should merge |
| Iqlab/Qalb | ن ساكنة/تنوين before م | ن instead of م-sound |
| Ikhfaa | ن ساكنة/تنوين before 15 letters | clear ن where hiding expected |
| Waqf | stop markers in text | pause at non-stop, or no pause at mandatory |

### Success criteria
- [ ] `RuleEngine.analyze()` runs end-to-end on Al-Fatihah
- [ ] All 4 Class A rules implemented
- [ ] 30 unit tests pass (mixed correct/violation fixtures)
- [ ] Violation format includes `char + char_index + expected + detected`
- [ ] False-positive check: zero violations on 10 correct recitations

---

## Phase 4 — Class B Acoustic Classifiers (Jul 9–22)

The repo already has `train.py`, `classifier.py`, `export_onnx.py`, `inference.py`. Modify, don't rewrite.

### Model swap (the critical change)
```python
# from:
backbone = Wav2Vec2Model.from_pretrained("facebook/wav2vec2-base")
# to:
backbone = Wav2Vec2Model.from_pretrained("jonatasgrosman/wav2vec2-large-xlsr-53-arabic")
# freeze entire backbone (large model + small data = head-only training):
for p in backbone.parameters():
    p.requires_grad = False
```
`classifier.py` adapts automatically (reads `config.hidden_size` = 1024 for the Arabic model).

### Training (on Kaggle GPU)
```bash
python -m src.train --rule ghunnah --data-dir data/processed --epochs 50
python -m src.train --rule madd    --data-dir data/processed --epochs 50
```
**Target:** val F1 ≥ 0.75 per rule

### ONNX export
```bash
python -m src.export_onnx --rule ghunnah --checkpoint checkpoints/ghunnah_best.pt --output models/ghunnah.onnx
python -m src.export_onnx --rule madd    --checkpoint checkpoints/madd_best.pt    --output models/madd.onnx
```

Input: float32 `[1, 192000]`, 16kHz mono  
Output: single logit → sigmoid → P(correct)

### Success criteria
- [ ] Arabic xlsr backbone, fully frozen
- [ ] Ghunnah val F1 ≥ 0.75
- [ ] Madd val F1 ≥ 0.75
- [ ] Both exported to ONNX, verification passed
- [ ] `TajweedInference.predict()` returns P(correct) on a test ayah on backend

---

## Phase 5 — Hybrid Decision Layer (Jul 23–29)

Merges Class A + Class B into one structured output. **Engine localizes; classifier judges.**

### Key metric: Feedback Grounding Rate
```
FGR = |violations with char_index + expected| / |total violations|
```
IQRA 2026 teams: FGR = 0%. Bayaan target: FGR ≥ 90%.

### Success criteria
- [ ] `HybridDetector.analyze()` runs end-to-end
- [ ] Output merges Class A + Class B in one format
- [ ] FGR computed per analysis
- [ ] Zero violations on 10 professional reciter clips

---

## Phase 6 — LLM Feedback + TTS (Jul 23–29, parallel with Phase 5)

RAG is required. Supervisor reviewed the Al-Azani et al. (2026) paper recommending RAG as a religious safeguard.

### RAG index
Matn Al-Jazariyyah + Tuhfat Al-Atfal rule descriptions + 5–10 teacher-written corrections per rule (supervisor reviews before user study).

### Pipeline
Violation report → retrieve → generate (Claude/Gemini) → TTS (ElevenLabs `eleven_multilingual_v2`, Arabic voice)

### Success criteria
- [ ] Feedback names char + position specifically
- [ ] RAG context present for every supported rule
- [ ] TTS produces audible Arabic
- [ ] 5 examples reviewed by native speaker / supervisor

---

## Phase 7 — Backend Integration (Jul 23 – Aug 5)

`POST /audio/analyze` endpoint in Ktor. ML runs as FastAPI microservice at `localhost:8001`. Ktor calls it, adds feedback + TTS, saves to Supabase. Both deploy on Railway via Docker.

---

## Phase 8 — Android UI (Issa + Osama, ongoing July–Aug)

Abdalrahman provides the `/audio/analyze` JSON schema + a mock server so Android develops before ML is ready.

---

## Phase 9 — Evaluation on QuranMB.v2 (Aug 6–13)

**Now** open QuranMB.v2. Run `HybridDetector` over every clip.

Report:
- F1 per rule, per class (A/B), overall
- Comparison to IQRA 2026: whu-iasp 0.7201, UTokyo 0.7170, RAM 0.7157
- FGR (theirs = 0%, ours target ≥ 90%)
- Latency median + p95

**Framing:** contribution is FGR + hybrid, not beating overall F1. Class A deterministic rules should outperform ML baselines when alignment is accurate. If Class A underperforms, the culprit is alignment — investigate Whisper, not the rules.

---

## Phase 10 — User Study (Aug 13–20)

5–10 users. 15-min session. Recite Al-Fatihah ayahs 1–4, read feedback, retry, fill 5-question Likert survey. Report mean ± SD. Mark "preliminary."

---

## Phase 11 — Write the Paper (Aug 24 – Sep 13)

By now every section writes itself from artifacts produced in Phases 2–10.

- **Week 1:** §1 Intro + §2 Related Work + §3 Class A/B taxonomy
- **Week 2:** §4 Architecture + §5 Setup + §6 Results
- **Week 3:** §7 User Study + §8 Conclusion + Abstract (fill [X] and [W] from Phase 9) + polish

Submit to ArabicNLP 2027, or EMNLP 2026 workshop if a deadline fits. arXiv preprint first.

---

## Phase 12 — Graduation Defense (Sep 14+)

Live demo: recite with a deliberate Madd error → see it highlighted on the character → hear the TTS correction → retry clean.

Defense headline: *"19 teams worldwide, all FGR = 0%. Bayaan = ~92%."*

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Arabic xlsr overfits small QDAT | Medium | High | Full backbone frozen, head-only; pos_weight; early stopping |
| QDAT single-verse → poor generalization | Medium | High | Mix EveryAyah + Iqra clips; validate on held-out Al-Fatihah |
| Whisper timestamps too coarse for a rule | Low | Medium | Class A uses transcript comparison — timing only needed for Class B positions |
| Clip-level Class B can't disambiguate 2 occurrences | Medium | Low | Rare in Al-Fatihah; note as limitation; positions still correct |
| LLM gives wrong Tajweed advice | Medium | High | RAG grounding + supervisor reviews 10 corrections before user study |
| Paper F1 < IQRA overall | Medium | Low | Contribution is FGR + hybrid; F1=0.65 with FGR=90% still enables what nobody else does |
