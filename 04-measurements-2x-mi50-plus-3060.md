# Measurements: 2x AMD Instinct MI50 32 GB (gfx906) + RTX 3060 12 GB

**Status: first public version (11 Oct 2026). Numbers cross-checked against the raw benchmark outputs; single rig, mostly one run per configuration.**

All numbers below come from one measurement session (11 Oct 2026), plus a few reference runs from 09 Oct 2026 where stated. One run per configuration unless a repetition count is given. Speeds are in tokens per second (t/s) unless noted; llama-bench values are mean ± standard deviation as reported by llama-bench.

**Power cap:** unless a row says otherwise, every MI50 number was measured with the MI50 power cap (`power1_cap`) at **200 W per card**. The 180 W section is explicitly labelled.

---

## 1. Hardware and software as tested

### Hardware

- Older X79-era desktop platform. No GPU peer-to-peer (P2P) between the cards; IOMMU not enabled.
- 2x AMD Instinct MI50 32 GB (gfx906), blower-style cards with their own turbine. Each reports 32752 MiB to llama.cpp.
- 1x NVIDIA GeForce RTX 3060 12 GB (compute capability 8.6), reports 11909 MiB to llama.cpp.
- Closed tower case with intake and exhaust case fans (6 intakes / 4 exhausts). Case configurations used during the session are noted per run (closed with BIOS fan curve, open, or closed with top exhaust forced to 100 % and intakes at ~70 %).
- MI50 turbine setpoint was 100 % on all loaded runs.

### Software

- **No ROCm or CUDA installed on the host.** Everything ran in Docker containers. Host NVIDIA driver 580.178.04 (advertises CUDA 13.0 max), NVIDIA container runtime present.
- **llama.cpp, upstream**, commit `88dcc460d628698bb8305b98c200c34f1edfdc04` (build b11483). Two images built from it:
  - **HIP-only image** ("L0"): used for most 2x MI50 llama-bench runs.
  - **HIP + CUDA image** ("P1"): same commit, both back-ends in one build (details in section 4). Used for the 3-card runs, llama-server runs and some 2x MI50 runs.
  - Base image for both: `rocm/dev-ubuntu-22.04`, pinned by digest, ROCm 6.3.3-74, gcc 11.4.
- **mx-llama.cpp** ("L3"), a gfx906-oriented llama.cpp fork, compared in tensor mode only.
- **vLLM 0.12 gfx906 fork** (nalanzeyu image), used for the 72B AWQ tensor-parallel runs.
- Third-party image also tried: `mixa3607/llama.cpp-gfx906:v0.6.0-rocm-7.14-cuda-13.4.2` (failed on the 3060, see section 7).

### Common llama.cpp settings

- `-ngl 99 -fa 1 -ctk q8_0 -ctv q8_0` everywhere (KV cache q8_0).
- llama-bench: `-p 512,4096 -n 128`, `-r 5` (`-r 2` for the 72B models to limit heat).
- Default `-ts` (split proportional to free memory).

### Models

| Label | Model | Quant |
|---|---|---|
| 14B | Qwen2.5-Coder-14B-Instruct | Q4_0 (8.5 GB) |
| 27B | 27B dense model | Q4_0 |
| 30B-A3B | 30B-A3B MoE (Qwen3 family) | Q4_0 |
| 72B | Qwen2.5-72B-Instruct | Q4_0, Q4_K_M (GGUF); AWQ (vLLM) |
| Draft | Qwen2.5-0.5B-Instruct | Q8_0 (531 MB) |

### Server benchmark contract

Server runs (vLLM and llama-server) use `vllm bench serve` with random data: **2,048 input tokens, 256 output tokens forced (`--ignore-eos`)**, a separate warm-up step, one seed per concurrency level, **prefix caching disabled** unless the row is a prefix-cache test. "Output throughput" = generated tokens / total duration of the level.

### Safety guard

A watchdog polled the cards every 2 s and killed the load if **junction temperature reached 90 °C**, if the power cap read back differed from the expected cap, or if the fan regulator was in fault/absent/stale (fail-closed). Many runs below were cut by this guard. For reference, the fan regulator's thresholds (derived from the card's critical temperature) are 85 °C alert, 95 °C fault.

---

## 2. Dense models on 2x MI50: `-sm layer` vs `-sm tensor` (200 W)

llama-bench, t/s, mean ± std. All on 2x MI50, no P2P, KV q8_0.

