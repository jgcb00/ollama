# Olala (Dragon architecture) on Ollama

This branch runs Olala / Dragon GGUFs (`general.architecture = dragon`) through
Ollama's llama-server runner. The model code lives in the jgcb00 llama.cpp fork:
`LLAMA_CPP_VERSION` names a tag of https://github.com/jgcb00/llama.cpp
(`olala-ollama-*`, the `olala-master` commits rebased on the upstream tag
Ollama pins), and `OLLAMA_LLAMA_CPP_GIT_REPOSITORY` in
`llama/server/CMakeLists.txt` / `cmake/local.cmake` points the fetch at that
fork. The `llama/compat/` patches apply unchanged.

## Build

```sh
cmake -B build .
cmake --build build --parallel 16
```

To iterate on a local llama.cpp checkout instead of the tag, add
`-DFETCHCONTENT_SOURCE_DIR_LLAMA_CPP=/path/to/llama.cpp` (the compat patches
are applied to that tree).

## Import a GGUF

GGUFs come from the fork's `convert_hf_to_gguf.py` (bf16) and
`llama-quantize`; q5_k_m is the recommended CPU quant.

```sh
cat > Modelfile <<'EOF'
FROM /path/to/olala-dpo99k-q5_k_m.gguf
EOF
ollama create olala -f Modelfile
ollama run olala
```

No Go TEMPLATE is needed: chat requests go to llama-server's
`/v1/chat/completions`, which renders the GGUF's jinja template and uses the
fork's Olala chat parser. The `analysis` channel is returned as `thinking`, the
`tools` channel as `tool_calls`. `think` maps to the template's
`reasoning_effort` (`false` -> `none`, `"low"`/`"medium"`/`"high"` passed through).

## Windows

The `olala-windows` workflow (`.github/workflows/olala-windows.yml`) builds an
x64 payload on every push to `olala`: CPU (all ggml-cpu variants, built with
llvm-mingw clang) + CUDA 13 backend + `ollama.exe`. Download the
`ollama-olala-windows-amd64` artifact from the run in the Actions tab.

1. Unzip it, e.g. to `C:\ollama-olala` (`ollama.exe` next to `lib\ollama\`).
2. If the official Ollama is installed, quit its tray app (it holds port
   11434), or run this one elsewhere with `$env:OLLAMA_HOST="127.0.0.1:11500"`
   in every PowerShell window that uses it. Both share
   `%USERPROFILE%\.ollama\models` unless `OLLAMA_MODELS` is set.
3. Start the server and import the GGUF:

   ```powershell
   cd C:\ollama-olala
   .\ollama.exe serve          # keep this window open
   # second window:
   Set-Content Modelfile 'FROM C:\models\olala-dpo99k-q5_k_m.gguf'
   .\ollama.exe create olala -f Modelfile
   .\ollama.exe run olala
   ```

NVIDIA GPUs need a driver recent enough for CUDA 13 (R580 or newer); without
one Ollama falls back to CPU. The Dragon ops exist for CPU, CUDA and Metal
only, so no ROCm/Vulkan payload is built. To build locally instead: Visual
Studio 2022 (Desktop C++), Go, CMake, Ninja, llvm-mingw in
`C:\Program Files\`, the CUDA 13 toolkit for GPU, then from a VS Developer
PowerShell with `$env:CMAKE_GENERATOR="Ninja"`:
`.\scripts\build_windows.ps1 cpu cuda13 ollama` (output in `dist\windows-amd64`).

## Notes

- Parallel requests (`OLLAMA_NUM_PARALLEL`) are supported; prefix caching
  works through llama-server's recurrent-state checkpoints.
- Attention layers have no RoPE, so llama-server's context shift only drops
  attention KV entries; the Mamba3 state keeps the full history.
- Converting safetensors directly with `ollama create` is not supported for
  this architecture; import a GGUF.
- Model behaviour, identical with plain llama.cpp: with `think: false` the
  DPO-99k checkpoint often never closes its final channel, and some sampled
  generations reason without end. Set `num_predict` for unattended use.
