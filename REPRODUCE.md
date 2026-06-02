# Reproduce the result

Two ways: with the committed state.json (zero compute, just verify) or
from scratch on a GPU host.

## Verify the committed result (no GPU required)

```bash
cat results/bench/ppl/smollm2-1.7b/wikitext-2/state.json | python -c "
import json, sys
s = json.load(sys.stdin)
oracle = s['phase0']['ppl']
rt = s['phase1']['ppl']
ratio = s['phase1']['compression_ratio']
pool = s['phase1']['pool_size_bytes']
delta = s['phase1']['delta_ppl_pct']
print(f'oracle_ppl        = {oracle:.10f}')
print(f'roundtrip_ppl     = {rt:.10f}')
print(f'delta_ppl_pct     = {delta:+.2f}%')
print(f'compression_ratio = {ratio:.4f}x')
print(f'pool_size_bytes   = {pool:,} ({pool/1e6:.1f} MB)')
"
```

Expected output:

```
oracle_ppl        = 4.7607620871
roundtrip_ppl     = 4.7607620871
delta_ppl_pct     = +0.00%
compression_ratio = 11.1304x
pool_size_bytes   = 36,175,872 (36.2 MB)
```

You can also `cat results/bench/ppl/smollm2-1.7b/wikitext-2/report.md`
for the human-readable version, or read `pool_manifest.json` for just
the poly-kv pool manifest extracted from the 1.1GB `roundtrip.bin`.

## Reproduce from scratch on a GPU host (5-10 minutes)

Requirements: ~2GB VRAM (we used a 7.91GB GTX 1070), ~3GB disk for the
model, internet access to download `HuggingFaceTB/SmolLM2-1.7B-Instruct`
and `Salesforce/wikitext`.

```bash
# 1. Clone
git clone https://github.com/RecursiveIntell/kv-lossless-11x
cd kv-lossless-11x

# 2. Build the Rust CLI (FibQuant codec + poly-kv pool + fast roundtrip)
cargo build --release --example poly_kv_fast_roundtrip

# 3. Run the full Phase 0 / Phase 1 / Phase 2 validation
cd poly-kv/scripts
mkdir -p ../../results/bench/ppl/smollm2-1.7b/wikitext-2
PYTORCH_ALLOC_CONF=expandable_segments:True \
  python3 ppl_validate.py \
    --model HuggingFaceTB/SmolLM2-1.7B-Instruct \
    --corpus wikitext-2 \
    --n-tokens 1024 \
    --ppl-frac 0.3 \
    --output ../../results/bench/ppl/smollm2-1.7b/wikitext-2/state.json
```

The script writes three files at the output directory:

- `state.json` — the receipts (compare against the committed one)
- `report.md` — the human-readable report
- `roundtrip.bin` — the poly-kv pool + decompressed layer blobs (1.1 GB)

If you only want to verify the smoke (Phase 0 forward pass + PPL, no
compression) use `ppl_smoke.py` instead — runs in ~30 seconds.

## Verify the build independently

```bash
# Check the workspace builds clean
cargo check --workspace

# Run the roundtrip-parity tests (these prove fib-quant encode==decode to f32 epsilon)
cargo test --release -p fib-quant --no-default-features --test decode_batch_fast_parity

# Run the compact-bytes tests (these prove the new wire format roundtrips)
cargo test --release -p fib-quant --no-default-features --test compact_bytes_roundtrip
```

All six tests should pass. The two test files are the unit-level proof
that the codec math is correct, independent of the end-to-end PPL run.

## Reproduce all four committed runs

The committed validation matrix uses the same `ppl_validate.py` script
with different `(model, corpus, n_tokens)` triples. Each is a single
no-hup command; the roundtrip.bin is gitignored and rewritten each run.

```bash
# Primary run
python3 ppl_validate.py \
  --model HuggingFaceTB/SmolLM2-1.7B-Instruct \
  --model-slug smollm2-1.7b \
  --corpus wikitext-2 \
  --n-tokens 1024 \
  --ppl-frac 0.3 \
  --output ../../results/bench/ppl/smollm2-1.7b/wikitext-2/state.json

# Cross-model run (TinyLlama 1.1B)
python3 ppl_validate.py \
  --model TinyLlama/TinyLlama-1.1B-Chat-v1.0 \
  --model-slug tinyllama-1.1b \
  --corpus wikitext-2 \
  --n-tokens 1024 \
  --ppl-frac 0.3 \
  --output ../../results/bench/ppl/tinyllama-1.1b/wikitext-2/state.json

# Cross-model run (Qwen2.5 0.5B with GQA)
python3 ppl_validate.py \
  --model Qwen/Qwen2.5-0.5B-Instruct \
  --model-slug qwen2.5-0.5b \
  --corpus wikitext-2 \
  --n-tokens 1024 \
  --ppl-frac 0.3 \
  --output ../../results/bench/ppl/qwen2.5-0.5b/wikitext-2/state.json

# Cross-corpus run (code-source slice)
echo "poly-kv README + Cargo.toml + first 5 src files" > /tmp/code_corpus.txt
# (See poly-kv/scripts/build_poly_kv_corpus.py invocation; the corpus
# is any UTF-8 text ≥ n_tokens tokens)
python3 ppl_validate.py \
  --model HuggingFaceTB/SmolLM2-1.7B-Instruct \
  --model-slug smollm2-1.7b \
  --corpus 'file:/tmp/code_corpus.txt' \
  --n-tokens 1024 \
  --ppl-frac 0.3 \
  --output ../../results/bench/ppl/smollm2-1.7b/code-source/state.json

# Longer-context run (1280 tokens; OOMs at 1536 on 7.91GB GPU)
python3 ppl_validate.py \
  --model HuggingFaceTB/SmolLM2-1.7B-Instruct \
  --model-slug smollm2-1.7b \
  --corpus wikitext-2 \
  --n-tokens 1280 \
  --ppl-frac 0.3 \
  --output ../../results/bench/ppl/smollm2-1.7b/wikitext-2-n1280/state.json
```

Each run takes ~3-5 minutes on the 7.91 GB test GPU. The
`state.json` files are byte-comparable against the committed copies
at the same paths; the only field that varies is `*_seconds` (wall
time on the test host).

## What the script does NOT verify

- Multi-agent sharing (the pool is built once but the multi-agent
  injection path via `materialize_shell` is not exercised). The
  `materialize_shell` API exists in `poly-kv/src/shell.rs:68` and
  is called via `pool.materialize_shell(...)` in
  `poly-kv/src/pool.rs:276`. Open work.
- Cross-model beyond Qwen2.5-0.5B-Instruct (SmolLM2-1.7B,
  TinyLlama-1.1B, Qwen2.5-0.5B are validated; Qwen-7B+ OOMs on
  7.91GB; Llama-3.2-1B is gated and needs HF auth)
- Cross-corpus beyond code-source (WikiText-2 + code-source are
  validated; C4, PG-19, cnn_dailymail, etc. are not)
- Long context beyond 1280 tokens (7.91GB GPU OOMs at 1536; A100
  needed for 2K+)
- Comparison against Google's TurboQuant at matched bit rate
  (fib_k4_n32 is at b=1.25; TurboQuant is at b=8; the bit-rate
  gap is 6.4×, so they cannot be directly compared at matched b)

These are open work. See the "Open work" section in the top-level
README for the full claim boundary.