| Model | Engine | Split | pp512 | pp4096 | tg128 |
|---|---|---|---|---|---|
| 27B Q4_0 | upstream (L0) | layer | 293 ± 4 | 479 ± 1 | 24.0 ± 0.1 |
| 27B Q4_0 | upstream (L0) | **tensor** | 462 ± 2 | 443 ± 1 | **32.3 ± 2.1** |
| 27B Q4_0 | mx-llama.cpp (L3) | tensor | 358 ± 0 | 342 ± 0 | 25.2 ± 0.3 |
| 72B Q4_K_M | upstream (L0) | layer | 94.9 ± 0.5 | 155.9 ± 0.2 | 10.03 ± 0.02 |
| 72B Q4_0 | upstream (L0) | layer | 120.6 ± 1.3 | 197.1 ± 0.3 | 11.36 ± 0.01 |
| 72B Q4_0 | upstream (L0) | **tensor** | 207.7 ± 0.5 | 195.9 ± 1.1 | **16.77 ± 0.37** |
| 72B Q4_0 | mx-llama.cpp (L3) | tensor | 155.9 ± 0.0 | 149.3 ± 0.0 | 14.47 ± 0.10 |

What this shows:

- **Tensor beats layer in generation**: 27B 32.3 vs 24.0 t/s (+35 %), 72B Q4_0 16.8 vs 11.4 t/s (+48 %). Tensor also improves pp512; pp4096 stays roughly equal (27B −7 %, 72B −1 %).
- **It works without P2P and with a q8_0 KV cache** (`type_k: q8_0` confirmed in the llama-bench JSON). We had been told f16 KV was required for `-sm tensor`; on this build it is not.
- **mx-llama.cpp in tensor mode was slower than upstream** on all three test models (72B: 14.5 vs 16.8 t/s). It ran without P2P and without crashing.
- **Q4_0 vs Q4_K_M** (72B, layer): tg +13 % (11.36 vs 10.03), pp +27 %.
- **Reproducibility**: the 72B Q4_K_M layer run (94.9 / 155.9 / 10.03) matches a run from 09 Oct (95.0 / 155.8 / 10.01) within 0.3 %.

**27B on a single MI50 could not be measured at 200 W**: it hit 90 °C and was cut on both cards (see section 8). Its only 180 W result is in section 9.

---

## 3. MoE 30B-A3B: one instance per card (200 W)

llama-bench, t/s, mean ± std, `-r 5`.

| Engine | Mode | pp512 | pp4096 | tg128 |
|---|---|---|---|---|
| upstream (L0) | **1 card** | 1198 ± 164 | 1230 ± 2 | **92.8 ± 0.6** |
| upstream (L0) | layer, 2 cards | 1370 ± 73 | 1885 ± 37 | 73.0 ± 0.3 |
| upstream (L0) | tensor, 2 cards | 1577 ± 109 | 1407 ± 4 | 71.8 ± 3.6 |
| mx-llama.cpp (L3) | tensor, 2 cards | 1236 ± 10 | 1085 ± 1 | 42.2 ± 1.7 |

**Generation is fastest on one card**: 92.8 t/s vs 73.0 (layer) and 71.8 (tensor) on two. For this model, running one instance per card is the better use of 2x MI50. The MoE runs were also the only sustained loads that stayed well clear of 90 °C (max 77 °C on one card).

llama-server, 30B-A3B Q4_0, **one MI50**, `-np 4`, server contract (prefix cache off):

| Concurrency | Requests | Median TTFT | Median TPOT | Output throughput |
|---|---|---|---|---|
| c1 | 4 | 1,501 ms | 13.79 ms | 50.98 t/s |
| c4 | 8 | 3,851 ms (P99 6,187 ms) | 38.37 ms | **75.15 t/s** |

0 errors at c4. This run happened while the case was being opened (transition).

---

## 4. Three cards in one llama.cpp process (2x MI50 + RTX 3060)

### The build

One llama.cpp process, one container, sees all three cards (`CUDA0` = 3060, `ROCm0` and `ROCm1` = MI50) and splits a model by layers across them. No RPC needed.

The working image was built from upstream llama.cpp at the same commit as above:

```
cmake -DGGML_BACKEND_DL=ON -DBUILD_SHARED_LIBS=ON -DGGML_NATIVE=OFF \
      -DGGML_CPU_ALL_VARIANTS=ON \
      -DGGML_HIP=ON -DGPU_TARGETS=gfx906 \
      -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=86
```

