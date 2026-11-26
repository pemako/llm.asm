# Building and Running `llm.asm` on macOS (Apple Silicon)

This repo's documented system requirements (`README.md`) are Linux x86-64 +
GNU `binutils`. Everything below records what it actually takes to build
*and run* on a macOS arm64 (Apple Silicon) host, and why.

## 1. `make all` failed at the link step

**Symptom:**

```
ld --gc-sections -s -z noexecstack -z noseparate-code build/syscalls.o ... -o build/asm-llm
ld: warning: -s is obsolete
ld: unknown options: --gc-sections -z -z
make: *** [build/asm-llm] Error 1
```

`nasm` assembled every `.o` file fine — only the link stage broke.

**Root cause:** on macOS, `/usr/bin/ld` is Apple's linker (`ld-prime`/cctools),
not GNU `ld`. It:

- doesn't understand the GNU-only flags `--gc-sections`, `-z noexecstack`,
  `-z noseparate-code` → the exact error above
- produces Mach-O, not ELF, so it couldn't link these ELF64 `.o` files into
  an ELF64 executable even if the flags were fixed

macOS also ships no `objcopy` at all (needed for the Phase 2 OS build's
ELF64→ELF32 Multiboot conversion).

This is not a code bug — it's the documented Linux/GNU-binutils requirement
not being met on macOS.

**Fix:**

```bash
brew install x86_64-elf-binutils
```

`Makefile` now auto-detects `Darwin` and switches `LD`/`OBJCOPY` to
`x86_64-elf-ld`/`x86_64-elf-objcopy` when present (falls back to plain
`ld`/`objcopy`, with a `$(warning ...)`, when not installed; Linux behavior
is unchanged). A command-line override (`make LD=... OBJCOPY=...`) still
takes precedence as before.

One Make gotcha hit along the way: `LD`/`OBJCOPY` have built-in **default**
values inside GNU Make itself, so `LD ?= x86_64-elf-ld` is a silent no-op —
Make already considers the variable "set". The fix uses plain `:=` inside
the `Darwin`/else branches instead.

**Verified:** clean `make all` (no flags), idempotent re-run
(`make: Nothing to be done for 'all'`), and output sizes match the README
exactly:

```
build/asm-llm          14,712 B   ELF 64-bit LSB executable, x86-64, statically linked, stripped
build/test_math         6,360 B
build/asm-llm-os.elf    24,252 B
build/asm-gpt-train     32,944 B
```

## 2. The built binaries still can't *run* natively on macOS

`build/asm-llm` (Phase 1) and `build/asm-gpt-train` (Phase 3) are static
**Linux x86-64** ELF executables using raw Linux syscalls (`mmap`, `clone`,
`futex`, `write`, ...). A macOS arm64 host can't execute that directly —
wrong OS and wrong architecture.

**Workaround used:** this machine has **OrbStack** (`docker info` →
`OrbStack aarch64`), which transparently emulates `linux/amd64` containers.
The static binary (zero libc, zero dynamic linking) runs inside one with no
extra setup:

```bash
docker run --rm --platform linux/amd64 -v "$(pwd)":/work -w /work debian:bookworm-slim \
  ./build/asm-llm -h
```

This was confirmed working (prints the usage banner, so CPUID/AVX2
detection and the basic syscall path survive emulation).

**Also confirmed working, end to end:** the zero-libc `clone()` + futex
thread pool at `-t 16` — full single-shot inference, multi-threaded
inference, and `--chat` all produced correct, coherent output under
OrbStack's `linux/amd64` emulation (see §4 for the actual runs and
numbers). So no, this wasn't the fragile part after all — at least not on
OrbStack. Throughput is obviously far below the README's native-hardware
numbers (EPYC/Xeon), since every instruction is being translated, but it's
fully functional.

An alternative if you ever need native-hardware throughput: a real x86-64
Linux VM (cloud or local) — this is what the README's own benchmarks (KVM
EPYC, Proxmox Xeon) were run on.

## 3. Getting and exporting the model weights

