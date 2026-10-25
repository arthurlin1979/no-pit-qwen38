# v7 Release Notes

Qwen3.8 Jinja v7 is out.

Thanks first to DeepSeek 4.1 Flash. DS runs too slowly on my 5090 box,
but it is smart and its output discipline is excellent — so I had DS help
tune Qwen3.8's output to its best: everything outside raw intelligence,
removed as far as possible; low-level output errors, pressed down flat.
That is v7.

These constraints started as proxy injection, but moving them into the
Jinja chat template turned out better: one file covers every host — my
Strata boxes and oMLX all run it — and it ports anywhere without
per-route installs.

Measured: on a 7-item subset of suite 114, format-class errors went to
zero (6/7). The one remainder is a semantic-comprehension item, outside
what a template can fix. Honestly stated.

Technical details: [BLOCKS.md](../BLOCKS.md) item 17; diff vs v6.2:
[v7_vs_v62.diff](./v7_vs_v62.diff).