- Base: `rocm/dev-ubuntu-22.04` (ROCm 6.3.3-74), pinned by digest.
- CUDA toolkit **12.8** installed **inside the container** from NVIDIA's repo, minimal packages: `cuda-nvcc-12-8 12.8.93-1`, `cuda-cudart-dev-12-8 12.8.90-1`, `libcublas-dev-12-8 12.8.5.5-1`, `cuda-driver-dev`, `cuda-cccl`. 12.8 rather than 13.x because the host driver (580.178.04) advertises CUDA 13.0 max.
- gcc 11.4 (Ubuntu 22.04), so the "nvcc refuses GCC 15" problem does not apply.
- Resulting image: 41.7 GB. Build time: 1411 s with 6 jobs.
- `HIP_VISIBLE_DEVICES` was not needed; device order was `CUDA0` then `ROCm0/1`; `-ts` left at default. `-ot` across back-ends was not tested.

### Proof the 3 cards worked (14B Q4_0, `-sm layer`, 3 cards, 200 W on the MI50)

llama-bench `-r 3`: **pp512 = 647.3 ± 26.1 t/s, tg128 = 40.2 ± 0.1 t/s**.

| Card | Max power | Max temperature | Memory |
|---|---|---|---|
| RTX 3060 | 60.4 W (17 W idle) | 46 °C (GPU temp) | 1610 MiB max (16 MiB idle) |
| MI50 #1 | 141 W | 47 °C junction | — |
| MI50 #2 | 201 W | 54 °C junction | — |

The 3060 got ~1.6 GiB of the 8.5 GB model, consistent with a split proportional to free memory (11.8 GiB vs ≈ 24-25 GiB free per MI50).

For scale: a 09 Oct run of the same model on **one** MI50 gave tg ≈ 48 t/s. So 3 cards at 40.2 t/s is **slower than one MI50** for a model that fits on one card — expected with layer split (cards work one after another, plus inter-card transfers). The third card only helps to **fit** a bigger model.

### Config A (2x MI50) vs config B (3 cards), layer split (200 W)

llama-bench, t/s, mean ± std. B always uses the HIP+CUDA image. Case closed, top exhaust 100 %, intakes ~70 %.

| Model | Config | Image | pp512 | pp4096 | tg128 | Reps |
|---|---|---|---|---|---|---|
| 30B-A3B Q4_0 | A: 2x MI50 | HIP-only | 1370.1 ± 73.4 | 1884.5 ± 36.8 | 73.0 ± 0.3 | 5 |
| 30B-A3B Q4_0 | **B: 3 cards** | HIP+CUDA | 1422.6 ± 82.0 | 1854.9 ± 1.7 | **76.7 ± 0.4** | 5 |
| 27B Q4_0 | A: 2x MI50 | HIP-only | 292.6 ± 3.5 | 478.5 ± 1.2 | 24.0 ± 0.05 | 5 |
| 27B Q4_0 | **B: 3 cards** | HIP+CUDA | 287.2 ± 12.5 | 478.9 ± 3.3 | **22.6 ± 0.05** | 5 |
| 72B Q4_K_M | A: 2x MI50 | **HIP+CUDA** (same binary) | 94.6 ± 0.8 | 156.1 ± 0.1 | 10.01 ± 0.04 | 2 |
| 72B Q4_K_M | **B: 3 cards** | HIP+CUDA | 103.0 ± 3.0 | 165.5 ± 0.1 | **9.30 ± 0.06** | 2 |
| 72B Q4_0 | A: 2x MI50 | HIP-only | 120.6 ± 1.4 | 197.1 ± 0.3 | 11.36 ± 0.01 | 2 |
| 72B Q4_0 | **B: 3 cards** | HIP+CUDA | 131.1 ± 1.5 | 201.6 ± 2.8 | **10.55 ± 0.01** | 2 |

Binary check: 72B Q4_K_M on 2x MI50 gives 94.9 / 155.9 / 10.03 with the HIP-only image (`-r 5`) and 94.6 / 156.1 / 10.01 with the HIP+CUDA image. The two binaries are equivalent on that model, so that A/B pair is clean.

### What the 3060 costs (72B Q4_K_M, same binary)

- tg128: 10.01 → 9.30 t/s, **−7.1 %**.
- pp512: +8.9 %; pp4096: +6.0 %.
- The MI50s ran a little cooler: 82/81 °C junction max (B) vs 85/85 °C (A).
- Likely explanation (our inference, not measured): the 3060 takes about 1/6 of the layers, which offloads the MI50s during prefill; in generation its lower memory bandwidth (≈ 360 GB/s vs ≈ 1 TB/s for an MI50) and one more PCIe hop cost speed.

### Power and temperature in config B (200 W cap on the MI50)

"Mean under load" = samples ≥ 100 W (2 s sampling). 3060 mean is over the run window.