`hf download` (the bundled CLI, v2.4.0) fails on this machine with:

```
ImportError: Using SOCKS proxy, but the 'socksio' package is not installed.
```

because `all_proxy=socks5://127.0.0.1:7897` is set in the shell environment
and `hf`'s bundled `httpx2` client needs the `socksio` extra to use a SOCKS
proxy. Workaround — drop just that one proxy var (the plain
`http_proxy`/`https_proxy` vars still apply and don't need `socksio`):

```bash
mkdir -p models
env -u all_proxy hf download Qwen/Qwen3-0.6B \
  config.json model.safetensors tokenizer.json \
  --local-dir models/qwen3-0.6b-bf16
```

Then export to the compact `.asmbin` format (pure Python + numpy, runs fine
natively on macOS — no architecture dependency here):

```bash
python3 scripts/export_weights.py \
  --model-dir models/qwen3-0.6b-bf16 \
  --out models/qwen3-0.6b-q8_0.asmbin \
  --scale-dtype fp16
```

## 4. Running it — verified

All three invocations below were actually run (not just copy-pasted from
the README) against `models/qwen3-0.6b-q8_0.asmbin` under OrbStack
`linux/amd64` emulation on this machine, confirming the full pipeline
works.

### Single-shot, 1 thread

```bash
docker run --rm --platform linux/amd64 -v "$(pwd)":/work -w /work debian:bookworm-slim \
  ./build/asm-llm -m models/qwen3-0.6b-q8_0.asmbin \
  -p "Explain how a CPU cache works in 3 concise sentences." -n 128 -t 1
```

```
A CPU cache is a small amount of memory used to store frequently accessed data and instructions. It helps speed up processing by storing data that is likely to be used again. The cache is organized in a way that allows it to quickly retrieve data from memory when needed.
[asm-llm] prefill: 25 tok in 1555.60 ms (16.07 tok/s)
[asm-llm] decode : 53 tok in 3708.90 ms (14.02 tok/s, threads=1)
```

### Single-shot, 16 threads (the documented default path)

```bash
docker run --rm --platform linux/amd64 -v "$(pwd)":/work -w /work debian:bookworm-slim \
  ./build/asm-llm -m models/qwen3-0.6b-q8_0.asmbin \
  -p "What is 17 * 23?" -n 64 -t 16
```

```
[asm-llm] prefill: 22 tok in 495.25 ms (44.42 tok/s)
[asm-llm] decode : 64 tok in 1934.73 ms (32.56 tok/s, threads=16)
```

Confirms the raw `clone()`/futex thread pool survives emulation and scales
(32.56 tok/s @ 16T vs 14.02 tok/s @ 1T) — not just "doesn't crash."

### Interactive multi-turn chat

```bash
docker run --rm -it --platform linux/amd64 -v "$(pwd)":/work -w /work debian:bookworm-slim \
  ./build/asm-llm -m models/qwen3-0.6b-q8_0.asmbin --chat -t 16
```

(`-it` is required so the container gets a TTY/stdin for the raw
`read(0)`/`write(1)` chat loop; omit `-i`/`-t` and it'll just hang waiting
on stdin.) Verified non-interactively by piping input:

```bash
printf 'Hi, who are you?\n/quit\n' | docker run --rm -i --platform linux/amd64 \
  -v "$(pwd)":/work -w /work debian:bookworm-slim \
  ./build/asm-llm -m models/qwen3-0.6b-q8_0.asmbin --chat -t 16
```

```
Interactive Multi-Turn Chat (/clear, /think, /nothink, /quit)

asm-llm> Hello! I'm a language model designed to assist with various tasks. If you have any questions or need help with something, feel free to ask!
[asm-llm] prefill: 18 tok in 1176.75 ms (15.30 tok/s)
[asm-llm] decode : 30 tok in 1244.65 ms (23.30 tok/s, threads=16) ctx=50/2048

asm-llm>
```

KV-cache context tracking (`ctx=50/2048`) and `/quit` both worked as
documented. For real use, drop `-it` into your own terminal and type —
that's the live chat.
