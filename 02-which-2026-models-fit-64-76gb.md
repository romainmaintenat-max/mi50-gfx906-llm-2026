# A 27B dense model takes on the giant MoEs: which 2026 open models fit in 64–76 GiB on 2x MI50

**Scope.** Survey as of 11 October 2026, based on web research (no new benchmarks run for this document). Intelligence scores use the Artificial Analysis index v4.3.2 scale. The target rig is 2x AMD Instinct MI50 32 GB (gfx906) = **config A (64 GiB ≈ 68.7 GB)**, plus an RTX 3060 12 GB = **config B (≈75.6 GiB ≈ 81.2 GB)**, with 35 GB of system RAM and ROCm used only inside containers. The question: which open model released in 2026 is the most capable that fits, and does the hypothesis "bigger = smarter" hold in this memory window?

**Units (correction after review).** An earlier draft mixed units: GGUF file sizes are decimal GB (as Hugging Face shows them), but the "64 / 76" budgets are GiB. Measured on the rig: each MI50 exposes 34,342,961,152 bytes (32,752 MiB); the 3060 12,288 MiB. Budgets are therefore **config A = 64 GiB ≈ 68.7 GB** and **config B ≈ 75.6 GiB ≈ 81.2 GB**. All headroom figures and fit verdicts below are recomputed in GB against these budgets. This widens every margin by about 7 % and changes four verdicts (Qwen3.5-122B IQ4_XS/IQ4_NL, MiniMax-M2.7 IQ1_M and gpt-oss-120b now plausible in A).

**Status: first public version (11 Oct 2026). Web survey; links checked on that date.**

---

**Short answer.** On the independent data available in October 2026, the most intelligent open model that fits in 64 or 76 GiB is **Qwen3.8-27B, a dense model from August 2026**. It scores 34 on the Artificial Analysis index v4.3.2 [M]. All the 2026 MoEs of 117 to 128B that fit on this rig score between 11 and 18 [M, partly estimated].

The hypothesis "bigger = smarter" is therefore **wrong within this size window**, but not absurd. It becomes partly true again at equal generation and vendor: Qwen claims Qwen3.8-Flash-Next (125B MoE) beats the 27B on its own tests [A]. That large model is experimental, barely supported on gfx906, and its 39 GB n-gram table exceeds the rig's 35 GB of RAM.

Two other facts weigh heavily:

- in 64 GiB, the 27B runs in **Q8_0 (29 GB)**, almost lossless, while a 120B MoE must drop to IQ4_XS or IQ4_NL, close to the memory limit;
- no independent source measures **French** or **long-form writing** for these models. On those two use cases, everything remains to be proven on the bench.

The report ends with **five ranked candidates to test**, each with the measurement that would settle it. This is not a decision: that comes after bench tests and an adversarial review.

---

## How to read the numbers

Every number carries a label. It says where the number comes from, not whether it is true.

- **[M]**: measured by a third party (Artificial Analysis, Vals, llama-bench published with a protocol).
- **[A]**: announced by the model vendor or a fork author, about their own work.
- **[R]**: anecdotal, rounded, or read in a forum thread.
- **[D]**: deduction or calculation by the author of this report.

Method limits to keep in mind:

