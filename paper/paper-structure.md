# Paper Structure

**Title:** From Phonemes to Feedback: A Hybrid Rule-Acoustic System for Learner-Facing Tajweed Mispronunciation Detection

**Target Venue:** ArabicNLP 2027 (primary) · EMNLP 2026 workshops (alternative)

---

## Section 1 — Introduction (~1 page)

- Motivation: 1.8B Muslims, Tajweed learning without qualified teachers
- Cite IqraEval 2025 and IQRA 2026 as current SOTA
- Quote the IQRA 2026 gap statement explicitly
- State the 4 contributions

**Contributions:**
1. First principled partition of Tajweed rules into text-determinable (Class A) vs acoustically-dependent (Class B)
2. First system to produce character-mapped violation reports with natural language pedagogical feedback
3. Evaluation on QuranMB.v2 with comparison to IQRA 2026 baselines
4. Lightweight ONNX classifiers (Arabic XLSR, frozen backbone) — backend-deployed for MVP, on-device-capable for future mobile Tajweed evaluation

---

## Section 2 — Related Work (~0.5 pages)

Seven papers, one paragraph each. Lead with TajweedAI (nearest hybrid prior work), then position Bayaan's broader scope.

See [`literature-review.md`](literature-review.md) for the full summaries.

**Order:**
1. TajweedAI (NeurIPS 2025) — nearest hybrid prior, Qalqalah only
2. QuranMB.v1 benchmark (El Kheir et al., 2025) — primary eval set
3. Large-scale automated pipeline (arXiv:2509.00094) — contrast: server-grade, not mobile
4. IqraEval 2025 shared task — first shared task, 15 teams, all pure ML
5. Hafs2Vec — top IqraEval system, wav2vec2 + custom phonemizer
6. IQRA 2026 — current SOTA, explicit gap statement
7. Critical Review (arXiv:2510.12858) — calls for hybrid, justifies our approach

---

## Section 3 — Background: Tajweed Rules and Detectability (~0.5 pages)

The Class A / Class B partition with full justification. This is the theoretical foundation of the paper.

**Class A — Text-Determinable:**

| Rule | Detection Logic |
|------|----------------|
| Idgham | ن ساكنة or تنوين before يرملون → assimilation. Text context sufficient. |
| Ikhfaa | ن ساكنة before 15 letters → nasalization. Phonological context. |
| Iqlab/Qalb | ن ساكنة before م → labial assimilation. Deterministic substitution. |
| Waqf violations | Stopping at non-marked positions. Check against standard pause markers. |

**Class B — Acoustically-Dependent:**

| Rule | Why Acoustic |
|------|-------------|
| Ghunnah | 2 beats (~500ms) of nasalization. Duration + nasality is acoustic only. |
| Madd | Vowel prolongation: 2/4/6 counts. Duration is purely acoustic. |
| Qalqalah | Echo/plosive-release on قطبجد at sukoon. Vibration quality is acoustic. |
| Makhraj | Articulation point errors. Same letter, different production. |

**Why this matters:** Every IQRA 2026 team fed all rules into the same model. Our architecture is principled — we use the cheapest sufficient tool per rule class, and the rule engine maintains text context throughout, solving the phoneme-to-character mapping problem natively.

---

## Section 4 — System Architecture (~1.5 pages)

### 4.1 ASR and Alignment (Tarteel Whisper)
Quran-fine-tuned Whisper via Groq API or `MaddoggProduction/whisper-l-v3-turbo-quran-lora`. Word-level timestamps. One model, one job.

### 4.2 Deterministic Rule Engine (Class A)
Pure Python. Given verse text + Whisper transcript + word timings, compares transcribed phonemes to expected post-Tajweed phonemes. Detects Idgham, Iqlab/Qalb, Ikhfaa, Waqf. No ML.

### 4.3 wav2vec2 Acoustic Classifiers (Class B)
`jonatasgrosman/wav2vec2-large-xlsr-53-arabic` — frozen backbone + 2-layer classification head. Binary per rule (Ghunnah, Madd; Qalqalah optional for TajweedAI comparison). ONNX export for backend deployment.

### 4.4 Hybrid Decision Layer
Merges Class A (deterministic) + Class B (acoustic) into a unified violation report. Every violation carries: `{rule, class, char, char_index, verse, start_ms, end_ms, expected, detected, severity}`.

### 4.5 LLM Feedback Generation (RAG-grounded)
Structured violation → retrieved Tajweed rule context (Matn Al-Jazariyyah / Tuhfat Al-Atfal) → Claude/Gemini → 1–2 sentence Arabic correction → ElevenLabs TTS. RAG is mandatory to prevent hallucination (see [`rag-research/`](../rag-research/rag-for-quranic-content.md)).

---

## Section 5 — Experimental Setup (~0.5 pages)

- **Training data:** QDAT (obadx/qdat) primary — Ghunnah (the_tight_noon) + Madd (separate_tide), 1,509 clips, 16kHz mono, MIT license. Augmented with EveryAyah professional recitations (correct = label 1).
- **Evaluation:** QuranMB.v2 (held-out, locked until Phase 9)
- **Baselines:** IQRA 2026 top systems — whu-iasp F1=0.7201, UTokyo F1=0.7170, RAM F1=0.7157
- **Metrics:** F1 per rule, per class (A/B), overall F1; Feedback Grounding Rate (FGR)

**FGR definition:**
```
FGR = |violations with char_index + expected description| / |total violations|
```
Every IQRA 2026 system has FGR = 0%. Bayaan target: FGR ≥ 90%.

---

## Section 6 — Results and Analysis (~0.5 pages)

Per-rule F1 breakdown. Hypothesis: Class A rules (deterministic) outperform pure ML baselines when alignment is accurate. Class B acoustic rules — compare directly to IQRA 2026 baselines.

**Results table (to fill after Phase 9):**

| Rule | Class | Bayaan F1 | Best IQRA 2026 |
|------|-------|-----------|----------------|
| Idgham | A | [TBD] | — |
| Iqlab | A | [TBD] | — |
| Ikhfaa | A | [TBD] | — |
| Ghunnah | B | [TBD] | 0.7201 |
| Madd | B | [TBD] | 0.7201 |
| Overall | — | [TBD] | 0.7201 |
| **FGR** | — | **[TBD]** | **0%** |

---

## Section 7 — Pedagogical Feedback Evaluation (~0.5 pages)

5–10 users (mix: Tajweed-knowledgeable for accuracy validation; beginners for clarity). 15-min session: recite Al-Fatihah ayahs 1–4, read feedback, retry, fill 5-question Likert survey.

**Survey questions:**
1. Was the error correctly identified?
2. Was the explanation clear?
3. Did it tell me what to do differently?
4. Did I know how to fix it after reading?
5. Would I use this system again?

Report: mean ± SD per question. Mark "preliminary."

---

## Section 8 — Conclusion

- Addressed the two gaps IQRA 2026 explicitly identified
- First character-mapped feedback system for Tajweed
- Mobile deployment path (ONNX, backend now, on-device future)
- Future: more rules, larger user study, dedicated Tajweed phoneme inventory
