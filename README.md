# RECOM: Reddit Evaluation for Correspondence of Models

A contamination-free evaluation dataset for assessing how Large Language Models (LLMs) align with community perspectives on open-ended, opinion-driven questions.

This repository accompanies our paper:
**"RECOM: A Validity–Discrimination Tradeoff in Automatic Metrics for Open-Ended Reddit Question Answering."**

## Headline finding

We define and measure two distinct things any evaluation metric is asked to do:

- **Validity** — does a high score reflect genuine content alignment, or could it equally arise from surface coincidence? Measured as the real-vs-random effect size against an explicit derangement baseline.
- **Discriminative power** — can the metric rank one system above another? Measured as the largest inter-model paired Cohen's *d*.

Across the lexical, semantic, and inference-based metrics we evaluate, **no metric does both jobs well**. Cosine similarity has the largest real-vs-random separation (*d* ≈ 2) but cannot rank the five models (|*d*| < 0.1). BERTScore precision *appears* to rank them (raw |*d*| up to 0.63), but once response length is residualized this collapses to **|*d*| = 0.09** — so its apparent discrimination is largely a length effect. Three independent LLM judges (Phi-4, Gemma-3-27B, Yi-1.5-34B) reproduce the validity gap with large effect sizes (paired *d* = 2.0–3.5; Krippendorff's α ≥ 0.85) and likewise separate the five models only weakly.

## Dataset statistics

| Statistic | Value |
|---|---|
| Total questions collected | 132,728 |
| Filtered to top-engagement pool | 25,000 |
| Sampled for evaluation | 15,000 |
| After refusal filtering (released here) | **11,528** |
| Retention rate | 76.9% |
| Source | r/AskReddit (September 1–30, 2025) |
| Reference set | Depth-1 community replies (mean 56.2 / median 14 per post) |
| Total (reply, answer) comparisons | 647,973 per model |
| Evaluated LLMs | Mistral-7B-Instruct-v0.3, Qwen2.5-7B-Instruct, Granite-3.1-Dense-8B-Instruct, Falcon3-10B-Instruct, Llama-3.1-8B-Instruct |

## Repository layout

```
recom/
├── recom.xlsx                   11,528 rows: post_id, question, 5 model answers
├── recom_replies.parquet        647,973 rows: post_id, reply_text (long format)
├── recom_replies.xlsx           11,528 rows: post_id, depth_1_replies (joined; truncated where >32k chars)
├── llm_judge/
│   ├── inputs.parquet           10,000 rows: judge_row_id, post_id, model, condition, question, gen_answer, replies (the 1,000-post analysis subsample × 5 models × 2 conditions)
│   ├── outputs_phi4.parquet     Phi-4 (14B) raw scores per judge_row_id (10,000 rows)
│   ├── outputs_gemma3.parquet   Gemma-3-27B-Instruct raw scores (10,000 rows)
│   ├── outputs_yi.parquet       Yi-1.5-34B-Chat raw scores (10,000 rows)
│   └── prompts/
│       ├── relevancy_prompt.txt
│       └── faithfulness_prompt.txt
├── LICENSE
└── README.md
```

## File schemas

### `recom.xlsx` (11,528 rows × 7 columns)

| Column | Type | Description |
|---|---|---|
| `post_id` | string | Reddit post ID (anonymous; usable for rehydration via the Reddit API) |
| `question` | string | The r/AskReddit post title — the open-ended question |
| `generated_answer_mistral_7b` | string | Mistral-7B-Instruct-v0.3 response (≤50 words, see prompt below) |
| `generated_answer_qwen2.5_7b` | string | Qwen2.5-7B-Instruct response |
| `generated_answer_granite3.1-dense_8b` | string | Granite-3.1-Dense-8B-Instruct response |
| `generated_answer_falcon3_10b` | string | Falcon3-10B-Instruct response |
| `generated_answer_llama3.1_8b` | string | Llama-3.1-8B-Instruct response |

### `recom_replies.parquet` (647,973 rows × 2 columns, long format)

| Column | Type | Description |
|---|---|---|
| `post_id` | string | Joins to `recom.xlsx` |
| `reply_text` | string | Single depth-1 community reply. Authors removed. Reddit's `[removed]` / `[deleted]` markers preserved as-is. |

Each post has a variable number of replies (median 14, mean 56). Join `recom.xlsx` to `recom_replies.parquet` on `post_id` to reconstruct the full multi-reference scoring set used in the paper.

`recom_replies.xlsx` provides the same data in wide format (one row per post, replies joined by `\n\n---\n\n`) for convenience; 395 rows are truncated at ~32,000 characters to fit Excel's cell limit (full text in the parquet).

### `llm_judge/`

Reproducibility bundle for the three-judge validation.

- **`inputs.parquet`** (10,000 rows = 1,000 posts × 5 models × 2 conditions) — the analysis subsample described in the paper. Columns: `judge_row_id`, `post_id`, `model` ∈ {mistral_7b, qwen2.5_7b, granite3.1-dense_8b, falcon3_10b, llama3.1_8b}, `condition` ∈ {real, random}, `question`, `gen_answer`, `replies` (list of up to 50 reply strings). The judges were actually run over a larger 5,000-post seed-42 random draw; we release only the 1,000-post analysis slice that the paper reports on.
- **`outputs_*.parquet`** (10,000 rows each) — raw per-row judge scores. Join to `inputs.parquet` on `judge_row_id`. Columns: `judge_row_id`, `relevancy_score`, `relevancy_reasoning`, `relevancy_raw_response`, `relevancy_error`, `faithfulness_score`, `faithfulness_reasoning`, `faithfulness_raw_response`, `faithfulness_error`. Phi-4 and Gemma-3 have valid integer scores on all 10,000 rows; Yi-1.5 has 9,964 valid Relevancy scores and 9,999 valid Faithfulness scores (the remainder are JSON-parsing errors captured in the `*_error` columns).
- **`prompts/`** — verbatim prompt templates passed to each judge (Relevancy: question + answer only; Faithfulness: question + answer + up to 50 chronologically-earliest replies). Both prompts request JSON output with `reasoning` and integer score fields; judges run at temperature 0 with JSON-constrained output.

## Answer-generation protocol

Every model was prompted with identical instructions:

```
You are answering this question.
Give a direct answer without restating or summarizing the question.
Rules:
- Answer ONLY
- Be clear and concise
- Keep it under 50 words
```

Inference: Ollama 0.12.3, temperature 0.8, top-p 0.9, top-k 40, seed 42, context 8,192 tokens, max generation 512 tokens. All five models served at uniform Q4_0 quantization on NVIDIA RTX A5000 GPUs.

## Refusal filtering

Questions on which **any** of the five models produced one of the following patterns were removed entirely (so every model is compared on the same 11,528 posts):

| Category | Patterns |
|---|---|
| AI self-references | "as an AI", "I'm a language model", "as an artificial intelligence" |
| Capability disclaimers | "I don't have", "I cannot experience", "I lack emotions" |
| Role disclaimers | "I'm not human", "I am not capable of" |

15,000 → 11,528 (76.9% retention).

## Evaluation protocol

For each post, every model's answer is scored against **every depth-1 community reply** (mean 56.2 / median 14 replies per post). Per-post scores are aggregated across replies with the family's literature convention — **max** for BLEU / ROUGE / BERTScore (standard multi-reference protocol; a candidate need only match one reference well), **mean** for cosine similarity, MoverScore, and NLI (community-level alignment). Both aggregations are computed for every metric.

A **random-derangement baseline** pairs each post with a different post's generated answer (seed 42, no fixed points), providing an empirical noise floor on the same scale for every metric. Validity = real-vs-random paired Cohen's *d*; discriminative power = max inter-model paired |*d*| in the real condition.

**Metric families:**

- **Lexical** — BLEU-1…4 (Method-1 smoothing), ROUGE-1/2/L F1 (Porter stemming).
- **Semantic** — BERTScore precision/recall/F1 (RoBERTa-large, layer 12), raw and baseline-rescaled; MoverScore over DistilBERT embeddings; cosine similarity with `all-mpnet-base-v2` and `all-MiniLM-L6-v2`.
- **Inference** — NLI classification (BART-large-MNLI), per-sentence entailment/contradiction/neutral.

## Length-residualized discriminative power

A central finding: inter-model discrimination is largely a response-length effect. Residualizing each candidate's own word count out of the metric (single pooled OLS across all five models) collapses:

| Metric | Raw max \|*d*\| | Length-residualized \|*d*\| | Pooled length *R²* |
|---|---|---|---|
| BERTScore precision | 0.63 | **0.09** | 0.169 |
| BERTScore F1 | 0.53 | 0.11 | 0.133 |
| MoverScore | 0.37 | **0.34** | 0.0003 |

MoverScore is the one metric whose discrimination survives length control — so genuine, length-independent system separation exists, but it is small and on a metric with only modest validity.

## Temporal contamination control

The September 2025 collection window postdates the training cutoffs of every evaluated model by at least nine months:

| Model | Training cutoff | Source |
|---|---|---|
| Mistral-7B-Instruct-v0.3 | ≤ May 2024† | [HuggingFace metadata](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.3) |
| Qwen2.5-7B-Instruct | ≤ Dec 2023‡ | [Developer disclosure](https://github.com/QwenLM/Qwen2.5/issues/525) |
| Granite-3.1-Dense-8B-Instruct | Apr 2024† | [Ollama model card](https://ollama.com/library/granite3.1-dense:8b) |
| Falcon3-10B-Instruct | ≤ Dec 2024† | [HuggingFace Falcon blog](https://huggingface.co/blog/falcon3) |
| Llama-3.1-8B-Instruct | Dec 2023 | [HuggingFace model card](https://huggingface.co/meta-llama/Llama-3.1-8B-Instruct) |

†Cutoff not officially disclosed; model release date used as a conservative upper bound. ‡Stated by the Qwen team.

## Ethical use

Per the paper's Ethics statement, RECOM is intended **for research use only** and is released under **CC BY-NC 4.0**. All usernames have been removed; the release contains only question text, depth-1 reply text, and non-personally-identifying Reddit post IDs. Community replies should not be interpreted as authoritative ground truth — they reflect the views of users active in each thread and may over-represent dominant viewpoints. r/AskReddit is predominantly safe-for-work; we did not perform explicit NSFW or toxicity screening, which we note as a limitation of the release.

## Citation

```
@misc{recom2026,
  title  = {RECOM: A Validity-Discrimination Tradeoff in Automatic Metrics for Open-Ended Reddit Question Answering},
  author = {Krishnappa, Pushwitha and Das, Amit and Jain, Vinija and Chadha, Aman and Mukherjee, Tathagata},
  year   = {2026},
  url    = {(https://github.com/pushwithak/recom)(https://github.com/pushwithak/recom)}
}
```
