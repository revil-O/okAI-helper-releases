# Measurements

The same figures as charts: [BENCHMARKS.md](BENCHMARKS.md) (and site/benchmarks/ for the website), written by
`npm run benchmarks` from tools/benchmarks_data.mjs.

All numbers on the Mac mini M4 with 16 GB (target machine) unless noted. llama.cpp build 9660 (Homebrew, ggml 0.15.1),
mlx 0.32.4 / mlx-lm 0.32.2. Prompt 512 and 128 generated tokens unless noted; 3 runs, averages.

## Spike 2026-11-05: models that fit into memory

| Model | Engine | Prompt t/s | Generation t/s | Peak memory |
| --- | --- | ---: | ---: | ---: |
| Ministral-3 8B Instruct 2512, Q4_K_M (4.83 GiB) | llama.cpp, Metal | 211.9 | 19.7 | – |
| Ministral-3 8B Instruct 2512, MLX 4 bit (5.63 GB) | mlx-lm | 220.0 | 21.7 | 5.3 GB |
| Qwen3-4B, MLX 4 bit (2.28 GB) | mlx-lm | 434.2 | 41.1 | 2.9 GB |

The two Ministral files are quantized differently (Q4_K_M vs. MLX 4 bit), so the engine comparison is approximate:
MLX about 10 % faster in generation for this model.

## Spike 2026-11-05: gateway overhead

`llama-server` (Ministral-3 8B Q4_K_M, context 4096) direct vs. through a minimal Node streaming proxy with a token
check (stand-in for the planned gateway). 128 tokens, temperature 0, 5 runs each, two rounds interleaved.

| Path | Time to first token | Total | Tokens/s |
| --- | ---: | ---: | ---: |
| direct | 58.9 / 59.1 ms | 6667 / 6675 ms | 19.4 / 19.3 |
| through proxy | 60.7 / 60.2 ms | 6662 / 6719 ms | 19.4 / 19.2 |

About 1.5 ms more to the first token, no measurable loss in throughput.

## Spike 2026-11-05: external SSD

`dd` of 4 GB from a file not read before on the external USB SSD (APFS, /Volumes/2TB_FangX): 0.97 GB/s.

## Spike 2026-11-05: models bigger than memory (mmap streaming)

Run from 14:06 (Qwen3-32B Q4_K_M 18.40 GiB dense and Qwen3-30B-A3B Q4_K_M MoE, both from the external SSD, first
with `-ngl 99`, then `-ngl 0`, then MoE with `-ncmoe 99`) ended in a **kernel panic** at 14:31: `watchdog timeout:
no checkins from watchdogd in 93 seconds`. The panic report shows 1,011,622 wired pages of 16 KB = **16.6 GB wired**,
compressor 7 % and swap OK — the whole memory was wired, so nothing could be compressed or swapped. Metal reports
`recommendedMaxWorkingSetSize = 12713 MB` on this Mac. Most likely cause: a model bigger than memory handed to Metal
(GPU offload of an mmap'd file wires its pages; see the ik_llama.cpp RFC #1833 note that wired memory on Apple Silicon
is neither compressed nor swapped). Which of the four runs was active is not known: the log was on the external SSD,
which was not detected after the reboot.

Repeat with a guard (kills llama-bench above 11 GB wired, 20 min per run), log on the internal disk, prompt 32 and
8 generated tokens, 1 run:

| Model, disk, setting | Prompt t/s | Generation t/s | Wall time | Max RSS | Max wired |
| --- | ---: | ---: | ---: | ---: | ---: |
| Qwen3-30B-A3B Q4_K_M (17.28 GiB, MoE), external, CPU only (`-dev none -ngl 0`) | 3.02 | 1.24 | 934 s | 8.5 GB | 2.9 GB |
| same, `-ngl 99 -ncmoe 99` (experts on CPU, rest on Metal) | failed | – | 70 s | 12.5 GB | **10.8 GB** |

The `-ncmoe` run failed with `failed to decode prompt batch, res = -3` and wired 10.8 GB although the experts stayed on
the CPU. Metal runs of models bigger than memory were dropped from the spike for this reason; the rest ran CPU only
with the guard at 6 GB wired:

| Model, disk, setting | Prompt t/s | Generation t/s | Wall time | Max RSS | Max wired |
| --- | ---: | ---: | ---: | ---: | ---: |
| Qwen3-32B Q4_K_M (18.40 GiB, dense), external, CPU only | 0.32 | not reached | 1202 s (time limit) | 13.2 GB | 1.9 GB |
| Qwen3-30B-A3B Q4_K_M (MoE), internal, CPU only | 3.12 | 1.50 | 124 s | 12.7 GB | 1.8 GB |
| Qwen3-32B Q4_K_M (dense), internal, CPU only | 0.28 | not reached | 1202 s (time limit) | 13.6 GB | 1.8 GB |

The internal runs followed right after copying the files (`cp`, 37 GB in 1:40 min), so part of each file was still in
the page cache; the shorter wall time of the internal MoE run (124 s vs. 934 s) is not the disk alone. Not measured:
a cold internal run (needs `sudo purge` or a reboot).

Conclusions for a 16 GB Mac:

- A MoE model bigger than memory runs from the SSD on the CPU at about 1.2–1.5 tokens/s generation: usable for short
  answers, slow for chat.
  (Superseded 2026-11-06: with llama.cpp's weight repacking switched off the same model runs at 6.66 tokens/s, see
  "Streaming a model bigger than memory: llama.cpp's weight repacking" below.)
- A dense model bigger than memory is not usable (prompt 0.3 t/s, no generated token within 20 min).
- Never hand a model bigger than memory to Metal, not even partly (`-ncmoe`): it wires almost all memory.

Rule for the app: Rule for the app: never offload more layers to Metal than fit
into the working set (`recommendedMaxWorkingSetSize` minus KV cache and reserve); the rest stays on the CPU (mmap).

A first attempt at 13:53 (Qwen3-32B, `-ngl 99`, prompt 32: 0.25 t/s) is void: a VulKara test and a second copy of the
benchmark ran at the same time (swap 39.7 GB).


## Speculative decoding without a draft model (n-gram), 2026-11-05

Mac mini M4 16 GB, llama.cpp b12131, Ministral 3 8B Instruct Q4_K_M on Metal, context 8192, temperature 0, seed 1, six
tasks each once per mode, the server restarted per mode (no answer remembered from an earlier run). Tokens per second
of the answer; drafts accepted of proposed; "=" the answer is the same as without, "≠" it is not.

