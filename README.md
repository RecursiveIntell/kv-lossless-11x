# ⚠️ Archived — superseded by [proveKV](https://github.com/RecursiveIntell/proveKV)

> **Historical quality-claim correction — October 10, 2026.** The original lossless K/V and ΔPPL +0.00% interpretation below is withdrawn as admissible quality evidence. Full-input/full-cache scoring did not establish held-out continuation quality or bit-exact K/V reconstruction. The codecs are approximate quantizers. The preserved storage receipts remain historical byte measurements, not proof of quality neutrality. The successor's producer-recorded cache-aligned shared-prefix continuation check instead reported oracle PPL 7.203125 versus roundtrip PPL 24.234375 (+236.44%); that aggregate fixture does not establish eight independent-prompt or reconstructed-shell quality. Read the [commit-pinned successor methodology and claim limits](https://github.com/RecursiveIntell/proveKV/tree/9ef62cab05d44f00585ec286557849993912e0f4). Original text, source, and receipts below are retained for audit; they are not current endorsed claims. The published successor crate is named `provekv`, not the historical `prove-kv` spelling in the table below.

This repository is **frozen**. The active development of the two-tier,
receipted, content-addressed KV-cache pool continues at:

> **https://github.com/RecursiveIntell/proveKV**

The codec math (`fib_k4_n32`) is a clean-room Rust port of the
[FibQuant paper, Lee & Kim 2026](https://arxiv.org/abs/2605.11478).
The 5 cross-validated PPL measurements, the committed `state.json`
receipts, and the multi-agent N=2..8 sweep are preserved here for
reproducibility — the receipts will keep loading against this code.

**Final commit on the poly-kv era line:** `7d27f73`
("Mark repo as superseded by proveKV").

**The rename that bridges the two repos:**

| Layer | Old name (this repo) | New name (proveKV) |
|---|---|---|
| Crate (system) | `poly-kv` (lib = `poly_kv`) | `prove-kv` (lib = `prove_kv`) |
| Codec (cold tier) | `fib_k4_n32` | `fib_k4_n32` (unchanged — FibQuant identity) |
| Codec (hot tier) | `turbo_8bit` | `turbo_8bit` (unchanged — upstream `turbo-quant`) |
| Receipt schema | `poly_kv_receipt_v1` | `prove_kv_receipt_v1` (wire-format break) |

The renaming decision is recorded at
[`docs/SYSTEM_NAMING_AND_BRANDING.md`](docs/SYSTEM_NAMING_AND_BRANDING.md)
in the new repo.

---

> **Withdrawn historical text.** The original section below is preserved verbatim for audit. Its lossless-reconstruction and PPL-neutrality assertions are withdrawn and are not endorsed current findings. See the dated correction above for the evidence boundary.

# kv-lossless-11x

**Lossless KV-cache compression at 11.13× on a real 1.7B-parameter LLM.**

This repository is a self-contained, runnable proof of a single measured result:

> On `HuggingFaceTB/SmolLM2-1.7B-Instruct` with the first 1024 tokens of
> the WikiText-2 test split, the **fib_k4_n32** codec (clean-room Rust
> port of the [FibQuant paper](https://arxiv.org/abs/2605.11478), Lee & Kim
> 2026) achieves:
>
> - **Compression ratio: 11.13×** vs fp32 raw (5.6× vs fp16 raw)
> - **Pool size: 36,175,872 bytes (36 MB)**, down from 201,341,281 bytes
>   (201 MB) raw fp16 cache
> - **ΔPPL: +0.00%** — the roundtrip K/V cache is bit-exact to the oracle
>   forward pass at 4-decimal PPL precision

The claim is **honest lossless at 11× on real LLM K/V** — not a synthetic
benchmark, not a 50× headline, not a lossy codec at higher compression.

## Reproduce it in five minutes

```bash
git clone https://github.com/RecursiveIntell/kv-lossless-11x
cd kv-lossless-11x
cargo build --release --example poly_kv_fast_roundtrip
cd poly-kv/scripts
PYTORCH_ALLOC_CONF=expandable_segments:True \
  python3 ppl_validate.py \
    --model HuggingFaceTB/SmolLM2-1.7B-Instruct \
    --corpus wikitext-2 \
    --n-tokens 1024 \
    --ppl-frac 0.3 \
    --output ../../results/bench/ppl/smollm2-1.7b/wikitext-2/state.json
```

The script writes `state.json` (machine-readable) and `report.md`
(human-readable) at the output path. The reference run from
2026-06-02 12:52–12:56 CDT is checked in at
[`results/bench/ppl/smollm2-1.7b/wikitext-2/`](results/bench/ppl/smollm2-1.7b/wikitext-2/).

(Reproduce notes above use the **pre-rename** paths and binary names
because the receipts in `results/` were produced against the
`poly_kv` binary. To run against the new system, use the `proveKV`
repo instead.)
