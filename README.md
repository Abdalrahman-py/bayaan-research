# Bayaan — Hybrid Tajweed Mispronunciation Detection

Research repository for **Bayaan**, a hybrid rule-acoustic system for learner-facing Tajweed mispronunciation detection in Quranic recitation.

---

## What This Is

Over 1.8 billion Muslims learn Quranic recitation governed by Tajweed — a phonological rule system where violations can alter meaning. Two successive shared tasks (IqraEval 2025, IQRA 2026) have produced systems reaching F1 = 0.72, yet every participating system operates at the phoneme level and produces no learner-facing feedback.

**Bayaan** addresses the explicitly-identified gap: the absence of phoneme-to-character mapping that prevents practical learner use.

---

## Key Contribution

We partition Tajweed rules into two classes:

- **Class A (Text-Determinable):** Violations detected by a deterministic rule engine from ASR transcript + text comparison. No ML needed.
- **Class B (Acoustically-Dependent):** Violations requiring audio signal analysis, handled by Arabic-pretrained wav2vec2 (XLSR-53) binary classifiers.

The rule engine localizes every error from the verse text — so the system **preserves Arabic script context throughout**, producing character-mapped violation reports. This solves the phoneme-to-character gap natively.

---

## Repository Structure

```
bayaan-research/
├── paper/
│   ├── abstract.md           # Paper abstract (to be filled after evaluation)
│   ├── paper-structure.md    # Full paper outline with section descriptions
│   └── literature-review.md  # 7 cited papers with detailed summaries
├── architecture/
│   ├── system-overview.md    # Full system architecture with diagram
│   └── class-a-b-taxonomy.md # Theoretical foundation — the Class A/B partition
├── datasets/
│   └── datasets.md           # QDAT, EveryAyah, Iqra_train, QuranMB.v2
├── rag-research/
│   └── rag-for-quranic-content.md  # RAG as mandatory safeguard — 6 papers
└── implementation/
    └── roadmap.md            # Phase-by-phase implementation plan (June–Sept 2026)
```

---

## Paper

**Working Title:** *"From Phonemes to Feedback: A Hybrid Rule-Acoustic System for Learner-Facing Tajweed Mispronunciation Detection"*

**Target Venue:** ArabicNLP 2027 (primary), EMNLP 2026 workshops (alternative)

**Novel Metric:** Feedback Grounding Rate (FGR) — the percentage of violations successfully mapped to their Arabic character with LLM-generated pedagogical correction. Every IQRA 2026 system has FGR = 0%.

---

## Graduation Project

This repository documents the research component of the Bayaan graduation project at UCAS (2026). The app is the artifact; the paper is the measurement and write-up of that artifact.

**Team:** Abdalrahman (AI/backend), Issa (Android UI), Ramzi (infra/DB), Osama (voice/core loop)  
**Supervisor:** Eng. Mahmoud Abu Jadallah
