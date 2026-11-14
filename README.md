# RECOM: Reddit Evaluation for Correspondence of Models

A post-cutoff evaluation dataset for assessing how Large Language Models (LLMs) align with community perspectives on open-ended, opinion-driven questions.

This repository accompanies our paper:
**"RECOM: A Validity–Discrimination Tradeoff in Automatic Metrics for Open-Ended Reddit Question Answering."** IEEE International Conference on Big Data (IEEE BigData 2026).

## Headline finding

We define and measure two distinct things any evaluation metric is asked to do:

- **Validity**: does a high score reflect genuine content alignment, or could it equally arise from surface coincidence? Measured as the real-vs-random effect size against an explicit derangement baseline. Because a deranged answer comes from an unrelated question, even a merely on-topic answer clears this floor.
- **Discriminative power**: can the metric separate one system from another? Measured as the largest inter-model paired Cohen's *d*. A larger separation shows that the metric responds to differences between systems, not that its ordering is correct.

Across the lexical, semantic, and inference-based metrics we evaluate, **no metric does both jobs well**. Cosine similarity has the largest real-vs-random separation (*d* ≈ 2) but barely separates the five models on raw scores (|*d*| < 0.1). BERTScore precision *appears* to rank them (raw |*d*| up to 0.63), but once response length is controlled this falls to **0.09–0.18**, so its apparent discrimination is largely a length effect. The two axes are not strongly rank-correlated across the 13 metrics (Spearman ρ = −0.18, p = 0.57): validity varies widely while discrimination is limited, and no metric is strong on both. Three independent LLM judges (Phi-4, Gemma-3-27B, Yi-1.5-34B) reproduce the validity gap with large effect sizes (paired *d* = 2.0–3.5; Krippendorff's α ≥ 0.85 over real and random items, 0.39 and 0.43 within the real items alone) and likewise separate the five models only weakly.

## Dataset statistics

| Statistic | Value |
|---|---|
| Total questions collected | 132,726 |
| Top posts by number of depth-1 replies | 25,000 |
| Sampled for evaluation | 15,000 |
| After refusal filtering (released here) | **11,528** |
| Retention rate | 76.9% |
| Source | r/AskReddit, September 1–30, 2025: monthly Reddit dump files (RS_2025-09, RC_2025-09, Pushshift format) downloaded from Academic Torrents, since Pushshift API access has been restricted since 2023 |
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
├── appendix/
│   └── RECOM_appendix.pdf       supplementary appendix to the paper
├── LICENSE
└── README.md
```

## File schemas

### `recom.xlsx` (11,528 rows × 7 columns)

| Column | Type | Description |
|---|---|---|
| `post_id` | string | Reddit post ID; links back to the original thread on Reddit, including its authors, unless the thread has been deleted |
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

- **`inputs.parquet`** (10,000 rows = 1,000 posts × 5 models × 2 conditions) — the exact analysis subsample the paper reports on: the first 1,000 posts of the seed-42 5,000-post draw, in post-ID order, for which all three judges returned valid scores on all ten items (37 earlier posts were skipped because Yi-1.5 returned unparseable output). Columns: `judge_row_id`, `post_id`, `model` ∈ {mistral_7b, qwen2.5_7b, granite3.1-dense_8b, falcon3_10b, llama3.1_8b}, `condition` ∈ {real, random}, `question`, `gen_answer`, `replies` (list of up to 50 reply strings).
- **`outputs_*.parquet`** (10,000 rows each) — raw per-row judge scores. Join to `inputs.parquet` on `judge_row_id`. Columns: `judge_row_id`, `relevancy_score`, `relevancy_reasoning`, `relevancy_raw_response`, `relevancy_error`, `faithfulness_score`, `faithfulness_reasoning`, `faithfulness_raw_response`, `faithfulness_error`. All three judges have valid integer scores on all 10,000 rows.
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

Inference: Ollama 0.12.3, temperature 0.8, top-p 0.9, top-k 40, seed 42, context 8,192 tokens, max generation 512 tokens. All five models served at uniform Q4_K_M quantization on NVIDIA RTX A5000 GPUs. The judges are also 4-bit: Phi-4 and Gemma-3-27B at Q4_K_M, Yi-1.5-34B at Q4_0.

## Refusal filtering

Questions on which **any** of the five models' answers matched one of the following 19 regular expressions were removed entirely, so every model is compared on the same 11,528 posts. Matching is case-insensitive and searches anywhere in the answer (`re.IGNORECASE | re.DOTALL`).

```
as an ai
i'm an ai
i am an ai
as a language model
i'm a language model
i am a language model
as an ai language model
i'm a large language model
i am a large language model
i don't (have|experience|eat|feel|possess)
i lack (emotions|feelings|consciousness|physical form)
i cannot (eat|feel|experience)
as an ai.*?i (don't|lack|cannot)
i (don't|lack|cannot).*?as an ai
i'm not (human|a person|capable of)
i am not (human|a person|capable of)
as an artificial intelligence
i'm just (a|an) (ai|bot|language model)
i am just (a|an) (ai|bot|language model)
```

15,000 → 11,528 (76.9% retention). Per-model refusal rates range from 3.5% (Qwen2.5-7B) to 12.2% (Granite-3.1-Dense-8B).

## Evaluation protocol

For each post, every model's answer is scored against **every depth-1 community reply** (mean 56.2 / median 14 replies per post). Per-post scores are aggregated across replies with the family's literature convention — **max** for BLEU / ROUGE / BERTScore (standard multi-reference protocol; a candidate need only match one reference well), **mean** for cosine similarity, MoverScore, and NLI (community-level alignment). Both aggregations are computed for every metric.

A **random-derangement baseline** pairs each post with a different post's generated answer (seed 42, no fixed points), providing an empirical noise floor on the same scale for every metric. Validity = real-vs-random paired Cohen's *d*; discriminative power = max inter-model paired |*d*| in the real condition.

**Metric families:**

- **Lexical** — BLEU-1…4 (Method-1 smoothing), ROUGE-1/2/L F1 (Porter stemming).
- **Semantic** — BERTScore precision/recall/F1 (RoBERTa-large, layer 12), raw and baseline-rescaled; MoverScore over DistilBERT embeddings; cosine similarity with `all-mpnet-base-v2` and `all-MiniLM-L6-v2`.
- **Inference** — NLI classification (BART-large-MNLI), per-sentence entailment/contradiction/neutral.

## Discriminative power under length control

A central finding: inter-model discrimination is largely a response-length effect. Residualizing each candidate's own word count out of the metric (single pooled OLS across all five models) gives:

| Metric | Raw max \|*d*\| | Length-residualized \|*d*\| | Pooled length *R²* |
|---|---|---|---|
| BERTScore precision | 0.63 | **0.09** | 0.169 |
| BERTScore F1 | 0.53 | 0.11 | 0.133 |
| MoverScore | 0.37 | **0.34** | 0.0003 |

Model identity and mean response length are nearly collinear (across the ten model pairs, BERTScore-precision |*d*| tracks the mean length gap at *r* = 0.99), so these residualized values are estimates rather than a clean decomposition; the same correction raises cosine similarity's largest |*d*| from 0.08 to 0.24. A model-free check agrees for BERTScore: on posts where two models' answers differ by at most three words (not a random sample), the maximum |*d*| falls to 0.17 for precision and 0.12 for F1, while cosine rises to 0.23–0.28.

Some discrimination survives length control (MoverScore, NLI contradiction 0.49 → 0.44, BERTScore recall 0.28 → 0.31), but it is small and found in metrics that do not also lead on validity.

## Post-cutoff collection

The September 2025 collection window postdates the training cutoffs of every evaluated model by at least nine months, so every reference reply postdates all cutoffs. Question phrasings do recur: 8.6% of questions have a near-verbatim match among November 2022 r/AskReddit questions, falling to 2.7% among questions of nine or more words.

| Model | Training cutoff | Source |
|---|---|---|
| Mistral-7B-Instruct-v0.3 | ≤ May 2024† | [HuggingFace metadata](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.3) |
| Qwen2.5-7B-Instruct | ≤ Dec 2023‡ | [Developer disclosure](https://github.com/QwenLM/Qwen2.5/issues/525) |
| Granite-3.1-Dense-8B-Instruct | ≤ Apr 2024† | [Ollama model card](https://ollama.com/library/granite3.1-dense:8b) |
| Falcon3-10B-Instruct | ≤ Dec 2024† | [HuggingFace Falcon blog](https://huggingface.co/blog/falcon3) |
| Llama-3.1-8B-Instruct | Dec 2023 | [HuggingFace model card](https://huggingface.co/meta-llama/Llama-3.1-8B-Instruct) |

†Cutoff not officially disclosed; model release date used as a conservative upper bound. ‡Stated by the Qwen team.

## Ethical use

Per the paper's Ethics statement, RECOM is intended **for research use only** and is released under **CC BY-NC 4.0**. The data are publicly posted Reddit content obtained from third-party monthly dump files distributed via Academic Torrents, not from the Reddit API; users did not consent to this specific study, and we do not deanonymize users. The release contains the 11,528 post-filter questions, the five models' generated answers, depth-1 reply text, and Reddit post IDs. Author fields are removed, but reply text is released as posted (a small number of replies mention other users by handle or contain links), and anyone with the post IDs can look up the original threads on Reddit, including their authors' usernames, unless those threads have since been deleted. Community replies reflect the views of users active in each thread, may over-represent dominant viewpoints, and should not be interpreted as authoritative ground truth. r/AskReddit is a general-audience subreddit, but we did not perform NSFW or toxicity screening of the released questions or replies, which we note as a limitation of the release.

## Citation

```
@inproceedings{recom2026,
  title     = {RECOM: A Validity-Discrimination Tradeoff in Automatic Metrics for Open-Ended Reddit Question Answering},
  author    = {Krishnappa, Pushwitha and Das, Amit and Jain, Vinija and Chadha, Aman and Mukherjee, Tathagata},
  booktitle = {2026 IEEE International Conference on Big Data (BigData)},
  year      = {2026},
  url       = {https://github.com/pushwithak/recom}
}
```

## Changelog

- **2026-11-14 (camera-ready).** README and appendix updated to match the published paper. `llm_judge/` now contains exactly the 1,000 posts the paper analyses; the previous release held the first 1,000 posts in row order, 32 of which were outside the analysis set (posts with a Yi-1.5 parse failure). The 968 posts common to both versions are unchanged.
