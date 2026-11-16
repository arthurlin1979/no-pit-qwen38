# No-Pit Qwen3.8

[中文版](./README.md) | English

> 🆕 v8 out (2026-11-17): v7＋expanded triggers (JSON skeleton / Chinese phrasings / post-tool half)＋CODE contract＋no-empty-params,
> EXT-036＋EXT-044 FAIL→PASS. See the [v8 release notes](./v8/RELEASE.en.md).
> v7 (2026-10-26) retained.

This project improves the output templates (chat template) of the Qwen3.8
model series. Qwen3.8 models are highly capable, but their format following
is unstable: the heavier the quantization, the more frequent tool call
format errors become, often accompanied by anomalous characters and
aborted conversations. This project addresses these issues through
customized output templates.

Scope: any inference engine accepting Jinja templates (Strata / LM Studio /
llama.cpp / vLLM / SGLang / oMLX). Strata is this project's primary
validation platform.

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
| v7 | Qwen3.8-Next-Flash needing JSON discipline | v6.2＋conditional output contract (fires on JSON tasks only) | `v7/` |
| v8 | v7＋trigger gaps＋code tasks＋empty-param ban (current default; needs Strata ≥0.2.13.1 / engine ≥0.1.59.1; tuned for UD-IQ4_XS-v31 rev d9620119, also eats IQ3_S) | v7＋JSON-skeleton / Chinese-phrasing triggers＋post-tool-half contract＋CODE contract＋mandatory non-empty params | `v8/` |

## Tested scope

All targets are Qwen3.8-Next-Flash; Qwen3.6-35B serves as a large-parameter control.

| Hardware | Model | Quantization | Engine |
|---|---|---|---|
| RTX PRO 6000 | Qwen3.8-Next-Flash uncensored-modelopt | NVFP4 (uncensored) | SGLang v3.0.1 |
| RTX 5090 (host 5090) | Qwen3.8-Next-Flash | IQ3_S (Strata-verified, 114: 85/89) | Strata |
| RTX 5090 (host 5094) | Qwen3.8-Next-Flash | UD-IQ4_XS-v31 (Strata-verified, 114: 88/89, house 94/110) | Strata |
| RTX 5060 Ti 16G | Qwen3.8-Next-Flash | IQ2_XS | Strata |
| Mac Studio (oMLX) | Jundot OQ4-mtp | OQ4 | oMLX |
| Mac Studio (LM Studio) | Qwen3.8-27B (incl. JonathanColetti uncensored) | Q6_K / IQ4_XS | LM Studio |
| Mac Studio (LM Studio, control) | Qwen3.6-35B | Q6_K | LM Studio |

