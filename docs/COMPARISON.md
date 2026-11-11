# ge vs llama.cpp: where it wins, ties, and falls short

**ge** is a GGUF inference engine written in pure Rust: one ~4 MB executable, no runtime dependencies in the core (the HTTP server is an optional actix-web feature), CPU and Vulkan GPU, and hybrid CPU+GPU. This report compares it with llama.cpp on the same machine and the same model files, and lists the places where llama.cpp is still better.

Test machine: AMD Ryzen 5 5600H (6 cores, DDR4, about 31 GB/s), NVIDIA RTX 3050 Laptop GPU (4 GB, 35 W default power limit, 40 W when checked, 162 GB/s VRAM read, 6.7 GB/s host-to-GPU), Windows 11. Full specs: [README](../README.md#test-machine). llama.cpp build 9102. Both engines were run back to back; laptop thermals and background load move numbers by roughly 10–30% between runs, so differences under about 5% are ties.

## Summary

| Area | Result |
|---|---|
| Memory | **ge uses 1.7–3× less RAM** in every mode tested |
| CPU-only prompt processing (3B) | **ge is 25–40% faster** |
| GPU decode, 3B fully on GPU | **ge is about 10% faster** at short context, and the gap grows with context |
| Hybrid decode, 8B | **Tie** (13.5–14.0 vs 13.0 tok/s); llama.cpp drops to 4 tok/s at the split ge chooses (`-ngl 22`), where ge fits 22 layers automatically |
| GPU prompt processing, 3B, same Vulkan path | **Tie** (1,240–1,340 vs 1,315–1,330 tok/s) |
| GPU prompt processing, 3B, NVIDIA-specific path | **llama.cpp is ~35% faster** (1,765–1,800 tok/s) |
| Hybrid prompt processing, 8B | **llama.cpp is ~20% faster** (535 vs 410–440 tok/s) |
| Serving many users | **Built, but not compared head-to-head with llama.cpp's server** |
| Breadth (architectures, quant types, platforms, users) | **llama.cpp is far ahead** |

## Where ge is better

### 1. Memory footprint
Peak RAM, measured during benchmark runs:

| Setup | llama.cpp | ge |
|---|---|---|
| Qwen2.5-3B, GPU | 1.96 GB | **0.64 GB** |
| Llama-3.1-8B, hybrid | 4.72 GB | **2.36 GB** |
| Qwen2.5-3B, CPU only | 3.33 GB | **1.96 GB** |

For the 8B hybrid case the number reflects that ge reads its CPU-side weights from a repacked cache and hands the GGUF pages back to the OS; llama.cpp keeps more resident. This matters on 8–16 GB machines running more than one thing.

### 2. CPU prompt processing (Qwen2.5-3B, prompt lengths 128–2,048)
ge: 69–86 tok/s. llama.cpp: 56–61 tok/s. About **25–40% faster** across the range.

### 3. GPU token generation, model fully on the GPU (Qwen2.5-3B)
| Context depth | llama.cpp | ge |
|---|---|---|
| 128 | 53.2 tok/s | **58.8 tok/s** |
| 512 | 50.6 tok/s | **58.2 tok/s** |
| 1,024 | 49.3 tok/s | **56.7 tok/s** |
| 2,048 | 46.7 tok/s | **53.5 tok/s** |

The advantage comes from attention that processes each group of query heads sharing a key/value head in one pass, and from recording the next token's GPU commands while the current one runs. The 8-bit KV cache (perplexity 4.2078 vs 4.2081 with f16) reduces memory traffic without a measurable quality cost on the tested model.

### 4. Hybrid CPU+GPU placement of large models
Llama-3.1-8B does not fit in 4 GB of VRAM. ge's planner measures the options and picks the layer split: 22 of 32 layers on the GPU with the 8-bit KV cache. llama.cpp at the same split (`-ngl 22`) ran at 4 tok/s, because it overflowed VRAM. At `-ngl 21` llama.cpp runs at 13.0 tok/s, and ge runs at 13.5–14.0.

### 5. Prompt-lookup drafting
On a code-editing prompt, where the output repeats text from the prompt, ge reached **18.5 tok/s**. llama.cpp was not run on that prompt, so there is no matched comparison; its open-ended 8B hybrid decode is 13.0 tok/s. This is a workload-specific gain and does not apply to open-ended chat. Drafting is lossless: greedy output is byte-identical to non-drafted decoding.

### 6. Footprint and deployment
- Single executable, about 4 MB, with no separate runtime libraries in the core build.
- llama.cpp's Windows Vulkan build needs its DLLs next to it (about 12.6 MB together).
- Shaders are compiled at build time and embedded; Vulkan is loaded at runtime.

## Where ge ties

- **3B prompt processing on the GPU, same Vulkan path:** ge 1,240–1,340 tok/s, llama.cpp 1,315–1,330 tok/s (llama.cpp's `GGML_VK_DISABLE_COOPMAT2=1`). Earlier in this work ge was 25% behind on this path; a port of llama.cpp's matrix-multiply design closed the gap.
- **8B hybrid token generation:** 13.5–14.0 (ge) vs 13.0 (llama.cpp at `-ngl 21`). That is within run-to-run variance.
- **CPU-only token generation:** both are limited by memory bandwidth (about 31 GB/s on this machine), so they are within a few percent.

## Where llama.cpp is better

### 1. GPU prompt processing on NVIDIA: about 35% gap
llama.cpp uses the `VK_NV_cooperative_matrix2` extension and int8 tensor-core paths on NVIDIA. ge uses the cross-vendor `VK_KHR_cooperative_matrix` extension through naga's WGSL compiler, which can only express f16/f32 cooperative matrices. On the 3B, llama.cpp's NVIDIA path does 1,765–1,800 tok/s to ge's 1,240–1,340. Closing this needs either a hand-written NVIDIA SPIR-V path (outside naga) or naga support for those features.

### 2. Hybrid prompt processing on the 8B: about 20% gap
llama.cpp runs 535 tok/s at `-ngl 21`; ge runs 410–440 tok/s at 22 layers. ge's remaining cost is copying the CPU-resident layers (1.3 GB per prompt) to the GPU over PCIe. Cached-file runs are fast (410–440); right after another process has read the whole model file, ge's first prefill drops to 289–338 tok/s. The first prefill after loading is therefore slower than later ones. The fix, a prefetch that runs before the first request, is not finished.

### 3. Serving many users: not yet compared
ge has a batching server (`ge serve`: OpenAI and Anthropic APIs, continuous batching, prefix reuse, several models at once). Measured on the 8B hybrid with 4 slots:

| Concurrent clients | Total tok/s |
|---|---|
| 1 | 11 |
| 4 | 28 |
| 8 | 29 (4 slots, the rest queue) |

Two models served together ran at 13 + 10 tok/s with 4 clients each, against 12 tok/s for one model alone. **llama.cpp's own server was not benchmarked on this machine, so there is no head-to-head claim for serving.** The 8B result also reflects the VRAM limit: each model gets half the VRAM and only 6 GPU layers when two run together.

### 4. Breadth
- llama.cpp supports many more model architectures, quantization types, and backends (CUDA, Metal, ROCm, SYCL, and others). ge supports the architectures and quant types the tests cover, and Vulkan only.
- llama.cpp has been used by far more people; ge has been tested on one laptop, one GPU, and about four models.
- ge's Gemma 3 support is untested: its first run would build a multi-GB repack cache, and its 256-wide attention heads use ge's slowest GPU attention path.

### 5. Maturity
ge's correctness is checked against llama.cpp's perplexity on three models (Qwen2.5-3B, Llama-3.1-8B, Mistral-7B) and by greedy-output comparisons, but that is a small set of models. Edge cases like unusual chat templates, very long contexts, and many quant types have had little testing.

## How this was measured

- Same GGUF files for both engines. Perplexity: `ge perplexity` against `llama-perplexity`, 12 chunks of 512 tokens (3B) or 6 chunks (8B, Mistral).
- Speed: `ge bench` (prefill and decode) against `llama-bench`, run back to back, warm model files, with interleaved repeats when results were close.
- Memory: peak working set of each process during the benchmark.
- Correctness: ge's GPU kernels checked against an f64 CPU reference (`ge selftest`), 12 unit tests including batched vs single-sequence decoding, and identical greedy output between CPU-only and hybrid runs for the 8B.
- Honest limits: one machine, one GPU, a handful of models, Windows only. Background programs (browser, IDE) were running during the tests, which lowers absolute numbers for both engines equally but adds noise.

## Reproduce

```bash
ge bench -m models/qwen2.5-3b-instruct-q4_k_m.gguf -c 4096 --pp 512 --tg 128
ge perplexity -m models/qwen2.5-3b-instruct-q4_k_m.gguf -f models/ppl.txt -c 512 --chunks 12 -ngl all
ge serve -m models/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf --alias llama8b -np 4 -c 2048 --port 8080
```

```bash
llama-bench -m models/qwen2.5-3b-instruct-q4_k_m.gguf -dev Vulkan1 -ngl 99 -p 512 -n 128
GGML_VK_DISABLE_COOPMAT2=1 llama-bench -m models/qwen2.5-3b-instruct-q4_k_m.gguf -dev Vulkan1 -ngl 99 -p 512
```


## Spill case: Qwen3-4B Q4_K_M on a 4 GB GPU (16K context)

Model 2.5 GB of weights plus about 2.4 GB of f16 KV at 16K context, so about 4.9 GB total on a 4 GB card. Both engines were run as a server, one request at a time, 256-token generations.

| Engine | GPU layers | KV cache | Decode tok/s | Prefill tok/s (1,770–1,778 tokens) |
|---|---|---|---|---|
| llama.cpp (b9102), matched split | 33 of 37 | in RAM (`--no-kv-offload`) | ~22 | ~960 |
| ge serve | 33 of 36 | split by planner | ~37 | ~719 (first, uncached request) |

Notes:
- llama.cpp fails at all-GPU with 16K context (out of device memory when allocating KV); it only runs when layers or KV move to the CPU.
- The first llama.cpp request after load was cold (50 tok/s prefill); the numbers above are warm.
- ge's repeated requests reused its prefix cache, so only the first request measures prefill.
- ge's planner chose the split itself. The llama.cpp split was chosen to match ge's GPU layer count.
- Single run each, not repeated; llama.cpp numbers are the mean of three requests, ge's of one.
