# Archive Notice

**This repository is archived as of 2026-06-02.**

The active development of the two-tier, receipted, content-addressed
KV-cache pool continues at:

> **https://github.com/RecursiveIntell/proveKV**

## What is preserved here

- The 5 cross-validated PPL measurements (SmolLM2-1.7B, TinyLlama-1.1B,
  Qwen2.5-0.5B; WikiText-2 + code-source; 1024 + 1280 tokens)
- The committed `state.json` receipts (5 files, all at
  `results/bench/ppl/<model>/<corpus>/state.json`)
- The multi-agent N=2..8 contention sweep (all lossless; 1.80× → 7.19×
  memory reduction scaling linearly with N)
- The methodology (locked) at `poly-kv/scripts/ppl_validate.py`
- The full git history of the poly-kv era, ending at commit `7d27f73`

## What is not preserved here

- The `RECEIPT_SCHEMA` was bumped from `poly_kv_receipt_v1` to
  `prove_kv_receipt_v1` in the new repo. The receipts in this repo
  still validate against the old schema because the source code on
  this branch (`7d27f73`) still emits the old schema.
- The hot-tier (turbo_8bit per-agent shell) is defined in source but
  was not benchmarked in the 11.13× lossless run. The two-tier policy
  is the design; the shared-pool measurement is what this repo proves.

## For citations

If you reference the 11.13× lossless measurement, please cite both:

1. The FibQuant paper (Lee & Kim 2026) — the codec math.
2. The CITATION.cff of whichever repo you used (this one for the
   archived state, the new `proveKV` repo for the active line).

## Final commit

```
7d27f73 Mark repo as superseded by proveKV
```
