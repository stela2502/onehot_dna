# onehot_dna

Fast one-hot DNA encoding for barcode correction, primer matching, and fixed-length sequence lookup.

Designed for workflows such as:

- BD Rhapsody C1/C2/C3 barcode correction
- 10x barcode matching
- Primer matching
- Adapter matching
- Small fixed-length sequence dictionaries

## Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
onehot_dna = "0.1"
```

Or from a local workspace:

```toml
[dependencies]
onehot_dna = { git = "https://github.com/stela2502/onehot_dna" }
```

## Quick Start

The most common workflow is:

1. Encode your candidate barcode dictionary once at startup.
2. Encode observed barcodes.
3. Find the best match using Hamming-like mismatch counting.

```rust
use onehot_dna::{encode_candidates, OneHot9};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let candidates = encode_candidates::<9, _>(&[
        b"AAAAAAAAA".as_slice(),
        b"CCCCCCCCC".as_slice(),
        b"GGGGGGGGG".as_slice(),
    ])?;

    let observed = OneHot9::from_bytes(b"AAAAAAAAC")?;

    if let Some((index, distance)) = observed.best_match(&candidates, 1) {
        println!("best candidate: {index}, mismatches: {distance}");
    }

    Ok(())
}
```

## Why One-Hot?

Bases are encoded as:

```text
A = 0001
C = 0010
G = 0100
T = 1000
N = 0000
```

Unknown bases count as mismatches.

This makes barcode correction simple and efficient using XOR and popcount operations.

## Pre-encoding Candidate Dictionaries

Encode barcode tables once and reuse them for millions of lookups.

```rust
let c1_candidates = encode_candidates::<9, _>(&c1_sequences)?;
let c2_candidates = encode_candidates::<9, _>(&c2_sequences)?;
let c3_candidates = encode_candidates::<9, _>(&c3_sequences)?;
```

## BD Rhapsody Example

BD Rhapsody cell barcodes consist of three independent 9 bp blocks:

```text
C1 (9 bp)
C2 (9 bp)
C3 (9 bp)
```

Each block can be corrected independently.

```rust
let observed_c1 = OneHot9::from_bytes(b"ACGTACGTA")?;

if let Some((idx, dist)) = observed_c1.best_match(&c1_candidates, 2) {
    println!("matched C1 barcode {idx} with {dist} mismatches");
}
```

## API Overview

```rust
OneHot::<N>::from_bytes(...)
OneHot::<N>::mismatches(...)
OneHot::<N>::best_match(...)
encode_candidates(...)
```

## Limitations

`OneHot<N>` stores four bits per base in a `u128`.

Maximum supported length:

```text
N <= 32
```

## Performance Notes

For a 9 bp barcode:

```text
9 bp × 4 bits = 36 bits
```

The encoded sequence is compact and supports very fast dictionary matching.