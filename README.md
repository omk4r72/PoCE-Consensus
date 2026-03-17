# PoCE — Proof of Clean Execution

> A blockchain consensus protocol that cryptographically excludes malware-compromised validators.

---

## The Problem

Every major blockchain today — Bitcoin, Ethereum, Solana, Avalanche — selects validators based on **computational power** or **token stake**. Neither mechanism checks whether the validator node itself has been compromised.

**A node running malware can validate blocks on any existing blockchain.**

PoCE solves this.

---

## What PoCE Does Differently

PoCE introduces the **AttestationChain** — a per-node, epoch-linked cryptographic record of binary integrity.

```
attest[e] = SHA-256( binary_hash || attest[e-1] || epoch || node_id )
```

- `binary_hash` = SHA-256 of the node's running executable (`/proc/self/exe`)
- Each epoch, the node re-attests and links to the previous attestation
- A node whose binary changes (malware injected) produces a different `binary_hash`
- This breaks the expected chain → node is **permanently excluded** from validator selection

**No existing mainnet blockchain implements this.**

---

## Algorithm: Validator Score

```
VS(v) = 0.40 × norm_stake
      + 0.30 × reputation
      + 0.20 × attest_bonus        ← clean_streak / 10
      + 0.10 × vrf_rank            ← sha256(epoch_seed || node_id)

HARD GATE: if attest_chain.is_eligible() == false → VS = 0 → never selected
```

A node is `eligible` only if:
1. It has at least `ATTEST_CHAIN_MIN` (3) epochs of history
2. Every recent attestation shows `binary_hash == expected_binary_hash`
3. The chain is cryptographically unbroken (each record links to the previous)

---

## Consensus Protocol

Three-phase BFT with VRF-based leader election:

```
epoch_seed  = SHA-256( prev_block_hash || epoch )
committee   = top-k nodes by VS (only eligible nodes considered)
proposer    = committee member with lowest VRF rank (verifiable by anyone)

PROPOSE     proposer  → all : Block
PREPARE     validator → all : PREPARE( block_hash )   if block valid
COMMIT      validator → all : COMMIT( block_hash )    if ≥ ⌈2k/3⌉+1 PREPARE
FINALIZE    any node         : block accepted          if ≥ ⌈2k/3⌉+1 COMMIT
```

Equivocation (double-vote) triggers automatic stake slashing.

---

## Comparison with Existing Algorithms

| Property                        | PoW | PoS | PBFT | Avalanche | **PoCE** |
|---------------------------------|-----|-----|------|-----------|----------|
| Binary integrity check          | ✗   | ✗   | ✗    | ✗         | **✓**    |
| Malware node exclusion          | ✗   | ✗   | ✗    | ✗         | **✓**    |
| Energy efficient                | ✗   | ✓   | ✓    | ✓         | **✓**    |
| No wealth centralization        | ✗   | ✗   | ✓    | ✗         | **✓**    |
| Cryptographic leader election   | ✗   | ✗   | ✗    | ✓         | **✓**    |
| Equivocation slashing           | ✗   | ✓   | ✗    | ✗         | **✓**    |
| Attestation chain linkage       | ✗   | ✗   | ✗    | ✗         | **✓**    |

---

## Results (20 nodes, 5 compromised, 10 rounds)

```
Eligible validators : 15/20   (5 malware nodes excluded)
Blocks finalized    : 10/10
Chain integrity     : PASS
Security check      : All 5 compromised nodes EXCLUDED from all 10 blocks. SECURE.
SHA-256 self-test   : NIST vectors — PASS
Time                : 0.005s
```

---

## Build & Run

**Requirements:** `g++` with C++17, Linux (uses POSIX sockets for P2P stub)

```bash
# Clone
git clone https://github.com/YOUR_USERNAME/PoCE-Consensus
cd PoCE-Consensus

# Build
g++ -O2 -std=c++17 poce_full.cpp -o poce

# Run
./poce
```

No external dependencies. SHA-256 implemented from scratch.

---

## Production Integration

To use real binary hashing (instead of simulated):

```cpp
// In poce_full.cpp, replace the binary_hash line with:
#include <fstream>
std::ifstream f("/proc/self/exe", std::ios::binary);
std::vector<uint8_t> buf(
    std::istreambuf_iterator<char>(f), {}
);
Hash256 actual_binary = sha::digest(buf.data(), buf.size());
```

For trusted execution environments (TEE), replace with remote attestation report from Intel SGX or AMD SEV.

---

## Repository Structure

```
PoCE-Consensus/
├── poce_full.cpp      — complete single-file implementation
├── README.md          — this file
└── WHITEPAPER.md      — full research paper
```

---

## Author

Omkar — Security Researcher  
Specialization: Malware Analysis, Reverse Engineering, Blockchain Security

---

## License

MIT License. Free to use, fork, and build upon.

---

## Citation

If you use PoCE in your research:

```bibtex
@misc{poce2024,
  title   = {PoCE: Proof of Clean Execution — A Malware-Resistant Blockchain Consensus Protocol},
  author  = {Omkar},
  year    = {2024},
  note    = {https://github.com/omk4r72/PoCE-Consensus/}
}
```