| Task | none | ngram-mod | ngram-map-k | ngram-cache |
|---|---|---|---|---|
| Java refactor | 18.0 | 18.0 = | 17.4 (5/96) = | 21.9 (64/206) ≠ |
| Python, new | 19.2 | 18.4 = | 18.0 = | 18.3 (51/425) ≠ |
| SQL change | 19.4 | 18.7 = | 18.0 = | 24.8 (35/88) = |
| JSON extract | 19.3 | 18.6 = | 18.0 = | 18.5 (26/195) = |
| Summary (German) | 19.4 | 19.1 = | 18.0 = | 21.1 (41/192) ≠ |
| Free text (German) | 19.3 | 18.3 = | 18.0 = | 18.0 (11/257) ≠ |
| **all** | **19.1** | **18.4** | **17.9** | **19.4** |

- ngram-mod and ngram-map-k found hardly anything to propose and cost 4–6 %.
- ngram-cache was faster where answers repeat the prompt (SQL +28 %, Java +22 %), slower elsewhere, +1.6 % in all.
- ngram-cache gave other answers in four of six tasks at temperature 0, from an early point on (e.g. character 113 of
  367 in the summary: "Netzkosten und Biomassepreise" against "Netzentgelte und Biomassekosten"). Whether that comes
  from the different arithmetic of checking several tokens at once or from a flaw of the method is **not settled**.
- First run (same prompt twice): ngram-simple 13 tokens/s instead of 19; ngram-mod 89 tokens/s only because it had
  kept the identical answer of the run before.

## Streaming a model bigger than memory: llama.cpp's weight repacking, 2026-11-06

Mac mini M4 16 GB, llama.cpp b12131 `llama-server --device none -ngl 0 -c 4096`, Qwen3-30B-A3B Q4_K_M (18.5 GB, MoE)
on the external USB SSD (APFS volume disk5 on the physical disk4), 64 tokens generated; reads counted over 10 s while
it answered (vm_stat Pageins/Swapins/Swapouts, iostat).

