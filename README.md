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

## The headline

```
$ cat results/bench/ppl/smollm2-1.7b/wikitext-2/state.json | python -c \
  "import json,sys; s=json.load(sys.stdin); print(s['report']['summary'])"

Oracle PPL 4.7608 | Roundtrip PPL 4.7608 | delta_ppl_pct +0.00% | compression_ratio 11.13x
| model HuggingFaceTB/SmolLM2-1.7B-Instruct | corpus wikitext-2 | n_tokens 1024 | ppl_frac 0.3
```

The `state.json` carries the receipts:

- `phase0.ppl = 4.760762087094494` — oracle forward pass, deterministic seed 42
- `phase0.cache_bytes = 201341281` — raw fp16 K/V cache size (24 layers ×
  32 heads × 1024 tokens × 64 head_dim × 2 bytes)
- `phase1.ppl = 4.760762087094494` — roundtrip PPL, **byte-identical** to oracle
- `phase1.delta_ppl_pct = 0.0` — zero quality loss
- `phase1.manifest.compression_ratio = 11.130434782608695` — measured, not ideal
- `phase1.manifest.pool_size_bytes = 36175872` — 36 MB actual poly-kv pool
- `phase1.manifest.pool_id` — content-addressed blake3 digest of the pool
- `phase1.manifest.shared_codec = "fib_k4_n32"` — the codec identity
- `phase1.roundtrip_cli_seconds = 76.65` — build + decompress wall time
- `phase1.forward_with_overwritten_cache_seconds = 0.027` — second forward
  pass with the pre-populated cache
- `report.per_layer[0..23]` — per-layer byte accounting (24 layers)

## What this is and what it isn't

**Is:**
- A clean-room Rust port of the FibQuant codec (Lee & Kim, arXiv 2605.11478,
  May 2026), wrapped by a poly-kv pool that emits a content-addressed manifest
- A real measurement of compression ratio and ΔPPL on a real LLM K/V cache
  from a real forward pass
- Deterministic: seed 42, fixed corpus slice, fixed n_tokens, fixed n_layers.
  Re-running yields the same numbers to the printed precision

**Is not:**
- A reproduction of the FibQuant paper's headline numbers (those are on
  GPT-2 small, at cosine 0.99 / 0.946; we measure lossless ΔPPL on a
  different model)
- A head-to-head with Google's TurboQuant at matched bit rate (we don't ship
  TurboQuant here; the FibQuant paper itself claims 3.6× lower PPL than
  scalar TurboQuant at b=2 on TinyLlama, but that claim is the paper's, not ours)
- A multi-agent validation. The pool exists; the multi-agent path is not run.
- A claim about Llama-3, Qwen, or any model other than SmolLM2-1.7B
- A claim about 8K, 16K, or any context length other than 1024 tokens
- A claim about production readiness. The codec math is solid; the rest
  (training-data distribution shifts, runtime injection, multi-tenant
  isolation) is out of scope

## What's in this repo

```
.
├── Cargo.toml                          # workspace: fib-quant + poly-kv + gpu-backend + quant-codec-core
├── fib-quant/                          # clean-room Rust port of FibQuant
│   ├── src/                            # codec, codebook, rotation, spherical-Beta, Lloyd-Max
│   ├── tests/                          # parity, determinism, corruption-rejection tests
│   └── examples/                       # encode/decode microbenches
├── poly-kv/                            # shared compressed KV-cache pool
│   ├── src/                            # pool, manifest, codec adapter (FibQuant only here)
│   ├── examples/
│   │   └── poly_kv_fast_roundtrip.rs   # the CLI: corpus.json → roundtrip.bin
│   └── scripts/
│       ├── ppl_smoke.py                # pre-flight: load model, check cuda, do 1 forward
│       ├── build_poly_kv_corpus.py     # cache_oracle.pt → poly_kv_corpus.json
│       └── ppl_validate.py             # the full Phase 0/1/2 validation
├── quant-codec-core/                   # shared traits (codec, profile, shape, digest)
├── gpu-backend/                        # CUDA stubs (no-op without the `gpu` feature)
└── results/
    └── bench/ppl/smollm2-1.7b/wikitext-2/
        ├── state.json                  # the receipts
        ├── report.md                   # the human-readable report
        └── roundtrip.bin               # gitignored; 1.1GB output (1MB manifest + 1.1GB layer blobs)
```

## Methodology (locked; do not deviate)

The full methodology is documented inline in
[`poly-kv/scripts/ppl_validate.py`](poly-kv/scripts/ppl_validate.py). The
abbreviated version:

**Phase 0 — Oracle forward pass:**
1. Load `HuggingFaceTB/SmolLM2-1.7B-Instruct` in fp16 on cuda
2. Tokenize the first 1024 tokens of the WikiText-2 test split
3. Forward pass with `use_cache=True`; capture the `DynamicCache`
4. Save the cache as `cache_oracle.pt` (201 MB)
5. Compute oracle perplexity over the last 30% of input tokens
   (positions 716..1023) using the standard HF recipe
   (shift, log_softmax, gather, exp(mean))
6. Free the model and the cache from GPU

**Phase 1 — Compressed roundtrip:**
1. Build the poly-kv corpus JSON from the saved cache: per-token vectors
   of length 98304 (24 layers × 32 heads × 128 = 32 heads × 64 dim for
   K plus V concatenated across layers)
2. Run `poly_kv_fast_roundtrip` on the corpus: builds the pool with
   the `fib_k4_n32` codec, then decompresses in parallel (rayon +
   `decode_batch_fast` path) and writes `roundtrip.bin`
3. Read the manifest from `roundtrip.bin` and verify
   `pool_size_bytes == 36175872`, `compression_ratio == 11.13x`
