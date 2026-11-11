# ge bench-all

- **Model:** Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf (4.92 GB)
- **CPU:** AMD Ryzen 5 5600H with Radeon Graphics · avx2 · 6 cores · 15.4 GB usable RAM
- **GPUs:** NVIDIA GeForce RTX 3050 Laptop GPU (4.2 GB VRAM) · AMD Radeon(TM) Graphics (0.3 GB VRAM)
- **OS:** windows x86_64
- **Workload:** 512-token prompt, 128 generated tokens, 4096 context, 5 run(s) each, median shown
- **ge:** v0.1.0

| Mode | Device | Placement | Prefill tok/s | vs CPU | Decode tok/s | vs CPU | Peak RAM | VRAM |
|---|---|---|---|---|---|---|---|---|
| CPU only | CPU | CPU | 44.0 ±6% | 1.0x | 7.5 ±7% | 1.0x | 4.90 GB | - |
| GPU, all layers | RTX 3050 Laptop | does not fit: needs 4.62 GB of VRAM, 3.53 GB free |  |  |  |  |  |  |
| Auto (planner) | RTX 3050 Laptop | hybrid 22/32 on GPU | 424.9 ±9% | 9.7x | 13.8 ±14% | 1.8x | 1.89 GB | 3.74 GB |

**Fastest generation: Auto (planner) on RTX 3050 Laptop (hybrid 22/32 on GPU): 13.8 tok/s, 1.8x the CPU**

**Fastest prompt processing: Auto (planner) on RTX 3050 Laptop (hybrid 22/32 on GPU): 424.9 tok/s, 9.7x the CPU**

- Decode is plain greedy generation (no speculative / prompt-lookup decoding).
- Peak RAM is the process's peak working set, including memory-mapped model pages. VRAM is ge's weights plus session buffers.
- AMD Radeon(TM) Graphics skipped: 0.26 GB of VRAM available.
