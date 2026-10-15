# No-Pit Qwen3.8

[中文版](./README.md) | English

> Qwen models are good, but their format following is shaky — the smaller the
> quantization, the worse it gets. Failed tool calls (tool call) loop forever,
> weird characters pop up, and conversations break. So we patch the output
> template (chat template) to bring the whole series back to where it should be.

Works anywhere Jinja is accepted (LM Studio / llama.cpp / vLLM / SGLang / oMLX).

## Which version

| Version | For | Contents | Path |
|---|---|---|---|
| v6 | Well-behaved models (NVFP4 etc.) | One Traditional-Chinese rule＋SERIAL RULE (one tool call per turn)＋output-contract | `v6/` |
| v6.2 | Small-quant + uncensored models, default | v6＋empty-think cleanup＋unclosed-think auto-split＋MUST-close-thinking | `v6.2/` |
| v6.2-alt | Reference experiments only | v6.2＋7 unproven extras (ban list, self-checks, etc.); use at your own risk | `v6.2-alt/` |

## What we changed (and only that)

Base: `froggeric/Qwen-Fixed-Chat-Templates` v22.5 (459 lines, Apache-2.0,
by Frédéric Guigand). v6 differs from upstream in only 7 hunks, see
`v6/v6_vs_upstream.diff`. The first-line version string is a watermark —
model updates can wash away custom templates, verify before load.

* SERIAL RULE: at most one `<tool_call>` per turn. Measured: stock fires 2,
  v6 fires exactly 1.
* Traditional Chinese rule, moved into the template (the old serve-side
  injection never fired on real traffic).
* Output-contract: raw JSON / source-only requests stay unwrapped.
* v6.2 adds: no empty `<think></think>` wrappers in history, auto-split
  unclosed thinking at `<tool_call>`, MUST close thinking before tool calls.

## Honest disclosures (measured, not claimed)

* Simplified-character leak (簡體滲漏): ~1 in 4 long-form essays leak,
  on every version tested. Templates can't fix it; short tasks are clean.
* Thinking-budget exhaustion (思考脹滿): uncensored models can spend
  thousands of tokens thinking with zero content on long tasks, under any
  template. Needs engine-side budgets.
* The 7 alt extras: none beat baseline in isolation. Reference only.

## Usage

```bash
# vLLM
vllm serve <model> --chat-template ./v6.2/chat_template.jinja --tool-call-parser qwen3_xml --reasoning-parser qwen3
# SGLang (0.5.x ignores in-dir templates; pass explicitly)
python -m sglang.launch_server --model-path <model> --chat-template ./v6.2/chat_template.jinja
# llama.cpp
llama-server -m <model.gguf> --jinja --chat-template-file ./v6.2/chat_template.jinja
# LM Studio: paste into Prompt Template / oMLX: overwrite model-dir copy and restart
```

For well-behaved models use `./v6/chat_template.jinja` instead.

## Files

* `v6/`: v6 build + diff vs upstream
* `v6.2/`: v6.2 build + diff vs v6 (`v62_vs_v6.diff`)
* `v6.2-alt/`: reference build
* `lab/strict-v6.1.jinja`: retired v6.1 experiment draft

Per-block notes: [BLOCKS.md](./BLOCKS.md).

PRs welcome.