| Run | MI50 #1: mean / max W / Jmax | MI50 #2: mean / max W / Jmax | 3060: mean / max W / Tmax |
|---|---|---|---|
| A 72B Q4_K_M (HIP+CUDA image, 2 cards) | 197.7 / 257 / 85 °C | 198.9 / 269 / 85 °C | 17.4 / 43.8 / 46 °C |
| B 72B Q4_K_M | 199.7 / 255 / 82 °C | 197.2 / 267 / 81 °C | 43.1 / 85.0 / 52 °C |
| B 72B Q4_0 | 188.2 / 231 / 82 °C | 198.9 / 242 / 83 °C | 47.1 / 87.1 / 47 °C |
| B 30B-A3B | 199.0 / 220 / 62 °C | 195.3 / 202 / 66 °C | 36.2 / 63.7 / 38 °C |
| B 27B | 195.6 / 216 / 78 °C | 186.1 / 243 / 78 °C | 51.4 / 69.3 / 50 °C |

No guard trip during the config B runs.

**Bottom line:** for models that already fit on 2x MI50, adding the 3060 changes speed by a few percent. The real point of the 3060 (+12 GB, i.e. a 64-76 GB model) **was not measured**: no model of that size was available.

---

## 5. Serving the 72B: vLLM AWQ TP2 vs llama-server tensor (200 W)

Server contract (2,048 in / 256 out, prefix cache off).

| Engine | Load time | Level | Requests | Median TTFT | Median TPOT | Output throughput |
|---|---|---|---|---|---|---|
| vLLM 0.12 gfx906, 72B AWQ, TP2 (run 1) | 511 s | warm-up c1 | 2 | 10,245 ms | **44.75 ms** | 11.82 t/s |
| | | c1 | — | **cut at 90 °C** | | |
| vLLM 0.12 gfx906, 72B AWQ, TP2 (run 2) | 568 s | c1 | 4 | **10,361 ms** | **44.68 ms** | **11.77 t/s** |
| | | c4 | 1/8 | **cut at 90 °C** (both cards) | | |
| llama-server, 72B Q4_0, `-sm tensor`, `-np 4` | 163 s | warm-up c1 | 2 | 10,145 ms | 61.47 ms | 9.91 t/s |
| | | c1 | — | **cut at 90 °C** | | |

- In single-stream generation, **vLLM TPOT 44.7 ms (22.3 t/s) vs llama-server tensor 61.5 ms (16.3 t/s)**. Prefill is the same on both sides: TTFT ≈ 10.1-10.4 s for 2,048 tokens.
- vLLM TPOT (44.7 ms) is identical to a run from 09 Oct (44.8 ms): stable day to day.
- vLLM command: `vllm serve <Qwen2.5-72B-Instruct-AWQ> --tensor-parallel-size 2 --max-model-len 8192 --gpu-memory-utilization 0.90 --dtype float16 --no-enable-prefix-caching`, container started with `--shm-size=16g` and **without `--ipc=host`** (private IPC). TP2 worked that way.
- Concurrency (c4) on the 72B could not be measured at 200 W: cut at 90 °C.

Power during these runs (mean under load / max W / junction max):

| Run | MI50 #1 | MI50 #2 |
|---|---|---|
| vLLM 72B, run 1 | 191 / 253 / 89 °C | 199 / 256 / **90 °C** |
| vLLM 72B, run 2 | 190 / **315** / **90 °C** | 198 / **307** / **90 °C** |
| llama-server 72B tensor | 190 / 241 / 89 °C | 190 / 228 / **90 °C** |
| llama-server 30B-A3B, 1 card | 161 / 219 / 77 °C | idle |

---

## 6. Speculative decoding and prefix caching (200 W)

### Speculative decoding (llama-server, 72B Q4_0, `-sm tensor`)

Server: `-sm tensor -fa 1 -ctk q8_0 -ctv q8_0 -c 4096 -np 1 -dev ROCm0,ROCm1`. Draft: Qwen2.5-0.5B-Instruct Q8_0, 3 draft tokens per step. Test: 3 identical sequential requests on a real French text (1,016 prompt tokens), `temperature 0`, `n_predict 256`, `cache_prompt false`; speed read from llama-server `timings`.

| Draft setup | tg (t/s), req. 1 / 2 / 3 | Time per request | Accepted / drafted |
|---|---|---|---|
| none | 16.01 / 16.08 / 16.47 | 20.6-21.4 s | — |
| `-md … -devd CUDA0`, **no `--spec-type`** | 16.45 / 16.59 / 16.78 | 20.2-20.6 s | none: **draft loaded but not used** |
| draft on 3060 + `--spec-type draft-simple` (see note) | **23.17 / 24.21 / 24.27** | 15.5-16.1 s | 170 / 252 (67.5 %) |
| draft on MI50 (`-devd ROCm0 --spec-type draft-simple`) | **22.67 / 23.57 / 23.72** | 15.8-16.4 s | 170 / 252 (67.5 %) |

