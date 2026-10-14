# No-Pit Qwen3.8

[中文版](./README.md) | English

> Qwen models are good, but their format following is shaky — the smaller the quantized build, the worse it gets. Failed tool calls loop forever, weird characters pop up, and the conversation just breaks. So we patch the output template to bring the whole series back to where it should be.

Works anywhere Jinja is accepted. I run the whole series myself — LM Studio / llama.cpp / vLLM / SGLang / oMLX.

## Base pin

* Upstream: `froggeric/Qwen-Fixed-Chat-Templates` main v22.5, 459 lines, fetched 2026-10-13
  * Source: https://huggingface.co/froggeric/Qwen-Fixed-Chat-Templates
  * License: Apache-2.0 inherited from Qwen, original author Frédéric Guigand
* Build: `chat_template.jinja`, 428 lines, 26292 bytes
* Watermark: `qwen3.8-froggeric-v22.5-arthur-strict-v6` + sha256 `3f092bdc58099b6ebcaab8698d6d71cb501c357cb90111d1708e57165c114871`. Verify the first line before load — model updates can wash away custom templates by hash.

Credits: Qwen structure / froggeric fixes, Apache-2.0, modifications under the same license.

## Why not upstream as-is

Upstream is full-featured; this is minimal and auditable. Only 7 hunks: persistent output-contract, SERIAL RULE (one call per turn), one Traditional Chinese rule. Everything else is byte-identical to upstream, so benchmarks stay comparable.

## 7 hunks (see `v6_vs_upstream.diff`)

1. H1 header: version string, drop `_tool_format` (never sent in practice)
2. H2 persistent: `_lang_rule` + `_strict_output-contract`, raw JSON / source-only without markdown fences
3. H3-H4 instructions: drop JSON instruction branch, keep XML + SERIAL
4. H5-H6 rendering: drop JSON render branch
5. H7 truncation: drop `_is_json_payload` exception, truncate uniformly

The Traditional Chinese rule moved from a dirty `frontend.py` patch (only injected when no system message exists — never fired on routed lanes that always send one) into the template so every client gets it.

## Usage

```bash
# vLLM
vllm serve <model> --chat-template ./chat_template.jinja --tool-call-parser qwen3_xml --reasoning-parser qwen3
# SGLang
python -m sglang.launch_server --model-path <model> --chat-template ./chat_template.jinja
# llama.cpp
llama-server -m <model.gguf> --jinja --chat-template-file ./chat_template.jinja --reasoning-format deepseek
# LM Studio: paste into Prompt Template / oMLX: overwrite the model dir copy and restart
```

## Self-test (5 items)

Thinking language / tool call / two tools yield exactly one / multi-system / raw JSON byte-exact. Offline jinja2 9/9 + 0 Simplified hits live.

## Lab v6.1

`lab/strict-v6.1.jinja` (27007B) adds hygiene + completion self-check, A/B showed no gain so v6 stays. The third candidate (error escalation) already exists upstream as two-tier recovery. PRs welcome.

## Known limits

SERIAL costs a round-trip; compensate with goal auto-continue + outer parallelism. temp-1 may leak a Simplified word. Qwen3.8 series only.

## Files

* `chat_template.jinja`: v6 build
* `v6_vs_upstream.diff`: 7-hunk diff vs upstream v22.5
* `lab/strict-v6.1.jinja`: v6.1 experiment draft
