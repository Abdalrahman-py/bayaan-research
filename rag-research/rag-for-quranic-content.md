# RAG for Qur'anic Content

RAG is a mandatory architectural safeguard for Bayaan's feedback generation — not a nice-to-have. Six papers and shared tasks establish this unanimously.

---

## Why RAG is Non-Negotiable

General-purpose LLMs (even Claude, Gemini, GPT-4) when prompted with Qur'anic content:
- Hallucinate verses and incorrect interpretations
- Ignore tafsīr context (asbāb al-nuzūl, scholarly tradition)
- Cannot cite authentic sources
- Generate confident-sounding but wrong religious rulings

Bayaan's feedback tells students what to fix in their recitation of the Quran. Incorrect feedback is not a UX problem — it is a religious accuracy problem. RAG grounds every generated response in verified Tajweed sources.

---

## Source Papers

### 1. Al-Azani, Abudalfa & Samma (2026) — Comprehensive Review

**Venue:** Information Processing & Management (ScienceDirect)  
**Scope:** 151 studies on AI in Qur'anic research

Strong recommendation: RAG is a necessity, not an optimization, when using LLMs with Qur'anic content. General models hallucinate verses, ignore tafsīr context, and cannot cite authentic sources.

**Bayaan relevance:** This is the paper Bayaan's supervisor shared. It is the primary motivator for the RAG requirement in the feedback pipeline.

---

### 2. AQRAG — Advanced Quranic Retrieval-Augmented Generation

**Abd Elhakeem et al. (2026)**  
**Venue:** Neural Computing and Applications 38:282 (Springer, April 2026)  
**DOI:** 10.1007/s00521-026-11886-7

Results:
- Baseline (no RAG, AraELECTRA fine-tuned): 64% pRR
- RAG + Gemini 2.5 Flash: 89.9%
- RAG + GPT-4.1-nano + reranking: **90.5%** — new SOTA on QRCD v1.2

The retrieve → rerank → generate pattern is directly applicable to Bayaan's feedback pipeline. Instead of sending raw verse + violation to the LLM, retrieve relevant tafsir passages, rerank by contextual relevance, then generate grounded feedback.

**Repo:** https://github.com/Mr-Array22/AI-Mufassir

---

### 3. 13 Open-Source LLMs with RAG on Quranic Studies (2025)

**Venue:** IJACSA 16(2), arXiv:2503.16581  
**Models tested:** Llama3:70b, Gemma2:27b, QwQ:32b down to Phi3:3.8b

RAG significantly reduces hallucination across all model sizes. Even small models (3B parameters) perform adequately with good RAG grounding.

**Key takeaway for Bayaan:** Model size is secondary to retrieval quality for domain-specific Qur'anic tasks. The retrieval index matters more than whether you use Claude or Gemini.

---

### 4. Amin et al. (2025) — Retrieval-Augmented LLMs for Qur'an and Hadith QA

**Venue:** ArabicNLP 2025 at EMNLP, pages 494-502  
**Task:** IslamicEval 2025 Subtask 2 — Passage Retrieval

Architecture: Fine-tuned BERT sentence transformer for dense retrieval + cross-encoder reranking + zero-answer detection.

Best result: Recall@30 = 0.541, MAP@10 = 0.1809

Multi-stage curriculum fine-tuning pattern: general Arabic → Tafseer → Quran QA. This is the pattern for Bayaan's retrieval index if expanded beyond the MVP RAG approach.

---

### 5. Samy et al. (2025) — RAG for Islamic Knowledge Assessment

**Venue:** ArabicNLP 2025 (QIAS Shared Task), pages 960-965  
**Task:** Multiple-choice Islamic knowledge QA

RAG works across Islamic knowledge domains beyond just Qur'anic text — applicable to Bayaan if expanded to broader Islamic knowledge feedback.

---

### 6. IslamicEval 2025 — Hallucination Benchmark

**Venue:** ArabicNLP 2025 at EMNLP  
**Scope:** First shared task for capturing LLM hallucination in Islamic content

Three subtasks:
1. Qur'anic verse hallucination detection
2. Passage retrieval for Qur'an and Hadith
3. Hallucination correction

The hallucination detection subtask provides evaluation methodology for Bayaan — how to measure whether a generated feedback statement is faithful to the source Tajweed rule.

---

## The Architecture Pattern (from literature consensus)

All successful systems follow the same three-step pattern:

```
Violation report from Hybrid Detector
        ↓
[Retrieval] — Dense embeddings + cosine similarity
  Index: Tajweed rule texts, Matn Al-Jazariyyah,
         Tuhfat Al-Atfal, teacher-written corrections
        ↓
[Reranking] — Filter irrelevant results
  Zero-answer check: if nothing retrieved, do not generate
        ↓
[Grounded Generation] — LLM generates from retrieved context
  Prompt: "Using only the following Tajweed rule text:
           {retrieved_context}
           Explain to the student in Arabic what they did wrong
           and how to correct the {rule} rule at {char} in verse {verse}."
        ↓
Feedback to student (Arabic, 1–2 sentences)
        ↓
TTS (ElevenLabs Arabic)
```

---

## What to Index for Bayaan

| Source | Type | Notes |
|--------|------|-------|
| Matn Al-Jazariyyah | Tajweed reference poem | Classical, authoritative, concise rule definitions |
| Tuhfat Al-Atfal | Tajweed reference | Common beginner text, same level as the app's users |
| Teacher-written corrections | Per-rule examples | 5–10 per rule, reviewed by supervisor |
| Tajweed rule descriptions | Technical | Per-rule: trigger condition + what correct sounds like + common mistake |

**Do NOT index:** Tafsīr, Hadith, or general Islamic content — Bayaan's feedback is about pronunciation, not theology. Keep the index narrow and domain-specific.

---

## What NOT to Do

- Do **not** send a verse to the LLM and ask "is the student's recitation correct?" without retrieved context
- Do **not** generate tafsīr-style explanations from raw LLM knowledge
- Do **not** generate feedback without citing the specific rule and character position
- Do **not** generate if the retrieval step returns nothing relevant — return a safe fallback instead

---

## MVP vs Product Roadmap

**For graduation (MVP):** Simple RAG — retrieve the relevant Tajweed rule passage → include in prompt → generate 1–2 sentence correction. Supervisor reviews 10 examples before user study.

**Product roadmap:** Advanced reranking (cross-encoder), multi-stage curriculum retrieval, hallucination rate tracking, zero-answer detection with graceful fallback.
