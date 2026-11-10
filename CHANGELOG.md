# Changelog

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
