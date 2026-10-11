# Compression does not catch up with Qwen3.8-27B: compressed Chinese flagships on 2x MI50

**Scope.** Survey as of 11 October 2026, based on web research (no new benchmarks run for this document). Intelligence scores use the Artificial Analysis index v4.3.2 scale. The target rig is 2x AMD Instinct MI50 32 GB (gfx906) = **config A (64 GiB ≈ 68.7 GB)**, plus an RTX 3060 12 GB = **config B (≈75.6 GiB ≈ 81.2 GB)**, with 35 GB of system RAM and ROCm used only inside containers. The question: can a 2026 Chinese flagship model, heavily quantized (1–2 bit) or pruned (REAP), fit in this window and beat the best dense model that fits at near-full precision?

**Units (correction after review).** An earlier draft mixed units: GGUF file sizes are decimal GB (as Hugging Face shows them), but the "64 / 76" budgets are GiB. Measured on the rig: each MI50 exposes 34,342,961,152 bytes (32,752 MiB); the 3060 12,288 MiB. Budgets are therefore **config A = 64 GiB ≈ 68.7 GB** and **config B ≈ 75.6 GiB ≈ 81.2 GB**. All headroom figures and fit verdicts below are recomputed in GB against these budgets. This widens every margin by about 7 % and changes four verdicts.

**Companion report:** [02 — Which 2026 open models fit in 64–76 GiB](02-which-2026-models-fit-64-76gb.md).

**Status: first public version (11 Oct 2026). Web survey; links checked on that date.**

---

**Short answer.** In 64 or 76 GB, **no 2026 Chinese flagship fits in compressed form** in a format the MI50s can load today. Unsloth's 1-bit files weigh 82.5 GB for DeepSeek V4 Flash, 93 GB for GLM-5.3-Flash and 77.7 GB for MiMo-V2-Flash: all too large. Kimi K3 and MiniMax M3 are out of reach.

What actually fits are **second-tier** Chinese models: MiniMax-M2.7 at 1–2 bits (IQ1_M tight in config A, 2-bit variants in config B), and pruned (REAP) versions of Step-3.5-Flash and MiniMax-M2.5. None of these has an independent score above 34. None has a published quality measurement in compressed form.

The hypothesis is not wrong **in general**: a 671B DeepSeek at 1 bit retains much of its quality [M]. But that was at 130–250 GB. At 64–76 GiB, the arithmetic does not work: a model that loses ~20% at 1–2 bits would need to start at ≈43 to stay above 34. The only Chinese models at that level are 744B and up.

Working conclusion: **at most two candidates deserve an evening on the bench**, with low expectations. This is not a decision.

---

## How to read the labels

- **[M]**: measured, with a published number (KLD, benchmark, tok/s).
- **[A]**: announced by the vendor, a fork author or a forum, without a verifiable protocol.
- **[D]**: deduction or calculation by the author of this report.

Rule used for "fits" [D]:

