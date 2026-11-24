# Changelog

## 0.3.1 (2026-11-24)

- Qwen3 models that switch thinking on and off (the hybrid ones, like Qwen3 1.7B, 4B, 8B, 30B-A3B): with thinking off
  they get the values their model card gives for that mode (temperature 0.7, top-p 0.8, top-k 20, min-p 0) and a
  presence penalty of 1.5, instead of the values for thinking. Small models answered endlessly less often in a test on
  an Intel Mac (Qwen3 1.7B: 4 of 5 before, 1 of 5 with these values), but not never; a bigger model or thinking on
  answers better. The model's own settings, a prompt profile's and a program's values still come first. Qwen3-2507
  models, which have no switch, keep their maker's values.
- A model planned on the CPU (an Intel Mac, or the GPU switched off) starts with `-ngl 0` only; a second `-ngl 999`
  after it had no effect, as the GPU was off anyway.

## 0.3.0 (2026-11-23)

- Intel Macs: a download of its own, `okAI-helper-0.3.0-x64.dmg`, next to `okAI-helper-0.3.0-arm64.dmg` for Apple
  silicon (two apps, no universal app). On an Intel Mac the models run on the CPU with llama.cpp and whisper.cpp; MLX,
  MLX image models and Parakeet exist for Apple silicon only and are hidden there, as are the GPU switches. The Models
  page says how fast an Intel Mac is (measured on a MacBook Pro from 2017 with 2 cores: Qwen3 1.7B about 16 tokens per
  second, Qwen3 4B about 7), and "Runs here" shows model files up to about 3 GB. The network guard blocks the engines on
  an Intel Mac the same way as on Apple silicon (checked on a real Intel Mac). Intel support is for a limited time:
  macOS 26 Tahoe is the last macOS for Intel Macs.
- The Intel app started on a Mac with Apple silicon by mistake runs under Rosetta; it notices this and installs the
  Apple silicon engines and, with the next update, the Apple silicon app.
- The update check picks the DMG that fits the Mac.

## 0.2.0 (2026-11-22)

- MCP server (Server page, off by default): programs that speak MCP use the local models at /mcp of the app's server,
  with the same token and the same allowed devices. Tools: list_models, ask_model and embed; the prompt profiles come as
  MCP prompts. Speaks both MCP generations on one address, 2025-11-25 and 2026-07-28 (official TypeScript SDK 2.4.0).
  Checked with the MCP Inspector in both generations and with a real MCP program; a device that is not allowed gets 403.
  Tips: a new entry "MCP server" and a step-by-step guide "MCP step by step" in five languages: the five steps with
  the port that is set, examples for MCP Inspector, Claude Code, VS Code, Cursor, Continue, LM Studio, Open WebUI and
  Claude Desktop, and what 401, 403, 404 and other answers mean. The full guide is in docs/MCP.md.

## 0.1.2 (2026-11-21)

- Downloads and the update check try a failed connection again, twice, after 2 and 5 seconds (GitHub refused
  connections for hours on 2026-11-21: installing a runtime, a download from Hugging Face and the update check failed
  until they were started again). Only a connection that got no answer is tried again; a switch under Privacy and
  network that is turned off in between, or a cancelled download, stops it at once.

## 0.1.1 (2026-11-21)

- Prompt cache per model (Settings, Hardware; 2 GB by default): llama.cpp keeps earlier prompts in memory, so going back
  to another chat or program does not compute its start again (16 GB Mac mini, Ministral 8B, two chats of 5,500 tokens:
  0.07 instead of 26 seconds). Before, llama.cpp kept up to 8 GB of them outside the memory plan, and a big model's
  server could vanish without an error. Now the cache is counted in the plan and takes only the memory left beside the
  model: it never pushes a model off the GPU, and it does not count against the GPU when several models are loaded.
- The guard measured its starting point right after unloading a model, before macOS had given that memory back: after a
  model switch its limit stood above the Mac's memory and it would not have stepped in. It now waits (at most five
  seconds) until the memory is free.
