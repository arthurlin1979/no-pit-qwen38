# v8 release notes

Qwen3.8 Jinja, eighth edition (v8).

## Pinned applicable versions

- Inference engine: Strata v0.2.13.1 and up, engine 0.1.59.1 and up. The Jinja syntax v8 uses (`~` concatenation, cross-loop `namespace` accumulation) needs Strata's embedded Jinja support.
- Model: Unsloth `Qwen3.8-Flash-Next-GGUF`, `UD-IQ4_XS-v31`, rev `d9620119` (full hash `d96201194f103c906994f283f504209dc8d2a10c`, 2026-11-04, 4 shards). The older rev `38bb39ee` (3 shards) also works; the triggers are revision-independent.
- Also applies to: GSQ-RCO IQ3_S (verified on 5090/5094 under the same Strata).
- Validation hardware: 5094 (Ryzen 9 9900X＋RTX 5090 32G＋128G RAM); 5090 identical.
- Handled for the new UD-Q4: Q5_0 down layers need `STRATA_PF_FUSED_Q50=1` (on Linux also `STRATA_PF_F16=1`), otherwise prefill halves. That is a serve-side env var, not template business — but v8's 114/house scores were all measured with the flag on.

## Motivation

Fix v7's two trigger blind spots (measured on 114: EXT-036, EXT-044).

## Changes (vs v7, see `v8_vs_v7.diff`)

1. Trigger scope widened from "last user message" to "all user messages".
2. Trigger signals: beyond the `json` keyword, JSON skeletons (`{"`＋`":`) and Chinese phrasings （只輸出／僅輸出／僅包含／只回傳／僅回傳）.
3. Tooled conversations included: once a conversation already has tool results (second half, time to wrap up) and any user message asks for JSON, the contract applies; the first turn (no tools called yet) is untouched, SERIAL RULE and tool-format protection unchanged.

## Measurements (5094, UD-IQ4_XS-v31, Strata, same seed and params)

- EXT-036 (multi-turn with tools): FAIL→PASS, raw JSON back.
- EXT-044 (774 chars, no "json" word): FAIL→PASS, raw JSON back.
- v7's 114-subset conclusions unchanged; v8 only treats trigger coverage, the rule text itself untouched.

## Added coverage (2026-11-17, same v8, no new edition)

- Code-task triggers: 只輸出程式碼／僅輸出程式碼／只回傳程式碼／僅回傳程式碼／code only／source code only → CODE contract (body only, no fences, no explanations).
- More Chinese phrasings: 輸出以下／回傳以下.
- One tool-branch rule: required params must be non-empty; if unknown, keep working instead of sending empty-string placeholders.
- Measured: V2-Q015 FAIL→PASS; code tasks (EXT-038/040) fences gone, raw code back (functional grading pending house code grader); V2-Q013 (should-refuse-without-ID) intentionally not forced — model behavior.
- Added: "實作＋def／函式／函數／class" triggers. Functional spot-checks all pass (two_sum 5/5, longest_unique 4/4); contract compliance is probabilistic, occasional fences remain.

## Known limitations

- The contract still depends on "seeing that you want JSON"; prompts that never hint at a format are invisible to the template — up to the model.
- Tooled conversations with zero JSON hints throughout get no contract (by design, tool-format first).
