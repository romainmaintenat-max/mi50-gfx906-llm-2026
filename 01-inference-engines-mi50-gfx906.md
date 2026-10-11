# Getting the most out of two MI50s in 2026: inference engines on gfx906

**Scope.** Survey as of 11 October 2026; web research, links checked at that date; reddit.com was read through a mirror. It covers the inference engines that run on the AMD Instinct MI50 (gfx906) and compares their published figures with the author's own measurements on this rig: 2x AMD Instinct MI50 32 GB (gfx906) + 1x RTX 3060 12 GB, Ubuntu host, ROCm only inside containers (no ROCm on the host), no VBIOS flash, and a power cap of 200 W per MI50 for every measurement quoted here.

**Status: first public version (11 Oct 2026). Web survey; links checked on that date.**

---

**Short answer.** On the MI50 (gfx906), only two engines are really alive in October 2026: **llama.cpp** (upstream and through about ten gfx906 forks) and a **line of vLLM forks** descended from the nlzy project, which was archived on 20 February 2026. Everything else is either missing or broken. Ollama survives only through community builds, SGLang produces unusable output, and nothing was found for TGI, ExLlama, Aphrodite or ktransformers.

The author's own measurements are **already at the level of the best published figures** for llama.cpp on one card (Qwen3-30B-A3B at 94 t/s generation, where most sources give 75 to 90). The gfx906 forks mostly promise gains in *prefill* (+12 to +37%), with very little in generation on MoE models.

The biggest gap on this rig is the **72B**: vLLM with TP2 decodes at about 22 t/s, against 10 t/s for llama.cpp with `-sm layer`. *(Superseded by 04: upstream llama.cpp `-sm tensor` was later measured at 16.77 t/s on the 72B Q4_0, and with speculative decoding at 22.7–24.3 t/s.)* The mechanics explain it: layer splitting makes the cards work one after the other, while tensor parallelism makes them work at the same time. The Vulkan result (0.8 t/s) is abnormal and looks like weights being read across PCIe.

Under concurrency, both vLLM tests scale only ×1.6 from 1 to 8 requests. The likely bottleneck is gfx906 **prefill** (no matrix cores), not decoding. That is the card's real ceiling for a local multi-agent setup that sends long contexts.

The final decision is left to the reader: the last section gives a **per-use-case matrix** rather than a single conclusion.

---

## How to read the figures in this report

Every figure carries a certainty label. The label says where the figure comes from, not whether it is true.

- **[M]**: published measurement with a readable protocol (llama-bench, commit, hardware). Good reliability.
- **[A]**: figure announced by a fork's author about their own code. Useful, but self-assessed.
- **[R]**: reported, rounded, anecdotal, or read through a Reddit mirror or a summarizer. Needs cross-checking.
- **[D]**: deduction or calculation made in this report. Stated as such.

Three method limits apply to the whole report:

- reddit.com was blocked during the research: r/LocalLLaMA threads were read through a mirror (reddit.sentinel-team.org);
- the GitHub API was unreachable: last-commit dates are often unknown;
- several 2026 model names (Qwen3.6, Qwen3.8, Gemma 4, Ornith, Kimi K2.6) come from the sources and could not be verified independently. They are reproduced as published.

---

## The author's measurements are already at the published ceiling for llama.cpp

This is the most useful section for deciding. It sets each of the author's measurements (llama-bench, 200 W per card) against the closest published figures.