- several leaderboards (LMArena, EQ-Bench creative, Fiction.LiveBench, official BFCL, compar:IA) **could not be read**: JavaScript-rendered pages;
- Artificial Analysis (AA) **changed scale on 7 September 2026** (v4.3). A "Qwen3.8-27B = 52" from August corresponds to the current 34. Never mix the two ([Leptidigital](https://www.leptidigital.fr/intelligence-artificielle-ia/meilleurs-modeles-ia-llm-55427/));
- many AA scores for mid-size models are marked "Estimate". A 1 to 3 point gap between two models is noise.

---

## The 2026 catalogue boils down to two families

Between 31B and 110B, **no 2026 dense model** was found. Labs release either 27–31B dense models, or 118–125B MoEs with 6 to 12B active parameters. The frontier models (DeepSeek V4, GLM-5.x, Kimi K3, Qwen3.8-Max) are open but far too large: even DeepSeek-V4-Flash at 1 bit weighs 76.87 GiB ([openclawdc](https://openclawdc.com/blog/best-local-llm-64gb-vram/)).

### The 118–125B MoEs that fit in this window

| Model | Release | Total / active | Context | License |
|---|---|---|---|---|
| **Qwen3.8-Flash-Next** | August 2026 | 125B / 6B + 51B n-gram table + 4B MTP | 262k (1M YaRN) | qwen-community-1.0 (not Apache) |
| **Qwen3.5-122B-A10B** | February 2026 | 122B / 10B | 262k (≈1M YaRN) | Apache-2.0 |
| **Mistral-Small-4-119B-2603** | March 2026 (inferred from the name) | 119B / 6.5B | 256k | Apache-2.0 |
| **Nemotron-3-Super-120B-A12B** | 11 March 2026 | 120B / 12B | 256k (1M max) | NVIDIA license |
| **Laguna S 2.1** (Poolside) | 21 July 2026 | 118B / 8B | 1M (secondary source) | OpenMDW |

Sources: [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next), [Qwen3.5-122B](https://huggingface.co/Qwen/Qwen3.5-122B-A10B), [Mistral-Small-4](https://huggingface.co/mistralai/Mistral-Small-4-119B-2603), [Nemotron-3-Super](https://huggingface.co/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-BF16), [Laguna S 2.1](https://thenextweb.com/news/poolside-laguna-s-2-1-open-weight-coding-model).

Key points:

- **Flash-Next** is presented by Qwen as an "experimental preview" of the Qwen4 architecture ([HF](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)), and described elsewhere as "intentionally under-trained" ([eesel.ai](https://www.eesel.ai/blog/qwen38-flash-next-review)).
- **Laguna S 2.1** is a **coding** model (Terminal-Bench ≈70%, SWE-Bench Pro ≈60% [A]). It does not target office-assistant tasks or writing.
- **No Qwen3.6-122B and no gpt-oss successor**: the corresponding Manifold markets resolved "no" ([Manifold Qwen](https://manifold.markets/NielsW/will-qwen-36-122b-a10b-be-open-weig), [Manifold OpenAI](https://manifold.markets/event/when-will-openai-release-their-next)).

### 2026 dense and small MoE models (baseline)

- **Qwen3.8-27B**: 27B dense, August 2026, Apache-2.0, 262k context, built-in MTP head ([HF](https://huggingface.co/Qwen/Qwen3.8-27B)).
- **Gemma 4 31B**: 30.7B dense, 2 April 2026, Apache-2.0, 256k, hybrid sliding-window + global attention ([HF](https://huggingface.co/google/gemma-4-31B-it), [Wikipedia](https://en.wikipedia.org/wiki/Gemma_(language_model))).
- **Qwen3.6-27B** and **Qwen3.6-35B-A3B**: April 2026; the 35B-A3B is the most used on MI50 ([computingforgeeks](https://computingforgeeks.com/open-source-llm-comparison/)).
- **GLM-4.7-Flash**: 30B / 3B active, January 2026, MIT ([GIGAZINE](https://gigazine.net/gsc_news/en/20260120-zai-glm-4-7-flash)).

### Older, for reference only

- **gpt-oss-120b** (August 2025): still OpenAI's latest open model. MXFP4 = 59.03 GiB, i.e. ≈63.4 GB ([TechCrunch](https://www.techcrunch.com/2025/08/05/openai-launches-two-open-ai-reasoning-models), [openclawdc](https://openclawdc.com/blog/best-local-llm-64gb-vram/)). AA = 12 [M].

---

## What actually fits in 64 and 76 GiB

### Calculation rules used

- **Gross headroom** = budget − weights, both in decimal GB. It must hold the KV cache, compute buffers and the ROCm context of each card.
- The author counts **about 1 to 1.5 GB of overhead per card**, excluding KV [D]. A gross headroom of 4 GB in config A is therefore close to zero.
- Hybrid architectures (Gated DeltaNet in Qwen, Mamba-2 in Nemotron) have a **small KV**, because only a minority of layers keep an attention cache. No per-token KV figure was found: this is a deduction [D].
- A real-world reference point: the author of the milpster fork reports fitting **Qwen3.8-27B Q6_K with 250k context in 40 GB** (KV f16 K / q8_0 V) [R] ([Level1Techs](https://forum.level1techs.com/t/glm-and-i-created-a-llama-cpp-fork-optimized-for-amd-gfx906-mi50-mi60-radeon-vii-gcn-hip/254257)).

### Feasibility table

A = 2x MI50 (68.7 GB). B = 2x MI50 + RTX 3060 (81.2 GB). GGUF sizes read on Hugging Face [M], in decimal GB.

| Model | Quant | Size | Headroom A | Headroom B | Verdict |
|---|---|---|---|---|---|
| **Qwen3.8-27B** | Q8_0 | 29.05 GB | 39.6 GB | 52.2 GB | **A, ample**, comfortable long context |
| | UD-Q4_K_M | 16.46 GB | 52.2 GB | — | A, but see the Q4 loss below |
| | BF16 | 55.59 GB | 13.1 GB | — | Pointless: gfx906 has no native bf16 |
| **Gemma 4 31B** | Q8_0 | 33.15 GB | 35.6 GB | — | A, with KV **mandatorily f16** |
| **Qwen3.5-122B-A10B** | UD-IQ4_XS | 60.23 GB | 8.5 GB | 21.0 GB | A **plausible but tight** once KV and per-card overhead are counted; B comfortable |
| | UD-IQ4_NL | 61.44 GB | 7.3 GB | 19.8 GB | A tight; B comfortable |
| | Q4_K_S | 71.71 GB | — | 9.5 GB | B fine |
| | Q3_K_M | 56.42 GB | 12.3 GB | — | A fine, degraded quality |
| **Mistral-Small-4** | UD-IQ4_XS | 58.13 GB | 10.6 GB | 23.1 GB | A fine, B comfortable |
| | UD-IQ4_NL | 59.20 GB | 9.5 GB | 22.0 GB | A fine |
| | UD-Q4_K_S | 69.47 GB | — | 11.7 GB | B fine |
| **Nemotron-3-Super** | IQ4_XS | 67.20 GB | 1.5 GB | 14.0 GB | **B only** (A far too tight) |
| | Q3_K_M | 64.65 GB | 4.1 GB | 16.6 GB | A very tight; B fine |
| **Laguna S 2.1** | UD-IQ4_XS | 53.61 GiB (≈57.6 GB) | ≈11.1 GB | ≈23.6 GB | A fine (secondary source) |
| **Qwen3.8-Flash-Next** | IQ4_XS (AtomicChat, split) | 45.8 GB GPU + 39.1 GB n-gram table | 22.9 GB | 35.4 GB | Fits on the GPU side, **not on the RAM side** (see below) |
| | UD-IQ1_S (Unsloth, whole) | 72.55 GB | — | 8.7 GB | B, heavily degraded quality |
| **gpt-oss-120b** (2025) | MXFP4 | ≈63.4 GB | ≈5.3 GB | ≈17.8 GB | A just possible, B fine |

Sizes re-read on Hugging Face on 11 Oct 2026 (BF16 and Gemma Q8_0 corrected from the first draft). Rule of thumb used for verdicts: below ~4 GB of gross headroom = too tight, 4–8 GB = tight, above 8 GB = fine (before long-context KV).

Sources: [Qwen3.8-27B GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF), [Qwen3.5-122B GGUF](https://huggingface.co/unsloth/Qwen3.5-122B-A10B-GGUF), [Mistral-Small-4 GGUF](https://huggingface.co/unsloth/Mistral-Small-4-119B-2603-GGUF), [Nemotron GGUF](https://huggingface.co/bartowski/nvidia_Nemotron-3-Super-120B-A12B-GGUF), [Flash-Next AtomicChat](https://huggingface.co/AtomicChat/Qwen3.8-Flash-Next-GGUF), [Unsloth Flash-Next](https://unsloth.ai/docs/models/qwen3.8-next), [openclawdc](https://openclawdc.com/blog/best-local-llm-64gb-vram/).

The Q8_0 size of Gemma 4 31B is **calculated** (30.7B × 8.5 bits): no file list was read [D].

### What config B really changes

On MoEs, the 3060 adds **capacity**, not speed:

- measured on Strix Halo + RTX 3060: **+24% generation on a 27B dense Q4_0**, but **only +1.6% on a 122B MoE MXFP4** [M] ([Strix Halo toolbox](https://github.com/DasDigitaleMomentum/strix-halo-cuda-combined-toolbox));
- that gain came from a slow iGPU being boosted by the 3060. With MI50s at ≈1 TB/s, the 3060 (≈360 GB/s) will be **the slowest card**: every layer it holds slows down each token [D];
- **no published measurement** of MI50 + NVIDIA in a single HIP+CUDA process. The only question asked about such a pairing (MI50 + V100) went unanswered ([discussion #19544](https://github.com/ggml-org/llama.cpp/discussions/19544)). *(Update: the author has since measured such a build working on 2x MI50 + RTX 3060 — see 04.)*

Practical consequence [D]: B is only useful to **make room for** Qwen3.5-122B in Q4_K_S, Mistral-Small-4 in Q4_K_S, or Nemotron-3-Super. For Qwen3.8-27B and Gemma 4 31B, B adds nothing. The 3060 then stays free for embeddings, Whisper or a small router.

### The Flash-Next case and 35 GB of RAM

This is the trickiest point in the report.

- The AtomicChat version keeps **45.8 GB on GPU** and leaves the **n-gram table (39.1 GB) on SSD via mmap** [M for the size] ([HF](https://huggingface.co/AtomicChat/Qwen3.8-Flash-Next-GGUF)).
- 39.1 GB exceeds the rig's 35 GB of RAM. The table will **never fit entirely in the page cache**. Each token will read a few random rows from the file [D].
- On an NVMe SSD, that may be acceptable. On an **HDD used for model storage**, random access will be catastrophic [D]. The file would need to sit on the system SSD.
- The **mx-llama.cpp** fork **requires `--no-mmap`** ([Docker Hub](https://hub.docker.com/r/mxxm/mx-llama.cpp)). With the table on CPU, it would then need 39 GB of RAM: **impossible** [D].
- AtomicChat's "table in host memory" path is off the main branch. Whether it works under ROCm is **unknown**.
- The only Flash-Next measurements on MI50 were taken on **4 cards (128 GB)**, all in VRAM.

---

## Support status on gfx906, model by model

### The SOLVE_TRI bug (#19972) affects the whole recent Qwen line

- llama.cpp's own kernel only covers n ≤ 64, k ≤ 32. Beyond that, it falls back to rocBLAS `strsm`, **absent for gfx906 in ROCm 7.x**. Result: `hipErrorInvalidDeviceFunction` [M] ([issue #19972](https://github.com/ggml-org/llama.cpp/issues/19972)).
- Affected models: those using **Gated DeltaNet** (Qwen3.5, Qwen3-Next, Kimi Linear). Qwen3.6 and Qwen3.8-27B go through the same `qwen3_5` code path ([sudoingX](https://github.com/sudoingX/qwen38-mtp)): they are **probably exposed** [D]. Flash-Next (qwen4exp) too.
- No upstream fix tied to the issue was found.

Three known workarounds:

1. `--no-warmup`, against hangs at startup ([arte-fact](https://github.com/arte-fact/llamacpp-gfx-906-turbo)).
2. Disable `GGML_OP_SOLVE_TRI` in the HIP backend (CPU fallback). Measured: Qwen3.5-35B-A3B on 2x MI50 at **85.7 pp / 48.3 tg t/s** [M], well below the usual ≈90 t/s for an A3B.
3. Use a gfx906 fork (furnace, mx-llama.cpp) that bypasses rocBLAS.

**Contradiction to flag**: the issue describes a crash on Qwen3.5-35B-A3B, but furnace measures Qwen3.6-35B-A3B on "stock master" without problems ([furnace](https://github.com/sixvolts/llamacpp-gfx906-furnace)). And the author's own Qwen3.5-27B Q4_0 already runs on this rig (293 pp / 25 tg). The bug therefore probably depends on block sizes and the model [D]. **Only a test with the exact model will settle it.**

### Support table

| Model | Upstream llama.cpp | gfx906 forks | vLLM gfx906 | Known risks |
|---|---|---|---|---|
| **Qwen3.8-27B** | Yes (qwen3_5 path); MTP via `--spec-type draft-mtp` | Martin1701, furnace, milpster, mx | David-Lzy: AWQ "functionally compatible" in TP4 only | SOLVE_TRI; MTP requires `--parallel 1`; issue #27623 (decoding collapse at very long context) |
| **Gemma 4 31B** | Yes, measured on stock MI50 | furnace (+24% tg) | Not documented | D=512 attention: FA unstable depending on the build; quantized KV very harmful |
| **Qwen3.5-122B-A10B** | Yes (Unsloth GGUF) | Probable (same architecture as 3.5) | Not documented | SOLVE_TRI; IQ4 = slow path on gfx906 |
| **Mistral-Small-4** | Unsloth GGUF → upstream support probable [D] | Not verified | Not documented | No gfx906 test found |
| **Nemotron-3-Super** | bartowski GGUF → upstream support probable [D] | Not verified | Not documented | Mamba-2 layers never tested on gfx906 |
| **Qwen3.8-Flash-Next** | qwen4exp PR merged on 27/08/2026 | furnace and mx (tested on 4x MI50) | None | Experimental; n-gram table; RAM |

Sources: [Martin1701](https://github.com/Martin1701/MI50-llama.cpp), [furnace](https://github.com/sixvolts/llamacpp-gfx906-furnace), [mx-llama.cpp](https://github.com/mxxm-t/mx-llama.cpp), [David-Lzy vLLM](https://github.com/David-Lzy/vllm-gfx906), [Traictory](https://traictory.com/news/2026-08-29-llama-cpp-qwen3-8-flash-next-support), [llama.cpp weekly report](https://buttondown.com/weekly-project-news/archive/weekly-github-report-for-llamacpp-august-17-2026-8985/), [machinesoftworks](https://machinesoftworks.com/posts/custom-gfx906-llamacpp-engine).

### Measured or expected speeds

| Model | Speed on MI50 | Label |
|---|---|---|
| Qwen3.8-27B Q8_0, 1 card | 22.8 t/s at 2k; 20.4 at 64k; MTP 28.8 in reasoning at 64k, 43–52 on code or short prose | [A] ([Martin1701](https://github.com/Martin1701/MI50-llama.cpp)) |
| Qwen3.8-27B, ROCm, milpster fork | ≈30 t/s (vs 18 on Vulkan) | [R] ([L1T](https://forum.level1techs.com/t/glm-and-i-created-a-llama-cpp-fork-optimized-for-amd-gfx906-mi50-mi60-radeon-vii-gcn-hip/254257)) |
| Qwen3.8-27B Q6_K at 120k | ≈15 t/s | [R] |
| Gemma 4 31B UD-Q4_K_XL, 1 card | 22.4 t/s stock → 27.9 furnace | [A] ([furnace](https://github.com/sixvolts/llamacpp-gfx906-furnace)) |
| Qwen3.8-Flash-Next Q4_K_XL, 4 cards | 44.8–48.3 t/s without MTP; 58.3–67.4 with MTP | [A] (two readings of the same README) |
| Qwen3.5-122B-A10B, ≈4.3 bits | ≈30 t/s expected | [D] |
| Mistral-Small-4, ≈4.3 bits | ≈45 t/s expected | [D] |
| Nemotron-3-Super, ≈4.3 bits | ≈25 t/s expected | [D] |

The author's calculation for the MoEs [D]: a Qwen3.6-35B-A3B in Q4_K_XL reads ≈1.8 GB of active weights per token and runs at ≈90 t/s [A]. That gives ≈160 GB/s effective for a MoE on MI50. The same efficiency is applied to the active weights of the large MoEs. It is an order of magnitude, not a promise.

### The quantization format matters as much as the model

- On gfx906, **Q4_0, Q4_1 and Q8_0** have dedicated kernels. K-quants and IQ quants go through a slower generic path: Q4_0 beats Q4_K_M by about 24% in generation on a 32B [M] ([wtarreau](http://wtarreau.blogspot.com/2025/12/amd-radeon-instinct-mi50-32gb-best-ai.html)).
- mx-llama.cpp repacks Q8_0, MXFP4, **IQ4_NL**, Q4_K, Q5_K, Q6_K and Q5_1 at load time. **IQ4_XS is not on the list** ([mx-llama.cpp](https://github.com/mxxm-t/mx-llama.cpp)).
- Consequence [D]: for a large MoE, prefer **IQ4_NL over IQ4_XS** when memory allows. For the 27B, **Q8_0** is both the most faithful format and a fast one on gfx906.

---

## "Bigger = smarter" against the data

### What the independent measurements say

AA index v4.3.2, October 2026, open models within this window [M] ([AA small](https://artificialanalysis.ai/models/open-source/small), [AA medium](https://artificialanalysis.ai/models/open-source/medium)):

| Model | Total / active | AA index |
|---|---|---|
| **Qwen3.8-27B** (reasoning xhigh) | 27B dense | **34** |
| Qwen3.8-27B (medium) | 27B dense | 28 |
| Qwen3.5-122B-A10B | 122B / 10B | 16–18 |
| Gemma 4 31B (reasoning) | 31B dense | 15 |
| Nemotron-3-Super | 120B / 12B | 13 |
| gpt-oss-120b (2025) | 117B / 5B | 12 |
| Mistral-Small-4 (reasoning) | 119B / 6.5B | 11 |

The sub-test breakdown confirms the gap for a "delegable agent" profile [M] ([AA comparison](https://artificialanalysis.ai/models/releases/comparisons/qwen3-8-27b-vs-qwen3-6-35b-a3b)):

- **AutomationBench** (tools, automation): Qwen3.8-27B **48.2%** vs 5.2% for Qwen3.6-35B-A3B;
- **AA-LCR** (long-context reasoning): **82%**;
- **GDPval** (agentic office tasks): Elo 1419;
- **AA-Omniscience** (knowledge minus hallucinations): **−10**, the least bad in the window. All models of this size are negative: when unsure, they are wrong more often than right.

The jump between generations is even sharper: Qwen3.6-27B (April 2026) scores **21** vs **8** for Qwen2.5-72B (2024) [M] ([AA](https://artificialanalysis.ai/models/comparisons/qwen2-5-72b-instruct-vs-qwen3-6-27b)).

### Why the "bigger = smarter" hypothesis is not absurd (steelman)

Four arguments support it. They deserve to be taken seriously.

1. **At equal vendor and generation, the bigger model wins.** On the Flash-Next model card, Qwen claims [A] ([HF](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)):

   | Test | Qwen3.8-27B | Qwen3.8-Flash-Next |
   |---|---|---|
   | GPQA-D | 89.2 | 91.7 |
   | SWE-bench Pro | 61.7 | 62.5 |
   | DeepSWE | 42.2 | **58.7** |

   No independent AA score for Flash-Next was found.

2. **AA mostly measures English and agentic work.** It measures neither French style nor the coherence of a 50-page narrative. The breadth of knowledge of a 120B could matter there. **Nobody has measured it** for these models.

3. **Several large-MoE scores are AA estimates**, and their sub-scores (Omniscience in particular) were not retrieved.

4. **Under the old index**, Qwen3.8-27B was said to be on par with DeepSeek V4 Flash (284B) [R] ([HN](https://hn.nuxt.dev/item/49334544)). Mostly, this shows that total size is a poor indicator, in both directions.

### The author's reading [D]

The right rule in 2026 is not "bigger", but:

- **more recent** (reinforcement-learning post-training accounts for most of the gain);
- **more active compute per token** (a MoE with 6–12B active reasons like a 6–12B model on hard tasks, with more knowledge);
- **less compressed**: in 64 GiB, the 27B fits in Q8_0 while a 120B has to go down to IQ4.

On that last point, Unsloth's measurement gives an idea of the gap [M] ([Unsloth](https://unsloth.ai/docs/models/qwen3.5/gguf-benchmarks)): mean divergence (KLD) of **0.0026 in Q8** vs **0.0192 in Q4_K_M**, roughly 7 times more. Measured on a ≈35B Qwen3.5, not on a 120B.

On Flash-Next, Unsloth measures first-token agreement of **94.1% in Q8_0**, **92.3% in Q4_K_XL** and **77.3% in IQ1_S** [M] ([Unsloth](https://unsloth.ai/docs/models/qwen3.8-next)). The only whole version that fits in B (IQ1_S) therefore loses a lot.

One testimony points the same way: on Qwen3.8-27B, Q4 reportedly produces **2 to 3 times more reasoning and more errors** than Q8 [R] ([HN](https://hn.nuxt.dev/item/49334544)).

---

## Use case by use case: what is known and what is not

### Tool calling and JSON

- **Qwen dominates user reports** as a Hermes agent: better at following structured instructions, reliable nested JSON, chains of 3 or more calls, recovery after errors [R] ([Reddit](https://reddit.sentinel-team.org/posts/1u2q75f/snapshots/2026-06-13T02%3A56%3A06.64152Z)).
- **Gemma 4 "sometimes hallucinates success"** for a tool call that failed [R]. Yet Gemma 4 claims 86.4% on τ2-bench Retail [A] ([Google](https://www.deepmind.google/models/gemma/gemma-4)). **Contradiction** between the vendor number and user reports.
- Many failures come from the **engine's parser or chat template**, not the model [R] ([r/hermesagent](https://r.datuan.dev/r/hermesagent/comments/1uc7rw5/)). Validate llama.cpp `--jinja` with the exact model.
- For JSON, llama-server can **constrain output with a grammar / JSON schema**. This guarantees syntax, not content [D, known llama.cpp feature].
- No official BFCL score was found for the models in this window.

### Long context

- Qwen3.8-27B: **AA-LCR 82%** [M], the best independent figure in the window.
- Real cost on MI50: ≈15 t/s at 120k [R], and issue #27623 reports a **decoding collapse at very long context** [R].
- **Quantized KV**: q8_0 is nearly lossless on Qwen 3.6 (KL 0.024) but not on Gemma 4 (0.108 dense, 0.377 MoE). In q4_0, Qwen suffers mostly on **long documents (KL 0.581)** and tool calls [M] ([localbench](https://localbench.substack.com/p/kv-cache-quantization-benchmark)).
- Resulting rule [D]: KV **f16 or q8_0 for Qwen**, **f16 only for Gemma**, never q4_0 for long-form fiction writing.

### Fixed cost of agents

Hermes consumes ≈**15k tokens** of context for a simple "hello" [R] ([r/hermesagent](https://r.datuan.dev/r/hermesagent/comments/1uc7rw5/)). On a 27B dense at ≈300 t/s prefill (the author's own Qwen3.5-27B measurement on this rig), that means **≈50 s before the first token** without caching [D]. Per-slot prefix cache reuse in llama-server is therefore essential.

Another constraint: **Qwen3.8's MTP requires `--parallel 1`** ([sudoingX](https://github.com/sudoingX/qwen38-mtp)). Several agents in parallel will have to choose between MTP and concurrency. One option [D]: **one Q8_0 instance per MI50** (29 GB each, context limited to ≈100k as in Martin1701's setup), i.e. two simultaneous agents.

### French, long-form writing, office-assistant tasks

- **No independent measurement of French** was found for 2026 models. AA itself states it is "mostly English" ([Leptidigital](https://www.leptidigital.fr/intelligence-artificielle-ia/meilleurs-modeles-ia-llm-55427/)).
- The only reference points are all **vendor** figures: Gemma 4 31B MMMLU 85.2% [A]; Qwen3.8-27B Belebele 93.2% [A] ([benchlm](https://benchlm.ai/md/models/qwen3-8-27b.md)). These are multiple-choice tests, not writing.
- **Writing**: Gemma 4 31B is considered better than Qwen at writing and translation [R]. No EQ-Bench creative score could be read.
- **Verbosity**: Flash-Next produced 200M output tokens on the AA index vs a 110M median [M]; Qwen3.8-27B has reported reasoning loops, sometimes over an hour [R] ([eesel.ai](https://www.eesel.ai/blog/qwen38-flash-next-review), [HN](https://hn.nuxt.dev/item/49334544)). For long-form fiction writing, a **non-reasoning** or reduced-effort mode will probably be preferable [D].
- Mistral (a French vendor) has **no French-language data** in the sources. Its European reputation remains a hypothesis.

---

## Contradictions between sources

| Topic | Version 1 | Version 2 | Likely reading |
|---|---|---|---|
| Flash-Next size | 125B-A6B (model card) | 177–180B (furnace, Traictory, HF counter) | 125B excluding n-gram table and MTP, ≈180B including them [D] |
| Minimal Flash-Next GGUF | 67.56 GiB (openclawdc) | 72.5 GB (Unsloth) | Same file, different units [D] |
| llama.cpp Flash-Next PR | #27742 (Traictory) | #27739 (X post) | To be checked in the repository |
| Qwen3.8 MTP PR | #22673 (sudoingX) | #28243 (furnace) | Possibly two separate PRs |
| Flash-Next on 4x MI50 | 44.8 / 58.3 t/s | 48.3 / 67.4 t/s | Two versions of the furnace README [D] |
| SOLVE_TRI | Crash on Qwen3.5-35B-A3B | Qwen3.6-35B-A3B runs on stock; the author's Qwen3.5-27B runs | Depends on block sizes [D] |
| Gemma 4 and FlashAttention | Works with `-fa 1` (furnace) | FA crash or hang without FA (machinesoftworks, b9549) | Depends on the build |
| Vulkan on MI50 | Better prefill than ROCm on MoE (2x MI50) | 18 vs 30 t/s on Qwen3.8-27B; ×8.7 on Gemma-4-26B | Model-dependent; ROCm by default |
| Qwen3.8-27B LiveCodeBench | 84.0% (Vals) [M] | 90.3% (vendor) [A] | The vendor overstates |
| AA index for Qwen3.8-27B | 52 (August) | 34 (October) | v4.3 scale change |
| Gemma 4 release | 2 April 2026 (Wikipedia) | March 2026 (computingforgeeks) | 2 April more likely |

---

## Candidates to bench-test

Ranked by expected value / risk. **This is not a decision.** Each entry gives the measurement that would confirm or eliminate it.

A common test battery is recommended [D]:

- 30 to 50 real tool calls via Hermes and Claude Code (valid-JSON rate, success rate, honesty after failure);
- 10 office-assistant tasks in French (email, summaries, rewording), scored blind;
- one chapter of 8,000 to 15,000 words of fiction in French, scored blind on coherence, repetition and style;
- speed at 2k, 32k and 64k context, and time to first token with a 15k system prompt.

### 1. Qwen3.8-27B — the data's favourite

- **Quant**: Q8_0 (29 GB).
- **Config**: A. Either split across the 2 MI50s for very long context, or one instance per card for two agents.
- **Engine**: upstream llama.cpp (recent `mixa3607/llama.cpp-gfx906` image), then Martin1701 or furnace for MTP.
- **Main risk**: reasoning loops and verbosity, plus the SOLVE_TRI bug.
- **Deciding measurement**: tool-call success rate **and** number of reasoning tokens per task, medium vs xhigh effort.

### 2. Qwen3.5-122B-A10B — the best test of "bigger"

- **Quant**: UD-IQ4_NL (61.4 GB) or Q4_K_S (71.7 GB).
- **Config**: B preferred. In A, IQ4_XS (60.2 GB, ≈8.5 GB gross headroom) and IQ4_NL (≈7.3 GB) are plausible but tight once KV and per-card overhead are counted.
- **Engine**: llama.cpp built with `GGML_BACKEND_DL=ON` HIP+CUDA, `-sm layer`, `-DGGML_CUDA_NO_PEER_COPY=ON`.
- **Main risk**: mixed HIP+CUDA was unmeasured on MI50 when this survey was written (since measured working by the author, see 04); in a static build, CUDA silently disappears (issue [#29048](https://github.com/ggml-org/llama.cpp/issues/29048)).
- **Deciding measurement**: blind head-to-head with #1 on the French battery and long-form writing. If it does not win where AA does not measure, the size argument falls.

### 3. Gemma 4 31B — the challenger for writing and French

- **Quant**: Q8_0 (≈33 GB [D]) or UD-Q4_K_XL.
- **Config**: A, KV f16 mandatory.
- **Engine**: furnace (+24% generation claimed) or upstream llama.cpp.
- **Main risk**: D=512 FlashAttention unstable on gfx906, and false tool successes.
- **Deciding measurement**: blind score of the French chapter vs #1, plus a deliberately failing tool test (does the model report it?).

### 4. Qwen3.8-Flash-Next — the "big and recent" bet

- **Quant**: AtomicChat IQ4_XS (45.8 GB GPU + 39.1 GB n-gram table on SSD).
- **Config**: A is enough for the GPU part; the table must be on the **SSD**, not the HDD.
- **Engine**: furnace (the only one tested on MI50 with this model), otherwise the AtomicChat branch.
- **Main risk**: loading impossible or very slow with 35 GB of RAM; experimental, under-trained model.
- **Deciding measurement**: first, does it load and exceed 10 t/s? Only then, quality vs #1.

### 5. Mistral-Small-4-119B — the fast option to verify

- **Quant**: UD-IQ4_XS (58.1 GB) in A, or UD-Q4_K_S (69.5 GB) in B.
- **Config**: B preferably.
- **Engine**: upstream llama.cpp.
- **Main risk**: lowest AA score on the list (11), and no published gfx906 test.
- **Deciding measurement**: French quality vs Gemma 4. If it does no better, it drops off the list.

### Excluded from the bench, with reasons

- **Nemotron-3-Super**: B only, AA 13, Mamba-2 layers never tested on gfx906.
- **Laguna S 2.1**: code-specialised, outside the main need.
- **gpt-oss-120b**: 2025; keep only as a control if a known comparison point is wanted.

---

## Conclusion

In a 64 to 76 GiB window, the real choice is not between a small and a large model. It is between **a recent dense model kept almost intact (Q8)** and **a larger MoE compressed to the maximum**, on hardware where IQ formats and mixed AMD+NVIDIA are the least proven paths. Public data clearly favours the former, but it is blind to the two use cases that matter most here: French and long-form writing.

So the bench has to decide, with the same battery for every model. Two outcomes would change the picture: a Qwen3.5-122B or Flash-Next clearly better in French in blind tests, or conversely a Qwen3.8-27B unable to sustain tool calls without looping. Before committing to anything, an adversarial review of these results is still needed.
