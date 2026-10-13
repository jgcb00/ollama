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