A note on power: the default limit of an MI50 is 225 W. Capping at 180 W costs about **10% of prefill and almost nothing in generation** [M] ([W. Tarreau's blog](http://wtarreau.blogspot.com/2025/12/amd-radeon-instinct-mi50-32gb-best-ai.html)). The 200 W cap used here therefore barely biases the generation comparison.

### Comparison table

| Author's measurement | Comparable published figures | Verdict |
|---|---|---|
| **14B Q4_0**, 1 card: pp512 610 / tg128 48 | Custom Ollama qwen3:14b **44 t/s**, phi4:14b 40.4 [M]; Llama 2 7B Q4_0 pp512 1129 / tg128 106 [M] | Consistent. A 14B is about half the speed of a 7B: 48 t/s is expected. |
| **Qwen3-30B-A3B Q4_0**, 1 card: pp512 1390 / tg128 94 | Q5_K_M mainline 590 / 76.7 [M]; Q8_0 pp512 1122 / tg 87 after patch [A]; Qwen3.6-35B-A3B Q4_K_XL 864 → 968 / 89.9 [A]; "~1100 PP / 100 TG" [R] | **At the top of the range.** Q4_0 is the fastest format on gfx906. |
| **Qwen3.5-27B Q4_0**, 1 card: pp512 293 / tg 25 | vLLM Qwen3.5-27B-AWQ **25.6 t/s** [A]; "27B ~300 PP / 30 TG at 180 W" [R]; Qwen3-VL-32B Q4_0 260–278 / 24.5–25 [M] | Consistent. The "30 t/s" quoted refers to another model (Qwen3.8) or other settings. |
| **72B Q4_K_M**, 2 cards layer: pp512 95 / tg 10 | No sourced measurement. Extrapolation from the notes: 10–12 t/s [D]; vLLM Qwen2.5-72B GPTQ 2×MI50: 16–18 t/s [R] | Consistent with the extrapolation. Q4_K_M is the wrong format for gfx906 (see below). |
| **Vulkan 72B**: tg 0.8 | Vulkan ≈ ROCm on Llama 2 7B (tg 118 vs 106) [M]; 18 vs 30 t/s on a 27B [R]; 9.4 vs 82 t/s on a MoE [M] | **Abnormal.** Even the worst published gap (×8.7) does not explain it. |
| **vLLM 72B AWQ TP2**, c1: 11.78 t/s output, TTFT 10.3 s, TPOT 44.8 ms | Qwen2.5-72B GPTQ-Int4 2×MI50: 16–18 t/s (May 2025) [R] | The author's decode speed (≈22 t/s) **exceeds** the Chinese figure. The "output" throughput is dragged down by TTFT. |
| **vLLM 14B**, c1 26.47 t/s, TPOT 21.7 ms; c8 42.78 t/s | Qwen3.5-35B-A3B-AWQ: 66.1 t/s at c1, **193 t/s at N=8** (×2.9) [A] | Decoding on par with llama.cpp. Weak concurrency scaling (×1.6). |

Table sources: [MTLoser/ollama-mi50](https://github.com/MTLoser/ollama-mi50-rocm71-build), [llama.cpp #15021](https://github.com/ggml-org/llama.cpp/discussions/15021), [W. Tarreau's blog](http://wtarreau.blogspot.com/2025/12/amd-radeon-instinct-mi50-32gb-best-ai.html), [PR #21698](https://github.com/ggml-org/llama.cpp/pull/21698), [sixvolts furnace](https://github.com/sixvolts/llamacpp-gfx906-furnace), [Reddit mirror, 32x MI50](https://reddit.sentinel-team.org/posts/1t0b0ie/snapshots/2026-05-02T03%3A06%3A21.60641Z), [KIntegrated](https://github.com/KIntegrated/vllm-gfx906-mobydick), [Bilibili 纳兰泽雨](https://www.bilibili.com/opus/1062709216989413379), [DByte308](https://github.com/DByte308/MI50), [Level1Techs](https://forum.level1techs.com/t/glm-and-i-created-a-llama-cpp-fork-optimized-for-amd-gfx906-mi50-mi60-radeon-vii-gcn-hip/254257).

### What these figures say beyond the table

**1. The "70 t/s ceiling for A3B" is wrong for Q4_0.**
The DByte308 log concludes that about 70 t/s is the practical ceiling for an A3B MoE on gfx906 [R] ([DByte308](https://github.com/DByte308/MI50)). But it measures in Q4_K_M and Q5. The author's 94 t/s in Q4_0, sixvolts' 89.9 t/s and the 87 t/s of PR #21698 contradict it. The format matters more than the engine.

**2. The forks will add almost nothing to generation on these MoE models.**
- PR #21698 by iacopPBK: prefill +36%, generation 87.12 → 87.01 t/s [A] ([PR #21698](https://github.com/ggml-org/llama.cpp/pull/21698)).
- The furnace fork on Qwen3.6-35B-A3B: prefill 864 → 968, generation 89.9 → 89.9 [A] ([sixvolts](https://github.com/sixvolts/llamacpp-gfx906-furnace)).

The forks' generation gains apply to **dense models in K-quants** (furnace: +24% on Gemma 4 31B Q4_K_XL [A]) and to **multi-slot** serving.

**3. The 72B in layer split is slow by design, not because of bad tuning.** [D]
- With `-sm layer`, the two cards work one after the other. For each token, the ≈47 GB model has to be read at the speed of a single card.
- 47 GB × 10 t/s ≈ 470 GB/s effective, about 46% of the HBM2's theoretical 1 TB/s. That is actually a better efficiency than the 14B (≈8 GB × 48 ≈ 380 GB/s).
- With vLLM TP2, each card reads its half **at the same time**. ≈40 GB in AWQ × 22 t/s ≈ 880 GB/s combined, i.e. ≈440 GB/s per card.

Conclusion of the calculation: vLLM's ×2.2 factor comes from parallelism, not from a better kernel. Tarreau also observes that **2 cards in layer mode do not change generation**, only prefill on long prompts [M] ([wtarreau](http://wtarreau.blogspot.com/2025/12/amd-radeon-instinct-mi50-32gb-best-ai.html)).

**4. Prefill is the real bottleneck under concurrency.** [D]
- vLLM 72B: 2048 tokens in 10.3 s, i.e. **≈199 t/s of prefill**. About 2× the llama.cpp pp512 (95), but still slow.
- The 11.78 t/s at c1 matches a request with 256 output tokens almost exactly: 10.3 s + 256 × 44.8 ms ≈ 21.8 s, i.e. ≈11.8 t/s. **Nearly half of the request time is prefill.** (The output token count is a deduction of this report; it was not part of the author's stated test parameters.)
- At c8, 8 × 2048 = 16,384 tokens have to be prefilled. At ≈200 t/s, that is on the order of 80 s of prefill compute. Hence a gain of only ×1.6.
- The 8× MI50 case on Reddit confirms it: 4 requests of 10k tokens → **mean TTFT 32.9 s** [R] ([Reddit mirror](https://reddit.sentinel-team.org/posts/1tc9j6u/snapshots/2026-05-15T23%3A40%3A01.25497Z)).
- KIntegrated's 193 t/s at N=8 was measured on a MoE, with unclear input length. Its cold-cache figures are **11 to 34% lower** than warm-cache [A] ([KIntegrated](https://github.com/KIntegrated/vllm-gfx906-mobydick)).

For a multi-agent setup that shares long system prompts, **prefix caching** (vLLM) or **per-slot cache reuse** (llama-server) matters more than the choice of engine.

**5. Vulkan at 0.8 t/s looks like a memory spill.** [D, hypothesis]
- 47 GB / 0.8 t/s ≈ 59 GB/s. An order of magnitude compatible with part of the weights being read across PCIe or from host RAM.
- The notes found **no source** on a Vulkan heap limit (16 GiB or other) on the MI50. It is therefore not asserted here.
- This test is not worth digging into for MI50-only use. Vulkan is only interesting for mixing AMD and NVIDIA cards (see multi-GPU).

---

## llama.cpp: upstream runs, about ten forks push prefill

### What upstream does for gfx906

Upstream builds and runs on gfx906 with `-DGGML_HIP=ON -DGPU_TARGETS=gfx906`, but **does not target** this card. It uses the dp4a MMQ path and the generic tile/vec FA kernels ([exabit tuning](https://github.com/exabit-io/llama.cpp-gfx906-tuning)).

Things to know:

- **rocBLAS Tensile files.** ROCm 7.x no longer ships gfx906 kernels. You need either ROCm ≤ 6.4, a manual copy, or an image that bundles them (mixa3607, mxxm) ([countryboycomputersbg](https://countryboycomputersbg.com/dual-instinct-mi50-32gb-running-moe-models-with-self-built-llama-cpp-gpt-oss20b-qwen330b-and-gpt-oss120b/)).
- **SOLVE_TRI bug (issue #19972).** It affects hybrid SSM / linear-attention models: Qwen3.5-35B-A3B, Qwen3-Next, Qwen3-Coder-Next, Kimi Linear. The operation falls back to rocBLAS strsm, which is missing for gfx906, and crashes with `hipErrorInvalidDeviceFunction` [M] ([issue #19972](https://github.com/ggml-org/llama.cpp/issues/19972)). Documented workaround: `--no-warmup` ([arte-fact](https://github.com/arte-fact/llamacpp-gfx-906-turbo)). Qwen3.5-27B runs on this rig, so the author's build is not affected, or no longer is.
- **Issue #16424** (3× MI50): corrupted output with `-sm layer`, memory fault with `-sm row`. Fixed by disabling IOMMU/VT-d or P2P (`-DGGML_CUDA_NO_PEER_COPY=On`). The maintainer sees it as a platform/PCIe bug, not a llama.cpp one ([issue #16424](https://github.com/ggml-org/llama.cpp/issues/16424)).
- **Do not enable ROCWMMA**: gfx906 has neither WMMA nor MFMA ([machinesoftworks](https://machinesoftworks.com/posts/custom-gfx906-llamacpp-engine)). The nullkalahar guide recommends it anyway (`GGML_HIP_ROCWMMA_FATTN=ON`): a contradictory source ([nullkalahar](https://github.com/nullkalahar/mi50-rocm7)).
- **Maintainers are slowing down gfx906 optimizations.** PR #21698 (prefill +29 to +36%) is blocked: JohannesGaessler considers it "too convoluted and complex" ([PR #21698](https://github.com/ggml-org/llama.cpp/pull/21698)). An "S1 tile-table" patch from Exabit would give +29/+34% prefill on master, but had not yet been submitted as of 2026-09-09 [A] ([exabit tuning](https://github.com/exabit-io/llama.cpp-gfx906-tuning)).

### Upstream settings with a measured effect

- **`GGML_HIP_GRAPHS=ON`**: +12% generation [R] ([DByte308](https://github.com/DByte308/MI50)); +8 to 10% [A] ([arte-fact](https://github.com/arte-fact/llamacpp-gfx-906-turbo)).
- **Larger `-ub`**: ubatch 512 → 4096 = **+46% prefill** (833 → 1214 t/s) [R] ([DByte308](https://github.com/DByte308/MI50)); `-ub 2048` = +18% vs 512 [A] ([exabit](https://github.com/exabit-io/llama.cpp-gfx906-tuning)). A pp512 llama-bench run does not see this effect.
- **`-fa on`**: the only attention path measured by Exabit, and it also wins on Llama 2 7B (pp 1057 → 1129) [M] ([#15021](https://github.com/ggml-org/llama.cpp/discussions/15021)).
- **KV q8_0**: 15 to 36% slower decoding at large depth [A] ([exabit](https://github.com/exabit-io/llama.cpp-gfx906-tuning)). Use only if the context does not fit otherwise.
- **`--no-mmap`**: recommended by several sources (mxxm, DByte308, Tarreau).

### gfx906 forks

All are individual projects. The most followed has 140 stars.

| Fork | Base / date | Status | Main contribution | Announced gains |
|---|---|---|---|---|
| [iacopPBK/llama.cpp-gfx906](https://github.com/iacopPBK/llama.cpp-gfx906) | build 7924, ROCm 7.1.1, last batch "2602" | **Frozen** ("not updated"), upstreaming | FA Q8, MMVQ Q4_0/Q4_1/Q8_0, DPP, MXFP4 | Prefill +29 to +36% via PR [A] |
| [mxxm-t/mx-llama.cpp](https://github.com/mxxm-t/mx-llama.cpp) | upstream b10760 | **Active** (Sept.–Oct. 2026), 50 stars | GPU weight repack (Q8_0, MXFP4, IQ4_NL, Q4_K, Q5_K, Q6_K, Q5_1), **tensor parallel `-tps`**, custom AllReduce, MTP/DFlash | Measurements in FEATURES.md, not read |
| [exabit-io/mx-llama.cpp](https://github.com/exabit-io/mx-llama.cpp) | fork of mx-llama.cpp | Active; the old repo was archived on 2026-09-24 | Multi-GPU patches, MMQ tables | Qwen3.8-27B Q8_0 tp4: 48.1 → 57.9 t/s; **78.5** with MTP [A] |
| [Martin1701/MI50-llama.cpp](https://github.com/Martin1701/MI50-llama.cpp) | mx-llama.cpp b10760, 0 stars | Active | Single MI50 32 GB: GQA6 attention, MTP | Qwen3.8-27B Q8_0 at 64k: 14.2 → 20.4 t/s; MTP 28.8 [A] |
| [sixvolts/llamacpp-gfx906-furnace](https://github.com/sixvolts/llamacpp-gfx906-furnace) | upstream of 2026-09-22 | **Active**, 13 stars | Three-plane K-quant repack (**≈89% of HBM bandwidth vs ≈58%**), grouped MoE, host copies | Gemma 4 31B Q4_K_XL: 186.7/22.4 → 220.3/27.9 [A] |
| [milpster/gfx906-llama-cpp](https://github.com/milpster/gfx906-llama-cpp) | synced 2026-09-03, **ROCm 6.1** | Active | Wave64 MMQ retune, native q8_0 FA, cost-based ROCm+Vulkan split | PP16384 332 → ~410 (+23%) [A, partly compounded] |
| [arte-fact/llamacpp-gfx-906-turbo](https://github.com/arte-fact/llamacpp-gfx-906-turbo) | derived from iacopPBK, ROCm 7.1+ | Low activity (32 commits) | TurboQuant turbo3 KV (≈3.5 bits) | 3.3× more context for −18% generation [A] |
| [eslowney/llama.cpp-gfx906](https://github.com/eslowney/llama.cpp-gfx906) | unknown | Unknown (page unreadable) | gfx906 FA kernels | — |
| ollo12-prog solve-tri-fix | — | Archived on 2026-04-05 [R] | SOLVE_TRI fix for Qwen3.5 | — |
| Machine Softworks "Build33" | b9549 | Personal | TurboQuant + Gemma 4 FA fix | — ([machinesoftworks](https://machinesoftworks.com/posts/custom-gfx906-llamacpp-engine)) |

Lineages, as reconstructed here [D]:

- iacopPBK → arte-fact → Build33 (late 2025 to early 2026, frozen);
- mxxm-t → exabit-io → Martin1701 (summer–autumn 2026, active);
- furnace and milpster are two independent lineages.

Caution: Exabit's figures come from 4 Radeon Pro Vega II linked by **XGMI** in a Mac Pro. They **do not represent** PCIe MI50s like the ones on this rig ([exabit](https://github.com/exabit-io/llama.cpp-gfx906-tuning)).

### llama.cpp Docker images usable without ROCm on the host

- **`mixa3607/llama.cpp-gfx906`**: upstream rebuilt with gfx906 rocBLAS. Latest tag `b11513-rocm-7.14-…`, updated around 2026-10-10 ([Docker Hub](https://hub.docker.com/r/mixa3607/llama.cpp-gfx906)). The best-maintained base.
- **`mxxm/llama-cpp-gfx906`**: "vanilla" upstream, tags `rocm-633` and `rocm-727` ([Docker Hub](https://hub.docker.com/r/mxxm/llama-cpp-gfx906)).
- **`mxxm/mx-llama.cpp:gfx906`**: the tensor-parallel fork. It requires `--ipc=host`, unconfined seccomp and AppArmor, and **always `--no-mmap`** (mmap hangs on this stack) ([Docker Hub](https://hub.docker.com/r/mxxm/mx-llama.cpp)).
- **`mxxm/iacoppbk-llama.cpp-gfx906`**: the frozen iacopPBK fork ([Docker Hub](https://hub.docker.com/r/mxxm/iacoppbk-llama.cpp-gfx906)).
- **`knguyen298/llama-swap-gfx906`**: llama-swap packaged for gfx906, cited as the main provider on Level1Techs ([L1T](https://forum.level1techs.com/t/glm-and-i-created-a-llama-cpp-fork-optimized-for-amd-gfx906-mi50-mi60-radeon-vii-gcn-hip/254257/2)).
- Not inspected: `aceshigh1/llama-mi50`, `amstel8/llama-rocm`, `arkahnat/llama.cpp-gfx906`, `reekid/llama.cpp-rocm-vulkan`, `mxxm/llama-mi50`.

The cost of Docker compared with bare metal is measured at only **+0.2 to 0.3%** [R] ([DByte308](https://github.com/DByte308/MI50)). Running ROCm in containers, as on this rig, costs nothing in performance.

### Concurrency in llama-server

The only rich data comes from Exabit (XGMI, so treat with caution):

- decode only, 8 slots: 174 t/s; 16 slots: 204 t/s [A];
- **upstream trap**: at 9 slots, throughput drops from 160 to 97 t/s (threshold of the 16-wide MMQ tile) [A];
- in real serving (1300 / 256-token requests), throughput plateaus around **83 to 100 t/s**; two tp2 pairs give +20% over one tp4 [A];
- Exabit's recommendation: **cap `-np` at 16**.

Source: [exabit tuning](https://github.com/exabit-io/llama.cpp-gfx906-tuning).

On 2× MI50 with `-sm layer` and Qwen3.6-35B-A3B, a Reddit test gives **129 t/s at 4 requests**, then a collapse to about 27 t/s at 8 requests, whatever the backend [R] ([Reddit mirror](https://reddit.sentinel-team.org/posts/1typr7o/snapshots/2026-06-13T02%3A56%3A06.64152Z)).

---

## vLLM: a line of individual forks, useful for the 72B and for concurrency

### The original project is archived, with no official successor

`nlzy/vllm-gfx906` (image `nalanzeyu/vllm-gfx906`) has been **archived since 20 February 2026**. The author cites rising MI50 prices and growing model sizes, and simply writes "everyone is welcome to fork it" ([nlzy](https://github.com/nlzy/vllm-gfx906)).

Its recommendations still hold for the whole lineage:

- recommended formats: **GPTQ, AWQ** and llm-compressor W4A16 INT;
- "All MoE quantization models are significantly slow, and all unquantized models are slightly slow".

### Active forks

| Fork | Base | Stack | Status | Published figures |
|---|---|---|---|---|
| [ai-infos/vllm-gfx906-mobydick](https://github.com/ai-infos/vllm-gfx906-mobydick) | vLLM ≈0.19–0.20 | ROCm 6.3.4, PyTorch 2.11, triton-gfx906 3.6.0, flash-attention-gfx906 (Triton) | **De facto successor**, 84 stars, image `aiinfos/vllm-gfx906-mobydick` | None in the README |
| [KIntegrated/vllm-gfx906-mobydick](https://github.com/KIntegrated/vllm-gfx906-mobydick) | vLLM 0.29 | ROCm 7.14 + **amdgpu DKMS driver on the host** | **Most active** (27 Aug – 15 Sept 2026) | Many, see below [A] |
| [David-Lzy/vllm-gfx906](https://github.com/David-Lzy/vllm-gfx906) | v0.28.0-gfx906.1 | ROCm 7.2.1 | Active, **no image** | No tok/s; experimental TP4 |
| [ttdxq/gfx906-vllm](https://github.com/ttdxq/gfx906-vllm) | — | ROCm 6.3 or 7.2 | Separate branch | None; **TP2 requires an RCCL built for gfx906 (2.27.7)** |
| [ai-infos/vllm-gfx906-deepseek](https://github.com/ai-infos/vllm-gfx906-deepseek) | — | — | Archived on 2026-02-22, merged into mobydick | — |

Other images, not verified: `btbtyler09/vllm-rocm-gcn5`, `lykaos/vllm-gfx`, `christopherlin/vllm-gfx906`, `nickoptimal/vllm-gfx906`. The mixa3607 vLLM image is **paused** ([ML-gfx906](https://github.com/mixa3607/ML-gfx906)).

### KIntegrated figures (September 2026) [A]

- Qwen3.5-35B-A3B-AWQ, 1 card: **66.1 t/s** single request; **193 t/s at N=8**; 88.6 t/s with MTP k=2.
- Qwen3.5-27B-AWQ, 1 card: **25.6 t/s**; **39.4 t/s with MTP** k=2.
- Qwen3.8-27B-AWQ-INT4 in TP2: 59.2 t/s with MTP; prefill 443.9 t/s at 32k, 289 t/s at 110k.

Source: [KIntegrated](https://github.com/KIntegrated/vllm-gfx906-mobydick).

Other reference points:

- 8× MI50, TP8, Qwen3.6-27B fp16: 52.8 t/s generation; **≈34 t/s in TP2** [R] ([Reddit mirror](https://reddit.sentinel-team.org/posts/1tc9j6u/snapshots/2026-05-15T23%3A40%3A01.25497Z)).
- 2× MI50 with vLLM, Qwen3.5/3.6 35B at 50k context: prefill ≈400, decode ≈35 t/s [R, third-party screenshots] ([Bilibili](https://www.bilibili.com/video/BV1QY9YB7EZm/)).

### vLLM constraints and traps on gfx906

- **`--dtype float16` is mandatory**: gfx906 has no bf16. bf16 may fall back to fp32 ([ai-infos](https://github.com/ai-infos/vllm-gfx906-mobydick)).
- **`iommu=pt` on the host kernel**, otherwise RCCL can hang. An analogous case on MI100 was fixed this way, together with `HSA_FORCE_FINE_GRAIN_PCIE=1` ([ROCm/rccl #1129](https://github.com/ROCm/rccl/issues/1129)).
- **GPU hangs** ("wedge") that require a reboot. KIntegrated recommends a canary probe ([KIntegrated](https://github.com/KIntegrated/vllm-gfx906-mobydick)).
- Without KIntegrated's `.pth` shim, **one CPU core spins at 100% per TP worker**.
- The PyPI Triton 3.8.0 wheel crashes (segfault).
- **3 cards = pipeline parallel only**, with no speed gain ([Bilibili](https://www.bilibili.com/opus/1062709216989413379)).
- The V1 runner disappears from upstream vLLM 0.32.0. The forks will have to follow or stay frozen [A].
- Unstable history: the lineage moved from the V1 engine to V0 in August 2025 (regressions), then back to V1 in September 2025 ([ai-infos deepseek](https://github.com/ai-infos/vllm-gfx906-deepseek)).

**Conflict with the "no ROCm on the host" constraint**: KIntegrated requires the official amdgpu DKMS driver (6.19.14), because the stock Ubuntu driver would reportedly block RCCL P2P. Strictly speaking this is not ROCm, but it is a host-side install. ai-infos/mobydick, by contrast, only asks for the recommended 6.11 kernel and `iommu=pt`.

### Reading the author's vLLM figures [D]

- **14B**: TPOT 21.7 ms = 46 t/s, against 48 t/s for llama.cpp. **Parity in single-stream.**
- **Qwen3.5-27B**: 25 t/s with llama.cpp against 25.6 t/s with vLLM at KIntegrated. Parity as well.
- **72B**: vLLM wins ×2.2 in decoding and ×2 in prefill. It is the only case where vLLM clearly wins on a single request.

This matches the consensus in the notes: vLLM wins on parallelism (TP, concurrency), not on single-user use.

---

## Other engines: almost all missing or broken

| Engine | Status on gfx906 (Oct. 2026) | Effort | Certainty |
|---|---|---|---|
| **Official Ollama** | Support dropped after **0.12.3** (or 0.12.4 depending on sources); 0.13.0 removed the gfx906 files | — | [R] ([countryboy](https://countryboycomputersbg.com/dual-instinct-mi50-32gb-running-moe-models-with-self-built-llama-cpp-gpt-oss20b-qwen330b-and-gpt-oss120b/), [L1T](https://forum.level1techs.com/t/radeon-instinct-mi50-and-ollama-in-a-mv/239906)) |
| **Community Ollama** | `xxdoman/ollama-mi50`: v0.35.1 (llama.cpp b11232), ROCm 7.2, changelog dated 04.10.2026, MTP | Low (image) | [R] ([Docker Hub](https://hub.docker.com/r/xxdoman/ollama-mi50)) |
| MTLoser custom Ollama | v0.18.2, ROCm 7.1 + Tensile 6.3, SOLVE_TRI patch; qwen3:14b **44 t/s** | Medium | [M] ([MTLoser](https://github.com/MTLoser/ollama-mi50-rocm71-build)) |
| Ollama on Vulkan | `ollama/ollama:0.33.3` works without override | Low | [R] ([L1T](https://forum.level1techs.com/t/radeon-vii-in-2026-uuughhh/245630?page=2)) |
| **SGLang** | NnnHU/sglang-mi50 port: loads and generates, but **degenerate output** (bf16 activations overflowing in fp16) | High, unusable | [A] ([NnnHU](https://github.com/NnnHU/sglang-mi50)) |
| **MLC-LLM** | A "dual MI60 – impressive speeds" thread exists, figures not retrieved; seen as an "exotic" Vulkan option | Unknown | [R] ([webshinetech](https://community.webshinetech.com/t/performance-benchmarks-with-dual-mi60-cards-mlc-llm-delivers-impressive-speeds-for-amd-hardware/3946), [Lemmy](https://lemmy.mariusdavid.fr/post/420908/1342996)) |
| **KoboldCpp** (koboldcpp-rocm) | Recommended on Lemmy, **no confirmed test** on MI50 | Unknown | [R] ([Lemmy](https://lemmy.mariusdavid.fr/post/420908/1342996)) |
| **ExLlamaV2/V3, TabbyAPI** | No source. Engines designed for CUDA | Probably impractical | [D] |
| **TGI, Aphrodite, ktransformers, PowerInfer, MIGraphX** | No source found | — | Absence of evidence, not evidence of absence |
| **LM Studio, LocalAI** | Probably through their llama.cpp Vulkan backend | Not verified | [D] |
| **distributed-llama, exo, petals** | No MI50 use found | — | — |
| **ik_llama.cpp** | Suggested for MoE, no figures | Unknown | [R] ([Lemmy](https://lemmy.mariusdavid.fr/post/420908/1343016)) |
| **Lemonade** (llamacpp-rocm) | Publishes a "gfx90X-dcgpu" target that includes gfx906 | Low | [R] ([L1T](https://forum.level1techs.com/t/glm-and-i-created-a-llama-cpp-fork-optimized-for-amd-gfx906-mi50-mi60-radeon-vii-gcn-hip/254257/2)) |

Why the desert [D, consistent with the sources]:

- gfx906 has disappeared from **LLVM 21+** and from the **official Triton 3.0+ wheels** ([NnnHU](https://github.com/NnnHU/sglang-mi50));
- no native bf16, while recent models are published in bf16;
- no **hipBLASLt** for gfx906, neither in ROCm 6.3.3 nor in TheRock 7.14. INT8 paths (`torch._int_mm`) therefore fail ([mixa3607 HIPBLASLT-GFX906](https://github.com/mixa3607/ML-gfx906/blob/master/rocm/HIPBLASLT-GFX906.md)).

Any Triton-based or bf16-based engine needs a heavy fork. llama.cpp avoids all three problems, hence its dominance.

---

## The containerized ROCm stack: TheRock 7.14 via mixa3607

### Official status

- AMD moved gfx906 to **maintenance mode with ROCm 6.0**. ROCm 5.7 is the last fully supported version. Fixes were promised only until Q2 2024 ([ROCm 6.0 release notes](https://rocm.docs.amd.com/en/docs-6.0.0/about/release-notes.html)).
- The current compatibility matrix (ROCm 10.1.0) mentions **neither gfx906 nor the MI50**. The oldest Instinct listed is the MI100 ([ROCm matrix](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)).
- From ROCm 7.0, rocBLAS no longer ships the gfx906 Tensile kernels ([nullkalahar](https://github.com/nullkalahar/mi50-rocm7)).

The compiler can still target gfx906. What is missing are the precompiled compute libraries [D].

### The reference community solution: mixa3607/ML-gfx906

This is the project that best fits the constraint of ROCm only in containers:

- MIT license, ≈331 stars, actively maintained in 2026;
- images: `rocm-gfx906`, `rocm-toolkit-gfx906`, `llama.cpp-gfx906`, `pytorch-gfx906`, `comfyui-gfx906`;
- current base: **TheRock `7.14.0-gfx906+20260802001858`**, tag `7.14-complete`;
- old tags still available: 6.3.3, 6.4.4, 7.0.x, 7.1.x, 7.2.0 to 7.2.4;
- pre-TheRock builds have been **unsupported since 2026-08-02**;
- the "gfx906 Tensile files" sub-project is deprecated, since TheRock now compiles those kernels;
- PyTorch 2.13.0 on ROCm 7.14 (`pytorch-gfx906:v2.13.0-rocm-7.14`).

Source: [ML-gfx906](https://github.com/mixa3607/ML-gfx906), [rocm/README](https://github.com/mixa3607/ML-gfx906/blob/master/rocm/README.md), [pytorch/README](https://github.com/mixa3607/ML-gfx906/blob/master/pytorch/README.md).

Users confirm that these containers run **without ROCm on the host** [R] ([Reddit mirror](https://reddit.sentinel-team.org/posts/1syqoby/snapshots/2026-05-02T03%3A06%3A21.60641Z)). Only the amdgpu kernel driver is needed on the host side.

Warning from mixa3607: do not mix the official AMD repositories with the gfx906 repository, or packages will conflict.

### The old Tensile trick (for the record)

Install ROCm 7.0.2, then copy the ≈156 `*gfx906*` files from rocBLAS 6.4.3 into `/opt/rocm/lib/rocblas/library/`. To be redone after every update ([nullkalahar](https://github.com/nullkalahar/mi50-rocm7)). Tarreau did the same from ROCm 6.4.4, noting that it "might trigger bugs" ([wtarreau](http://wtarreau.blogspot.com/2025/12/amd-radeon-instinct-mi50-32gb-best-ai.html)).

On this rig, the TheRock images make this trick unnecessary.

### The sramecc trap: check once

gfx906 exists in two ISA variants. Recent libraries ship only `gfx906:sramecc+`. A card that reports `sramecc-` (ECC disabled) crashes immediately in rocBLAS, even if `rocminfo` works ([Level1Techs](https://forum.level1techs.com/t/guide-prebuilt-fix-rocm-math-libraries-segfault-on-gfx906-with-ecc-disabled-vega-ii-radeon-vii-some-mi50s/252873), [Intermountainh8ter fix](https://github.com/Intermountainh8ter/rocm-gfx906-sramecc-fix)).

- Diagnosis: `rocminfo | grep -o 'amdgcn-amd-amdhsa--gfx906[^ ]*'` inside the container.
- `HSA_OVERRIDE_GFX_VERSION` does not fix this problem.
- Real-world case: an MI50 flashed with Apple Vega Pro II firmware (ECC off) mixed with ECC cards gives "invalid device function" [R] ([L1T](https://forum.level1techs.com/t/glm-and-i-created-a-llama-cpp-fork-optimized-for-amd-gfx906-mi50-mi60-radeon-vii-gcn-hip/254257)).

Since the VBIOS on this rig is not flashed, the cards should be `sramecc+` [D]. The llama-bench runs work, which indirectly confirms it.

### Flash-attention

- **No composable_kernel and no CK flash-attention** for gfx906. No evidence of it working was found.
- The real path is **llama.cpp's** FA kernel (upstream and forks).
- On the vLLM side, ai-infos/mobydick bundles a **Triton flash-attention-gfx906**, and KIntegrated reuses Q8 FA kernels from [cassettesgoboom/gfx906-fa-vllm](https://github.com/KIntegrated/vllm-gfx906-mobydick). No independent measurement of their effect.

### Which ROCm version?

No rigorous comparison exists (same model, same build, several ROCm versions). Opinions differ:

- **ROCm 6.1**: "fastest ROCm version for these cards", according to milpster, with no figure [R] ([L1T](https://forum.level1techs.com/t/glm-and-i-created-a-llama-cpp-fork-optimized-for-amd-gfx906-mi50-mi60-radeon-vii-gcn-hip/254257));
- **ROCm 6.4** for stability, according to nullkalahar ([nullkalahar](https://github.com/nullkalahar/mi50-rocm7));
- **ROCm 7.1.1** judged the most stable by an L1T user ([L1T](https://forum.level1techs.com/t/radeon-vii-in-2026-uuughhh/245630?page=2));
- **ROCm 7.14 TheRock** used by mixa3607, KIntegrated, Martin1701 and Exabit.

Since gfx906 no longer receives tuning, a recent version should not speed up rocBLAS. Differences would come from the compiler and the runtime [D]. Only a measurement on these cards will settle it.

---

## Multi-GPU: layer split gives room, tensor parallel gives speed

### The three llama.cpp modes

From the upstream docs ([multi-gpu.md](https://raw.githubusercontent.com/ggml-org/llama.cpp/HEAD/docs/multi-gpu.md)):

- **`-sm layer`** (default): the most compatible, tolerates slow links. Gives capacity, not single-stream speed.
- **`-sm row`**: **deprecated** upstream for all backends. On the MI50 it also causes memory faults (#16424).
- **`-sm tensor`**: experimental. Constraints:
  - flash attention mandatory;
  - **no quantized KV** (f32, f16 or bf16 only);
  - **MoE and hybrid models fail**;
  - highly dependent on the GPU interconnect, can be slower than a single card.

On ROCm, RCCL is available via `-DGGML_HIP_RCCL=ON`, but it is disabled by default. HIP tensor parallel **crashes on PCIe MI50s for lack of P2P**, according to arte-fact and OpenClaw ([OpenClaw](https://openclawdc.com/blog/amd-mi50-32gb-local-llm/)). The forks work around this:

- **mx-llama.cpp**: custom AllReduce via peer-to-peer writes, which "beats the RCCL ring for generation over PCIe" [A] ([mx-llama.cpp](https://github.com/mxxm-t/mx-llama.cpp));
- **furnace**: copies routed through the host, because "gfx906 PCIe peer copies crash in the HIP runtime" [A] ([sixvolts](https://github.com/sixvolts/llamacpp-gfx906-furnace)).

**No published figures** for upstream `-sm tensor` or mx-llama.cpp on 2× MI50 over PCIe. For a dense 72B, it is the only llama.cpp path that could catch up with vLLM TP2. It requires f16 KV, so less context.

### vLLM in TP2

This is the only published and measured way to beat single-card generation on a dense model. The ≈22 t/s measured on this rig with the 72B AWQ confirms it. The requirements are listed above (`iommu=pt`, `--ipc=host`, gfx906-compatible RCCL).

Diverging opinions:

- "vLLM can split over multiple cards reasonably well, better than llama.cpp" [R] ([Lemmy](https://lemmy.mariusdavid.fr/post/420908/1343016));
- "≈50 t/s with MTP on 2× MI50 under llama.cpp", hence a recommendation for llama.cpp over vLLM on a 27B [R] ([Reddit mirror](https://reddit.sentinel-team.org/posts/1tc9j6u/snapshots/2026-05-15T23%3A40%3A01.25497Z)).

Both are compatible: llama.cpp with MTP wins on a 27B that fits on one card; vLLM TP wins on a dense model that does not [D].

### PCIe link and xGMI

- No **xGMI** (Infinity Fabric) bridge benchmark on the MI50 was found. The topic is described as "unexplored".
- No P2P-on versus P2P-off comparison.
- Practical advice: check link widths with `lspci -vv`. A card at x4 through the chipset hurts TP2 and `-sm tensor`, and barely affects `-sm layer` [D] ([Reddit mirror](https://reddit.sentinel-team.org/posts/1w6wcbs/snapshots/2026-09-04T21%3A20%3A12.3645Z)).

### Mixing the MI50s and the RTX 3060

Three paths exist. None has a published figure for MI50 + 3060.

1. **llama.cpp Vulkan on all 3 cards, `-sm layer`.**
   - The simplest. An RTX 5060 Ti + MI50 user describes it as "slightly faster than the slower card" [R] ([Reddit mirror](https://reddit.sentinel-team.org/posts/1w20wtp/snapshots/2026-09-04T21%3A20%3A12.3645Z)).
   - Tensor split across vendors is not possible.
   - Given the 0.8 t/s Vulkan result on this rig, that result would need to be understood first.
   - milpster specifically targets mixed ROCm + Vulkan rigs, with a cost-based split that helps "only sometimes" ([L1T](https://forum.level1techs.com/t/glm-and-i-created-a-llama-cpp-fork-optimized-for-amd-gfx906-mi50-mi60-radeon-vii-gcn-hip/254257)).
2. **A single binary with HIP and CUDA** (GGML_BACKEND_DL). Reported as working but "not optimal", and not for all models [R]. Painful to build in a container.
3. **RPC between a ROCm container and a CUDA container.**
   - **Critical vulnerability CVE-2026-34159** (CVSS 9.8): unauthenticated remote code execution in the RPC backend ([GitHub advisory](https://github.com/ggml-org/llama.cpp/security/advisories/GHSA-j8rj-fmpv-wcxw)).
   - **Contradictory sources** on the fix: GitHub says "patched: none", the Qi-An-Xin CERT says fixed in **b8492** ([secrss](https://www.secrss.com/articles/89833)).
   - llama.cpp's security policy **excludes RPC from its scope**.
   - If used: build ≥ b8492, listen on 127.0.0.1 or an internal Docker network, **never on a VPN overlay or the LAN**.

Reading of this report [D]: adding 12 GiB to 64 GiB only gains ≈19% capacity, with an overhead on every layer placed on the 3060. The 3060 is better used as a **separate CUDA node** for light roles (router, embeddings, small draft model or fast critic). This is a proposal, not a published result.

### Large published rigs, for scale

- **8× MI50, TP8**, Qwen3.6-27B fp16: 52.8 t/s, 1569 t/s prefill [R].
- **32× MI50 across 2 nodes**, Kimi K2.6 int4: **9.7 t/s** generation, 263 t/s prefill, **640 W idle, 4.8 kW peak**. The poster's verdict: "Is it worth? No, unless you've got solar panels or free energy" [R] ([Reddit mirror](https://reddit.sentinel-team.org/posts/1t0b0ie/snapshots/2026-05-02T03%3A06%3A21.60641Z)).
- **2× MI50**, Qwen3-235B-A22B in IQ1_S (50.67 GiB): 137 / **19.9 t/s** [M] ([wtarreau](http://wtarreau.blogspot.com/2025/12/amd-radeon-instinct-mi50-32gb-best-ai.html)).

---

## Models and quantization: Q4_0, Q4_1 and Q8_0 beat K-quants

### Why format matters so much on gfx906

The forks wrote dedicated kernels (MMVQ/MMQ) for **Q4_0, Q4_1 and Q8_0**. K-quants and IQ go through the generic path ([arte-fact](https://github.com/arte-fact/llamacpp-gfx-906-turbo), [iacopPBK](https://github.com/iacopPBK/llama.cpp-gfx906)).

Likely explanation [D]: gfx906 has the int8 `v_dot4` instruction but no matrix cores. Q4_0 and Q8_0 unpack simply to int8. K-quants (super-blocks, 6-bit scales) and IQ (lookup tables) cost much more compute per byte. On a card that uses its bandwidth poorly, that compute dominates.

Measurements that confirm it:

- Qwen3-VL-32B, mainline: **Q4_0 259.9 / 25.0** versus **Q4_K_M 229.5 / 19.9** (pp512 / tg128) [M]. That is ≈+25% generation for Q4_0 ([wtarreau](http://wtarreau.blogspot.com/2025/12/amd-radeon-instinct-mi50-32gb-best-ai.html)).
- Exabit: Q4_0 vs Q8_0 = +11% decode, +33% prefill; **Q4_K_M = +4% decode but −12% prefill** relative to Q8_0 [A] ([exabit](https://github.com/exabit-io/llama.cpp-gfx906-tuning)).
- Chinese community: "q4_K_M slower than q4_1 / q8_0" ([Bilibili](https://www.bilibili.com/opus/1062709216989413379)).

Exception: the **furnace** fork specifically fixes K-quants through repacking (≈89% of bandwidth versus ≈58%) [A]. If Q4_K_M is required for quality, that is the fork to test.

**Applied to the 72B** [D]: moving from Q4_K_M to **Q4_0** should gain on the order of +20 to +25% in generation (≈12 t/s) and noticeably more in prefill, based on the gap measured on the 32B. That is still far from vLLM TP2's 22 t/s.

**No quality evaluation** (SWE-bench, Aider) compares Q4_0 and Q4_K_M on code in these sources. The quality cost of Q4_0 is not documented here.

### On the vLLM side

- **AWQ and GPTQ (W4A16) in fp16** are the only recommended formats.
- No fast FP8, NVFP4 or MXFP4 ([David-Lzy](https://github.com/David-Lzy/vllm-gfx906)).
- Quantized MoE: discouraged by nlzy, but KIntegrated gets 66 t/s on an AWQ MoE in 2026. **Sources in tension**: nlzy's advice predates the recent optimizations.
- Known failures: Qwen3.5-35B-A3B-GPTQ-Int4 and cyankiwi/Qwen3.5-35B-A3B-AWQ-4bit on ttdxq ([ttdxq](https://github.com/ttdxq/gfx906-vllm)).

### Model families that perform best

- **MoE with ≈3B active** (Qwen3-30B-A3B, Qwen3-Coder-30B-A3B, Qwen3.6-35B-A3B): 75 to 94 t/s on 1 card. Unanimously the best compromise for a coding agent.
- **Qwen3-Coder-Next 80B-A3B** in Q4_1 on 4 cards: 51.6 to 57.4 t/s at 65–90k context [A] ([arte-fact](https://github.com/arte-fact/llamacpp-gfx-906-turbo)). On 2× MI50 (64 GiB), it should fit in Q4 [D], but it is a hybrid model exposed to the SOLVE_TRI bug.
- **Dense 27–32B**: 20 to 30 t/s on 1 card. Better suited to a critic or validator role.
- **gpt-oss-120b (MXFP4)** and **GLM-4.5-Air**: logical candidates for 2× MI50, but **no MI50 figures found**. MXFP4 has optimized paths in iacopPBK and mx-llama.cpp.
- No figures either for gpt-oss-20b, DeepSeek, Llama 3.3 70B or Qwen2.5-72B under llama.cpp.

### Speculative decoding, MTP and long context

Speculative decoding should work well on gfx906: verifying k tokens costs almost as much as one when the card is not saturated [D].

- **MTP on a dense 27B**: 25.6 → 39.4 t/s with vLLM (+54%) [A] ([KIntegrated](https://github.com/KIntegrated/vllm-gfx906-mobydick)); 21.6 → 28.8 t/s at 64k with Martin1701 [A].
- Exabit tp4: 57.9 → **78.5 t/s** with MTP [A].
- **Contradictions**:
  - one user sees no MTP gain on a single card with llama.cpp [R];
  - furnace: with 4 users, MTP loses (77 t/s versus 92 without) [A];
  - MTP and DFlash are slower on long prompts with vLLM TP8 [R].
- **DFlash2** (acceptance rate 0.691) is used in production on gfx906 ([prismix](https://prismix.dev/news/063ee83c05f6)).

Takeaway [D]: speculation helps in **single-stream** and can hurt under **concurrency**. For a single coding agent, it is the strongest lever available.

Long context:

- dense 27B: generation ≈÷2 between short and 120k (13.5 to 15.1 t/s), prefill −30% between 16k and 120k [R, partly mixed hardware];
- Qwen3-Coder-30B-A3B Q4_K_M: **73 t/s at 128 tokens, 21 t/s at 16k** [R] ([DByte308](https://github.com/DByte308/MI50)). This sharp drop should be checked on these cards with an agent working at 16–32k;
- turbo3 KV: 3.3× more context for −18% generation [A].

---

## What the forums say: good at generation, weak at prefill, expensive in 2026

### Consensus

- **Good generation, weak prefill.** A 20,000-token prompt takes ≈18 s on an MI50 versus ≈4 s on a recent R9700 [R] ([OpenClaw](https://openclawdc.com/blog/amd-mi50-32gb-local-llm/)). The article's title: "Only as a Hobby".
- **Rising prices.** ≈$75 to $150 in 2024, **$400 to $500 in 2026** [R] ([Reddit mirror](https://reddit.sentinel-team.org/posts/1t0b0ie/snapshots/2026-05-02T03%3A06%3A21.60641Z), [TechEnclave](https://techenclave.com/t/sourcing-mi50-32gb-in-india/419230)). The "best value" argument no longer holds for a new purchase, but cards already owned are a different matter.
- **Dominant use**: llama.cpp and forks (often via llama-swap), community Ollama. One L1T user tested more than 170 GGUF models against about a dozen in safetensors ([L1T](https://forum.level1techs.com/t/radeon-vii-in-2026-uuughhh/245630?page=2)).
- **Cultural difference**: the Chinese community organized around vLLM (TP, concurrency); the Western community around single-stream llama.cpp with speculation ([Bilibili](https://www.bilibili.com/opus/1062709216989413379)).

### Hardware practices

- **Cooling**: passive card. 100 °C in ≈60 s under load, then throttling from 1725 to ≈930 MHz [M] ([DByte308](https://github.com/DByte308/MI50)). A fan ramp from 45 to 70 °C holds 1600 to 1900 MHz without throttling ([wtarreau](http://wtarreau.blogspot.com/2025/12/amd-radeon-instinct-mi50-32gb-best-ai.html)). Printable shrouds exist ([Printables](https://www.printables.com/model/1413245-amd-radeon-instinct-mi50-triple-slim-80mm-fan-shro)).
- **Power draw**: ≈17 to 20 W per card at idle; a 2× MI50 machine rarely exceeds 500 W at the wall [M].
- **BIOS**: Above 4G Decoding is mandatory, otherwise the machine does not boot.
- **VM passthrough**: requires vendor-reset ([L1T](https://forum.level1techs.com/t/radeon-instinct-mi50-and-ollama-in-a-mv/239906)).
- **VBIOS flashing**: risky (signatures, no POST, a CH341 programmer sometimes needed) ([TechPowerUp](https://www.techpowerup.com/forums/threads/help-flash-mi50-to-radeon-vii-pro.329623)). Not flashing, as on this rig, also avoids the ECC trap.
- Kernel trap seen on another AMD card: `amdgpu.sched_policy=2` causes hangs. Stay at 1 ([TheRock #5793](https://github.com/ROCm/TheRock/issues/5793)).

### Negative opinions worth taking seriously

- "Real PITA", "not great at all for most AI/LLM": only llama.cpp and bare PyTorch work, Triton fails [R] ([TechEnclave](https://techenclave.com/t/sourcing-mi50-32gb-in-india/419230)).
- Tarreau fears that AMD's abandonment will turn these cards into "e-waste". A commenter replies that the community keeps gfx906 alive ([wtarreau](http://wtarreau.blogspot.com/2025/12/amd-radeon-instinct-mi50-32gb-best-ai.html)).

---

## Leads found outside the initial list

- **tp906-engine** (`skyne98/tp906-engine`): tensor-parallel engine dedicated to gfx906. Qwen3.5 4B Q8_0 in TP2: 1405 / 89 t/s [A] ([Docker Hub](https://hub.docker.com/r/skyne98/tp906-engine)). No information on maintenance.
- **"reinstinct" engine**: origin of the repack and DPP reductions ported into furnace (cited by sixvolts, no link).
- **mx-llama.cpp and its derivatives** (exabit-io, Martin1701): tensor parallel and MTP for gfx906.
- **sixvolts furnace**: the best lead for K-quants.
- **KIntegrated and David-Lzy**: 2026 vLLM forks.
- **xxdoman/ollama-mi50**: recent Ollama for gfx906.
- **sramecc fix** (Intermountainh8ter).
- **TurboQuant turbo3** (≈3.5-bit KV) and **DFlash2** (drafter).
- **Lemonade gfx90X-dcgpu**, **llama-swap-gfx906**.
- **Qwen3-Coder-Next 80B-A3B** as a 2–4 card candidate.
- **Wiki arkprojects.space/wiki/AMD_GFX906**, linked by mixa3607, not read.
- Sources to read from another network: [diegostrebel.com "Benchmarking the Instinct MI50 in 2026"](https://diegostrebel.com/posts/mi50_benchmarks/) (403 during the research), and the Reddit thread "[2x MI50 32GB quant speed comparison v2](https://www.reddit.com/r/LocalLLaMA/comments/1rmt315/2x_mi50_32gb_quant_speed_comparison_version_2/)" (ROCm vs Vulkan, not retrieved).

---

## Contradictory sources at a glance

| Topic | Position A | Position B | Reading of this report |
|---|---|---|---|
| Official support | AMD matrix: gfx906 absent ([AMD](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html)) | "AMD added the MI50 back" ([L1T](https://forum.level1techs.com/t/radeon-vii-in-2026-uuughhh/245630?page=2)) | AMD's documentation prevails. |
| Best ROCm version | 6.1 (milpster) | 6.4 (nullkalahar), 7.1.1 (L1T), 7.14 (mixa, KIntegrated) | Not settled. To be measured. |
| Vulkan vs ROCm | Vulkan ≥ ROCm on Llama 2 7B; better prefill on 2× MI50 MoE | ROCm ×8.7 on Gemma MoE; "never use Vulkan" (DByte308) | Depends on model and commit. The 0.8 t/s on this rig is outside every range. |
| A3B MoE ceiling | ≈70 t/s (DByte308) | 87–94 t/s (PR, furnace, the author) | The Q4_0 format makes the difference. |
| Fork vs upstream | ≈parity (Tarreau, Dec. 2025) | +30 to +37% prefill (Exabit, Sept. 2026) | Both are true at their date. |
| ROCWMMA FA | Enable it (nullkalahar) | Do not enable, no WMMA (machinesoftworks) | The hardware sides with B. |
| RPC CVE fix | "Patched: none" (GitHub) | Fixed in b8492 (Qi-An-Xin) | Treat RPC as unsafe. |
| Last official Ollama | 0.12.3 | 0.12.4 | Irrelevant with community images. |
| MTP | +30 to +54% single-stream | No gain on 1 card (one user); loss at 4 users | Helps solo, hurts under load. |
| Quantized MoE under vLLM | "Significantly slow" (nlzy, 2025) | 66 t/s, 193 t/s at N=8 (KIntegrated, 2026) | Dated advice. |
| mobydick stack | ROCm 6.3.4 (README) | ROCm 7.2.1 (Reddit post) | Several stacks possible. |
| Date of the 8× MI50 post | "mid-December 2025" (mirror) | Cites the "Qwen3.6" model | Date it as "no later than May 2026". |

---

## Measurements to run before deciding

These tests fill the gaps the literature does not cover for this configuration. They are not projects, only measurements.

1. **72B in Q4_0** on upstream llama.cpp, 2 cards `-sm layer`. Checks the ≈12 t/s estimate [D].
2. **72B with `-sm tensor`** using `mxxm/mx-llama.cpp:gfx906` (f16 KV). The only llama.cpp path that might approach vLLM TP2. No published figures.
3. **Qwen3-30B-A3B at 16k and 32k context.** Checks the drop to 21 t/s at 16k reported by DByte308.
4. **vLLM with short inputs and prefix caching enabled**, at c1, c4 and c8. Separates the prefill effect from the decode effect.
5. **llama-server `-np 4` and `-np 8`** on the MoE, with a shared system prompt. Compare with vLLM on the same scenario. Watch out for the 9-slot threshold upstream.
6. **MTP or draft** on the dense 27B in single-stream.
7. **sramecc and PCIe width check** (`rocminfo`, `lspci -vv`), once.

---

## Decision matrix by use case

The decision is up to the reader. Each row gives two defensible options, what to avoid, and the criterion that tips the balance.

| Use case | Option A | Option B | Avoid | What tips the balance | Certainty |
|---|---|---|---|---|---|
| **Coding agent 14–30B, single-stream** | **Upstream llama.cpp** (`mixa3607/llama.cpp-gfx906`), A3B MoE in **Q4_0**, `-fa on`, HIP graphs, `-ub 2048` or more, 1 card | **mx-llama.cpp** or **furnace** fork for prefill (+12 to +37% announced) | Vulkan; K-quants without furnace; vLLM (no single-stream gain) | The 94 t/s measured on this rig is already at the published generation ceiling. A fork is only justified if prefill (TTFT) is a problem. | High for generation, medium for the forks' prefill gain |
| **Coding agent, long context (≥32k)** | llama.cpp + **MTP / draft** on dense 27B, or A3B MoE with per-slot cache reuse | **Martin1701** fork (single card, optimized for 64k) or turbo3 if context runs short | KV q8_0 when not needed (−15 to −36% decode) | Measurement #3: if the MoE collapses at 16k, the dense 27B + MTP becomes competitive again. | Medium (little pure-MI50 data) |
| **72B dense, single-stream** | **vLLM TP2 AWQ** (ai-infos/mobydick): ≈22 t/s measured on this rig | llama.cpp **`-sm tensor`** (mx-llama.cpp), not measured *(superseded by 04: upstream `-sm tensor` measured at 16.77 t/s, mx-llama.cpp at 14.47)* | llama.cpp `-sm layer` in Q4_K_M (10 t/s); Vulkan; `-sm row` | TP makes both cards work in parallel. Layer split cannot. Q4_0 in layer mode will only give about +20%. | High for vLLM (measured on this rig), low for `-sm tensor` |
| **72B, multiple requests** | vLLM TP2 with **prefix caching**, short inputs, moderate `max-num-seqs` | Replace the dense 72B with an **80B-A3B MoE or gpt-oss-120b** under llama.cpp (not measured on MI50) | Firing 8 parallel 2k-token requests without cache (prefill saturated, only ×1.6) | Prefill at ≈200 t/s is the ceiling. The fix is fewer tokens to prefill, not more requests. | Medium |
| **Multi-agent concurrency (4–8 agents, 14–35B models)** | **vLLM, 1 card per model** (TP1), AWQ, prefix caching. KIntegrated: ×2.9 at N=8 on MoE | **llama-server `-np 4` to 8** on Q4_0 MoE, one instance per card (129 t/s at 4 requests published on 2 cards) | MTP under load; 9 slots upstream; TP for models that fit on one card | One instance per card avoids all PCIe traffic. vLLM wins if the agents share a long system prompt. | Medium (the c8 measurements on this rig show a prefill bottleneck) |
| **Light roles: router, embeddings, draft** | **RTX 3060 as a separate CUDA node** (proposal [D]) | Small model on one MI50 next to the coder | **Exposed llama.cpp RPC** (CVE 9.8); Vulkan mixing for only +19% capacity | The 3060 contributes more through independence than as capacity added to the MI50s. | Low (no published data on this mix) |
| **ROCm stack** | **mixa3607 TheRock 7.14** in a container; `iommu=pt` on the host for TP | Older 6.3.3 or 6.4.4 tags if an engine requires it (ai-infos vLLM on 6.3.4) | DKMS driver on the host (KIntegrated) if a host with no AMD packages matters; mixing AMD and gfx906 repositories | The "ROCm in containers only" constraint is already the recommended practice. Docker costs only 0.2 to 0.3%. | High |
| **Engines to set aside for now** | — | — | SGLang (degenerate output), ExLlama, TGI, Aphrodite, ktransformers, official Ollama | No bf16, no official Triton, no hipBLASLt on gfx906 | High for SGLang, medium for the others (absence of sources) |
