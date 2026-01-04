# RECOM: Reddit Evaluation for Correspondence of Models

## Overview

**RECOM** (Reddit Evaluation for Correspondence of Models) is a benchmark dataset for evaluating how Large Language Models (LLMs) align with community perspectives on temporally recent open-domain questions.

This repository accompanies our paper: **"Assessing LLM Reliability on Temporally Recent Open-Domain Questions"**

## Dataset Statistics

| Metric | Value |
|--------|-------|
| Total questions collected | 132,728 |
| High-engagement filtered | 25,000 |
| Sampled for evaluation | 15,000 |
| After response filtering | 11,515 |
| Retention rate | 76.8% |
| Source | r/AskReddit (September 2025) |
| Models evaluated | 4 (Llama-3.1-8B, Mistral-7B, Gemma-2-9B, GPT-OSS-20B) |

## Dataset Files

### 1. `recom_full.csv`
Complete dataset of 15,000 questions with reference summaries and all model responses.

| Column | Description |
|--------|-------------|
| `id` | Unique identifier |
| `original_post` | Original Reddit question |
| `summary` | Community consensus summary (Llama-3.1-8B) |
| `generated_answer_gemma2_9b` | Gemma-2-9B response |
| `generated_answer_gpt_oss_20b` | GPT-OSS-20B response |
| `generated_answer_llama3.1_8b` | Llama-3.1-8B response |
| `generated_answer_mistral_7b` | Mistral-7B response |

### 2. `recom_filtered.csv`
Filtered evaluation dataset (11,515 questions) after removing refusal responses.

Same schema as above, with refusal patterns removed.

## Filtering Criteria

Responses containing the following patterns were excluded:

| Category | Patterns |
|----------|----------|
| **AI self-references** | "as an AI", "I'm a language model", "as an artificial intelligence" |
| **Capability disclaimers** | "I don't have personal", "I cannot experience", "I lack emotions" |
| **Role disclaimers** | "I'm not human", "I am not capable of" |


## Key Findings

| Finding | Result |
|---------|--------|
|  **Semantic-Lexical Paradox** | >99% cosine similarity despite <8% BLEU-1 overlap |
|  **Scale ≠ Performance** | Mistral-7B (7B) outperforms GPT-OSS-20B (20B) |
|  **Low Contradiction** | <7% contradiction rate across all models |
|  **MoverScore Gradient** | 51–53% intermediate alignment scores |