JSON discipline re-test (7-item subset of suite 114): v7 eliminates all
format-class errors (6/7); v8 closes the trigger gaps for 7/7
(EXT-036, EXT-044 FAIL→PASS, measured on 5094 UD-IQ4_XS-v31).

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
* v8 adds three items: trigger scope widened to all user messages; trigger
  signals add JSON skeletons plus Chinese phrasings （只輸出／僅輸出／僅包含／
  只回傳／僅回傳／輸出以下／回傳以下）; post-tool-half of tooled
  conversations also gets the contract (first turn untouched). Plus a CODE
  contract (code-only tasks) and a ban on empty-string required params.
  Tool-format protections unchanged.

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
# Strata (primary): overwrite the pack's tokenizer template, then restart (verified: 5090/5094, Strata ≥0.2.13.1)
cp ./v8/chat_template.jinja Strata-data/packs/iq3_s/tokenizer/chat_template.jinja
# Back up the original first, and check the first-line watermark reads strict-v8
systemctl --user restart strata-iq3s
# UD-IQ4_XS-v31, same with pack name swapped to unsloth-ud-iq4_xs-v31; also needs STRATA_PF_FUSED_Q50=1＋STRATA_PF_F16=1 (without it prefill halves)
# Each pack has its own tokenizer dir: iq3_s and unsloth-ud-iq4_xs-v31 are independent; swapping one never touches the other
# vLLM
vllm serve <model> --chat-template ./v8/chat_template.jinja --tool-call-parser qwen3_xml --reasoning-parser qwen3
# SGLang (0.5.x ignores in-directory templates; pass explicitly)
python -m sglang.launch_server --model-path <model> --chat-template ./v8/chat_template.jinja
# llama.cpp
llama-server -m <model.gguf> --jinja --chat-template-file ./v8/chat_template.jinja
```

### LM Studio (two verified methods, 2026-11-17)

1. Model-directory method (GGUF dirs such as `bartowski/*`, `unsloth/*`): back up, then overwrite `chat_template.jinja`; takes effect on next load (verified v6.2→v8).
2. Per-model JSON override: in `~/.lmstudio/.internal/user-concrete-model-default-config/<publisher>/<repo>/<file>.gguf.json`, add `llm.load.promptTemplate` under `load.fields` (paste full text＋`stopStrings: []` — without it LM Studio silently falls back; see the v7 lesson). Then `lms unload`＋`lms load` (or next load).

### Pennyroyal (SGLang, verified 2026-11-17: 5092 Qwen3.8 NVFP4)

Pennyroyal runs SGLang, which ignores in-model-directory templates — all three pieces must change together (miss one and it silently won't apply):

```bash
# 1. Drop the template into the host template dir (back up the old one first)
cp ./v8/chat_template.jinja /home/arthur/pennyroyal/templates/froggeric-v22.5-arthur-strict-v8.jinja
# 2. Repoint chat-template.sh and update its SHA (it checksums; a mismatch refuses to start — back up chat-template.sh first)
#    CHAT_TEMPLATE=.../templates/froggeric-v22.5-arthur-strict-v8.jinja
#    set SHA to the new file's sha256sum
# 3. Add one mount to compose (mirror the existing v6/v6.2 entries) mapping the host file into the container at the same name:
#      - type: bind
#        source: ${PENNYROYAL_TEMPLATE_HOST:-/home/arthur/pennyroyal/templates}/froggeric-v22.5-arthur-strict-v8.jinja
#        target: /opt/pennyroyal/configs/pennyroyal/templates/froggeric-v22.5-arthur-strict-v8.jinja
# 4. Recreate and verify (SGLang takes minutes to load):
docker compose -f /home/arthur/pennyroyal/docker/pennyroyal/compose.yaml up -d
# Verify: a keyword-less JSON test returning raw means it took. Watch out for doubled filename prefixes (write templates/ only once).
```

### oMLX (verified 2026-11-17, no restart needed)

Overwriting the model directory's `chat_template.jinja` takes effect immediately (per-request template read; server PID unchanged and picked up; keep the v7 backup). Example: `~/.omlx/models/Jundot/Qwen3.8-Flash-Next-oQ4e-mtp/chat_template.jinja`.

Legacy reference: well-behaved models use `./v6/chat_template.jinja`.

## File layout

* `v6/`: v6 build + diff vs upstream
* `v6.2/`: v6.2 build + diff vs v6 (`v62_vs_v6.diff`)
* `v6.2-alt/`: experimental reference build
* `v7/`: v7 build + diff vs v6.2 (`v7_vs_v62.diff`) + release notes (`RELEASE.en.md`)
* `v8/`: v8 build (current default) + diff vs v7 (`v8_vs_v7.diff`) + release notes (`RELEASE.en.md`, with pinned applicable versions)
* `lab/strict-v6.1.jinja`: retired v6.1 experiment draft

## Measured speed

RTX 5060 Ti 16G (IQ2_XS, Strata), full 110-question run (Ability 100 + Think-off 10, 2026-10-19):

| Item | Value |
|---|---|
| 110-question generation speed (median) | 51.0 tok/s |
| 110-question generation speed (average) | 47.2 tok/s |
| Range | 7.8 – 72.3 tok/s |
| JSON-discipline recheck (7 questions) latency | 4–8s |

5094 RTX 5090 (UD-IQ4_XS-v31＋v8, Strata 0.2.13.1 / engine 0.1.59.1, 2026-11-17):

| Item | Value |
|---|---|
| 114 (ability 89) | 88/89 (only genuine miss EXT-030, a cross-model hard question) |
| house 100＋10 | 94/110 (Ability 85＋Agent 9/10; A07 partially completed 5/10) |
| 128K cold-read prefill | ≈5500 tok/s (Q5_0 needs `STRATA_PF_FUSED_Q50=1`, otherwise halves) |
| warm short-reply decode | ≈150–270 tok/s |

Pull requests welcome.

## License

Apache-2.0, see [LICENSE](./LICENSE) and [NOTICE](./NOTICE).
