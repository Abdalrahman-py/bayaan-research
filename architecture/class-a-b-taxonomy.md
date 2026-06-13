# Class A / Class B Taxonomy

The theoretical contribution of this paper. A principled partition of Tajweed rules based on what information is required to detect each violation.

---

## The Core Insight

Every Tajweed rule either:

- **Can be detected from the text + ASR transcript alone** (no audio signal needed), or
- **Requires analysis of the acoustic signal** (text alone is insufficient)

Every IQRA 2026 team fed all rules into the same ML model. Bayaan uses the cheapest sufficient tool per rule class.

This is not a performance optimization — it is architecturally correct. Using ML for Class A rules (where deterministic detection is exact) introduces unnecessary noise and errors. Using a rule engine for Class B (where the acoustic property is what defines the rule) produces meaningless results.

---

## Class A — Text-Determinable Violations

Given the Quranic text and a phoneme alignment (which audio segment maps to which character), violations can be detected by rule without analyzing acoustic features.

The Whisper transcript is compared against what the rule engine *expects* to be produced. If the student produced something different, that is a violation.

| Rule | Trigger (from text) | What correct sounds like | Violation signal |
|------|--------------------|--------------------------|--------------------|
| **Idgham** | ن ساكنة or تنوين followed by يرملون | ن fully assimilates into following letter — not pronounced separately | Distinct ن in transcript where it should have merged |
| **Ikhfaa** | ن ساكنة or تنوين followed by 15 specific letters | ن is hidden (not fully pronounced, not fully merged) with nasal quality | Clear, full ن transcribed where hiding was expected |
| **Iqlab / Qalb** | ن ساكنة or تنوين followed by م | ن transforms to م-sound with nasalization (labial assimilation) | ن transcribed instead of م-sound |
| **Waqf violations** | Waqf (stop) markers embedded in Quranic text (وقف, صلى, etc.) | Stop at obligatory markers; no stop at prohibition markers | Long pause at non-stop position, or no stop at mandatory marker |

### Why no ML for Class A

When alignment is correct, rule engine detection is exact for these rules. A statistical model cannot outperform a deterministic rule on a deterministic problem. The only failure mode for Class A is alignment error — if Whisper misaligns the transcript, the rule engine sees the wrong phoneme. The fix is better ASR, not ML.

---

## Class B — Acoustically-Dependent Violations

Rules that cannot be detected from text alone. The definition of these rules is inherently acoustic — duration, intensity, vibration quality, articulation point. No amount of text analysis can determine whether the student held a Madd for 4 counts instead of 2.

| Rule | What it means | Why acoustic | Detection method |
|------|--------------|--------------|-----------------|
| **Ghunnah** | 2-beat (~500ms) nasalization on ن مشددة and م مشددة | Duration + nasality — measurable only from audio | Binary classifier: P(correct Ghunnah) |
| **Madd** | Vowel prolongation: 2 counts (short), 4 counts (medium), 6 counts (obligatory) depending on context | Duration of vowel extension — purely acoustic | Binary classifier: P(correct Madd duration) |
| **Qalqalah** | Echo/plosive-release vibration on قطبجد at sukoon | Vibration quality at articulation — acoustic only | Binary classifier: P(correct Qalqalah) — paper-only, TajweedAI comparison |
| **Makhraj** | Articulation point violations (e.g., ح from throat vs ه) | Same letter, different anatomical production point | Future work — requires fine-grained acoustic features |

### How Class B works in Bayaan

The Tajweed Engine does the localization — it tells the classifier *where* in the text each Class B rule applies. The classifier is only asked: "was this clip acoustically correct for this rule, yes or no?"

This is different from what IQRA 2026 teams did. They asked one large model: "find all errors in this audio." Bayaan asks two specialized questions:
1. Engine: "where does this rule occur?" (from text)
2. Classifier: "was it correct at this location?" (from audio)

Because the engine answers question 1 from text, every Class B violation automatically inherits `char` and `char_index` — enabling FGR.

---

## The Known Limitation

Class B classifiers operate at clip level (one classification per ayah). If an ayah contains two Ghunnah positions and the classifier says "incorrect," both positions are flagged. Attribution of *which one* was wrong is coarse.

For Al-Fatihah (MVP scope), this is rare — most ayahs have at most one occurrence of each Class B rule. For the paper, this is noted in §6 (Analysis) and §8 (Conclusion) as future work: fine-grained position attribution within ayah.

The positions themselves are always correct (engine-derived). Only the attribution between multiple occurrences of the same rule within one ayah is coarse.

---

## Comparison to Every IQRA 2026 Team

| | IQRA 2026 teams | Bayaan |
|--|----------------|--------|
| Architecture | Single end-to-end model | Class A rules + Class B classifiers |
| Rule treatment | All rules → same model | Cheapest sufficient tool per class |
| Text context | Lost at input | Preserved by engine throughout |
| Output | Phoneme-level error flags | Character-mapped violation reports |
| Phoneme→char mapping | None | Native (engine never loses text) |
| FGR | 0% | ≥ 90% (target) |
| Mobile deployment | Not designed for | ONNX, backend now, on-device possible |
