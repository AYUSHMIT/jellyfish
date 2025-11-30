# Jellyfish cryptographic library

![example workflow](https://github.com/EspressoSystems/jellyfish/actions/workflows/build.yml/badge.svg)
![Crates.io (version)](https://img.shields.io/crates/dv/jf-plonk/0.1.0)
![GitHub](https://img.shields.io/github/license/EspressoSystems/jellyfish)

## Disclaimer

**DISCLAIMER:** This software is provided "as is" and its security has not been externally audited. Use at your own risk.

## Chatroom

For general discussions on Jellyfish PLONK, please join our [Discord channel](https://discord.gg/GJa4gznGfU).

## Crates

### Helper
- ['jf-utils'](utilities): utilities and helper functions.

### Primitives
- [`jf-prf`](prf): trait definitions for pseudorandom function (PRF).
- [`jf-crhf`](crhf): trait definitions for collision-resistant hash function (CRHF).
- [`jf-commitment`](commitment): trait definitions for cryptographic commitment scheme.
- [`jf-rescue`](rescue): Rescue hash function, and its subsequent PRF, CRHF, commitment scheme implementations.
- [`jf-elgamal`](elgamal): a Rescue-based ElGamal encryption scheme implementation.
- [`jf-signature`](signature): signature scheme trait definition, and BLS/Schnorr signature scheme implementations.
- [`jf-vrf`](vrf): verifiable random function trait definition and BLS-based implementation.
- [`jf-aead`](aead): authenticated encryption with associated data (AEAD) implementation.
- [`jf-merkle-tree`](merkle_tree): various (vanilla, sparse, namespaced) Merkle tree trait definitions and implementations.
- [`jf-pcs`](pcs): polynomial commitment scheme (PCS) trait definitions and univariate/multilinear KZG-PCS implementations.
- [`jf-poseidon2`](poseidon2): Poseidon2 permutation, and Sponge, CRHF, derived from this permutation.

The `jf-vid` crate is now moved to another repo [`jf-advz`](https://github.com/EspressoSystems/jellyfish-compat/tree/main/advz).

### Plonk
- [`jf-relation`](relation): Jellyfish constraint system for PLONK.
- [`jf-plonk`](plonk): KZG-PCS based TurboPlonk and UltraPlonk implementations.

## Usage

### Adding Jellyfish as a Dependency

Add the following to your `Cargo.toml` to use the PLONK proving system:

```toml
[dependencies]
jf-plonk = { git = "https://github.com/EspressoSystems/jellyfish", tag = "jf-plonk-v0.7.0" }
jf-relation = { git = "https://github.com/EspressoSystems/jellyfish", tag = "jf-relation-v0.5.0" }

# For testing with locally generated SRS (not for production)
[features]
test-srs = ["jf-plonk/test-srs"]
```

You'll also need arkworks dependencies for the specific curve you want to use:

```toml
[dependencies]
ark-bls12-381 = "0.5"
ark-ed-on-bls12-381 = "0.5"
ark-ff = "0.5"
ark-std = "0.5"
```

### Defining a Circuit

Jellyfish provides a constraint system for building PLONK circuits. You can compose circuits using built-in gadgets or build custom gates.

#### Using Built-in Gadgets

```rust
use ark_bls12_381::Fr;
use jf_relation::{Circuit, PlonkCircuit};

fn build_simple_circuit() -> Result<PlonkCircuit<Fr>, jf_relation::CircuitError> {
    // Create a TurboPlonk circuit
    let mut circuit = PlonkCircuit::<Fr>::new_turbo_plonk();

    // Create variables (private witnesses)
    let a = circuit.create_variable(Fr::from(3u64))?;
    let b = circuit.create_variable(Fr::from(5u64))?;
    
    // Create public input variable
    let c = circuit.create_public_variable(Fr::from(15u64))?;
    
    // Add constraint: a * b = c
    circuit.mul_gate(a, b, c)?;
    
    // Finalize the circuit for proving
    circuit.finalize_for_arithmetization()?;
    
    Ok(circuit)
}
```

#### Building Custom Gates

For more complex constraints, you can use the quadratic polynomial gate:

```rust
use ark_bls12_381::Fr;
use jf_relation::{Circuit, PlonkCircuit};
use jf_relation::constants::{GATE_WIDTH, N_MUL_SELECTORS};

fn build_custom_gate_circuit() -> Result<PlonkCircuit<Fr>, jf_relation::CircuitError> {
    let mut circuit = PlonkCircuit::<Fr>::new_turbo_plonk();
    
    // Create variables
    let a = circuit.create_variable(Fr::from(2u64))?;
    let b = circuit.create_variable(Fr::from(3u64))?;
    let c = circuit.create_variable(Fr::from(4u64))?;
    let d = circuit.create_variable(Fr::from(5u64))?;
    
    // Quadratic polynomial gate: q1*a + q2*b + q3*c + q4*d + q12*a*b + q34*c*d + q_c = q_o*e
    let q_lc = [Fr::from(1u64), Fr::from(2u64), Fr::from(0u64), Fr::from(0u64)]; // linear coefficients
    let q_mul = [Fr::from(1u64), Fr::from(0u64)]; // multiplication coefficients
    let q_c = Fr::from(0u64); // constant term
    
    // This computes: 1*a + 2*b + 1*a*b = e
    let e = circuit.gen_quad_poly(&[a, b, c, d], &q_lc, &q_mul, q_c)?;
    
    circuit.finalize_for_arithmetization()?;
    
    Ok(circuit)
}
```

#### Using ECC Gadgets

For elliptic curve operations (useful for signature verification, key derivation, etc.):

```rust
use ark_bls12_381::Bls12_381;
use ark_ec::twisted_edwards::Affine as TEAffine;
use ark_ed_on_bls12_381::{EdwardsAffine, EdwardsConfig, Fr};
use ark_std::UniformRand;
use jf_relation::{Circuit, PlonkCircuit};
use jf_relation::gadgets::ecc::TEPoint;

fn build_ecc_circuit(
    scalar: Fr,
    point: EdwardsAffine,
) -> Result<PlonkCircuit<<EdwardsConfig as ark_ec::CurveConfig>::BaseField>, jf_relation::CircuitError> {
    use jf_utils::fr_to_fq;
    
    let mut circuit = PlonkCircuit::new_turbo_plonk();
    
    // Create scalar variable (private witness)
    let scalar_fq = fr_to_fq::<_, EdwardsConfig>(&scalar);
    let scalar_var = circuit.create_variable(scalar_fq)?;
    
    // Create point variable (as constant)
    let point_jf: TEPoint<_> = point.into();
    let point_var = circuit.create_constant_point_variable(point_jf)?;
    
    // Perform scalar multiplication: result = scalar * point
    let result_var = circuit.variable_base_scalar_mul::<EdwardsConfig>(scalar_var, &point_var)?;
    
    circuit.finalize_for_arithmetization()?;
    
    Ok(circuit)
}
```

### Generating Proving and Verifying Keys (Setup Phase)

After defining your circuit, generate the proving and verifying keys:

```rust
use ark_bls12_381::Bls12_381;
use ark_std::rand::SeedableRng;
use jf_plonk::proof_system::{PlonkKzgSnark, UniversalSNARK};
use rand_chacha::ChaCha20Rng;

fn setup_keys<C: jf_relation::Arithmetization<ark_bls12_381::Fr>>(
    circuit: &C,
) -> Result<(
    <PlonkKzgSnark<Bls12_381> as UniversalSNARK<Bls12_381>>::ProvingKey,
    <PlonkKzgSnark<Bls12_381> as UniversalSNARK<Bls12_381>>::VerifyingKey,
), jf_plonk::errors::PlonkError> {
    let mut rng = ChaCha20Rng::from_seed([0u8; 32]);
    
    // Get the required SRS size from the circuit
    let srs_size = circuit.srs_size()?;
    
    // Generate universal SRS (for testing only - use MPC ceremony in production)
    #[cfg(feature = "test-srs")]
    let srs = PlonkKzgSnark::<Bls12_381>::universal_setup_for_testing(srs_size, &mut rng)?;
    
    // Generate proving and verifying keys
    let (proving_key, verifying_key) = PlonkKzgSnark::<Bls12_381>::preprocess(&srs, circuit)?;
    
    Ok((proving_key, verifying_key))
}
```

### Producing Proofs

Generate a proof for a satisfied circuit:

```rust
use ark_bls12_381::Bls12_381;
use ark_std::rand::SeedableRng;
use jf_plonk::{
    proof_system::{PlonkKzgSnark, UniversalSNARK, structs::ProvingKey},
    transcript::StandardTranscript,
};
use rand_chacha::ChaCha20Rng;

fn generate_proof<C: jf_relation::Arithmetization<ark_bls12_381::Fr>>(
    circuit: &C,
    proving_key: &ProvingKey<Bls12_381>,
) -> Result<<PlonkKzgSnark<Bls12_381> as UniversalSNARK<Bls12_381>>::Proof, jf_plonk::errors::PlonkError> {
    let mut rng = ChaCha20Rng::from_seed([0u8; 32]);
    
    // Generate proof using StandardTranscript for Fiat-Shamir
    let proof = PlonkKzgSnark::<Bls12_381>::prove::<_, _, StandardTranscript>(
        &mut rng,
        circuit,
        proving_key,
        None, // Optional extra transcript message
    )?;
    
    Ok(proof)
}
```

### Verifying Proofs

#### Off-chain Verification

```rust
use ark_bls12_381::{Bls12_381, Fr};
use jf_plonk::{
    proof_system::{PlonkKzgSnark, UniversalSNARK, structs::{Proof, VerifyingKey}},
    transcript::StandardTranscript,
};

fn verify_proof(
    verifying_key: &VerifyingKey<Bls12_381>,
    public_inputs: &[Fr],
    proof: &Proof<Bls12_381>,
) -> Result<(), jf_plonk::errors::PlonkError> {
    PlonkKzgSnark::<Bls12_381>::verify::<StandardTranscript>(
        verifying_key,
        public_inputs,
        proof,
        None, // Optional extra transcript message (must match prover)
    )
}
```

#### On-chain Verification (Solidity)

For Ethereum smart contract verification, use the `SolidityTranscript`:

```rust
use ark_bls12_381::{Bls12_381, Fr};
use jf_plonk::{
    proof_system::{PlonkKzgSnark, UniversalSNARK, structs::{Proof, VerifyingKey}},
    transcript::SolidityTranscript,
};

fn verify_proof_for_solidity(
    verifying_key: &VerifyingKey<Bls12_381>,
    public_inputs: &[Fr],
    proof: &Proof<Bls12_381>,
) -> Result<(), jf_plonk::errors::PlonkError> {
    PlonkKzgSnark::<Bls12_381>::verify::<SolidityTranscript>(
        verifying_key,
        public_inputs,
        proof,
        None,
    )
}
```

### Complete Example

See [`plonk/examples/proof_of_exp.rs`](plonk/examples/proof_of_exp.rs) for a complete working example that demonstrates:
- Building a proof-of-knowledge circuit for discrete log
- Setting up the proving system
- Generating and verifying proofs

Run it with:
```bash
cargo run --release --example proof-of-exp --features test-srs
```

### Available Gadgets

The `jf-relation` crate provides several built-in gadgets:

- **Arithmetic**: `add`, `sub`, `mul`, `enforce_constant`, `enforce_equal`
- **Boolean**: `enforce_bool`, `create_boolean_variable`, `logic_and`, `logic_or`
- **Comparison**: `is_less_than`, `enforce_less_than`
- **Range**: `add_range_check_variable` (UltraPlonk only)
- **ECC**: `variable_base_scalar_mul`, `fixed_base_scalar_mul`, `point_addition`
- **Lookup tables**: `create_table_and_lookup_variables` (UltraPlonk only)

### TurboPlonk vs UltraPlonk

- **TurboPlonk**: Standard PLONK with custom gates. Use `PlonkCircuit::new_turbo_plonk()`.
- **UltraPlonk**: Extended PLONK with lookup table support for efficient range checks and table lookups. Use `PlonkCircuit::new_ultra_plonk(range_bit_len)`.

## Development environment setup

We recommend the following tools:

- [`nix`](https://nixos.org/download.html)
- [`direnv`](https://direnv.net/docs/installation.html)

Run `direnv allow` at the repo root. You should see dependencies (including Rust) being installed.
Alternatively, enter the nix-shell manually via `nix develop`.

You can check you are in the correct development environment by running `which cargo`, which should print
something like `/nix/store/2gb31jhahrm59n3lhpv1lw0wfax9cf9v-rust-minimal-1.69.0/bin/cargo`;
and running `echo $CARGO_HOME` should print `~/.cargo-nix`.

## Build, run tests and examples

Build:

```
cargo build
```

Run an example:

```
cargo run --release --example proof-of-exp --features test-srs
```

This is a simple example to prove and verify knowledge of exponent.
It shows how one may compose a circuit, and then build a proof for the circuit.

### WASM target

Jellyfish is `no_std` compliant and compilable to WASM target environment, just run:

```
./scripts/build_wasm.sh
```

### Backends

To choose different backends for arithmetics of `curve25519-dalek`, which is currently
used by `jf-primitives/aead`, set the environment variable:

```
RUSTFLAGS='--cfg curve25519_dalek_backend="BACKEND"'
```

See the full list of backend options [here](https://github.com/dalek-cryptography/curve25519-dalek#backends).

You could further configure the word size for the backend by setting (see [here](https://github.com/dalek-cryptography/curve25519-dalek#word-size-for-serial-backends)):

```
RUSTFLAGS='--cfg curve25519_dalek_bits="SIZE"'
```

### Tests

```
cargo test --release
```

Note that by default the _release_ mode does not check integers overflow.
In order to enforce this check run:

```
./scripts/run_tests.sh
```

#### Test coverage

We use [grcov](https://github.com/mozilla/grcov) for test coverage

```
./scripts/test_coverage.sh
```

### Generate and read the documentation

#### Standard

```
cargo doc --open
```

### Code formatting

To format your code run

```
cargo fmt
```

### Updating non-cargo dependencies

Run `nix flake update` if you would like to pin other version edit `flake.nix`
beforehand. Commit the lock file when happy.

To update only a single input specify it as argument, for example

    nix flake update github:oxalica/rust-overlay

### Benchmarks

#### Primitives

Currently, a benchmark for verifying Merkle paths is implemented.
The additional flags allow using assembly implementation of `square_in_place` and `mul_assign` within arkworks:

```bash
RUSTFLAGS='-Ctarget-cpu=native -Ctarget-feature=+bmi2,+adx' cargo bench --bench=merkle_path
```

#### PLONK proof generation/verification

For benchmark, run:

```
RAYON_NUM_THREADS=N cargo bench
```

where N is the number of threads you want to use (N = 1 for single-thread).

A sample benchmark result is available under [`bench.md`](./bench.md).

## Git Hooks

The pre-commit hooks are installed via the nix shell. To run them on all files use

```
pre-commit run --all-files
```