- **16.0-16.5 → 22.7-24.3 t/s (+40 to +47 %)**, 67 % of drafted tokens accepted.
- Works **as well with the draft on an MI50** as on the 3060, so a 2x MI50 setup does not need the 3060 for this.
- Output text starts identically with and without the draft, as expected at temperature 0.
- Note on the "draft on 3060" row: another model was resident on the 3060 during the run (see section 10). A second-by-second review of the logs shows its only computation ended about 28 s before the measured requests started, so it held memory but was idle while we measured. The MI50 draft row has no such caveat and supports the same conclusion.
- MI50 mean power under load: no draft 190 and 192 W (junction 85 °C); draft on 3060 196 and 202 W (80-81 °C); draft on MI50 183 and 195 W (80-82 °C). The 3060 drew 45.8 W on average with the draft.

With speculative decoding, llama-server (22.7-24.3 t/s) is on par with vLLM (22.3 t/s) in single-stream on the 72B.

### Prefix caching (2,000-token shared prefix)

Dataset: `--random-prefix-len 2000 --random-input-len 48 --random-output-len 256 --ignore-eos`. Steady state (the client's initial test request warms the cache).

vLLM 0.12 gfx906, 72B AWQ TP2:

| Prefix cache | Level | Requests | Median TTFT | P99 TTFT | Median TPOT | Output throughput |
|---|---|---|---|---|---|---|
| ON | warm-up c1 | 2 | 200 ms | — | 44.96 ms | 21.94 t/s |
| ON | c1 | 4 | **194 ms** | 198 ms | 44.98 ms | **21.90 t/s** |
| ON | c4 | 4 | **927 ms** | 929 ms | 63.25 ms | **60.03 t/s** |
| OFF | warm-up c1 | 2 | **10,198 ms** | — | 44.95 ms | **11.82 t/s** |
| OFF | c1 | — | **cut at 90 °C** | | | |

vLLM log: `Prefix cache hit rate: 76.3%`. Mean power under load: ON 178 and 185 W, junction max 81 and 82 °C; OFF 193 and 200 W, junction max 88 and **90 °C**.

llama-server, 30B-A3B Q4_0, one MI50, `-np 4`:

| Prompt cache | Level | Requests | Median TTFT | P99 TTFT | Median TPOT | Output throughput |
|---|---|---|---|---|---|---|
| ON (`--cache-prompt`, 8,192 MiB RAM cache) | c1 | 4 | **50 ms** | 51 ms | 13.78 ms | **71.87 t/s** |
| ON | c4 | 8 | **864 ms** | 4,792 ms | 29.80 ms | **102.32 t/s** |
| OFF (`--no-cache-prompt -cram 0`) | c1 | 4 | 1,559 ms | 1,566 ms | 13.85 ms | 50.23 t/s |
| OFF | c4 | 8 | 3,988 ms | 6,368 ms | 38.61 ms | 73.92 t/s |

Gain: c1 TTFT ÷ 31, throughput +43 %; c4 median TTFT ÷ 4.6, throughput +38 %. Junction max 73 °C (ON) vs 79 °C (OFF).

⚠️ **Correction (found later the same evening):** with vLLM 0.12's `random` dataset, `--random-prefix-len 2000 --random-input-len 48` produces **identical prompts** (the 2,000 random prefix tokens re-encode to more than 2,048 tokens and the truncation cuts the 48 variable ones; verified: 1 distinct prompt out of 4). The gains above were therefore measured on **repeated identical requests**. With a `prefix_repetition` dataset (2,000 shared + 48 truly distinct tokens), llama-server on the 72B goes from 10.5 s to 0.42 s of prefill per request: **÷25** (180 W, section 9).

Overall (identical-prompt caveat above): TTFT divided by 30-50 at c1, throughput +38 to +86 %, and **less heat** — the only time the 72B on vLLM held both c1 and c4 at 200 W was with the prefix cache on (81-82 °C).

---

## 7. Pitfalls found

- **llama.cpp `-md` alone is no longer enough** (build b11483). The draft model loads, but nothing is drafted unless you add `--spec-type draft-simple`. The log gives no warning.
- **The mixa3607 HIP+CUDA image does not run on an RTX 3060.** Its CUDA module only contains `sm_120a` (string found in the binary; we had no `cuobjdump` to inspect the cubins, so this is a strong hint, not proof). The model loads, then the first compute fails: `CUDA error: no kernel image is available for execution on the device` (exit code 139). Building our own image with `-DCMAKE_CUDA_ARCHITECTURES=86` fixed it.
- **A background service loaded a model on the 3060 mid-run.** Another service on the host loaded an 8B model (8,192 MiB) on the 3060 via Ollama. It stayed resident (idle) during the "draft on 3060" speculative run; nothing prevented two programs from sharing the card. If you share the box with other services, pin them away from the cards you benchmark and tell your monitoring about your test containers.
- **`-sm tensor` across different back-ends (HIP + CUDA) is not supported** (upstream docs; not tested here). Config B was layer split only.
- **Docker daemon became unresponsive twice** (once during the image build, once without any obvious cause). A latency check during a later model load (100 samples, max 0.05 s) did not reproduce it. Cause not established.

---

## 8. Thermal and power findings at 200 W

### Any sustained dense load reaches 90 °C junction

At 200 W, **every sustained dense load hit 90 °C junction within 1 to 3 minutes and was cut by the safety guard**: 27B on one card (on either card), vLLM 72B c1/c4, llama-server 72B. Only short runs (llama-bench, 2 to 6 min) and the MoE passed. The guard tripped 10 times on junction ≥ 90 °C during the session.

### Not fan-limited, and not case-airflow-limited

- MI50 turbine at max speed run after run: 4,623 to 4,703 rpm on 07, 08, 09 and 11 Oct. No drift. Setpoint 100 % on all loaded runs; regulator reported the turbine OK.
- Before the 200 W cap was set (08 Oct afternoon, max power 227 to 258 W), single-card runs reached 92-97 °C. The 200 W cap gained about 5 °C.
- One card alone runs hotter than a pair: with layer split each card waits for the other half the time (78 to 89 °C), alone it works without pause (88 to 92 °C).
- Day-to-day: 72B Q4_K_M layer on two cards was 85/84, 86/84, 87/86 °C on 09 Oct vs 88/87 and 89/88 °C (Q4_0) on 11 Oct. +1 to +3 °C, consistent with slightly warmer case air (system sensor 27-28 °C under load vs 24-25 °C idle). Real but secondary.

Case airflow test, 27B Q4_0, one card, 200 W. Times from the first sample ≥ 100 W:

| Case setup | Junction at start | → 85 °C | → 90 °C |
|---|---|---|---|
| closed, BIOS fan curve | 36 °C | 38 s | **70 s** |
| open | 30 °C | 44 s | **108 s** |
| closed, top exhaust 100 % + intakes ~70 %, cold start | 30 °C | 46 s | **90 s** |

Opening the case or forcing the case fans delays the 90 °C point by 20 to 40 s (plateau at 88 °C instead of a continuous climb) but does not prevent it. One run per setup: noise not measured, indicative only. A fourth run (forced fans, warm start at 45 °C: 36 s / 72 s) is invalid and excluded. Our reading: the limit is the **card's own blower**, not the case air.

### Mean power vs transient peaks

- Mean power under sustained load: **167 to 203 W per card depending on the run** (table below; 2 s sampling, samples ≥ 100 W). The 200 W cap holds **on average**, within 1.5 %.
- Peaks of 220 to 269 W are transients seen by `power1_input`. One isolated reading of **315 / 307 W** occurred during vLLM 72B at c4. The card regulates its average, not every instant.
- Mean power and peaks per llama-bench run (mean under load / max / junction max):

| Run (2x MI50 unless noted) | MI50 #1 | MI50 #2 |
|---|---|---|
| 30B-A3B, 1 card | 172 / 205 / 77 °C | idle |
| 30B-A3B layer | 190 / 216 / 73 | 200 / 218 / 67 |
| 30B-A3B tensor | 155 / 198 / 67 | 156 / 178 / 68 |
| 27B layer | 195 / 240 / 83 | 195 / 222 / 81 |
| 27B tensor | 177 / 231 / 81 | 180 / 239 / 82 |
| 72B Q4_K_M layer | 199 / 261 / 88 | 197 / 261 / 87 |
| 72B Q4_0 layer | 200 / 233 / 86 | 203 / 246 / 86 |
| 72B Q4_0 tensor | 191 / 226 / 87 | 185 / 229 / 88 |
| 72B Q4_0 tensor, mx-llama.cpp | 167 / 211 / 78 | 171 / 225 / 80 |

With alert at 85 °C and fault at 95 °C, sustained single-card load at 200 W lives permanently between alert and the 90 °C guard, with 3 to 7 °C margin before fault. Too thin for an agent running for hours.

---

## 9. 180 W cap: results

Cap set to 180 W on both MI50 and read back before, during (every 2 s) and after. Cold start for each run (junction < 40 °C, stable ±1 °C over 2 min). Case closed, top exhaust forced to 100 %, intakes ~70 %. The forced top exhaust expired 83 s before the end of the server run (T3), which ran its last 83 s on the BIOS curve. The guard never tripped at 180 W.

### T1 — 27B Q4_0, one MI50, llama-bench `-r 5`

| | 200 W, closed, BIOS curve | 200 W, open | 200 W, forced fans | **180 W, forced fans** |
|---|---|---|---|---|
| Junction at start | 36 °C | 30 °C | 30 °C | 31 °C |
| → 80 °C | 20 s | 26 s | 24 s | **46 s** |
| → 85 °C | 38 s | 44 s | 46 s | **110 s** |
| → 88 °C | 52 s | 68 s | 66 s | **never** |
| → 90 °C | 70 s (cut) | 108 s (cut) | 90 s (cut) | **never** |
| Junction max | 90 °C | 90 °C | 90 °C | **85 °C** |
| Mean W under load / max | 196 / 217 | 195 / 220 | 194 / 226 | **179 / 220** |
| Outcome | cut, no result | cut | cut | **complete (177.6 s)** |
| pp512 / pp4096 / tg128 (t/s) | — | — | — | **281.65 ± 2.91 / 272.63 ± 0.39 / 24.55 ± 0.04** |

No 200 W run completed on this card. The only complete 200 W reference is from 09 Oct on the **other** MI50, same image and settings: 293.01 / 283.14 / 25.10 t/s (junction max 90 °C). Delta at 180 W: **−3.9 % / −3.7 % / −2.2 %** (indicative: different cards).

### T2 — 72B Q4_0, `-sm tensor`, 2x MI50, llama-bench `-r 2`

| | 200 W (closed, BIOS curve) | **180 W (forced fans)** |
|---|---|---|
| Junction at start (#1 / #2) | 43 / 42 °C | 31 / 30 °C |
| → 80 °C | 36 / 32 s | 68 / 54 s |
| → 85 °C | 64 / 48 s | never / never |
| → 88 °C | never / 82 s | never / never |
| → 90 °C | never | never |
| Junction max | 87 / 88 °C | **82 / 82 °C** |
| Mean W under load / max | 191 / 226 and 185 / 229 | **180 / 200 and 184 / 200** |
| pp512 | 207.69 ± 0.45 | 204.02 ± 0.50 (−1.8 %) |
| pp4096 | 195.90 ± 1.14 | 191.40 ± 1.24 (−2.3 %) |
| tg128 | 16.77 ± 0.37 | **16.71 ± 0.50 (−0.4 %, within noise)** |

Not equal conditions: the 200 W run started hotter (43/42 vs 31/30 °C), case closed with the BIOS curve.

### T3 — llama-server 72B Q4_0 tensor, sustained, c1

Server: `-ngl 99 -dev ROCm0,ROCm1 -sm tensor -fa 1 -ctk q8_0 -ctv q8_0 -c 12288 -np 4 --no-cache-prompt -cram 0`. Client: server contract, seed 1001.

| | 200 W (forced top from mid-run) | **180 W** |
|---|---|---|
| Load time | 163 s | 178 s |
| Warm-up c1, 2 req.: median TTFT / TPOT | 10,145 ms / 61.47 ms | 10,414 ms / 61.95 ms |
| **c1, 8 requests** | **cut at 90 °C** | **8/8, 0 failures**, 208.3 s |
| Median / P99 TTFT | — | 10,479 / 10,693 ms |
| Median / P99 TPOT | — | 60.77 / 61.73 ms (16.5 t/s) |
| Output throughput | — | 9.83 t/s (2,047 of 2,048 tokens) |
| Total load duration | 158 s before the cut | **328 s** |
| → 85 °C (#1 / #2) | 48 / 46 s | 190 / 166 s |
| → 88 / → 90 °C (#1; #2) | 76 s, never; 136 s, **158 s** | never; never |
| Junction max | 89 / **90 °C** | **87 / 86 °C** |
| Mean W under load / max | 189 / 241 and 190 / 228 | **177 / 209 and 178 / 220** |

T3 junction profile, max per 30 s slice:
- MI50 #1: 64, 76, 81, 81, 79, 83, 84, 85, 86, 86, 87, 87
- MI50 #2: 65, 77, 81, 83, 81, 83, 85, 85, 86, 86, 86, 86

The climb slows but **does not provably stop**: +2 °C over the last two minutes (the last 83 s without the forced top exhaust). A plateau around 86-88 °C is plausible but not demonstrated.

### Summary 180 W vs 200 W

- About −15 W real mean per card under load (177-184 W vs 189-196 W).
- About **−4 to −5 °C** junction max at equal load; time to 85 °C roughly **doubled**.
- Speed cost: generation −0.4 to −2.2 %, prefill −2 to −4 %, TTFT +3 %.
- First time in the session that a 72B server load (5.5 min) stayed under the 90 °C guard — but see the 30-min test below: it did not last.

**180 W was adopted as the operating cap after these tests.**

---

### Long sustained load — 72B Q4_0 llama-server tensor, c1, prompt cache ON, 30 min planned

Same server as T3 but with the prompt cache on; `prefix_repetition` dataset (2,000 shared + 48 distinct tokens, 256 out, `--ignore-eos`); cold start at 32-33 °C; blowers at 100 % from the first minute; case top exhaust on the BIOS curve (~34-51 %).

| | MI50 #1 (top card) | MI50 #2 |
|---|---|---|
| Time to 80 / 85 / 88 / 90 °C | 46 / 148 / 238 / **582 s** | 48 / 164 / 380 / never |
| Plateau | **none**: 88 °C from minute 3 to 7, then 89, then 90 | 87-88 °C from minute 4 to 9 (6 min only) |
| Samples ≥ 85 °C after minute 3 | 96.5 % | 91.6 % |
| Mean power under load | 177 W | 178 W |

- **The safety guard cut the run at 90 °C after 9 min 42 s of load** (8 min 43 s of the measured 30 min).
- No throttling visible before 90 °C: core clock steady (≈1,600 MHz mean), generation 17.04 → 17.03 t/s across the run.
- Prefill per request with the shared prefix cached: 0.42 s (vs 10.5 s for the full 2,048 tokens).
- One run, no control for the case-fan setting (T3 had the top exhaust forced to 100 %; this run had the BIOS curve).

**Conclusion so far:** on this rig, **180 W makes short runs safe but does not make a continuous single-stream server load sustainable on the hotter (top) card.** Next candidates, untested: 160 W, permanently forced case exhaust, duty-cycling requests.

## 10. Caveats / what is tainted or not measured

Copied from the reports' own reservations.

**Tainted or weak results**

- **Speculative decoding "draft on 3060"**: another model (8,192 MiB) was resident on the 3060 during the run (8,911 MiB occupied in total, ~700 for the draft). The blocking loop that kept it off had been stopped and was not re-armed for this test. Its only computation finished about 28 s before the measured requests began, so the figures are usable, but the card was shared. The "draft on MI50" run gives the same conclusion.
- **Case airflow comparison**: one run per configuration, thermal noise not measured; differences are indicative. The warm-start forced-fan run is invalid.
- **180 W runs**: one run per configuration, no repetition. T1 has no complete 200 W reference on the same card (reference is from the other card, two days earlier). T2's 200 W reference started hotter with a different fan setup. T3 ran its last 83 s without the forced top exhaust.
- **Config A vs B**: the A rows for 30B, 27B and 72B Q4_0 come from the HIP-only image, not the HIP+CUDA image. Binary equivalence is proven only on 72B Q4_K_M.
- **72B llama-bench runs use `-r 2`** (vs 5 elsewhere) to limit heat; measured std ≤ 3 %.
- **Three-card output quality not checked** (llama-bench produces no readable text).
- **mixa3607 image `sm_120a`**: only a string found in the binary plus the runtime error; cubins not inspected.
- **Client tokenizer**: the client used the Qwen2.5-72B-AWQ tokenizer also for the 30B-A3B (Qwen3) llama-server runs. The server counts its own tokens; the client's "total input tokens" figures are the client's.
- **Prefix cache P99 TTFT at c4 with cache (4.8 s)** probably comes from the first pass when each slot's cache is still empty (inference).
- **The 72B Q4_0 model on a second storage location** was run through a test-only override of the benchmark harness.
- **One isolated 315 / 307 W reading** (vLLM c4) is unexplained beyond being a transient; to be watched.
- **Two Docker daemon hangs**: cause not established.

**Not measured**

- Concurrency (c4) on the 72B at 200 W, in vLLM and llama-server: cut at 90 °C.
- llama-server 72B in layer mode, 27B in tensor mode (server), and any server in config B.
- 27B on one card at 200 W: impossible under the 90 °C rule, whatever the case setup.
- 72B Q4_K_M in tensor mode.
- `-sm tensor` across 3 cards (HIP + CUDA).
- The real benefit of config B: a model of 64-76 GB that does not fit on 2x MI50. None was available.
- Other vLLM gfx906 forks: not tested.
- Speculative decoding in vLLM; speculative at c4; deeper drafts (`--spec-draft-n-max` > 3); MTP-head models (`--spec-type draft-mtp`).
- Dedicated KV q8_0 vs f16 comparison (q8_0 was used everywhere).
- At 180 W: no c4 level, no long load (> 6 min), no open-case run. Whether 180 W allows hours of sustained load is still open.