- file size (decimal GB) + **≈6 GB** of KV cache and buffers ≤ 68.7 GB (A) or ≤ 81.2 GB (B);
- everything must fit **in VRAM**: with 35 GB of RAM, and `--no-mmap` required by mx-llama.cpp ([Docker Hub](https://hub.docker.com/r/mxxm/mx-llama.cpp)), there is almost no room to offload experts to the CPU.

---

## 1-bit files of the flagship models exceed the budget

### Feasibility table

| Model (total / active) | Variant | Size | Verdict [D] |
|---|---|---|---|
| **Qwen3.8-27B** (reference) | Q8_0 | 29 GB | **Fits in A**, ample |
| **MiniMax-M2.7** (229B / ≈10B) | UD-IQ1_M | 60.7 GB | **Fits in A, tight** (66.7 with KV) ; B comfortable |
| | UD-IQ2_XXS | 65.4 GB | **Fits in B** (71.4 with KV), not in A |
| | UD-IQ2_M | 70.1 GB | Fits in B (≈76.1) |
| | UD-Q2_K_XL | 75.3 GB | B at the limit (≈81.3) |
| **DeepSeek V4 Flash REAP 162B** (0xSero) | Mixed Q2, ds4 format | 52.6 GB | Fits in A on paper, **but ds4 engine only** |
| **DeepSeek V4 Flash** (284B / 13B), nisparks fork | IQ1_S-XL (teamblobfish) | 58 GB | Fits in A on paper (≈64), **non-standard fork** |
| **Step-3.5-Flash REAP 121B** (Cerebras) | Q3_K_M (calculated) | ≈55–58 GB | Fits in A **if the GGUF exists** |
| | Q4_K_M (calculated) | ≈73 GB | B on paper (≈79), if the GGUF exists |
| **Step-3.5-Flash** whole (196B / 11B) | ≈2.5 bits (calculated) | ≈61 GB | A tight / B, size not verified |
| **MiniMax-M2.5 REAP 139B** (Cerebras) | IQ2_M / Q3_K (calculated) | ≈47 / ≈61 GB | A / A tight, sizes not verified |
| **DeepSeek V4 Flash** (Unsloth) | UD-IQ1_S | 82.5 GB | **Does not fit** |
| **MiMo-V2-Flash** (309B / 15B) | UD-TQ1_0 | 77.7 GB | Does not fit |
| **GLM-5.3-Flash** (320B / 18B) | UD-IQ1_S | 93.09 GB | Does not fit |
| **GLM-5.3-Flash REAP 250B** | 1 bit (calculated, no GGUF) | ≈73 GB | B on paper (≈79), no GGUF exists |
| **DeepSeek V4 Flash REAP 180B** | NVFP4/MXFP4 | 103 GB | Does not fit |
| **MiniMax-M3** (≈428B / 23B) | UD-IQ1_M | 128 GB | Does not fit |
| **DeepSeek V4.1 Flash** (552B) / **V4 Pro** (1.6T) | ≈1.6 bit (calculated) | ≈110 / ≈320 GB | Does not fit |
| **Kimi K3** (2.8T / 104B), **MiMo-V2.6-Pro** (1.02T) | — | — | Does not fit |

Size sources: [MiniMax-M2.7 GGUF](https://huggingface.co/unsloth/MiniMax-M2.7-GGUF), [DeepSeek V4 Flash GGUF](https://huggingface.co/unsloth/DeepSeek-V4-Flash-GGUF), [0xSero 162B](https://huggingface.co/0xSero/DeepSeek-V4-Flash-162B-GGUF), [0xSero 180B](https://friendli.ai/models/0xSero/DeepSeek-V4-Flash-180B), [tinycomputers](https://tinycomputers.io/posts/running-deepseek-v4-flash-on-amd-strix-halo.md), [GLM-5.3-Flash](https://unsloth.ai/docs/models/glm-5.3-flash), [GLM REAP 250B](https://huggingface.co/OpenMOSE/GLM-5.3-Flash-REAP-250B-A18B), [MiMo-V2-Flash GGUF](https://huggingface.co/unsloth/MiMo-V2-Flash-GGUF), [MiniMax-M3](https://unsloth.ai/docs/models/minimax-m3), [Step REAP](https://huggingface.co/cerebras/Step-3.5-Flash-REAP-121B-A11B), [MiniMax REAP](https://huggingface.co/cerebras/MiniMax-M2.5-REAP-139B-A10B), [slash-digital](https://slash-digital.io/en/insights/open-weight-models-2026/).

Rows marked "calculated" have **no verified file**. The Unsloth page for Step-3.5-Flash could not be read.

### No official 2026 "little sibling"

- No GLM-5.x Air, MiMo Lite, Kimi Mini, or small Hunyuan/ERNIE/Seed-OSS was found for 2026 ([community gist](https://gist.github.com/BIGBALLON/5b5a4d5a053c7e73484fabea0c0e2466)).
- The only recent small Chinese models are **GLM-4.7-Flash** (30B / 3B, January 2026) and its 23B REAP pruning ([Cerebras](https://huggingface.co/collections/cerebras)). They fit in Q8, but they are 30B-A3B-class models, not compressed flagships.
- The search may have missed some: the vendors' Hugging Face organizations should be checked directly.

---

## Contradiction: 58 GB or 82.5 GB for DeepSeek V4 Flash in IQ1?

Both sources refer to the same 284B model, but not the same file.

- **58 GB**: "IQ1_S-XL" file from teamblobfish, loaded by a **nisparks fork** (branch `pr/01-deepseek-v4-arch`) on Strix Halo, ≈1.6 bits per weight [M for the size] ([tinycomputers](https://tinycomputers.io/posts/running-deepseek-v4-flash-on-amd-strix-halo.md)).
- **82.5 GB**: **Unsloth UD-IQ1_S** file, for standard llama.cpp ([Unsloth](https://huggingface.co/unsloth/DeepSeek-V4-Flash-GGUF)).
- The previous report (companion 02) cited **76.87 GiB** ([openclawdc](https://openclawdc.com/blog/best-local-llm-64gb-vram/)). That is the same Unsloth file expressed in GiB (82.5 GB ≈ 76.8 GiB) [D].

The author's reading [D]:

- 284B × 1.6 bits ≈ 57 GB. The 58 GB file therefore puts **almost everything** at 1 bit, attention included.
- Unsloth keeps attention and shared layers at high precision. Its "1 bit" is actually ≈2.3 bits on average. The ds4 engine confirms the order of magnitude: its 2-bit build that only touches the experts already weighs 81 GB ([ds4](https://github.com/539hex/ds4)).
- The 58 GB file is therefore **smaller because it is more destructive**. On Strix Halo it ran at 1–2 tok/s with **repetition loops** attributed to the compression [M, approximate].
- It also used tensor names specific to the fork. Nothing shows that it loads in upstream llama.cpp or in mx-llama.cpp.

**Consequence**: do not take "58 GB" as the reference size for V4 Flash at 1 bit. The size usable in standard llama.cpp is 82.5 GB.

---

## Starting scores are already at Qwen3.8-27B's level

### What is measured

Artificial Analysis index v4.3.2, full-precision models [M]:

| Model | AA index | Fits on this rig compressed? |
|---|---|---|
| GLM-5.3 (744B) | 45 | No |
| Kimi K3 (2.8T) | 44 | No |
| **GLM-5.3-Flash** (320B) | **42** | No (93 GB minimum) |
| DeepSeek V4 Pro | 36 | No |
| **DeepSeek V4 Flash 0731** (284B) | **34** | No (82.5 GB) |
| **Qwen3.8-27B** (Q8 on this rig) | **34** | Yes, 29 GB |
| MiniMax-M2.7, Step-3.5-Flash, MiMo-V2-Flash | **not found** | — |

Sources: [AA v4.3](https://artificialanalysis.ai/articles/artificial-analysis-intelligence-index-v4-3), [AA V4 Flash](https://artificialanalysis.ai/models/deepseek-v4-flash), [slash-digital](https://slash-digital.io/en/insights/open-weight-models-2026/).

Caution on versions:

- GLM-5.3, Kimi K3 and MiMo-V2.6-Pro come from slash-digital, and their index version is not confirmed.
- Qwen3.8-27B's 34 was read directly on the AA v4.3.2 pages during the previous research ([AA](https://artificialanalysis.ai/models/open-source/small)).
- Never compare with the old index (before 7 September 2026), which used a different scale.

### What compression costs [M]

- **GLM-5.3-Flash in UD-IQ1_S**: the first predicted token matches BF16 in only **70.9%** of cases (KLD 0.67). In UD-IQ2_XXS: 76.3% (KLD 0.45) ([Unsloth](https://unsloth.ai/docs/models/glm-5.3-flash)).
- For comparison, a Qwen3.5 in **Q8 has a KLD of 0.0026** ([Unsloth](https://unsloth.ai/docs/models/qwen3.5/gguf-benchmarks)). The gap is roughly **250 times**, across different models [D].
- **20% REAP pruning** on GLM-5.3-Flash, without quantization: KLD 0.45 and perplexity +2.8% ([OpenMOSE](https://huggingface.co/OpenMOSE/GLM-5.3-Flash-REAP-250B-A18B)). Pruning therefore costs as much as going to 2 bits.
- The two losses **add up**. A REAP + 1 bit would end up around KLD 0.8–1.0, worse than every published data point [D].
- **DeepSeek V3.1** on Aider Polyglot: 71.6 at full precision, **55.7 at 1 bit** (−22%), 65.8 in Q2_K_XL (−8%) ([Unsloth Aider](https://unsloth.ai/docs/basics/unsloth-dynamic-ggufs-on-aider-polyglot)).
- **No quality measurement** (KLD, perplexity, benchmark) exists for DeepSeek V4 Flash at ≤ 2 bits or for MiniMax-M2.7 at 1–2 bits.

### The arithmetic that settles it [D]

- V4 Flash starts at 34. With an 8 to 22% loss, a 1–2 bit version would land **between ≈27 and ≈31**. Qwen3.8-27B in Q8 stays at ≈34.
- To stay above 34 after a 20% loss, a model needs to start at **≈43**. Only GLM-5.3, Kimi K3 or MiMo-V2.6-Pro are there. They range from 744B to 2.8T.
- GLM-5.3-Flash (42) is the only "almost in the window" model that would have margin. But it needs 93 GB at 1 bit, or a pruning whose losses stack.

---

## The best argument for the hypothesis, taken seriously

The data does not say extreme compression is useless. It says it is not enough **at this memory size**.

What supports the hypothesis:

- **DeepSeek R1 at dynamic 1.58 bits** (131 GB) scored 6.92/10 on a game-coding test, while naive quantization scored 0 [M, single test] ([Unsloth](https://unsloth.ai/blog/deepseekr1-dynamic)).
- **DeepSeek V3.1 at 1 bit, thinking mode** (57.8) beat full-precision R1 (56.9) and nearly matched Qwen3-235B (59.6) [M] ([Unsloth Aider](https://unsloth.ai/docs/basics/unsloth-dynamic-ggufs-on-aider-polyglot)).
- **REAP on Step-3.5-Flash**: removing 40% of the experts costs only ≈2.5 points on HumanEval (98.2 → 95.7) [M] ([Cerebras](https://huggingface.co/cerebras/Step-3.5-Flash-REAP-121B-A11B)).
- AA mostly measures English and agentic work. A large MoE may retain more **knowledge** and more **French**, even when damaged. Nobody has measured it.

Why this does not transfer:

- those successes concern models of **671B at more than 130 GB**, compared **against themselves**, never against a 27B dense model at equal memory;
- no published comparison of "large MoE at 1.5–2 bits vs 27–32B dense at Q8" exists, neither in 2025 nor in 2026;
- REAP is only measured on **code**. Users report that aggressive pruning (≈50%) "can no longer hold a chain of reasoning" outside code [A] ([Level1Techs](https://forum.level1techs.com/t/glm-5-2-reap/251814));
- MiniMax-M2.7, the only flagship-ish model that fits, claims **56.2% on SWE-Pro** [A] ([Unsloth](https://huggingface.co/unsloth/MiniMax-M2.7-GGUF)). Qwen claims **61.7%** on SWE-bench Pro for Qwen3.8-27B [A]. If these are indeed the same tests, the 27B is ahead **even before** M2.7 is compressed [D].

---

## gfx906 support adds a second wall

- **DeepSeek V4 Flash**: the **mx-llama.cpp** fork supports it on MI50, with real measurements of 24–27 tok/s, but on **8 cards in MXFP4** (≈21 GiB per card) [A, fork author] ([mx-llama.cpp](https://github.com/mxxm-t/mx-llama.cpp/blob/master/FEATURES.md)). No file that fits in this budget has been tested there.
- **ds4 engine** (the only one that reads the 162B REAP): ROCm documented for Strix Halo only, multi-GPU in CUDA only, and "two cards do not have enough memory" according to its own README [A] ([ds4](https://github.com/539hex/ds4)). gfx906 support very unlikely [D].
- **GLM-5.3-Flash**: landed in llama.cpp on 30 September 2026 (PR #27773). Its 34 KDA layers expose it to the **SOLVE_TRI bug #19972**, which crashes gfx906 [D] ([aiweekly](https://aiweekly.co/alerts/llamacpp-merges-320b-glm-53-flash-hybrid-text-vision-support), [#19972](https://github.com/ggml-org/llama.cpp/issues/19972)).
- **IQ1/IQ2 formats**: no gfx906 fork has a dedicated kernel. They go through the generic path ([arte-fact](https://github.com/arte-fact/llamacpp-gfx-906-turbo)). They still run: Qwen3-235B in IQ1_S on 2x MI50 gives **≈20 tok/s** generation and 137 prefill [M] ([wtarreau](http://wtarreau.blogspot.com/2025/12/amd-radeon-instinct-mi50-32gb-best-ai.html)).
- **Config B**: mixing MI50 + RTX 3060 in a single process (HIP + CUDA) has **no published measurement** ([mx-llama.cpp](https://github.com/mxxm-t/mx-llama.cpp/blob/master/FEATURES.md)). Yet MiniMax-M2.7's 2-bit variants only fit in B. (Update: the author has since measured a mixed HIP + CUDA build working on 2x MI50 + RTX 3060 — see 04.)
- **MiniMax-M2** has been in upstream llama.cpp since 2025, but this is prior knowledge that was not re-verified [D]. **Step-3.5** support is uncertain.
- No measured MI50 speed was found for GLM, MiniMax, Kimi, MiMo or Step.

---

## Chinese candidates that can be bench-tested

This is not a decision. Expectations are **low** for both: no data places them above Qwen3.8-27B. They serve to test the hypothesis with the best available representatives.

### 1. MiniMax-M2.7 in UD-IQ2_XXS (65.4 GB), config B — or UD-IQ1_M (60.7 GB) in config A

- **Why this one**: the only recent Chinese model over 200B that fits with a verified file, and ranked on the quality/price frontier by AA ([AA](https://artificialanalysis.ai/articles/recent-open-weights-model-launches)).
- **Main risk**: mixed HIP + CUDA never measured on MI50, and no quality data at 2 bits.
- **Deciding measurement**: first, does it load across the three cards and exceed ≈10 tok/s? Then, a blind head-to-head against Qwen3.8-27B Q8 on the common test battery (tool calls, office-assistant tasks in French, long chapter). If it does not clearly win in French, the hypothesis falls.

### 2. Step-3.5-Flash-REAP-121B in Q3_K_M (≈55–58 GB), config A

- **Why this one**: the only pruning with a low measured loss (on code), and it would fit in A without the 3060.
- **Main risk**: nothing has been verified upstream. Not the GGUF size, not Step-3.5 support in llama.cpp, not quality outside code.
- **Deciding measurement**: verify that a GGUF exists and loads. If so, a chained tool-call test (3 calls or more): that is where pruning breaks first.

### Excluded, with reasons

- **DeepSeek V4 Flash**: no standard file under 82.5 GB; the 58 GB one loops and depends on a fork; the 52.6 GB one depends on ds4.
- **GLM-5.3-Flash** and its REAP: 93 GB minimum, or a hypothetical ≈73 GB with stacked losses and SOLVE_TRI risk.
- **MiMo, Kimi K3, MiniMax-M3, V4.1 Flash**: too large, even at 1 bit.
- **MiniMax-M2.5-REAP-139B**: alternative to M2.7 if that one does not load in B, but Cerebras' evaluation is marked "TBD" ([Cerebras](https://huggingface.co/cerebras/MiniMax-M2.5-REAP-139B-A10B)).

---

## Conclusion

The "bigger = smarter" hypothesis rests on a real phenomenon: very large MoEs withstand dynamic compression well. But that phenomenon has a **memory threshold**. It kicks in from ≈130 GB, when a model scoring 43+ on the index still keeps enough bits to stay ahead. At 64–76 GiB, the Chinese models that fit start at 34 at best, and compression drops them below Qwen3.8-27B in Q8.

What would change this conclusion: an official "little sibling" of GLM-5.x or DeepSeek around 100–150B, or a first published quality measurement of MiniMax-M2.7 at 2 bits. Until then, the bench can confirm or refute the hypothesis with just two candidates, and an adversarial review of the results is still needed before committing to any decision.
