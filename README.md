# 2x AMD Instinct MI50 (gfx906) + RTX 3060 in 2026: engines, models, measurements

A home-lab write-up, as of **11 October 2026**: what still runs on the MI50 (gfx906) in 2026, which 2026 open-weight models fit in 64 GiB (2x MI50, ≈68.7 GB) or ≈75.6 GiB (2x MI50 + RTX 3060, ≈81.2 GB), and what one such rig actually measured.

**Status: first public version, 11 October 2026.** Corrections are welcome: open an issue with a source or a measurement.

## Contents

| File | What it is | Kind |
|---|---|---|
| [01-inference-engines-mi50-gfx906.md](01-inference-engines-mi50-gfx906.md) | Survey of inference engines on gfx906: llama.cpp and its forks, vLLM forks, ROCm in containers, multi-GPU, quantization formats | Web research |
| [02-which-2026-models-fit-64-76gb.md](02-which-2026-models-fit-64-76gb.md) | Which 2026 open models fit in 64 / 76 GB, and whether "bigger = smarter" holds there | Web research |
| [03-compressed-chinese-flagships.md](03-compressed-chinese-flagships.md) | Can 1–2 bit Chinese flagship MoEs (DeepSeek V4 Flash, GLM-5.3-Flash, MiniMax…) beat a dense 27B at Q8 in that memory? | Web research |
| [04-measurements-2x-mi50-plus-3060.md](04-measurements-2x-mi50-plus-3060.md) | Measured results on the rig: `-sm tensor` without P2P, 3-card HIP+CUDA build, vLLM TP2, thermals at 200 W vs 180 W | Own measurements |

## Key findings

- **Upstream llama.cpp `-sm tensor` works on 2x MI50 over PCIe without P2P**, and beats `-sm layer`: 27B 32.3 vs 24.0 t/s; 72B Q4_0 16.77 vs 11.36 t/s (200 W cap). See 04.
- **One llama.cpp binary with HIP + CUDA back-ends** (`GGML_BACKEND_DL`) drives 2x MI50 + RTX 3060 together. Cost: up to ~7 % generation on the tested models. See 04.
- **vLLM (gfx906 fork) 72B AWQ TP2** gives the best single-stream decode measured here (~22 t/s), with a ~10 s time to first token on a 2k prompt. See 04.
- **At 200 W, sustained dense load reached 90 °C junction** on this rig even with the blower at full speed; **180 W cost 0.4–2.2 % generation and 2–4 % prefill**, with no cut in three short 180 W runs (≤ 6 min each) — **but a continuous 72B server load at 180 W was still cut at 90 °C after ~10 min on the hotter card** (one run). See 04.
- **Format matters as much as the engine on gfx906**: Q4_0 / Q4_1 / Q8_0 have dedicated kernels; K-quants and IQ formats take slower paths unless a fork repacks them. See 01.
- **In this memory window, a recent dense 27B at Q8 beats every 2026 MoE that fits** on independent indices; compressed Chinese flagships do not fit in a standard-llama.cpp format. See 02 and 03 — and note what they do *not* measure (French, long-form writing).

## How to read the numbers

Research files tag every figure:

- **[M]** measured by a third party with a readable protocol;
- **[A]** announced by the author of a model or fork about their own work;
- **[R]** reported, rounded or anecdotal (forum posts, mirrors);
- **[D]** a deduction or calculation by this write-up.

A tag says where a number comes from, not whether it is true.

## Rig

- 2x AMD Instinct MI50 32 GB (gfx906, stock VBIOS), 1x RTX 3060 12 GB, 35 GB system RAM, older X79-era platform, no GPU P2P.
- Ubuntu host. **ROCm and CUDA only inside Docker containers**; only the kernel drivers on the host.

## Limits

- Web research done on 11 October 2026; projects and links move fast.
- reddit.com was read through a mirror; some leaderboards could not be read (JavaScript pages).
- Measurements come from a single rig and mostly a single run per configuration.

## License

Text: CC BY 4.0. Cited sources belong to their authors.