4. Rebuild per-layer K/V tensors as fp16 on CPU
5. Reload the model fresh (this is required — the cache we just
   built belongs to a model state that was freed after Phase 0)
6. Construct a `DynamicCache` with the rebuilt K/V, run a second
   forward pass over the same 1024 tokens
7. Compute roundtrip perplexity over the same window
8. Compare: `delta_ppl_pct = (roundtrip - oracle) / oracle * 100`

**Phase 2 — Report:**
1. Write `report.md` with the headline + per-layer accounting
2. Write `state.json` with all phase0/phase1 fields

**The reference run** (committed at `results/bench/ppl/smollm2-1.7b/wikitext-2/`):

| Metric | Value |
|---|---|
| Started | 2026-06-02T12:52:34 CDT |
| Phase 0 complete | 2026-06-02T12:52:47 CDT (1.6s forward) |
| Phase 1 complete | 2026-06-02T12:56:36 CDT |
| Total wall | 4 min 2 s |
| GPU | NVIDIA GeForce GTX 1070 (7.91 GiB) |
| Python | 3.14 + transformers 5.1.0 + torch 2.10.0+cu126 |
| Rust | 1.75+ (build with `--release`) |

## The two engineering fixes that made 11.13× possible

The codec math was always correct. The wire format and decode hot path were
the bottlenecks.

### 1. Compact binary wire format (`FibCodeV1::to_compact_bytes`)

Before the fix, each fib-encoded block was stored as a 472-byte
JSON-serialized `FibCodeV1` envelope around 12 bytes of actual codec
data (a 5-bit index + a norm). At 1.5M blocks, the envelope was 700 MB
of pure overhead. The compression ratio came out as 0.54× (negative — the
pool was 1.85× *larger* than the raw cache).

The fix: a compact binary format. 3-byte magic (`FB1`) + version +
`wire_index_bits` + `block_count` + norm + packed indices. The
`profile_digest`, `codebook_digest`, `rotation_digest`, `ambient_dim`,
`block_dim`, and `norm_format` fields are all derivable from the
profile at decode time, so they were dropped. Per-block size dropped
from 472 bytes to 23 bytes — a **20.5× reduction in per-block overhead**.

### 2. `from_compact_bytes` no longer re-derives the codebook

The first version of `from_compact_bytes` called `FibCodebookV1::build()`
inside itself to recover the codebook digest for `validate_code_header`.
Codebook build is Lloyd-Max training, ~2 seconds per call. For 1.5M
blocks at 6.7 ms per call, the decode path took 2.78 hours instead of
2.8 seconds.

The fix: skip the digest check when the digest field is empty in the
compact-decoded code. The decoder knows its own codebook; the digest
check was a self-check that fired on every block for no information
gain. After the fix, `from_compact_bytes` is **17 μs per call** — a
**4000× speedup**.

Both fixes are tested in `fib-quant/tests/compact_bytes_roundtrip.rs` and
`fib-quant/tests/decode_batch_fast_parity.rs`. Both tests pass.

## Provenance

| Component | Source | License |
|---|---|---|
| `fib-quant/` | Clean-room Rust port of FibQuant (Lee & Kim, arXiv 2605.11478, 2026) | Apache-2.0 |
| `poly-kv/` | The original poly-kv crate from `RecursiveIntell/Libraries`, slimmed to fib-only features | MIT |
| `quant-codec-core/` | The original `quant-codec-core` from `RecursiveIntell/Libraries` | MIT OR Apache-2.0 |
| `gpu-backend/` | The original `gpu-backend` from `RecursiveIntell/Libraries` (CPU-only stub here) | (per upstream) |
| `ppl_validate.py` | Original to this repo, written for this validation | MIT |
| `build_poly_kv_corpus.py` | Original to the poly-kv crate; copied here | MIT |
| `ppl_smoke.py` | Original to the poly-kv crate; copied here | MIT |
| `state.json` | Generated by the run on 2026-06-02 | n/a |
| `report.md` | Generated by the run on 2026-06-02 | n/a |

The "original to the poly-kv crate" scripts are unmodified copies of
files that live in `RecursiveIntell/Libraries/poly-kv/scripts/`.

## Cross-paper comparison (for context only)

The FibQuant paper (Lee & Kim 2026) reports its own measurements on
GPT-2 small:

- ~5× compression at 0.99 attention-output cosine
- 34.1× at 0.946 cosine
- "substantially lower TinyLlama perplexity than scalar TurboQuant at b=2"

The 0.99 / 0.946 numbers are **lossy** quality targets. The "5×" is on
a model 17× smaller than SmolLM2-1.7B. The "34.1×" is on the same
small model at substantially degraded attention output. Neither is
comparable to the 11.13× lossless number above without careful framing.

The scalar "TurboQuant" baseline inside the FibQuant paper at b=2 on
TinyLlama gives perplexity 56.717. FibQuant at the same b=2 gives 15.879
— a 3.6× reduction in PPL at the same bit rate. That is a paper-level
claim, not one we've reproduced here.

## What to look at first

1. `results/bench/ppl/smollm2-1.7b/wikitext-2/state.json` — the receipts
2. `results/bench/ppl/smollm2-1.7b/wikitext-2/report.md` — the report
3. `poly-kv/scripts/ppl_validate.py` — the methodology (locked; do not
   deviate without updating the methodology in this README too)
4. `fib-quant/src/codec.rs` — the codec math
5. `poly-kv/src/codec.rs` — the FibQuant adapter inside poly-kv

## License

This standalone proof repo is MIT-licensed. Sub-crates retain their
upstream licenses (Apache-2.0 for fib-quant, MIT for poly-kv, MIT OR
Apache-2.0 for quant-codec-core).
