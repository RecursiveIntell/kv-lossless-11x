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

## What the script does NOT verify

- Multi-agent sharing (the pool is built once but the multi-agent
  injection path is not exercised)
- Cross-model parity (only SmolLM2-1.7B is run; not Llama, Qwen, etc.)
- Cross-corpus (only WikiText-2 is run; not C4, PG-19, etc.)
- Long context (>1024 tokens; the 7.91GB GPU OOMs at 2048)
- Comparison against Google's TurboQuant at matched bit rate (we don't
  ship TurboQuant here; the FibQuant paper itself claims to beat it at
  b=2 on TinyLlama, but that is the paper's claim, not ours)

These are open work. See the top-level README for the full claim
boundary.
