# Paper Abstract

**Title:** From Phonemes to Feedback: A Hybrid Rule-Acoustic System for Learner-Facing Tajweed Mispronunciation Detection

---

## Abstract

Over 1.8 billion Muslims learn Quranic recitation governed by Tajweed — a phonological rule system where violations can alter meaning. Two successive shared tasks (IqraEval 2025, IQRA 2026) have produced systems reaching F1 = 0.72, yet every participating system operates at the phoneme level. The IQRA 2026 organizers explicitly flag the absence of phoneme-to-character mapping as the primary obstacle to learner-facing applications.

We present **Bayaan**, a hybrid system that partitions Tajweed rules into two classes: text-determinable violations detected by a deterministic rule engine from ASR transcription and word-level alignment, and acoustically-dependent violations judged by Arabic-pretrained wav2vec2 (XLSR-53) binary classifiers exported to ONNX.

The deterministic engine localizes every error from the verse text — the classifier only judges acoustic correctness — so the system preserves Arabic script context throughout, producing **character-mapped violation reports** with the exact script position and expected pronunciation.

This is the first system to solve the phoneme-to-character gap natively. Evaluated on QuranMB.v2, Bayaan achieves F1 = **[X]** overall, with Class A rules outperforming pure ML baselines. A Feedback Grounding Rate of **[W]%** confirms that every flagged violation is mapped to its Arabic character with LLM-generated pedagogical correction.

> **Note:** [X] and [W] will be filled after QuranMB.v2 evaluation in August 2026.

---

## Core Claims

1. First principled partition of Tajweed rules into text-determinable (Class A) vs acoustically-dependent (Class B)
2. First system to produce character-mapped violation reports with natural language pedagogical feedback
3. Evaluation on QuranMB.v2 with comparison to IQRA 2026 baselines
4. Lightweight ONNX classifiers (Arabic XLSR, frozen backbone) — backend-deployed for MVP, on-device-capable for future mobile deployment

---

## The Gap We Fill

From the IQRA 2026 overview paper (arXiv:2603.29087):

> *"There is no trivial or universal mapping from a predicted erroneous phoneme back to the corresponding Arabic character, limiting practical learner-facing applications. Current systems operate at the phoneme level, but learners interact with Arabic script."*

Two years of shared tasks. 19+ participating teams. All pure ML pipelines. None produce learner-facing script-level feedback. Bayaan addresses both identified gaps through a hybrid architecture.
