# No-Pit Qwen3.8

[中文版](./README.md) | English

> 🆕 v7 out (2026-10-26): v6.2＋conditional output contract; format-class
> errors down to zero on the JSON re-test (6/7). See the [v7 release notes](./v7/RELEASE.en.md).

This project improves the output templates (chat template) of the Qwen3.8
model series. Qwen3.8 models are highly capable, but their format following
is unstable: the heavier the quantization, the more frequent tool call
format errors become, often accompanied by anomalous characters and
aborted conversations. This project addresses these issues through
customized output templates.

Scope: any inference engine accepting Jinja templates (LM Studio /
llama.cpp / vLLM / SGLang / oMLX).

## Upstream

* Base template: [froggeric/Qwen-Fixed-Chat-Templates](https://huggingface.co/froggeric/Qwen-Fixed-Chat-Templates) (v22.5, `855bffc`, Apache-2.0, by Frédéric Guigand)
* Original model template: [Qwen/Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B) (Qwen team, Alibaba Cloud)
* This project is an independent derivative work, not a GitHub fork; license chain in [NOTICE](./NOTICE).

## Versions

| Version | Target | Contents | Path |
|---|---|---|---|
| v6 | Well-behaved models (NVFP4 etc.) | Traditional-Chinese rule＋SERIAL RULE (one tool call per turn)＋output-contract | `v6/` |
| v6.2 | Small-quant + uncensored models (recommended default) | v6＋empty-think cleanup＋unclosed-think auto-split＋MUST-close-thinking | `v6.2/` |
| v6.2-alt | Experimental reference | v6.2＋7 unverified extras; effectiveness varies, verify independently | `v6.2-alt/` |
| v7 | Qwen3.8-Next-Flash needing JSON discipline (current default) | v6.2＋conditional output contract (fires on JSON tasks only) | `v7/` |

## Tested scope

All targets are Qwen3.8-Next-Flash; Qwen3.6-35B serves as a large-parameter control.

| Hardware | Model | Quantization | Engine |
|---|---|---|---|
| RTX PRO 6000 | Qwen3.8-Next-Flash uncensored-modelopt | NVFP4 (uncensored) | SGLang v3.0.1 |
| RTX 5090 | Qwen3.8-Next-Flash | IQ3_S (verified on two Strata hosts) | Strata |
| RTX 5060 Ti 16G | Qwen3.8-Next-Flash | IQ2_XS | Strata |
| Mac Studio (oMLX) | Jundot OQ4-mtp | OQ4 | oMLX |
| Mac Studio (LM Studio) | Qwen3.8-27B (incl. JonathanColetti uncensored) | Q6_K / IQ4_XS | LM Studio |
| Mac Studio (LM Studio, control) | Qwen3.6-35B | Q6_K | LM Studio |

JSON discipline re-test (7-item subset of suite 114): v7 eliminates all
format-class errors (6/7; the one remainder is semantic).

## Changes (project modifications only)

Base: `froggeric/Qwen-Fixed-Chat-Templates` v22.5 (459 lines, Apache-2.0,
by Frédéric Guigand). v6 differs from upstream in only 7 hunks; everything
else is byte-identical — see `v6/v6_vs_upstream.diff`. The first-line version
string serves as a watermark: model updates may overwrite custom templates,
so verify it before loading.

* SERIAL RULE: at most one `<tool_call>` block per turn. Measured: stock
  emits 2 calls in one turn vs exactly 1 under v6, preventing format
  corruption from concurrent calls by small models.
* Traditional-Chinese rule: Chinese output uniformly uses Traditional Chinese
  (Taiwan conventions). The rule previously lived in the serve layer with a
  faulty trigger condition and never fired; moved into the template so all
  clients are covered.
* Output-contract: raw JSON / source-only requests return only the requested
  payload, without Markdown fences or commentary.
* v6.2 adds three items: omit empty `<think></think>` wrappers in history,
  auto-split unclosed thinking at `<tool_call>`, and mandatory thinking
  closure before tool calls.
* v7 adds one item: conditional output contract. When the last user message
  contains `json` and no tools are requested, 7 JSON output rules are
  appended to system; tool and plain-chat requests are unaffected.

Per-block details and evidence status: [BLOCKS.md](./BLOCKS.md).

## Known limitations (verified by measurement)

* Simplified-character leak (簡體滲漏): roughly 1 in 4 long-form Chinese
  outputs contain Simplified characters, on every version tested. This is a
  model-level limitation templates cannot fix; short outputs are clean.
* Thinking-budget exhaustion (思考脹滿): uncensored models may spend their
  entire token budget thinking on long tasks with zero content output, under
  any template. Requires engine-side budget enforcement.
* The 7 alt extras: none beat baseline in isolation tests. Reference only.

## Usage

```bash
# vLLM
vllm serve <model> --chat-template ./v7/chat_template.jinja --tool-call-parser qwen3_xml --reasoning-parser qwen3
# SGLang (0.5.x ignores in-directory templates; pass explicitly)
python -m sglang.launch_server --model-path <model> --chat-template ./v7/chat_template.jinja
# llama.cpp
llama-server -m <model.gguf> --jinja --chat-template-file ./v7/chat_template.jinja
# LM Studio: paste into Prompt Template and save / oMLX: overwrite the model-directory copy and restart
```

Well-behaved models: use `./v6/chat_template.jinja` instead.

## File layout

* `v6/`: v6 build + diff vs upstream
* `v6.2/`: v6.2 build + diff vs v6 (`v62_vs_v6.diff`)
* `v6.2-alt/`: experimental reference build
* `v7/`: v7 build + diff vs v6.2 (`v7_vs_v62.diff`) + release notes (`RELEASE.en.md`)
* `lab/strict-v6.1.jinja`: retired v6.1 experiment draft

Pull requests welcome.

## License

Apache-2.0, see [LICENSE](./LICENSE) and [NOTICE](./NOTICE).
