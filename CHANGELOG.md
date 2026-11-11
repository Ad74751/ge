# Changelog

## v0.1.0-alpha.1

First public release.

- **CPU engine:** int8 matrix kernels (AVX2, AVX-VNNI, AVX-512-VNNI) with an 8-row interleaved
  weight layout, a per-machine auto-tuner, a repacked weight cache, and lossless prompt-lookup
  speculative decoding.
- **Vulkan GPU engine:** quantized weights in VRAM, tensor-core matrix multiplies and attention
  (`VK_KHR_cooperative_matrix`), an 8-bit KV cache, and pre-recorded decode steps.
- **CPU+GPU hybrid:** an automatic placement planner, and weight streaming so CPU-resident layers
  also run on the GPU for long prompts.
- **`ge serve`:** OpenAI- and Anthropic-compatible HTTP API with streaming, continuous batching of
  many users, prompt-prefix reuse, API keys, and serving two models at once.
- **`ge bench-all`:** the same benchmark on CPU, each GPU and automatic placement, as a report
  with Markdown and JSON output.
- **Checks:** `ge selftest` verifies every CPU and GPU kernel against a reference on the user's machine.

Known limitations are listed in the [README](README.md#known-limitations).
