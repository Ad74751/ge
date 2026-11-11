# Benchmarking ge

This guide explains how to measure ge on your machine, how to compare it fairly with llama.cpp,
and how to share results. Results from many different machines are the most useful thing you can
contribute during the alpha.

## 1. Run the benchmark

```bash
ge bench-all -m MODEL.gguf --runs 5 --out my-results
```

`bench-all` runs the same workload in every configuration your machine supports:

| Configuration | What it measures |
|---|---|
| CPU only | every layer on the CPU |
| GPU, all layers | every layer on one GPU (reported as "does not fit" when VRAM is too small) |
| Auto (planner) | the placement ge picks by itself: CPU, full GPU, or a CPU+GPU split |

GPUs with less than 1 GB of usable VRAM (most integrated GPUs on Windows) are skipped and listed in
the notes. Each configuration runs as a separate process, so memory is fully released between runs.

The report shows, per configuration:

- **Prefill tok/s:** prompt processing speed, `--pp` tokens (default 512) processed at once.
- **Decode tok/s:** generation speed, `--tg` tokens (default 128) generated one at a time.
- **vs CPU:** speedup over the CPU-only row.
- **Peak RAM:** the process's peak working set, including the parts of the model file it has mapped.
- **VRAM:** ge's GPU weights plus its session buffers.
- **±%:** half the spread between the fastest and slowest run, relative to the median.

`--out my-results` also writes `my-results.md` (a table to paste into an issue) and
`my-results.json` (machine-readable, see [benchmark-schema.json](benchmark-schema.json)).

Other options: `--pp N`, `--tg N`, `--ctx N` (context length, default 4096), `--runs N`.

## 2. Rules that keep results comparable

- **Use `--runs 5`.** Single runs on laptops vary by 10–30%.
- **Plug in laptops and use a performance power mode.** Battery saver and quiet modes lower clocks.
- **Close heavy programs** (games, video calls, browsers with many tabs, other AI tools).
- **Run once before measuring.** The first run on a new model writes the weight cache and tunes ge;
  later runs are the ones to report. `bench-all` warms up inside each run, but not across model downloads.
- **Report the exact model file.** Quantization matters: a `Q4_K_M` result is not comparable with
  `Q8_0`. Include the file name and, if you can, its SHA-256 (`certutil -hashfile FILE SHA256` on
  Windows, `sha256sum FILE` on Linux).
- **Decode numbers are plain greedy generation.** `bench` and `bench-all` never use ge's
  prompt-lookup drafting, so they compare directly with `llama-bench`'s `tg` numbers.

## 3. Compare with llama.cpp (optional)

Use the same model file and run the commands back to back, with nothing else running.

```bash
# CPU only
llama-bench -m MODEL.gguf -ngl 0 -dev none -p 512 -n 128 -r 5

# All layers on the GPU (Vulkan build of llama.cpp; pick your device from `llama-bench --list-devices`)
llama-bench -m MODEL.gguf -ngl 99 -p 512 -n 128 -r 5

# CPU+GPU split: raise -ngl until it no longer fits, and report the fastest value that fits
llama-bench -m MODEL.gguf -ngl 20 -p 512 -n 128 -r 5
```

On NVIDIA GPUs, llama.cpp's Vulkan backend uses the NVIDIA-only `VK_NV_cooperative_matrix2`
extension for prompt processing. Setting `GGML_VK_DISABLE_COOPMAT2=1` makes it use the same
cross-vendor path as ge, which separates the engine design from that extension. Please report both
if you can, and say which is which.

## 4. Share your results

1. Open a [benchmark issue](https://github.com/Ad74751/ge/issues/new?template=benchmark.yml).
2. Paste the contents of `my-results.md` and attach `my-results.json`.
3. Fill in the driver version and power mode, which ge can't detect yet.

Nothing is uploaded automatically: ge has no telemetry, and the files contain only what you see in
them (CPU and GPU model names, RAM size, OS, model file name and the measurements). Review them
before posting.

## 5. Check correctness, not just speed

Speed only counts if the output is right. Two quick checks:

```bash
ge selftest                                   # every CPU and GPU kernel against a reference
ge perplexity -m MODEL.gguf -f text.txt -c 512 --chunks 12   # compare with llama-perplexity on the same text
```

Perplexity from ge and `llama-perplexity` on the same file and text should agree to within about
1%. A much larger gap is a bug: please report it.
