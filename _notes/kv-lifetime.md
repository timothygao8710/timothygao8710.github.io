---
title: "KV's know their own lifetime"
date: 2026-07-03
description: "Can a key vector predict how long it will keep receiving attention from future queries? Across Llama, Qwen 3, and Gemma 3 (1B-8B) the answer is yes, and it yields a content-based KV-cache eviction policy that ties attention-based baselines on fluent generation and beats them on needle-in-a-haystack."
tags: [llm, inference, efficient-ai-models]
---

# KV's know their own lifetime

An LLM's Query–Key space is known to be low-rank. The following is Llama 3.2 Instruct 1B's QK space projected onto its 3 principal components on the Hamlet Soliloquy (an interactive version can be found [here](https://timothygao8710.github.io/QK-Visualizer/)):

![Llama-3.2-1B's QK space projected onto its top-3 principal components](/images/notes/kv-lifetime/attnsink.gif){: .kv-fig}

*Fig 1. Llama-3.2-1B's query–key space projected onto its 3 principal components on the Hamlet soliloquy*
{: .kv-cap}

Playing around with the visual, we see that keys closest in angle to the current query tend to be in its top-K, and in fact, the token-0 key located near the centroid of the queries is always relevant — this is the attention sink. This makes sense as the attention score `<q, k> = ||q|| ||k|| cos(theta)`, and in Llama, the `cos(theta)` term dominates.\*

In fact, we can plot, for each key, its contribution to the post-softmax attention score of future queries (Fig 2). More specifically, we plot for a fixed `k_i`, `softmax(<q_j, k_i> / sqrt(head dim))`, where `softmax(*)` denotes multiplying it by query `j`'s respective softmax normalization constant. We call this a key's "lifetime". The x-axis in the following diagrams is the difference in index `j − i`.

![A key's lifetime: key 0 (the BOS sink), a random key (#50), and all keys overlaid](/images/notes/kv-lifetime/fig_lifetime_curves.png){: .kv-fig}

*Fig 2. A key's lifetime — the post-softmax attention it receives from future queries (Llama-3.2-3B, layer 10). **(a)** Key 0 **(b)** A typical key (50th key) **(c)** All keys across 10 diverse prompts*
{: .kv-cap}

In this work, we investigate whether there is some direction in query–key space that correlates with how long a particular key will be "alive" — keep receiving non-trivial attention from future queries. We ask if this direction is independent of the query: conditioned on only the information from the key vector alone, can we predict how long it will be "alive"?

We definitively confirm the answer to the question is "yes", across model families, parameter count, quantization, and prompts (Table 1). Furthermore, this mechanistic observation can be used to construct a KV cache eviction algorithm that matches baseline on easy narration tasks and beats baseline on hard needle-in-a-haystack tasks:

![Our method vs existing eviction: perplexity (left) and needle retrieval (right)](/images/notes/kv-lifetime/fig_learned_evict.png){: .kv-fig}
![Needle-in-a-haystack retrieval under eviction](/images/notes/kv-lifetime/fig_learned_niah.png){: .kv-fig}

*Fig 3. Our method vs existing eviction policies (Recent / StreamingLLM / H2O / oracle). **(a)** Fluent generation (perplexity) **(b)** Needle-in-a-haystack*
{: .kv-cap}

![Cross train/eval transfer heatmap](/images/notes/kv-lifetime/explore_transfer.png){: .kv-fig .narrow}

*Fig 4. The predictability transfers across tasks shows the lifetime direction is not a per-task artifact.*
{: .kv-cap}

Unlike current approaches which make eviction decisions based on access patterns — using the heuristic that more accessed = more important — our method learns importance from content. This is desirable because keys corresponding to important "needles" might not be accessed for a long time (i.e., dormant) but "revived" many tokens later — a main failure mode of access-based approaches. However, one downside is that we must decide how important a key will be at birth instead of being able to make this decision lazily in the future, which requires some degree of foresight into what future queries will look like. However, one can argue the creation of a key — for future queries to attend to — encodes this anyway.

Mechanistically analyzing what our predictor learned, we find it's largely a linear, layer-specific direction, but contrary to the QK visual, it lives in a low-variance subspace, and truncating to the top PCA axes kills the signal (Fig 9). Lexically, we find the longest-lived keys are the BOS sink, delimiters, and rare or numeric tokens, while the shortest-lived are common words and continuations — and raw token frequency is an uninformative cue, *which* token it is matters far more than how common it is (Fig 11). Finally, the direction is causal: steering a key along `+w` raises the attention it later receives (+27%) and along `−w` lowers it (−33%).

---

## Eviction is usually a *post-hoc* problem

`Recent` keeps a sliding window of the latest tokens. `StreamingLLM` keeps first four tokens (attention-sink) plus a recent window. `H2O` adds "heavy hitters". The thought is that relevance is observed; a key earns its place in the cache by having *already* been attended to.

![Forward-from-the-key vs backward-from-the-query](/images/notes/kv-lifetime/kv_eviction_1a.png){: .kv-fig .medium}

*Fig 5. Existing eviction policies are from the perspective of the query*
{: .kv-cap}

Unlike existing methods, we predict a key's future lifetime from its content at birth, before any attention has accrued. This seems counterintuitive as we're making decisions at a time when we have less information. Also it's surprising that the same predictor works across vastly different prompts (Fig 4, Fig 6). This suggests the "lifetime direction" is intrinsic to the geometry of the QK space independent of the prompt, and we can condition on this instead of the access pattern. 

Additionally, unlike existing methods that iteratively have to add more inductive biases based on empirical observations of how attention operates (e.g., modifying to take into account attention sinks (StreamingLLM), observations about different layers (PyramidKV), the votes of a recent end-of-prompt query window (SnapKV), the persistence of importance over time — a token pivotal once tends to stay pivotal (Scissorhands), and the distinct attention structure of each individual head (FastGen)), our method hard-codes none of them. A learned content predictor can learn these biases instead of bolting them on one by one.

![Generalization across task regimes and model families](/images/notes/kv-lifetime/fig_scorecard.png){: .kv-fig}

*Fig 6. Our method generalizes across task regimes — and across model families (Llama, Qwen 3, Gemma 3) and scales (1B–8B).*
{: .kv-cap}

## Mechanistically

To further validate our findings, we mechanistically analyzed what it learned.

A token's key vector k_i (shape (head_dim,)) predicts whether that token will still receive attention beyond a series of token horizons (64, 256, 1024) at AUC 0.949. Instead of using a nonlinear MLP, a single hyperplane per layer gets 0.902, so the readout is mostly linear: a lifetime direction.

![Keys projected onto the per-layer lifetime direction, colored by lifetime, for Llama/Gemma/Qwen](/images/notes/kv-lifetime/fig_numberline.png){: .kv-fig}

*Fig 7. Keys projected onto the per-layer lifetime direction `w` form a near-monotone number line, colored by each key's actual lifetime, for Llama / Gemma / Qwen*
{: .kv-cap}

The direction seems to be causal. Pushing a subset of keys along `+w` yields +27% attention, while along `−w` yields −33%, well past an equal-norm random direction (+10% / −15%).

![Causal steering by magnitude and layer, averaged over all heads](/images/notes/kv-lifetime/fig_steer_sweep.png){: .kv-fig}

*Fig 8. Causal steering. We perturb a random subset of keys by `±ε·||k||` along `±w` across magnitudes `ε ∈ {0.5, 1, 2}` and layers, measuring the attention they then receive (averaged over all heads). At layer 1 the lifetime direction changes the attention a key receives by ~30% (up for `+w`, down for `−w`) while an equal-norm random direction barely moves it; deeper layers are messier but `−w` still destroys attention well beyond `−random`.*
{: .kv-cap}

The direction is layer-specific (the 8 per-layer directions are near-orthogonal), aligned with the sink/norm structure, yet lives in low-variance key directions — PCA-truncating the keys destroys it.

![Intrinsic dimensionality: probe AUC vs number of key principal components](/images/notes/kv-lifetime/explore_dim.png){: .kv-fig .narrow}

*Fig 9. Intrinsic dimensionality — per-layer probe AUC vs the number of key principal components retained. The signal is spread across ~112 of 128 dims, truncating to the high-variance axes destroys it.*
{: .kv-cap}

![Cross-layer cosine of the per-layer lifetime directions](/images/notes/kv-lifetime/cross_layer_cosine.png){: .kv-fig .narrow}

*Fig 10. The 8 per-layer lifetime directions are near-orthogonal, different QK space / lifetime geometries.*
{: .kv-cap}

Lexical observations. Longest lived = BOS sink, delimiters, rare/numeric tokens. Shortest = common words and continuations. Token frequency itself is a weak cue, *which* token matters far more than how common it is.

![Word clouds: longest-lived vs shortest-lived tokens](/images/notes/kv-lifetime/fig_wordclouds.png){: .kv-fig}

*Fig 11. Word clouds of the longest-lived (left) vs shortest-lived (right) key tokens.*
{: .kv-cap}

Layer trend. Decodability is strongest at layer 1 and dipping mid-stack at layer 12 before recovering deeper. We investigated where the dip happens, and found it's because a cluster of many tokens' lifetimes are centered around the cutoff 1024 at this depth, hard for predictor to distinguish, this should be an easy fix for next iteration.

![Lifetime decodability across depth](/images/notes/kv-lifetime/fig_layer_trend.png){: .kv-fig .medium}

*Fig 12. Lifetime decodability across depth (per-layer key-only linear probe AUC@1024).*
{: .kv-cap}

\*Across different prompts, `corr(cos(theta), <q,k>) ~= 0.0939`, `corr(||k||, <q, k>) = 0.02`.

---

## Appendix

The main post gives the core result. This appendix gives the supporting numbers, methods, and failure cases.

### A1 — Eviction benchmarks

We replace H2O's accumulated-attention score with an at-birth lifetime score, while keeping the rest of the cache policy fixed: sinks + recent tokens + heavy slots. The score uses only information available when the key is created: position, key norm, and the key vector. It has seen no future queries and no accumulated attention.

On language modeling, learned eviction matches H2O across budgets. Full-cache late PPL is **14.110**. At 12.5% cache, H2O gets **14.81** and learned gets **14.82**; at 25%, H2O gets **14.28** and learned gets **14.25**.

On needle retrieval, the gap is much larger. With the needle ~2636 tokens before the question, learned eviction reaches perfect retrieval by a **512-token cache**, while H2O is still near zero:

| budget | H2O | learned | oracle |
|---:|---:|---:|---:|
| 64 | 0.00 | 0.08 | 1.00 |
| 128 | 0.00 | 0.50 | 1.00 |
| 256 | 0.00 | 0.67 | 1.00 |
| 512 | 0.08 | 1.00 | 1.00 |
| 1024 | 0.25 | 1.00 | 1.00 |

This is the failure mode we care about. A dormant needle has received almost no attention, so H2O has no reason to keep it. The key vector does: among dormant keys, revival is predicted at **AUC 0.931**, versus **0.871** for early attention, with base rate **0.068**.

![Dormant-key revival is content-predictable where H2O is blind](/images/notes/kv-lifetime/fig_dormant.png){: .kv-fig}

### A2 — Cross-model results

We ran the same capture, training, and evaluation pipeline across Llama 3, Qwen 3, and Gemma 3, from 1B to 8B parameters. The main question is whether the key vector adds lifetime signal beyond key norm.

It does, on all nine models:

| family | model | R2 AUC@1024 | key-vector gain |
|---|---|---:|---:|
| Llama 3 | 1B | 0.9488 | +0.041 |
| Llama 3 | 3B | 0.9543 | +0.034 |
| Llama 3 | 8B | 0.9529 | +0.043 |
| Llama 3 | 3B 4-bit | 0.9534 | +0.034 |
| Qwen 3 | 1.7B | 0.9559 | +0.044 |
| Qwen 3 | 4B | 0.9443 | +0.055 |
| Qwen 3 | 8B | 0.9418 | +0.061 |
| Gemma 3 | 1B | 0.9731 | +0.018 |
| Gemma 3 | 4B | 0.9546 | +0.034 |

Every confidence interval excludes zero. The effect does not disappear with scale, and it survives quantization: Llama-3.2-3B bf16 and 4-bit both give **+0.034**.

QK-normalized models are interesting. In Llama, key norm is already useful. In Qwen and Gemma, norm becomes weaker, but the direction signal remains strong. So on modern QK-norm models, lifetime is mostly in key direction, not key magnitude.

### A3 — Calibration

The eviction score is optimized for ranking, not calibrated probabilities. During training, useful-but-rare keys are upweighted so the bad error, evicting a key that will matter later, is less common. This makes the raw survival probabilities overconfident.

For example, at `S(W=256)`, the weighted model has **ECE 0.116** and **Brier 0.109**. The top bin predicts **0.976** survival, but the observed survival rate is **0.686**.

A single temperature on the hazard logits fixes much of this. With **T≈3.7**, ECE improves from **0.116 → 0.082**, and Brier from **0.109 → 0.075**. Ranking is unchanged, since temperature scaling is monotone.

So eviction uses the raw score. If we want to read the score as an actual probability, we temperature-scale it first.

![Reliability of S(W=256)](/images/notes/kv-lifetime/fig_ceiling_reliability.png){: .kv-fig .narrow}

### A4 — Training

The main mechanistic run uses `Llama-3.2-3B-Instruct`, 4-bit weights, full-precision attention math, context length `N=4096`, and 8 dumped layers: `[1,5,8,12,16,20,23,27]`.

The data has four regimes: prose, code, synthetic NIAH, and key→value repeat. We train on about **1.76M** per-key examples and test on **348k**, split by whole window. A per-key split leaks.

A monkeypatched eager-attention forward records each sampled key's vector, value vector, token id, position, early attention, relevance labels, censoring, and death distance at threshold `1e-2`. With capture disabled, attention is byte-identical to stock attention.

The predictor is a small censored hazard model, about **100k parameters**, with layer and kv-head embeddings. It predicts survival at horizons `64`, `256`, and `1024`.

For the ablations, we use the following ladder:

`R0 position → R1 +norm → R2 +key → R3 +value → R4 +token id → R5 +early attention`

In the main text, the “at-birth content score” is **R2**. It uses position, key norm, and the key vector. **R5** is also causal, but waits to observe early attention. Neither uses future queries.

### A5 — Ablations

Adding the key vector gives the main jump.

| rung | features | AUC@64 | AUC@256 | AUC@1024 |
|---|---|---:|---:|---:|
| R0 | position | 0.778 | 0.824 | 0.857 |
| R1 | + norm | 0.847 | 0.889 | 0.920 |
| R2 | + key vector | 0.924 | 0.945 | 0.953 |
| R5 | + early attention | 0.942 | 0.942 | 0.946 |

At horizon 1024, the key vector adds **+0.034 [+0.032, +0.036]** beyond position and norm. The key vector alone reaches **0.949 AUC**, which is **99.6%** of the full feature model.

Early attention helps at short horizons, but not long ones. R5 beats R2 at 64, ties/slightly loses at 256, and loses at 1024. This explains the needle result: when the important key is dormant, early attention is mostly a misleading feature.

The ceiling looks information-bound, not capacity-bound. An 18k-param probe already reaches **0.946 AUC@1024**; a 407k-param probe reaches **0.9516**; longer training does not move much. Adding the query projection barely helps: key **0.955**, key+query **0.957**.

The signal is also spread across the key vector. About **112 / 128** principal components are needed for 95% of the signal, so PCA truncation destroys it. The residual stream is worse than the keys it produces: residual `h_i` gets **0.770**, while all-head keys get **0.826**. `W_k` seems to extract the right subspace.

![Feature-set sufficiency](/images/notes/kv-lifetime/explore_sufficiency.png){: .kv-fig .medium}
![Architecture / capacity sweep](/images/notes/kv-lifetime/fig_ceiling_arch.png){: .kv-fig}

### A6 — Context length and layers

The effect remains at 8k context. At `N=8192`, the ladder gives:

`R0 0.904 → R1 0.946 → R2 0.960`

at horizon 4096, with **R2−R1 = +0.013 [+0.009, +0.017]**.

Layers do not share one universal rule. A predictor trained on 6 of 8 layers and tested on the 2 held-out layers collapses to **0.701 AUC@1024**. Those same layers reach **0.954–0.957** when trained directly.

So the mapping is layer-specific. This matches the near-orthogonal lifetime directions in Fig 10: each depth reads lifetime from a different part of key space.

### A7 — Lifetime statistics

Different workloads have different lifetime distributions. Overall median death is **9 tokens**, and the fraction living past 1024 tokens is **0.073**.

| regime | median death | frac >1024 |
|---|---:|---:|
| code | 16 | 0.094 |
| niah | 11 | 0.077 |
| prose | 10 | 0.059 |
| repeat | 6 | 0.069 |

The simple correlations are stable: position `corr(birth, death) = -0.255`, norm `corr(||K||, death) = -0.370`, and early attention `corr(early128, death) = +0.503`.

Gemma3 sliding-window layers are the special case. Their window structurally caps lifetime: median death **127**, `S@1024 = 0.0`. Full-attention Gemma layers look normal: median death **671**, `S@1024 = 0.534`.

![Per-regime lifetime distributions](/images/notes/kv-lifetime/fig_dataset.png){: .kv-fig}

### A8 — Token examples

The longest-lived keys are mostly BOS, delimiters, structural tokens, rare words, and numbers. The shortest-lived keys are mostly common words and continuations.

Raw token frequency is not the explanation. Frequency has roughly zero correlation with lifetime; which token it is, through the key geometry, matters much more.

Examples of long-lived tokens: `<|begin_of_text|>`, `[\`, `·board`, `_e`, `·cheeks`, `908`, `750`.

Examples of short-lived tokens: `·struggled`, `·uncertain`, `·enthusiasm`, `·curse`, `·news`, `·checkout`.
