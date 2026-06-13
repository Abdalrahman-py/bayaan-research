# Datasets

Four datasets used in Bayaan. Their roles are distinct — do not conflate them.

---

## 1. QDAT — Primary Training Data

**Source:** `obadx/qdat` on HuggingFace (cleaned version of original Kaggle QDAT)  
**Paper:** Harere, A. A., & Al Jallad, K. *"QDAT: A dataset for Reciting the Quran"*  
**License:** MIT ✅ (free to use, modify, redistribute, commercial use allowed)

### Why the HuggingFace version

The `obadx/qdat` version is pre-processed:
- Downsampled to **16kHz mono PCM WAV** — exactly what wav2vec2 expects
- Duplicate entries removed
- Mislabeling corrected (original had errors in the `s18` recording group)

### Columns Used by Bayaan

| Column | Tajweed Rule | Clips | Balance |
|--------|-------------|-------|---------|
| `the_tight_noon` | Ghunnah (النون المشددة) | 289 incorrect + 1,220 correct = 1,509 total | ~4:1 correct-heavy |
| `separate_tide` | Madd Munfasil (المد المنفصل) | 654 incorrect + 856 correct = 1,510 total | ~1.3:1 (balanced) |
| `concealment` | Ikhfaa (إخفاء) | Out of scope for MVP | — |

**Label semantics (settled):** `label = 1 → correct recitation`, `label = 0 → incorrect`. The classifier outputs P(correct). A violation is flagged when `P(correct) < threshold`.

### Key Facts

- 1,509 total clips
- 160 speakers, one verse: *"قالوا لا علم لنا إلا ما علمتنا إنك أنت العليم الحكيم"*
- Each speaker recorded the same verse 9–10 times
- Class imbalance on Ghunnah (4:1) requires `pos_weight` in BCEWithLogitsLoss during training

### Citation

```
Harere, A. A., & Al Jallad, K. "QDAT: A dataset for Reciting the Quran."
Dataset: https://huggingface.co/datasets/obadx/qdat (MIT License)
```

---

## 2. EveryAyah — Correct Recitation Diversity

**Source:** everyayah.com — professional recitations of the full Quran  
**License:** Download for personal/research use (verify per reciter)  
**Role:** Additional `correct` (label=1) examples to prevent the model from overfitting to QDAT's single verse

### Why We Need It

QDAT contains one verse, 160 speakers. A classifier trained only on QDAT might learn verse-specific patterns rather than the rule itself. EveryAyah provides 20–28 professional reciters across many verses, all correctly recited.

**Usage:** Download clips → label as `correct` (label=1) → mix into QDAT training set. Do not use for incorrect examples (EveryAyah contains no annotated mistakes).

### How many to use

Download 20–28 reciters. Extract Al-Fatihah + a few additional surahs. Aim for 300–500 additional correct clips per rule to balance out QDAT's correct-heavy distribution without drowning QDAT.

---

## 3. Iqra_train — Verse and Speaker Diversity + Qalqalah

**Source:** IqraEval 2025 HuggingFace space (`iqraeval/iqra-eval-2025`)  
**Size:** 79 hours, ~74K clips, diverse speakers  
**Role in Bayaan:** Two purposes:
1. Additional verse/speaker diversity for Ghunnah + Madd classifiers
2. The **only available labeled source for Qalqalah** (paper-only, optional)

### Qalqalah from Iqra_train

TajweedAI (NeurIPS 2025) overfit their Qalqalah classifier to a single word and achieved only 57.14% accuracy on external data. Bayaan avoids this by training across **at least 3–4 different surahs**. Iqra_train + EveryAyah provides the diversity needed.

**Only download if:** the Qalqalah classifier is in scope. For the graduation MVP, Ghunnah + Madd are sufficient. Qalqalah is added if it strengthens the TajweedAI comparison in the paper.

---

## 4. QuranMB.v2 — Held-Out Test Set (LOCKED)

**Source:** IQRA 2026 challenge (Interspeech 2026)  
**What it is:** The benchmark used by all 19 IQRA 2026 teams — real human mispronounced MSA speech, professionally annotated  
**Role:** Paper evaluation only — held out until Phase 9

### CRITICAL: Do Not Touch Until Phase 9

QuranMB.v2 is locked from the moment it is downloaded. It is not used for:
- Debugging classifiers
- Threshold calibration
- Any exploratory analysis

Only opened when the full system is ready for evaluation. Using it earlier would invalidate the paper's comparison to IQRA 2026 baselines.

### What it enables

- Direct comparison to IQRA 2026 top systems (whu-iasp F1=0.7201, UTokyo F1=0.7170, RAM F1=0.7157)
- Proof that Bayaan's results are on the same benchmark, not a cherry-picked test set
- FGR metric computed on the same data as IQRA 2026 F1 → the comparison is apples-to-apples

---

## Dataset Summary

| Dataset | Role | License | Status |
|---------|------|---------|--------|
| QDAT (`obadx/qdat`) | **Primary training** — Ghunnah + Madd labeled clips | MIT ✅ | Download + use |
| EveryAyah | Correct-clip diversity for training | Per reciter | Download 20–28 reciters |
| Iqra_train | Verse diversity + Qalqalah (optional) | Research use | Download if Qalqalah in scope |
| QuranMB.v2 | **Paper evaluation only — LOCKED** | IQRA 2026 challenge | Download, do not open until Phase 9 |