- The version in the header of every page, as in VulKara: it opens the About page; when a newer version is out it reads
  "Update to …" and brings the update notice back, also after Later.
- Settings: Appearance sits next to Memory.
- Downloads show progress, speed and time left once a second, averaged, instead of with every piece of data.
- uv 0.13.6 for installing the MLX runtimes: uv 0.12.23 trusted a compromised certificate authority (GHSA-hq85-5cpj-mr62).
  The packages were and are checked against fixed checksums.

## 0.1.0 (2026-11-10)

First version.

- Model library: GGUF files and MLX folders in any number of folders, on the internal SSD or external drives (a drive
  that is not connected is skipped). Reads the shape, the vision projector and what the chat template can do (tools,
  thinking) from the files; picture models in GGUF (VulKara's) are left out.
- Memory plan per model, from measurements on a 16 GB Mac mini: on the GPU when it fits, streamed from the SSD on the
  CPU for mixture-of-experts models bigger than memory, refused for dense models bigger than memory and MLX models that
  do not fit. Metal's working set is asked from llama.cpp at start.
- Engines: llama.cpp (llama-server) and MLX (mlx-lm), one model at a time, on 127.0.0.1 with their own key. A guard stops
  an engine that holds too much wired memory; quitting the app waits until the engine is gone.
- Expert cache for SSD streaming (Settings, Hardware; off by default, experimental): a MoE model bigger than memory on the
  internal SSD reads its experts into a fixed cache instead of mapping the file, with llama-stream, llama.cpp with the
  unmerged PR #25294 and a macOS patch, built into the app (tools/build_llama_stream.sh). On a 16 GB Mac mini with 8 GB of cache
  15–18 tokens per second from a cold start (mapped: 6–14 on the first answer), in fixed memory, and other models may
  stay loaded beside it; a model used alone answers faster mapped once the page cache is warm (33 tokens per second). From an external drive the
  file stays mapped (faster there).
- Models without network (Settings → Privacy and network, on by default): every engine (llama.cpp, MLX, speech) runs
  in macOS's sandbox (sandbox-exec) and reaches only this Mac's loopback; the internet, the local network and other
  ports are blocked by the kernel. The app checks the sandbox before the first engine and starts none if it does not
  hold. Other programs still reach the models through the app's server. The app's own traffic can be turned off by
  purpose (Hugging Face; installing runtimes) and reaches only those hosts over HTTPS, every redirect checked; the
  windows load nothing from the network. The server can be limited to allowed devices (addresses or IPv4 networks;
  others get 403 before the token, shown in the server log); the guard monitor shows per engine whether its network is
  blocked.
- About page with the version and the changes; the app looks for a newer version once a day and from the menu (the
  public release list on GitHub, switchable under Privacy and network), downloads it, checks its checksum, signature and
  notarization and restarts into it.
- Chat with streaming, system prompt, folded thinking and a thinking switch. While the model works, a fixed line above the
  input says what it does, from the click on send (also while the model loads), with the seconds and the tokens so far;
  the chat scrolls only up and down (long code lines wrap).
- Library as a table: architecture, parameters, format, quantization, publisher, abilities, size and where it runs; the gear opens a model's settings in a window of its own, with all its facts on top, without scrolling; only one such window at a time, closed when it loses the focus. Abilities have a colour each.
- Settings: hardware overview (chip, cores, memory, GPU cores, Metal working set, load) and a switch to run models without the GPU.
- Prompt profiles: system prompt with choosable fields, example dialogs, sampling and thinking, for chosen models or all; eleven templates; chosen in the chat or on the server as `model@profile` (listed in /v1/models).
- Tips & tricks: speculative decoding explained (faster, not different answers; exact acceptance), with sources.
- THIRD_PARTY_NOTICES.txt with the licences of the bundled packages, shipped with the app.
- Thinking level for models whose template has one (gpt-oss): low, medium or high in the chat, per model and per profile.
- Status bar: "No model loaded into memory or ready to stream", wrapping when space is short.
- Streamed models run without llama.cpp's weight repacking: read from the drive they are on instead of swapped to the internal SSD; Qwen3-30B-A3B 6.66 instead of 2.89 tokens/s on a 16 GB Mac, loaded in 30 instead of 109 s.
- Models → Loaded: every model in memory with its plan, the memory its engine holds and the rate its engine reads from disk (per process; without the built-in probe the whole drive of a streamed model). The library's table takes the window's whole width.
- Several models at once, within one memory budget: a model that does not fit beside the others makes the longest unused one go; a model streamed from the SSD runs alone. Unload per model.
- Tokens and speed per model since the app started, from the chat and the server, in the library.
- Benchmarks: docs/BENCHMARKS.md with charts of all measurements, and the same page for the website.
- Runtimes: MLX vision (mlx-vlm 0.7.10) as an optional runtime of its own: MLX models with vision read pictures.
- Runtimes: the llama.cpp build can be chosen among tested ones (b9660 default, b12131), each checked by its SHA-256 and kept in its own folder.
- Model search: abilities of MLX models read from chat_template.jinja too, pictures from the repo's files; the maker's values from the model a repo was converted from, and a model card's recommended sampling as a hint to take at the download.
- Model search: download size of every hit (GGUF from the smallest to the largest quantization, MLX the whole model), sorting by downloads or size; vision projectors named x.mmproj-… or x-mmproj-… are found and downloaded too.
- Status bar: CPU and GPU load as a two-row LED meter, like a cassette deck's level meter.
- Guard monitor, a window of its own (status bar, Models → Loaded, the app menu and the menu bar icon): per loaded model PID,
  engine, CPU, GPU, memory, disk read and tokens per second, each live against what was measured on this Mac, with a
  coloured delta and a sparkline of the last 60 s; CPU and GPU are compared only while the model answers. Above, this
  Mac as a table: CPU, GPU, memory used, wired memory against the guard's limit, file cache. Remembers its place.
- App menu with About, Settings… (⌘,) and the guard monitor; an icon in the menu bar with the loaded models, the server,
  settings and quit. Closing the main window keeps the app, its server and its models running; quit from the icon or
  with ⌘Q. Edit, View and Window are in the app's language too; the released app's View menu has no Reload and no
  developer tools. The menu bar menu is set on the icon (it opens while another app is in front), with the loaded
  models and the server under headers of their own; the server's address is the one in the local network while it
  listens there; quitting closes it if it is open. The menu code is the same file as VulKara's.
- Model browser for Hugging Face (GGUF and MLX): cards with downloads, likes, last update, licence, base model, context
  and abilities as icons with explanations; filters by ability (several at once), cards or rows; details with every file judged by the memory plan from the model's real header, the model
  card as text, the prompt template and the maker's values. Downloads resume and are checked by SHA-256; a GGUF gets
  its maker's values next to it. Token in the macOS keychain.
- Settings per model: sampling for chat and server (a client's own values win), context length, 8-bit KV cache and
  flash attention; *Model defaults* applies the maker's values. A changed load setting reloads the model.
- Tips & tricks for every model setting: on hover at the setting, and in detail with sources on the new Tips page.
- MLX answers run up to the context length, not to mlx-lm's default of 512 tokens.
- Local OpenAI-compatible server: `/v1/models`, `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`; 127.0.0.1
  only, access token, requests from web pages refused, models loaded on demand.
- English, German, Spanish, Portuguese (Brazil) and Portuguese (Portugal).
- The app installs its engines itself: llama.cpp b9660 and mlx-lm 0.32.2 with its own Python, pinned and checked by
  checksum (every Python package by its hash); Homebrew is no longer needed.
- App icon (a chat bubble streaming tokens, with VulKara's spark) and a DMG for Apple silicon.