| | repacking on (llama.cpp's default) | `--no-repack` |
| --- | ---: | ---: |
| loaded after | 109 s | 30 s |
| generation | 2.89 tokens/s | 6.66 tokens/s |
| prompt | 0.87 tokens/s | 1.25 tokens/s |
| read from the model file (Pageins) | 0 MB/s | 205 MB/s |
| external SSD (disk4) | 0 MB/s | 195 MB/s |
| internal SSD (disk0) | 412 MB/s | 0.4 MB/s |
| swap in / out | 218 / 213 MB/s | 0 / 0 MB/s |
| phys_footprint / RSS | 18 GB / 4.7 GB | 489 MB / 12.4 GB |

With repacking llama.cpp copies the weights into memory of its own (for the CPU's kernels): they are no longer a mapped
file, and macOS swaps them to the internal SSD, which then writes about 210 MB/s while the model answers. The model's
own drive is not read at all. Without repacking the weights stay mapped and are read from the drive they are on, with no
swap, at 2.3 times the speed. The earlier read rates "from the internal SSD" (overview of loaded models, 2026-11-06)
were the swap. The spike's 1.24–1.5 tokens/s (llama-bench, prompt 32, 8 tokens) are not comparable with these
(llama-server, 64 tokens).

## gpt-oss-20b and the Harmony format, 2026-11-06

Mac mini M4 16 GB, okAI-helper's server, llama.cpp b12131, `ggml-org/gpt-oss-20b-GGUF` MXFP4 (12.11 GB, MoE, 3.6 B
active): the plan streams it on the CPU (needs 12.15 GB, Metal's share is 11.8 GB), so with `--no-repack`.

| Test | Result |
| --- | --- |
| answer with thinking | content `391`, thinking apart in `reasoning_content`, no raw Harmony tokens (`<\|channel\|>` …); 21.9 tokens/s |
| `chat_template_kwargs.reasoning_effort` | low: 69 characters of thinking, 38 tokens; high: 472 characters, 149 tokens |
| tools | `finish_reason: tool_calls`, `get_weather({"city":"Berlin"})`; the answer after the tool result is right |
| streaming | 51 events, content and thinking apart, no raw tokens |

llama.cpp renders and parses Harmony itself: OpenAI's Harmony library is not needed for GGUF. Not tested: gpt-oss on MLX.

## Speech: speech recognition (STT) and speech output (TTS), 2026-11-06

Mac mini M4 16 GB. Test audio: FLEURS dev ([google/fleurs](https://huggingface.co/datasets/google/fleurs), CC BY 4.0),
the first 120 clips each in German (24 min) and English (19 min), read speech, 1 to 35 s per clip. Every clip is sent on
its own (as in push-to-talk); the first clip is a warm-up and not counted. Error rate (WER/CER, jiwer 4.0.0) against the raw
transcription, both in lower case without punctuation. Digits and spelled-out numbers count as different words for
every engine. Whisper got the language (`de`/`en`); Parakeet detects it itself. Weights came from the page cache (not cold).
Memory: `/usr/bin/time -l` peak footprint, whisper.cpp via `footprint` on the server process.

| Engine, model | RTF de / en | median per clip | WER de / en | CER de / en | peak memory | load |
| --- | --- | --- | --- | --- | --- | --- |
| whisper.cpp v1.9.5 (Metal, own build), large-v3-turbo q8_0 (874 MB), `whisper-server` | 0.083 / 0.102 | 0.99 s | 4.8 / 5.0 % | 1.5 / 2.2 % | 1.1 GB | 0.4 s |
| mlx-whisper 0.4.3, large-v3-turbo (1.6 GB) | 0.078 / 0.093 | 0.92 s | 4.6 / 5.1 % | 1.1 / 2.2 % | 3.4–4.0 GB | 1.6 s |
| parakeet-mlx 0.7.0, parakeet-tdt-0.6b-v3 (2.5 GB, bf16 in memory) | **0.024 / 0.025** | **0.26 s** | 4.8 / 6.3 % | 1.5 / 2.4 % | 12.4 GB; **3.0 GB** with `mx.set_cache_limit(512 MB)`, 2.5 GB with 0 | 0.1 s |

RTF = compute time / audio length (0.024: one minute of speech in 1.4 s). Parakeet's MLX peak was only 2.1–2.7 GB
(`mx.get_peak_memory`); the rest of the 12.4 GB was MLX's buffer cache. A cache limit of 512 MB keeps the speed (RTF
0.024/0.026) and the same text; 0 costs 20–25 % speed.

TTS, Piper 1.8.0 (onnxruntime 1.31.1, CPU), 10 sentences from FLEURS per language:

| Voice | RTF | load | peak memory | round trip WER (whisper.cpp / mlx-whisper / parakeet) |
| --- | --- | --- | --- | --- |
| de_DE-thorsten-medium (63 MB, CC0) | 0.021 | 0.8 s | 0.49 GB | 4.5 / 5.0 / 3.5 % |
| en_US-kristin-medium (64 MB, public domain) | 0.021 | 0.8 s | 0.49 GB | 4.0 / 4.5 / 8.1 % |

Round trip = Piper speaks the sentence, the STT engine writes it down again (how understandable the voice is).

Notes: whisper.cpp publishes no macOS programs, only an xcframework ([release b5454](https://github.com/ggml-org/whisper.cpp/releases/tag/b5454)).
The build takes 24 s on the mini with Homebrew's cmake. `whisper-server` has `--inference-path`, so an OpenAI-style path
is possible. mlx-whisper pulls in torch (132 MB). parakeet-mlx has no prompt, but it can boost a word list (`boost.words`, only with beam decoding; the measurements
above used greedy decoding; beam and boosting are measured below); it also has `transcribe_stream` for live text. Not tested: the user's own voice and microphone, a cold start.

### Built into the app: the gateway's /v1/audio/transcriptions, 2026-11-06

Test DMG on the Mac mini, the first 20 FLEURS dev clips per language sent with curl to the app's gateway (port 1235, with
the server key), **no language given**. Time per clip includes curl and the gateway; the first clip includes loading
the model. Word list (prompt): `Kubernetes, Photosynthese, Mitochondrien, Quantenmechanik` (words FLEURS does not contain:
this measures what the list costs, not what it gains).

| Engine | prompt | WER de / en | median per clip | first clip |
| --- | --- | --- | --- | --- |
| whisper.cpp (`-l auto`) | – | 3.2 / 3.8 % | 1.98 s | 6.9 s |
| whisper.cpp (`-l auto`) | word list | 3.5 / 4.0 % | 1.96 s | 2.6 s |
| Parakeet, greedy | – | 4.6 / 5.7 % | 0.33 s | 0.4 s |
| Parakeet, beam (`patience=2`) with the boosted words | word list | 4.6 / 5.5 % | 0.82 s | 1.3 s |

Whisper's language detection found German and English on every clip. Parakeet's beam with a word list costs 2.5× the
time of greedy decoding.

parakeet-mlx's beam (`Beam()`, beam_size 5, patience 1) cut the last word on a Piper sentence (`… und Quantenmech`), with
or without boosted words; it stops once 5 hypotheses finish. `patience=2.0` keeps the word; on the 40 FLEURS clips it
gives the same WER and time as patience 1 (beam with boosted words, direct in Python: 4.6 / 5.5 %, median 0.70 / 0.65 s;
greedy 4.6 / 5.7 %, 0.26 / 0.25 s). The app's server uses patience 2.

The app's own path (IPC `speech:transcribe`, a Piper WAV of a sentence with Kubernetes, Mitochondrien and Quantenmechanik):
both engines wrote all three words without a list. With the profile's list whisper.cpp wrote "Kluster" for "Cluster" and
"mitochondrien" in lower case: a whisper prompt steers the style as well, so a list is worth it only for words the engine
gets wrong without it (one sentence, not measured over many). With patience 2 Parakeet wrote the whole sentence with
the list too.

whisper-server v1.9.5 has no key, answers every origin (`Access-Control-Allow-Origin: *`) and has `POST /load` (loads
any model file by path). The app starts it with a secret `--request-path` (new per start, prefixed to every path):
without it `/`, `/health`, `/load` and the inference path answer 404 (checked on the mini).

## Embeddings with llama-server: bge-m3, 2026-11-06

Mac mini M4 16 GB, llama.cpp b9660, `gpustack/bge-m3-GGUF` Q8_0 (0.63 GB), `-c 8192 --jinja -ngl 999`, German Markdown
from `plans/` cut by bge-m3's own tokenizer (`/tokenize`, `/detokenize`).

| Start | 302 tokens | 702 | 1502 | 3002 | 10 × 1000 in one request | RSS | wired over baseline (peak) |
|---|---|---|---|---|---|---|---|
| as the app started it (no `--embeddings`) | 501 | 501 | 501 | 501 | 501 | 0.76 GB | – |
| `--embeddings` (ub 512) | 0.20 s | 500 | 500 | 500 | 500 | 1.06 GB | – |
| `--embeddings -b 2048 -ub 2048` | 0.10 s | 0.22 s | 0.52 s | 500 | 3.0 s | 2.7 GB | +0.33 GB |
| `--embeddings -b 8192 -ub 8192` | 0.10 s | 0.22 s | 0.52 s | 1.25 s | 3.2 s | 4.7 GB | +0.74 GB |

An encoder takes each input in one micro-batch: the 500 says "input (702 tokens) is too large to process. increase the
physical batch size". About 3,100-3,300 tokens/s for ten 1000-token inputs. The app now starts embedding models with
`--embeddings -b N -ub N`, N the context, at most the trained one. The memory plan counted 1.9 GB for bge-m3 at 8192
(weights, KV cache, 0.5 GB compute) while the process holds 4.62 GiB, little of it wired. It now counts an encoder's
compute buffer as 0.75 GiB + 0.35 MiB per micro-batch token (fitted to the two runs: 1.32 GiB at 2048, 3.28 GiB at
8192, after weights 0.59 GiB and KV cache 0.75 GiB), 4.89 GiB for bge-m3 at 8192.

## MoE experts streamed into a fixed RAM cache (PoC tools/moestream), 2026-11-07

Mac mini M4 16 GB, Qwen3-30B-A3B Q4_K_M (17.28 GiB, 48 layers, 128 experts, 8 per token) on the external USB SSD
(/Volumes/2TB_FangX), `tools/moestream/moestream.m` built on the mini: the non-expert weights (0.93 GiB) read into
memory once, the experts per layer in an LRU cache of fixed size, each miss read with `pread` (three slabs per expert,
in parallel). It reads only (no matmul), so its tokens/s are the ceiling the SSD allows; the routing is synthetic
(uniform: worst case; skew: Zipf over a fixed order per layer), so the hit rates are not the model's. 64 tokens per
run, "warm" = the second half. Checked first: self-test (synthetic GGUF: offsets, LRU order, slab bytes) and 864 slabs
of the real file byte for byte against mmap, with and without F_NOCACHE: all equal.

| Cache | Reads | Routing | Hit rate | Read per token | SSD read | Tokens/s (ceiling) | Footprint | Page cache during the run |
| ---: | --- | --- | ---: | ---: | ---: | ---: | ---: | --- |
| 7.93 GiB (61 slots) | F_NOCACHE | uniform | 47.5 % | 0.575 GB | 0.46 GB/s | 0.80 | 8.93 GiB | 2.65 → 2.51 GiB (does not grow) |
| 8.97 GiB (69 slots) | F_NOCACHE | uniform | 53.7 % | 0.508 GB | 0.46 GB/s | 0.91 | 9.97 GiB | 2.51 → 1.85 GiB |
| 7.93 GiB | F_NOCACHE | skew | 77.5 % | 0.247 GB | 0.46 GB/s | 1.85 | 8.93 GiB | 1.85 → 1.89 GiB |
| 8.97 GiB | F_NOCACHE | skew | 81.4 % | 0.204 GB | 0.45 GB/s | 2.23 | 9.97 GiB | 1.89 → 1.86 GiB |
| 7.93 GiB | page cache | uniform | 47.5 % | 0.575 GB | 0.45 GB/s | 0.78 | 8.93 GiB | 1.86 → max 5.60 GiB |
| 8.97 GiB | page cache | uniform | 53.7 % | 0.508 GB | 0.21 GB/s | 0.41 | 9.97 GiB | 3.86 → max 5.52 GiB |
| 7.93 GiB | page cache | skew | 77.5 % | 0.247 GB | 0.42 GB/s | 1.68 | 8.93 GiB | 3.18 → max 5.57 GiB |
| 8.97 GiB | page cache | skew | 81.4 % | 0.204 GB | 0.41 GB/s | 2.02 | 9.97 GiB | 4.13 → max 5.66 GiB |
| 7.93 GiB as a Metal buffer | F_NOCACHE | uniform | 47.5 % | 0.575 GB | 0.46 GB/s | 0.80 | 8.97 GiB | wired 1.62 → 9.67 GiB |

- With F_NOCACHE the memory is exactly the cache plus the resident weights (8.93 GiB for 8 GB), the page cache does not
  grow: about 6 GB stay for the context and macOS. Without it the page cache grows by up to 3.7 GiB on top and the
  reads got slower (0.21 GB/s once); F_NOCACHE is the better choice.
- 1 GiB is too small for this model: 8 experts in each of 48 layers need about 1.06 GB (2.77 MB per slot and layer).
- A cache of 8.97 GiB as one Metal buffer was refused (`newBufferWithLength` nil; 7.93 GiB worked). The Metal buffer
  wires the whole cache (+8.05 GiB wired), the guard (11 GiB) did not trigger. Why 8.97 GiB fails is open (a limit per
  buffer?).
- The external SSD gives 0.46 GB/s with these reads: at 0.2–0.6 GB per token that is 0.8–2.2 tokens/s before any
  compute. The same model with mmap (`--no-repack`, page cache ~12 GB) generated 6.66 tokens/s on 2026-11-06, so the
  real routing hits far more than the synthetic one (6.66 tokens/s at 0.2 GB/s is about 30 MB per token). Open: the
  model's real hit rate in a cache of 8 GB (needs the router's choices: llama.cpp PR #25294 or a trace), the internal
  SSD (not tested: 25 GB free for an 18.5 GB file).
- llama.cpp PR #25294 measured: see below.

### The same with llama.cpp PR #25294 (`--moe-stream-cache`), 2026-11-07

PR head 1248fd8 (unmerged), built with `tools/moestream/llama-pr25294-fnocache.patch` (macOS has no O_DIRECT: the
patch opens the streamed file with F_NOCACHE; without it `--moe-stream-direct` falls back to buffered reads on macOS),
`-DLLAMA_OPENSSL=OFF -DCMAKE_OSX_DEPLOYMENT_TARGET=14.0` (the first build linked Homebrew's OpenSSL and did not start on
the mini). `llama-server -c 4096 --no-warmup`, the same model and SSD, one prompt (18 tokens), 64 tokens at
temperature 0; RSS and system wired sampled every 0.5 s, a guard kills the server above 11 GiB wired.

| Run | Generation | Prompt | Max RSS | Max wired | Answer |
| --- | ---: | ---: | ---: | ---: | --- |
| mmap, CPU, `--no-repack` (no streaming) | 5.66 / 5.88 tokens/s (2 runs) | 1.01 / 0.91 | 11.96 GiB | 1.68 GiB | A |
| stream, 8 GiB cache, F_NOCACHE, CPU, `--no-repack` | **6.38 tokens/s** | 1.74 | **8.18 GiB** | 1.67 GiB | A (identical) |
| stream, 8 GiB, F_NOCACHE, CPU (repacking on) | 4.67 | 1.13 | 8.85 GiB | 1.71 GiB | B |
| stream, 9 GiB, F_NOCACHE, CPU (repacking on) | 5.61 | 1.13 | 10.41 GiB | 1.71 GiB | B |
| stream, 8 GiB, page cache, CPU (repacking on) | 4.87 | 1.15 | 6.48 GiB + page cache | 1.70 GiB | B |
| stream, 8 GiB, F_NOCACHE, Metal `-ngl 99` | 4.70 | 1.11 | 9.57 GiB | **10.99 GiB** | C |
| stream, 9 GiB, F_NOCACHE, Metal | stopped by the guard at 11.57 GiB wired while loading | | | | – |
| stream, 1 GiB | refused: "need an expert cache of at least 3*n_expert_used slots (have 8, need 24)"; with `-b 1 -ub 1` `GGML_ASSERT(n_outputs_max <= cparams.n_outputs_max)` | | | | – |

- Streamed into 8 GiB the model answered faster than with mmap (6.38 vs 5.66–5.88 tokens/s) with 3.8 GiB less
  memory — **corrected in round 4: that run came right after an mmap run of the same file, whose pages were still in
  the page cache (F_NOCACHE does not evict pages already cached); from a cold page cache it gave 2.50 tokens/s. The other external runs of this table did not record the page cache
  and may be partly warm too (the tool's buffered runs before them left up to 5.6 GiB of this file cached).**
  The answer is the same text as mmap's (SHA-1 of the content equal). Repacking costs here as with mmap
  (2026-11-06): 4.67 tokens/s; its answer B differs from A but is the same in all repacked runs (other CPU kernels,
  not the streaming).
- Metal brings nothing (4.70 tokens/s, the SSD is the limit) and wires the cache: 10.99 GiB with 8 GiB, 9 GiB went
  over the guard. Streaming stays on the CPU.
- 1 GiB does not work with this PR; at least 24 slots per layer (~3.2 GB for this model) at the default batch size.
- Not shown in the log (INFO level hidden): whether F_NOCACHE was active; the fallback warning did not appear.

### Round 3: internal SSD, a long prompt, gpt-oss-20b (PR #25294, CPU, `--no-repack`), 2026-11-07

Same build and harness, 64 tokens at temperature 0; "long" = 466 prompt tokens. Page cache = file-backed pages
(vm_stat) at the start and at most during the run. Order: the streamed runs before mmap (mmap fills the page cache).

| Model, SSD, prompt | Run | Generation | Prompt | Max RSS | Page cache start → max | Answer |
| --- | --- | ---: | ---: | ---: | --- | --- |
| Qwen3-30B-A3B, internal, short | stream 8 GiB | **15.31** | 4.68 | 9.33 GiB | 10.64 → 10.64 GiB | same |
| | mmap | 11.60 | 2.44 | 11.45 GiB | 2.22 → 12.61 GiB | same |
| Qwen3-30B-A3B, internal, long | stream 8 GiB | 9.59 | 17.04 | 9.48 GiB | 2.22 → 2.29 GiB | same |
| | mmap | 9.38 | 19.11 | 11.87 GiB | 12.57 → 12.59 GiB | same |
| Qwen3-30B-A3B, external, long | stream 8 GiB | 2.98 | 11.55 | 9.29 GiB | 12.53 → 12.53 GiB | same |
| | mmap | 4.58 | 8.90 | 11.90 GiB | 3.17 → 12.53 GiB | same |
| gpt-oss-20b MXFP4 (12.1 GB, fits), internal | stream 4 GiB | 7.07 | 4.14 | **5.36 GiB** | 12.65 → 12.65 GiB | same |
| | stream 8 GiB | 18.55 | 5.98 | 9.73 GiB | 6.80 → 6.93 GiB | same |
| | mmap | **26.56** | **40.07** | 11.51 GiB | 2.85 → 12.35 GiB | same |

- F_NOCACHE works with the patch: in every streamed run the page cache does not grow (at most +0.13 GiB), with mmap it
  fills to ~12.5 GiB. The answers are identical within each prompt and model (SHA-1 of the content).
- From the internal SSD streaming into 8 GiB beats mmap for the model bigger than memory: 15.31 vs 11.60 tokens/s
  (short prompt), 9.59 vs 9.38 (long), with 2.1–2.4 GiB less memory; the long prompt's prefill is slightly slower
  (17.0 vs 19.1 tokens/s).
- From the external SSD with the long prompt mmap was faster (4.58 vs 2.98 tokens/s generation), with the short prompt
  streaming (6.38 vs 5.66, round 2). Why is open (mmap had ~12.5 GiB of page cache against the 8 GiB cache; the PR's
  eviction after a long prefill?).
- A model that fits in memory loses with streaming: gpt-oss-20b 26.56 → 18.55 tokens/s (8 GiB) and the prompt 40 → 6
  tokens/s. What it gains is memory: with 4 GiB it runs in 5.36 GiB instead of 11.51 (7.07 tokens/s), room for a second
  model or a long context.

### Round 4: why mmap beat streaming from the external SSD (PR #25294, CPU, `--no-repack`), 2026-11-07

Same harness, external SSD, `-lv 3` for the server's own timings; "cold" = the page cache held no pages of this file.

| Run | Prompt | Generated | Generation | Prompt t/s | Max RSS | Page cache start → max |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| stream 8 GiB (repeat of round 3) | 466 | 64 | 3.00 | 11.60 | 9.48 GiB | 12.16 → 12.14 (other file) |
| stream 8 GiB | 466 | 256 | 5.87 | 11.56 | 9.48 GiB | 2.73 → 2.80 |
| stream 9 GiB | 466 | 64 | 3.51 | 11.00 | 10.26 GiB | 2.66 → 2.73 |
| stream 8 GiB, page cache (no F_NOCACHE) | 466 | 64 | 3.00 | 11.71 | 6.60 GiB | 1.93 → 7.04 |
| stream 8 GiB, short prompt, cold | 17 | 64 | 2.50 | 0.58 | 8.57 GiB | 3.92 → 4.99 |
| mmap, cold | 466 | 64 | 4.33 | 8.70 | 11.83 GiB | 3.73 → 12.65 |
| mmap, warm (right after) | 466 | 64 | 3.52 | 8.80 | 11.72 GiB | 12.51 → 12.66 |
| mmap | 466 | 256 | 7.92 | 5.46 | 11.35 GiB | 12.13 → 12.38 |

- No riddle in the streaming itself: both ways warm up during the first tokens (stream 3.00 → 5.87, mmap 4.33 → 7.92
  tokens/s with 256 instead of 64 tokens), and mmap stays about 1.35–1.45 times faster from this drive. With a slow
  drive (0.46 GB/s) the size of the cache decides: mmap uses ~12 GiB of page cache, the stream 8 GiB (9 GiB: +17 %).
- Round 2's "faster than mmap from the external SSD" was a warm page cache (see the correction there). From the
  internal SSD (round 3, both cold for their file) streaming was ahead: 15.31 vs 11.60 tokens/s. Single runs vary by up
  to ±20 % (mmap 4.33 / 3.52), so differences below that are noise.
- Prefill of a long prompt: the stream is faster (11.6 vs 8.7 tokens/s).
- F_NOCACHE costs no speed (3.00 both) and keeps 5 GiB of page cache from growing.
- Not stats from the PR (`print_stats` hit rates): the server did not print them at `-lv 3`.

### Round 5: llama-stream inside the app (test DMG 3e6d968+, Mac mini M4 16 GB), 2026-11-07

The app's own llama-stream (tools/build_llama_stream.sh: PR head 1248fd8 + the F_NOCACHE patch, static, without the web
UI) packaged in the test DMG, setting "expert cache" on, Qwen3-30B-A3B Q4_K_M from the internal SSD, context 8192,
one chat of 64–110 tokens through the app (`chat:send`), then a small model loaded beside it. Memory sampled every
second; wired before the load 1.95 GiB.

| Expert cache | Plan (need) | Generation t/s | Max RSS llama-stream | Max wired | Small model beside it |
| ---: | ---: | ---: | ---: | ---: | --- |
| 8 GB | 10.75 GiB | 14.06 | 8.47 GiB | 2.12 GiB | Qwen3-Embedding-0.6B (plan 5.0 GiB): does not fit, the stream goes (right) |
| 6 GB | 8.75 GiB | 11.79 | 7.68 GiB | 2.08 GiB | Qwen3-0.6B MLX (plan 4.3 GiB): 0.07 GB over the budget, the stream goes (right) |
| 5 GB | 7.75 GiB | 9.16 | 6.79 GiB | 2.10 GiB | Qwen3-0.6B MLX: both stay loaded |

- The app starts llama-stream with `--device none -ngl 0 --no-repack --moe-stream-cache N --moe-stream-direct
  --no-warmup` (seen in `ps`); nothing goes to Metal (wired +0.17 GiB at most).
- The plan's need is above the measured RSS by about 1–2 GiB (the KV cache of 8192 tokens is counted in full): safe.
- A model with a fixed cache shares the Mac by the budget like a model on the CPU; mapped from the file it still runs
  alone. Speed falls with the cache: 8 GB 14.1, 6 GB 11.8, 5 GB 9.2 tokens/s (single runs, ±20 %).

### Round 6: cache hit rates and a long session (Mac mini M4 16 GB), 2026-11-07

Qwen3-30B-A3B Q4_K_M from the internal SSD, CPU (`--device none -ngl 0 --no-repack --no-warmup --moe-stream-direct`),
context 8192, temperature 0. llama-server never prints the PR's cache statistics; only `llama-completion` does
(`common_perf_print`), and only at `-lv 4`, because common/log.cpp shows library INFO messages at trace verbosity. So the
hit rates come from `llama-completion` built from the same tree (PR head 1248fd8 + F_NOCACHE), 256 tokens, one prompt.

| Expert cache | Generation t/s | Hit rate | Misses (of which cold) | Load stall | Preloads ready on arrival |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 5 GB | 8.92 | 85.65 % | 14409 (4423) | 20.8 s of 33.0 s | 439 of 1796 |
| 6 GB | 11.21 | 89.66 % | 10380 (4423) | 15.0 s of 27.0 s | 719 of 1608 |
| 8 GB | 16.56 | 94.29 % | 5738 (4423) | 7.6 s of 19.5 s | 28 of 1232 |

- The 4423 cold misses are the first touch of every expert used; at 8 GB three quarters of all misses are those, at
  5 GB the cache keeps throwing experts out (10 000 misses after the first touch).
- The PR's preloading helps little at 8 GB (28 of 1232 ready in time).

Long session: the app's llama-stream (runtimes/llama-stream, `--moe-stream-cache 8`), 12 requests in a row on different
topics (history, code, chess, biology, networking, literature, medicine, grammar, finance, astronomy), 128 tokens each,
no prompt cache:

- Generation 10.99 (the code question) to 22.50 tokens/s, no decline over the session; the first question asked again
  as the 12th: 17.57 instead of 16.61 tokens/s (with changing topics little of the start stays in the cache).
- Max RSS 9.74 GiB (one request: 8.47–8.77 GiB), wired 2.07 GiB; the plan counts 8 + 1.5 GB + KV + 0.5 GB. Whether RSS
  keeps growing in sessions longer than 12 requests is not measured.

### Round 7: cold start after `sudo purge` (Mac mini M4 16 GB), 2026-11-07

The page cache emptied by `sudo purge` (file-backed pages 0.61–0.75 GiB before the start), then the app's llama-stream
with 8 GB of cache, two requests (128 tokens), then the same binary without streaming (mmap). The stream reads past the
page cache (F_NOCACHE), so mmap after it still starts nearly cold (0.74–1.82 GiB cached). Two cold runs; two more runs
right after them were warm (12 GiB cached) and are listed for comparison.

| Run | Load | 1st request t/s | 2nd request t/s | Max RSS | Page cache after |
| --- | ---: | ---: | ---: | ---: | ---: |
| stream 8 GB, cold (2 runs) | 2–3 s | 15.39 / 15.46 | 18.05 / 18.25 | 9.73 GiB | 0.74 / 1.82 GiB |
| mmap, nearly cold (2 runs) | 14 s | 6.35 / 14.29 | 22.99 / 33.59 | 11.88 / 12.01 GiB | 11.98 / 12.32 GiB |
| stream 8 GB, page cache warm | 1 s | 14.97 / 15.54 | 18.39 / 19.05 | 8.77 GiB | 3.4–3.5 GiB |
| mmap, page cache warm | 13–14 s | 13.70 / 14.08 | 33.20 / 33.57 | 12.04 GiB | 12.0–12.2 GiB |

- The stream does not care whether the page cache is cold or warm: 15 then 18 tokens/s either way, ready in 1–3 s.
- mmap needs 14 s to load and is slow on the first request when cold (6.35 tokens/s in one run); once the page cache
  holds 12 GiB of the file it reaches 33 tokens/s, almost twice the stream with 8 GB of cache, but takes all free memory.
- So: the stream wins on a cold start and when memory is needed for something else (fixed 9–10 GiB instead of 12 GiB);
  a model used alone and often answers faster mapped once the page cache is warm.

### Round 8: a long prompt with the model alone (Mac mini M4 16 GB), 2026-11-07

Qwen3-30B-A3B Q4_K_M from the internal SSD, the app's llama-stream binary, context 8192, a prompt of 5980 tokens
(110 sentences on MoE topics, then "summarize"), 128 tokens, no prompt cache, two requests per run. Is mmap still ahead
when the KV cache and the page cache compete for memory?

| Run | Load | Prompt t/s | Generation t/s | Wall per request | Max RSS | Page cache after |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| stream 8 GB | 2 s | 15.95 / 15.93 | 8.04 / 8.74 | 391 s | 9.48 GiB | 2.01 GiB |
| mmap (cache filling) | 14 s | 36.47 / 37.09 | 6.95 / 6.70 | 181 s | 11.92 GiB | 11.74 GiB |
| mmap (cache warm) | 9 s | 36.89 / 37.20 | 7.07 / 6.78 | 180 s | 11.93 GiB | 11.73 GiB |

- With a long prompt mmap is over twice as fast end to end: the prompt runs at 37 against 16 tokens/s (a batch of
  prompt tokens touches nearly every expert, which the stream has to read through its 8 GB cache again and again).
- Generation at 6000 tokens of context is slightly faster streamed (8.0–8.7 against 6.7–7.1 tokens/s).

### Round 9: a second MoE model bigger than memory, GLM-4.7-Flash (Mac mini M4 16 GB), 2026-11-07

unsloth/GLM-4.7-Flash-GGUF Q4_K_M (18.3 GB, MIT, revision 0d32489, SHA-256 checked) on the internal SSD. llama-stream
(PR head 1248fd8) runs it without changes. Hit rates with `llama-completion -lv 4` (256 tokens), then the app's binary,
short prompt, 128 tokens, two requests per run:

| Expert cache | Generation t/s | Hit rate | Misses (of which cold) | Load stall |
| ---: | ---: | ---: | ---: | ---: |
| 6 GB | 9.27 | 84.72 % | 7377 (2563) | 17.4 s of 31.4 s |
| 8 GB | 11.54 | 90.28 % | 4693 (2563) | 11.4 s of 26.3 s |

| Run | Load | 1st request t/s | 2nd request t/s | Max RSS | Page cache after |
| --- | ---: | ---: | ---: | ---: | ---: |
| stream 8 GB | 2 s | 11.27 | 14.74 | 9.52 GiB | 2.87 GiB |
| mmap (cache filling) | 15 s | 9.49 | 10.77 | 12.02 GiB | 12.48 GiB |
| mmap (cache warm) | 10 s | 8.89 | 11.28 | 11.87 GiB | 12.21 GiB |

- Unlike Qwen3-30B-A3B, GLM-4.7-Flash streamed stays ahead of mmap even with the page cache warm (14.7 against 11.3
  tokens/s), in 2.5 GiB less memory. Its hit rate at 8 GB is lower (90 against 94 %).
- So whether mmap or the stream is faster for a model alone depends on the model and the prompt length; no simple rule
  by the pool (plans/MOE_STREAMING_PLAN.md, proposal 2) holds. Answers were not compared with mmap in this round.

### Round 10: 40 requests in a row, and GLM-4.7-Flash's answers streamed against mapped (Mac mini M4 16 GB), 2026-11-08

The app's llama-stream binary, context 8192, 128 tokens per request, temperature 0, no prompt cache.

- Qwen3-30B-A3B, 8 GB of cache, 40 requests on 20 topics (each topic twice): RSS after each request grows by about
  0.01 GiB per request up to 9.03 GiB (request 21–23), then falls back to 8.54–8.78 GiB and stays at 8.78 GiB for the
  last 10 requests; generation 12.85–21.16 tokens/s, the second pass of every topic within ±0.6 tokens/s of the first
  (binary search 13.42 / 13.41, TCP 12.98 / 12.85, plate tectonics 21.16 / 21.14). No growth without bound, no slowdown.
- GLM-4.7-Flash Q4_K_M, three prompts (MoE in English, a Python function, a German grammar question): the same answer
  streamed (cache 8 GB) and mapped, SHA-1 of the text equal for all three; streamed 7.36–10.93, mapped 5.75–8.46 tokens/s.

## Guard monitor, step 0: per-process sources without root (Mac mini M4 16 GB), 2026-11-08

plans/MONITOR_PLAN.md step 0. tools/probe/okai-probe.c (`proc_pid_rusage(RUSAGE_INFO_V4)`, built with clang, arm64,
macOS 14) and `ioreg`, sampled once a second while llama-stream answered: Ministral 3 8B Q4_K_M on Metal (`-ngl 999`,
19.89 tokens/s) and Qwen3-30B-A3B streamed with 8 GB of cache (17.77 tokens/s).

- **CPU per process**: `ri_user_time + ri_system_time` are mach time units, not ns: times `mach_timebase_info`
  (125/3 on the M2 and the M4) they equal `ps -o time` (0.62 s against 0:00.62). Metal engine 2–3 % of one core while it
  generates, the streamed one 331–371 %.
- **Disk per process**: `ri_diskio_bytesread` per second against `iostat -I disk0` of the whole drive while streaming:
  1708 / 1766, 1672 / 1670, 920 / 858, 577 / 587, 522 / 538 MB/s — within a few percent (the drive also serves other
  processes). Metal engine after loading: 0 MB/s.
- **GPU per process**: `ioreg -r -c IOGPUDeviceUserClient -l -w0`, one block per Metal client (`AGXDeviceUserClient`);
  in a block `"AppUsage"` comes before `"IOUserClientCreator" = "pid N, name"`, so a parser has to pair them per block
  (the first parser paired line by line and found 0). `accumulatedGPUTime` is in ns: +2.162 s of GPU time in 2.216 s
  (`lastSubmittedTime` +2.216 s) = 97.5 %, while "Device Utilization %" showed 98.
- **Memory**: `ri_phys_footprint` of the Metal engine 0.64 GiB against RSS 5.41 GiB (the weights live in Metal buffers,
  as engine_stats.ts notes); streamed: footprint 9.30 GiB, RSS 9.05 GiB.
- **Cost per sample** (mean, ms): probe 7–10, GPU clients (ioreg) 19–25, "Device Utilization %" (ioreg) 21–24, ps 8–11,
  iostat 11–15. One monitor tick with probe + GPU clients + utilization ≈ 50 ms, mostly process starts.

## Guard monitor, step 1: okai-probe in the app (Mac mini M4 16 GB), 2026-11-08

plans/MONITOR_PLAN.md step 1. The test DMG (okai-probe in Resources/runtimes/probe), `engines:stats` read once a second
over the devtools port while the model answered; beside it on the mini once a second `ps -o %cpu,rss`, `iostat -I disk0`
and "Device Utilization %".

- **Ministral 3 8B Q4_K_M on Metal**: CPU 23–29 % (ps 24.5–28.9), GPU 89–100 %, mean about 96 (utilization 96–98),
  resident 7.37 GB (ps RSS 7 200 976 KB = 7.37 GB), footprint 2.23 GB, read 0–0.15 MB/s once loaded.
- **Qwen3-30B-A3B Q4_K_M streamed (mapped: the test profile has the expert cache off)**: CPU 52 % while reading the
  prompt, 366–397 % while generating (ps 361–400), GPU 0, read 528–585 MB/s at the start (iostat 501–589), then
  6–56 MB/s (iostat 6.5–51); resident 12.3–12.4 GB (ps 12.0–12.1 GB), footprint 0.92 GB; 23.4 tokens/s.
- One `engines:stats` call with the probe takes about 14 ms in the app (polled every 1.014 s with a 1 s sleep). The first
  call after a model is loaded has no rates (they need a previous sample). The test profile was unchanged afterwards.

## Guard monitor, step 5: two models side by side, streamed, the monitor's cost (Mac mini M4 16 GB), 2026-11-09

plans/MONITOR_PLAN.md step 5, test DMG of 7b8aed7, two runs. `monitor:stats` once a second over the devtools port;
beside it on the mini `top -l 2 -s 1` (CPU of the last second, every process of the app and the engines),
`iostat -I disk0` and "Device Utilization %". A streamed model runs alone by design (pool_plan.ts), so "side by side"
is Ministral 3 8B Q4_K_M on Metal (llama.cpp) with Qwen3-4B 4 bit (MLX). Activity Monitor itself was not read (no
screen recording permission on the mini); its CPU column has the same source as top.

| Phase | Model | CPU app / top | GPU app | GPU sum / device | Tokens/s |
| --- | --- | ---: | ---: | ---: | ---: |
| MLX alone, generating | Qwen3-4B | 15 / 15 % | 92 % | 66.5 / 67.3 % (with the start) | 37.9 |
| llama alone, generating (Qwen3-4B loaded, idle) | Ministral 8B | 56 / ≈ 56 % | 87–94 % | 75.7 / 84.8 % (with the start) | 18.0 |
| both answering | Ministral 8B | 32 / 31 % | 8–9 % | 92.8 / 93.5 % | 13.2 (part of it alone) |
| both answering | Qwen3-4B | 21 / 20 % | 85 % | | 18.7 |
| streamed (mapped), cache warm | Qwen3-30B-A3B | 387 % (last 15 s) | 0 | 0 / 8 % | – |

- CPU per second: the app against top within 1.8–3.4 points for the GPU models, 19 points (about 5 %) while
  streaming, mostly in the warm-up where it climbs by 100 points a second.
- GPU per process: the processes' sum matches the device's utilization (92.8 against 93.5 % with both answering; MLX
  alone 66.5 against 67.3), so mlx-lm's `accumulatedGPUTime` counts correctly. The split between two processes that
  use the GPU at once does not: Ministral got 8–9 % yet kept 13.2 tokens/s (alone it needs 88 % for 18), MLX got 85 %.
- Disk: streamed, the processes' reads match the drive (195.8 against iostat 203.9 MB/s, 700 MB/s in the warm-up).
  The probe also counts reads served from the file cache: MLX's first answer read 680–720 MB/s for 3 s while disk0 read
  3.9 MB/s.
- Memory: llama.cpp's resident (RSS) 7.20–7.40 GB as expected (7.37). MLX keeps its weights in Metal buffers: RSS
  1.2–1.3 GB, footprint 2.76 GB, the "peak memory" of the 2026-11-05 spike 2.9 GB.
- Ministral generating alone with Qwen3-4B loaded beside it: CPU 56 % and 18.0 tokens/s, against 26 % and 19.7 alone
  in memory (step 1, the spike). The cause is not measured.
- The open monitor's cost, models idle, the app's own processes (top): closed 1.8–2.5 %, open 4.0–4.2 %, so about
  1.7 points of one core.


## SteuerLLM (dense 28 B) in own low-bit quants, 2026-11-06 to 2026-11-09

Purpose: to judge whether training a mixture-of-experts model from SteuerLLM would make it streamable on the
mini (as in "MoE experts streamed into a fixed RAM cache" above). The dense quants below are the reference that
such a MoE model would have to reach in quality and speed; no MoE model was trained in these runs.

`windprak/open_steuerllm` (llama architecture, 28.02 B, 48 layers, 8 KV heads, head dim 128), quantized on the M2
24 GB from bf16 with llama.cpp b9660 and an own importance matrix (100 chunks × 512). The mixes keep every sixth
layer (5, 11, … 47) at a higher type (`--tensor-type-file`), token embeddings q4_k, output q6_k. Perplexity: eval.txt,
40 chunks × 512 tokens, CPU; the M2 reference runs (CPU, `-ngl 0`) stop after 16 chunks, so the table gives the
running value at chunk 16 as well. Speed: llama-bench pp512 / tg128, all layers on Metal.

| Quant | Size | PPL 40 chunks | PPL at chunk 16 | pp512 t/s | tg128 t/s |
| --- | --- | --- | --- | --- | --- |
| IQ2_M | 8.92 GiB | 3.153 ± 0.076 | 3.262 | 60.6 | 8.02 |
| mix1: IQ2_S, every 6th layer q4_k | 9.47 GiB | 3.845 ± 0.102 | 4.034 | 59.9 | 7.80 |
| mix2: IQ2_M, every 6th layer q5_k | 10.58 GiB | 3.107 ± 0.074 | 3.173 | 58.4 | 7.35 |
| IQ3_XXS | 10.22 GiB | **2.533 ± 0.052** | 2.605 | 59.5 | 7.95 |
| Q4_K_M (M2 only) | – | – | 2.191 | – | – |
| bf16 (M2 only) | – | – | 2.151 | – | – |

mix1 is worse than plain IQ2_M although it is larger: IQ2_S for the other layers costs more than the q4_k layers win.
IQ3_XXS is the best quant that fits; it is still clearly above Q4_K_M (2.61 against 2.19 at chunk 16).

### SteuerEx answers for the upload (steuerllm.i5.ai.fau.de)

115 questions, question text only, temperature 0, `max_tokens` 4096, llama-server `-fa on -c 7168 -np 1`, KV q8_0.
The scores from the upload are open.

| Quant | Server | Answers | finish | Mean t/s | Longest answer | Largest context |
| --- | --- | --- | --- | --- | --- | --- |
| IQ3_XXS | `-ngl 99` | 115, none empty | 112 stop, 3 length | 7.73 | 4096 (cut) | 5012 |
| mix2-IQ2_M | `-ngl 44 --no-mmap --cache-ram 0` | 115, none empty | 115 stop | 5.55 | 2133 | 4337 |

- mix2 with all layers on Metal: "Insufficient Memory / Compute error" from `-c 5120` on (512 works). Fewer layers alone
  did not help (earlier diagnosis run, its log was not kept); `-ngl 44` and `-ngl 40`, each with `--no-mmap`, run at
  `-c 7168` (diag.log, 2026-11-07). Why fewer layers alone did not help is not proven.
- mix2 the first time (`-ngl 44 --no-mmap`, default `--cache-ram 8192`): llama-server disappeared after 94 answers
  without an error or crash report. Its last log line shows the prompt cache at 70 prompts, 8070 MiB of its 8192 MiB
  limit, on top of the weights read into RAM. Memory exhaustion is likely but not proven (nothing in the system log).
  With `--cache-ram 0` the remaining 21 answers ran through.
- After a server stops, Metal frees its memory a moment later: mix2 started 0.3 s after IQ3_XXS and failed every
  answer with "Insufficient Memory"; the script now waits until the wired memory is under 4 GB.
