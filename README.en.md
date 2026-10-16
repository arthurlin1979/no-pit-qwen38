# No-Pit Qwen3.8

[中文版](./README.md) | English

Qwen models are smart, but their format following is a mess.
The smaller the quantization, the worse it gets: failed tool calls loop
forever, weird characters pop up, conversations die halfway. So I patched
the output template (chat template) to bring them back to what they should be.

Works anywhere Jinja is accepted (LM Studio / llama.cpp / vLLM / SGLang / oMLX).
I run the whole series myself.

## Pick one of three

| Version | For | Contents | Path |
|---|---|---|---|
| v6 | Well-behaved models (NVFP4 etc.) | One Traditional-Chinese rule＋SERIAL RULE (one tool call per turn)＋output-contract | `v6/` |
| v6.2 | Small-quant + uncensored models, the default | v6＋empty-think cleanup＋unclosed-think auto-split＋MUST-close-thinking | `v6.2/` |
| v6.2-alt | Experimenters | v6.2＋7 unproven extras. Some say they work, some say they don't. Your call | `v6.2-alt/` |

## What I actually changed (nothing else)

Base: `froggeric/Qwen-Fixed-Chat-Templates` v22.5 (459 lines, Apache-2.0,
by Frédéric Guigand). I touched exactly 7 hunks, everything else is
byte-identical — proof in `v6/v6_vs_upstream.diff`. The first-line version
string is a watermark: model updates can wash away custom templates, so check
it before load.

* SERIAL RULE: at most one `<tool_call>` per turn. Honestly this one hits
  hardest — measured stock firing 2 at once vs v6 firing exactly 1. Small
  models mangling parallel calls is gone.
* Traditional Chinese rule. Frankly I first put it in the serve layer with a
  wrong condition so it never fired; moved into the template so every client
  gets it.
* Output-contract: raw JSON / source-only requests stay unwrapped, no chatter.
* v6.2 adds three: no empty `<think></think>` wrappers in history, auto-split
  unclosed thinking at `<tool_call>`, MUST close thinking before tool calls.

Per-block notes with honest verdicts: [BLOCKS.md](./BLOCKS.md).

## Bad news first (measured, not claimed)

* Simplified-character leak (簡體滲漏): ~1 in 4 long-form essays leak, on
  every version tested. Templates can't fix it; short tasks are clean.
* Thinking-budget exhaustion (思考脹滿): uncensored models can burn thousands
  of tokens thinking with zero content on long tasks, under any template.
  Needs engine-side budgets.
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

Well-behaved models: use `./v6/chat_template.jinja` instead.

## Files

* `v6/`: v6 build + diff vs upstream
* `v6.2/`: v6.2 build + diff vs v6 (`v62_vs_v6.diff`)
* `v6.2-alt/`: reference build
* `lab/strict-v6.1.jinja`: retired v6.1 experiment draft

PRs welcome.
