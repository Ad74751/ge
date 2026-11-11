# Security and privacy

ge's source code isn't public during the alpha, so this page lists exactly what the program does on
your machine, and how you can check it yourself instead of taking our word for it.

## What ge does on your computer

**Network**
- ge makes **no outbound connections**: no telemetry, no analytics, no update checks, no model
  downloads. It works fully offline.
- The only network activity is `ge serve`, which **listens** for HTTP requests. By default it binds
  to `127.0.0.1` (this computer only). It's reachable from other machines only if you pass
  `--host 0.0.0.0` or another address yourself.

**Files it reads**
- The model files you pass with `-m`, and text files you pass with `-f`.

**Files it writes**

| File | Where | Why |
|---|---|---|
| `MODEL.gguf.gecache` | next to the model file | weights repacked for fast loading (about the model's size). Skip with `--no-cache`; delete any time. |
| `profile.txt` | cache folder (below) | the CPU tuning results (a few lines of text) |
| `gpu-plan.txt` | cache folder (below) | the chosen CPU/GPU split per model (a few lines of text) |
| `NAME.md`, `NAME.json` | where you say | only when you run `ge bench-all --out NAME` |

The cache folder is `%LOCALAPPDATA%\gguf-engine` on Windows and `$XDG_CACHE_HOME/gguf-engine` (or
`~/.cache/gguf-engine`) on Linux. Set `GE_CACHE_DIR` to use another folder. Deleting it is safe.

**Programs it starts**
- Only `ge bench-all`, which runs copies of the same `ge` executable once per configuration.

**What it doesn't do**
- No installer, no administrator rights, no registry changes, no services, no drivers, no
  scheduled tasks, no browser or credential access.
- It uses your GPU through the Vulkan library that comes with your graphics driver.
- It adjusts only its own process: its threads' priority and CPU affinity, and on Windows it opts
  itself out of power throttling (EcoQoS) so laptops don't run it on slow clocks.

## Check it yourself

1. **Verify the download.** Each release has a `SHA256SUMS.txt`. Compare it with the archive you
   downloaded:
   ```powershell
   certutil -hashfile ge-v0.1.0-alpha.1-windows-x86_64.zip SHA256
   ```
   ```bash
   sha256sum -c SHA256SUMS.txt --ignore-missing
   ```
2. **Scan it.** Run your antivirus on the archive, or upload it to [VirusTotal](https://www.virustotal.com).
3. **Watch the network.** On Windows, open Resource Monitor → Network while ge runs: `ge chat` and
   `ge bench` show no connections, and `ge serve` shows only its listening port. On Linux,
   `ss -tunap | grep ge` shows the same. Or simply disconnect from the internet: everything works.
4. **Isolate it.** Try it first in [Windows Sandbox](https://learn.microsoft.com/windows/security/application-security/application-isolation/windows-sandbox/windows-sandbox-overview)
   or a virtual machine. Note that GPU acceleration usually isn't available there, so ge will use the CPU.

**About warnings:** the Windows binary is not code-signed yet, so Windows SmartScreen may say
"Windows protected your PC" or "unknown publisher" the first time. That's expected for unsigned
programs; check the checksum before choosing **More info → Run anyway**.

## Running the server safely

- Keep the default `127.0.0.1` binding unless other machines need access.
- When exposing it, always set `--api-key KEY` (or `GE_API_KEY`), and put it behind a reverse proxy
  with HTTPS: ge serves plain HTTP.
- Request size is limited to 64 MB and the queue to `--max-queue` requests (default 1024).

## Reporting a vulnerability

Please don't open a public issue for security problems. Use GitHub's
[private vulnerability reporting](https://github.com/Ad74751/ge/security/advisories/new) instead,
with the version (`ge --version`), your OS and steps to reproduce. You'll get a reply as soon as possible.
