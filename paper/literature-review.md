# Literature Review

7 papers cited in the Bayaan research paper. Each addresses a distinct aspect of the problem.

---

## 1. QuranMB.v1 — The Primary Benchmark

**El Kheir, Y. et al. (2025).** *Towards a Unified Benchmark for Arabic Pronunciation Assessment: Quranic Recitation as Case Study.*  
Interspeech 2025 & ArabicNLP Shared Task 2025. arXiv:2506.07722.

First publicly available test set for Quranic mispronunciation detection. Introduced a 68-phoneme MSA inventory and a data processing pipeline.

**Role in Bayaan:** QuranMB.v1 is the dataset that the IqraEval 2025 shared task used. QuranMB.v2 (from IQRA 2026) is Bayaan's primary evaluation benchmark — held out until Phase 9.

---

## 2. Large-Scale Automated Pipeline

**Anonymous. (2025).** *Automatic Pronunciation Error Detection and Correction of the Holy Quran's Learners Using Deep Learning.*  
arXiv:2509.00094.

850+ hours of expert-reciter audio, ~300K annotated utterances. Introduced Quran Phonetic Script (QPS) — two-level encoding of phoneme identity + Tajweed articulation properties. wav2vec2-BERT for segmentation + Tasmeea algorithm for transcript verification. Achieves 0.16% PER on test set.

**Role in Bayaan:** Direct contrast. Requires server-grade infrastructure and expert-reciter data. Not deployable on mobile. Bayaan achieves a comparable use case with QDAT (1,509 clips, MIT license) + ONNX classifiers.

---

## 3. IqraEval 2025 — First Shared Task

**El Kheir, Y. et al. (2025).** *Iqra'Eval: A Shared Task on Qur'anic Pronunciation Assessment.*  
In Proceedings of ArabicNLP 2025. ACL Anthology: 2025.arabicnlp-sharedtasks.61.

First shared task on Quranic pronunciation assessment. 15 participating teams. Used QuranMB.v1. All teams used pure ML approaches — no deterministic component, no learner feedback.

**Role in Bayaan:** Establishes the baseline landscape and confirms that the entire community defaulted to pure ML before Bayaan.

---

## 4. Hafs2Vec — Top IqraEval 2025 System

**Ibrahim, A. (2025).** *Hafs2Vec: A System for the IqraEval Arabic and Qur'anic Phoneme-level Pronunciation Assessment.*  
ACL Anthology: 2025.arabicnlp-sharedtasks.62.

Second-best system at IqraEval 2025. Based on wav2vec2-xls-r-1b, trained on 94 hours of Quranic recitations (EveryAyah/QUL — 28 professional reciters, 54K clips) + IqraEval training set (79h, 74K clips). Custom Quranic phonemizer generating context- and Tajweed-aware phoneme sequences. Phoneme Error Rate below 9.50% on MSA speech.

**Role in Bayaan:** Confirms wav2vec2 (XLSR family) as the proven backbone for Quranic acoustic classification. Hafs2Vec validates the model choice; Bayaan uses the smaller `xlsr-53-arabic` (300M vs 1B) to stay ONNX-feasible.

---

## 5. IQRA 2026 — Current SOTA and Explicit Gaps

**(IQRA 2026 Organizers). (2026).** *IQRA 2026: Interspeech Challenge on Automatic Pronunciation Assessment for Modern Standard Arabic.*  
arXiv:2603.29087.

19 participating teams. Released QuranMB.v2 + Iqra_Extra_IS26 (first dataset of real human mispronounced MSA speech). Top systems:

| Team | F1 | Approach |
|------|----|----------|
| whu-iasp | 0.7201 | wav2vec2-xls-r-300m + TCN + curriculum learning |
| UTokyo | 0.7170 | CROTTC optimal transport reformulation |
| RAM | 0.7157 | agreement-based data filtering |
| Kalimat | 0.6702 | Qwen3-ASR 1.7B + LoRA (first generative approach) |

**Gaps explicitly identified by organizers:**
1. No phoneme-to-character mapping → limits practical learner-facing use
2. Systems output phoneme sequences; learners need Arabic script feedback
3. Authentic mispronunciation data still scarce

**Role in Bayaan:** The source of the problem statement. The gap quote is the paper's opening argument. Bayaan's FGR metric directly answers gap #1 and #2.

---

## 6. TajweedAI — Nearest Prior Hybrid System

**Mazid, N. et al. (2025).** *TajweedAI: A Hybrid ASR-Classifier for Real-Time Qalqalah Detection in Quranic Recitation.*  
NeurIPS 2025 (Workshop). OpenReview: AauWmDPOIf.

Closest prior work to Bayaan's approach. Hybrid pipeline: Whisper ASR for temporal alignment + binary classifier for phonetic rule verification. Case study: Qalqalah only (one rule, one specific word). Hard negative mining on the team's own mispronunciations.

**Results:** 100% accuracy internal / 57.14% accuracy external — significant overfitting to the training word.

**Limitations:** No learner feedback output; no character mapping; no multi-rule generalization; no mobile deployment.

**Role in Bayaan:** Validates the hybrid architecture. Bayaan differentiates by covering 7+ rules across both classes, producing Arabic script-level character-mapped feedback, deploying on-device via ONNX, and applying the Class A/B principled partition. TajweedAI validates the hybrid approach; Bayaan operationalizes it at scale.

---

## 7. Critical Review — Calls for Hybrid

**(Authors). (2025).** *A Critical Review of the Need for Knowledge-Centric Evaluation of Quranic Recitation.*  
arXiv:2510.12858.

> *"The future of automated Quranic recitation assessment lies in hybrid systems that combine linguistic expertise with advanced audio processing, built on canonical pronunciation principles and articulation points (Makhraj), rather than depending purely on statistical patterns."*

**Role in Bayaan:** Provides the theoretical justification for the entire approach. Published before Bayaan — our paper is the implementation answer to its call. This paper is cited in §1 (Introduction) and §2 (Related Work) to frame the hybrid approach as a principled response to community consensus, not an arbitrary design choice.

---

## Summary Table

| Paper | Year | What it establishes |
|-------|------|-------------------|
| QuranMB.v1 | 2025 | Benchmark and 68-phoneme inventory |
| Large-Scale Pipeline | 2025 | SOTA at scale — requires 850h, server-grade |
| IqraEval 2025 | 2025 | First shared task; all 15 teams = pure ML |
| Hafs2Vec | 2025 | wav2vec2-xls-r as proven backbone |
| IQRA 2026 | 2026 | Current SOTA (F1=0.72); explicit gap statement |
| TajweedAI | 2025 | Validates hybrid; overfits to single word |
| Critical Review | 2025 | Theory: hybrid is the right path |
