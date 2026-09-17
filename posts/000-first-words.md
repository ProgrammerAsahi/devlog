**English** | [简体中文](000-first-words.zh-CN.md)

# Teaching My 4090 to Speak: Two Days, Five Gates

*devlog #0 · September 16, 2026 · Amsterdam*

---

*Dear you, six months from now:*

*You might be packing your bag in the morning light, or reading by the window with familiar footsteps nearby. Let me use this letter to quietly send you all the small moments of tenderness and growth from these six months…*

*(The full letter was written in Chinese by the model — read it in the [中文版本](000-first-words.zh-CN.md). It signed off with a literal `[你的名字]` placeholder, which I kept verbatim: the innocence of a 1.7B model, and the first piece of authenticity in this log.)*

---

Those words were spoken by my RTX 4090 — more precisely, by Qwen3-1.7B running on it, streamed token by token through vLLM's OpenAI-compatible API (it drafted half a page inside `<think>` before writing a single word). Getting it to say them took me two days and five gates.

## Starting point

- Windows 11 + WSL2 (Ubuntu 24.04, kernel 6.18)
- RTX 4090, driver 615.65 (CUDA 13.4)
- Python 3.12 venv built with uv: torch 2.13, vllm 0.29
- Goal: `vllm serve Qwen/Qwen3-1.7B`, then one streaming curl request

Sounds like a single step. In reality:

## Gate 1: UVA is not available

`RuntimeError: UVA is not available` — a misleading message. The truth: vLLM 0.29's new V2 model runner needs a pinned-memory staging buffer that the GPU can read and write directly, and vLLM disables pinned memory on WSL by default (a legacy of incomplete support in old WSL2 kernels). The fix was one environment variable: `VLLM_WSL2_ENABLE_PIN_MEMORY=1`.

## Gate 2: The GPU vanishes

While diagnosing gate 1, I found `nvidia-smi` reporting *"GPU access blocked by the operating system"* and `torch.cuda.is_available()` returning False — WSL couldn't see the card at all. `dmesg` held the remains of a kernel crash in dxg, WSL's GPU paravirtualization channel. The fix was on the Windows side: `wsl --shutdown`, then back in.

## Gate 3: Python.h does not exist

This error was hiding *above* the several-hundred-line traceback: `fatal error: Python.h: No such file or directory`. vLLM uses `torch.compile` → Inductor → Triton by default, and Triton compiles a small C extension **on first run** with gcc, which needs Python's development headers. On Ubuntu those live in a separate package: `sudo apt install python3.12-dev`.

## Gate 4: One machine, three CUDAs (the maddening one)

FlashInfer's sampling kernels are JIT-compiled on first use — and nvcc failed with `Unknown option '--compress-mode=size'`, a flag that only exists in CUDA 12.8+. The apt-installed nvcc on my system was 12.0.

The investigation revealed **three versions of CUDA** living on one machine: the 13.4 driver (mapped into WSL from Windows), apt's nvcc 12.0, and pip's nvcc 13.4 inside the venv. I won't pretend I stayed calm — setting up the environment was already far more complicated than I'd imagined.

Pointing the build at the venv's nvcc 13.4 exposed an even deeper layer: the pip packages were fighting a civil war — the compiler was 13.4, but the CUDA headers right next to it claimed 13.0, and CCCL's strict compatibility check rejected the combination with an `#error`.

## Gate 5: Fix it properly, don't route around it

I chose to fix the environment once and for all: purge apt's 12.0, install a self-consistent `cuda-toolkit` from NVIDIA's official WSL repository — learning in passing that the WSL repo didn't even have 13.4, only up to 13.3. **Version numbers in tutorials expire; what `apt-cache policy` shows is the truth.** The end state: the driver (13.4) untouched, one single toolchain at `/usr/local/cuda-13.3` (with `CUDA_HOME` in `~/.bashrc`), and torch's pip runtime treated as its private dependency, left alone.

After that, `vllm serve` went through in one run: 20s of torch.compile, 3s of CUDA graph capture, 49s of FlashInfer compiling with the new nvcc — and 17.03 GiB of VRAM turned into a KV cache of **159,424 tokens**. `Application startup complete`.

## The most valuable lesson

Don't panic at failures and hundred-line stack traces. The root cause is usually **one concise line** — and often not in the Python traceback at all, but in the output of the tool it called. Four gates, same technique four times: filter for ERROR lines first, then look inside the error block for what the tool itself said (gcc's `fatal error`, nvcc's `fatal`, CCCL's `#error`). Go to the deepest error position; the root cause is usually right there.

## Planned 3 hours, actual 4.5

Worth it. The hardest CUDA environment problem is solved once and for all; I won't pay that time again. Writing the summaries by hand made me a real participant. Beginnings are hard — going slower made it stick better.

## On the "AI mentor + my hands" way of learning

An AI (Kimi-K3) mentored the whole process while I did the work. Extremely efficient — the same debugging could have taken me far longer alone. But the cost is real: I didn't find the errors myself, so the lessons may not stick as deeply. Starting with the next task, the boundary moves one step forward: **troubleshooting should be guided, not done for me.** A new agreement with my AI mentor, written here on the record.

## The first token, and the letter

Honestly, when the first token appeared I was happy, but not thrilled — I had seen streaming output before, when I did frontend integration for a DeepSeek-R1 deployment on Nebius's token factory. But this time was different: every layer from driver to toolchain was opened by me (with my mentor).

To be honest, when I reread the letter, my first reaction was that it was funny. I'm an adult, yet it wrote to me as if I were a clueless little baby — and in the tone of a father writing to his son, no less. It unilaterally adopted a dad voice, which means I've somehow gained a "cyber little dad". Being taken advantage of like that is mildly annoying, haha. And the wording is honestly too flowery and pretentious. But considering it came from a small model with zero context, I'll let it go — watching a model I deployed with my own hands produce such ornate prose, the joy outweighs everything.

The small model taught me something too: 1.7B counts as "tiny", yet its 3.78 GiB of weights still took over a minute to download, with a pile of prerequisite files — large models really are large. And for the first time I felt, firsthand, that CUDA is mission-critical for LLM deployment.

## To my fellow travelers

Don't be scared by unfamiliar terms or complicated environment setup. You don't need to understand or memorize any of it on day one — all of that comes later, gradually. **As long as you get a model deployed and producing output, you've already taken the first step.**

## What's next

This was only task one of phase P0. Next stop: building a serving baseline measurement on this 4090 — prediction written first, as always.

Both repos, if you'd like to follow along:

- Main project (experiment code + reproduction notes): [ProgrammerAsahi/serving-lab](https://github.com/ProgrammerAsahi/serving-lab)
- Public writing (this repo): [ProgrammerAsahi/devlog](https://github.com/ProgrammerAsahi/devlog)
