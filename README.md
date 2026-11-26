![CI](https://github.com/Primex-tech-labs/Educhain-Academy/actions/workflows/ci.yml/badge.svg)

# Educhain-Academy

A decentralized learning-credential and skill-scoring protocol built on Stellar. Learners own their academic identity as a cryptographic keypair, collect verifiable credentials (certificates, course completions, skill badges) from trusted issuers, and receive a portable **Skill Score** computed transparently on-chain — no central registrar, no transcript lock-in, no unverifiable CVs.

**Status:** This project is in active development. See [Contributing](#contributing) to get started.

## Table of contents

- [The problem](#the-problem)
- [How it works](#how-it-works)
- [Architecture](#architecture)
- [Contracts](#contracts)
- [Deployed contracts](#deployed-contracts)
- [Scoring formula](#scoring-formula)
- [Quick start](#quick-start)
- [Running tests](#running-tests)
- [Project structure](#project-structure)
- [TypeScript SDK](#typescript-sdk)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## The problem

Learners everywhere complete courses, bootcamps, certifications, and job experience that never translate into portable, verifiable proof of skill. Diplomas sit in siloed registries, certificates are screenshots that anyone can fake, and employers have no way to check whether a candidate's stated skills are real. The people who suffer most are self-taught developers, learners in emerging markets, and career-switchers without formal transcripts — they have the skills but no institution willing to vouch for them.

The result: talent is filtered by paperwork instead of ability. Employers over-rely on brand-name degrees because they cannot verify anything else, and capable learners are locked out.

This protocol flips the model. Learning identity and achievement history are owned by the learner, anchored on a public ledger, and verifiable by any employer, school, or verifier without a central intermediary.

---

## How it works

The protocol has three steps:

**1. Get a decentralized identity (DID)**
A learner generates a Stellar keypair. Their public key becomes their DID: `did:stellar:testnet:G...`. They publish a [DID document](docs/did-spec.md#23-complete-example-document) to IPFS and anchor its content hash to the Stellar ledger via the identity-oracle contract. No registration required — the keypair is the identity. See [DID Document Schema](docs/did-spec.md#2-did-document-schema) for the required JSON-LD structure.

**2. Collect verifiable credentials (VCs)**
Trusted issuers — universities, bootcamps, certifying bodies, training providers, and employers — sign JSON-LD credentials attesting to facts about the learner (course completed, skill mastered, assessment passed, certificate earned). The SHA-256 hash of each credential is anchored on-chain. The credential itself stays off-chain, preserving privacy.

**3. Skill Score computed on-chain**
The credit-oracle Soroban contract aggregates anchored VC hashes, on-chain learning-activity statistics, and completion records into a composite **Skill Score** from 300 to 850. Any employer, school, or verifier can query the score permissionlessly. The scoring weights are governed and upgradeable.

---

## Architecture

```mermaid
graph TB
    subgraph Off-chain
        USER[Learner / DID keypair]
        ISSUER[Credential Issuer]
        IPFS[(IPFS — DID docs & VCs)]
    end

    subgraph Stellar Ledger
        IO[identity-oracle\nDID anchor · VC hash registry]
        CO[credit-oracle\nSkill Score computation · Completion history]
        RR[revocation-registry\nVC status list]
    end

    subgraph Consumers
        EMPLOYER[Employer / Hiring Platform]
        SCHOOL[School / Admissions]
        VERIFIER[Third-party Verifier]
    end

    USER    -->|anchor_did CID| IO
    ISSUER  -->|anchor_vc hash| IO
    ISSUER  -->|store full VC| IPFS
    USER    -->|store DID doc| IPFS
    IO      -->|is_verified check| CO
    RR      -->|revocation check| IO
    ISSUER  -->|revoke| RR
    CO      -->|get_score| EMPLOYER
    CO      -->|get_score| SCHOOL
    IO      -->|verify_vc| VERIFIER
```

---

## Contracts

The protocol is composed of three Soroban smart contracts deployed on the Stellar network.

### identity-oracle

Manages decentralized learner identifiers and verifiable credential anchoring.

| Function                                    | Description                                    |
| ------------------------------------------- | ---------------------------------------------- |
| `initialize(admin)`                         | Sets the contract admin                        |
| `register_issuer(admin, issuer)`            | Adds a trusted VC issuer                       |
| `deregister_issuer(admin, issuer)`          | Revokes a trusted issuer (existing VCs persist) |
| `anchor_did(subject, did_doc_cid)`          | Stores the IPFS CID of a DID document          |
| `anchor_vc(issuer, subject, vc_hash)`       | Anchors a VC hash from a trusted issuer        |
| `is_verified(subject)`                      | Returns true if subject has ≥ 1 non-revoked VC |
| `get_vc_count(subject)`                     | Returns the number of anchored VCs             |
| `verify_vc(subject, vc_hash)`               | Checks if a specific VC hash is valid          |
| `mark_vc_revoked(issuer, subject, vc_hash)` | Marks a VC as revoked                          |
| `upgrade(admin, new_wasm_hash)`             | Upgrades the contract WASM in-place            |

### credit-oracle

Computes and stores Skill Scores based on on-chain data.

| Function                                             | Description                                        |
| ---------------------------------------------------- | -------------------------------------------------- |
| `initialize(admin)`                                  | Sets admin and default scoring weights (40/30/30)  |
| `register_feeder(admin, feeder)`                     | Registers a trusted activity-stats feeder          |
| `deregister_feeder(admin, feeder)`                   | Revokes a trusted feeder (no retroactive effect)   |
| `register_lender(admin, lender)`                     | Registers a trusted issuer for completion recording |
| `deregister_lender(admin, lender)`                   | Revokes a trusted issuer (no retroactive effect)   |
| `update_tx_stats(feeder, subject, stats)`            | Updates 30-day learning-activity statistics        |
| `record_repayment(lender, subject, amount, on_time)` | Records a course/assessment completion outcome     |
| `compute_score(subject)`                             | Computes and persists the Skill Score              |
| `get_score(subject)`                                 | Returns the latest ScoreRecord                     |
| `propose_weights(weights)`                           | Proposes new weights with 24h timelock             |
| `apply_weights()`                                    | Applies pending weights after timelock expires     |
| `get_scoring_weights()`                              | Returns current scoring weights                    |

### revocation-registry

Maintains an on-chain list of revoked credential hashes.

| Function                          | Description                                     |
| --------------------------------- | ----------------------------------------------- |
| `initialize(admin)`               | Sets the contract admin                         |
| `revoke(issuer, vc_hash)`         | Revokes a credential by hash                    |
| `batch_revoke(issuer, vc_hashes)` | Revokes multiple credentials in one transaction |
| `is_revoked(vc_hash)`             | Returns true if the credential has been revoked |
| `upgrade(admin, new_wasm_hash)`   | Upgrades the contract WASM in-place             |

---

## Deployed contracts

### Testnet

| Contract            | Address                                                    | Explorer                                                                                                          |
| ------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| identity-oracle     | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` | [view](https://stellar.expert/explorer/testnet/contract/CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX) |
| credit-oracle       | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` | [view](https://stellar.expert/explorer/testnet/contract/CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX) |
| revocation-registry | `CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` | [view](https://stellar.expert/explorer/testnet/contract/CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX) |

Full deployment record: [deployments.testnet.json](deployments.testnet.json). Run `bash scripts/deploy.sh` to deploy your own instance.

---

## Scoring formula

The **Skill Score** ranges from 300 (no history) to 850 (exceptional). It is computed from three weighted components:

```
vc_score    = min(vc_count × 20, 100)
tx_score    = min(volume_30d_stroops ÷ 100_000_000, 100)   # 1 point per XLM of activity, cap 100
repay_score = (on_time_count × 10000 ÷ total_count) ÷ 100  # 0–100 completion rate, integer division

composite   = (vc_score × vc_weight
             + tx_score × tx_weight
             + repay_score × repayment_weight) ÷ 100

final_score = clamp(300 + composite × 550 ÷ 100, 300, 850)
```

Default weights: `vc_weight = 40`, `tx_weight = 30`, `repayment_weight = 30`

**Example scores** (all arithmetic uses integer division, matching the contract):

| Profile          | VCs | 30d Activity | Completion rate | Score |
| ---------------- | --- | ------------ | --------------- | ----- |
| New learner      | 0   | 0 XLM        | —               | 300   |
| Early stage      | 1   | 5 XLM        | 70%             | 465   |
| Established      | 2   | 20 XLM       | 85%             | 558   |
| Strong           | 3   | 50 XLM       | 95%             | 668   |
| Exceptional      | 5   | 100+ XLM     | 100%            | 850   |

Full formula documentation with worked examples: [docs/scoring-spec.md](docs/scoring-spec.md)

---

## Quick start

### Prerequisites

- Rust stable — `rustup update stable`
- `stellar-cli` 21+ — `cargo install --locked stellar-cli --features opt`
- Node.js 18+ and pnpm — `npm install -g pnpm`
- A funded Stellar testnet account — `stellar keys generate --global deployer --network testnet`

### Setup

```bash
# Clone the repo
git clone https://github.com/Primex-tech-labs/Educhain-Academy
cd Educhain-Academy

# Install TypeScript dependencies
pnpm install

# Run all tests
pnpm test
```

### Deploy to testnet

```bash
# Fund your deployer key
curl "https://friendbot.stellar.org/?addr=$(stellar keys address deployer)"

# Deploy all three contracts
bash scripts/deploy.sh
```

Contract addresses will be saved to `deployments.testnet.json`.

---

## Running tests

Run all Rust and TypeScript tests:

```bash
pnpm test
```

For individual commands:

```bash
# Run all Rust contract tests (including integration tests)
cargo test --workspace

# Run with output for debugging
cargo test --workspace -- --nocapture

# Lint Rust contracts and TypeScript
pnpm lint

# Build release binaries
pnpm build

# Run a specific contract's tests
cargo test -p identity-oracle
cargo test -p credit-oracle
cargo test -p revocation-registry

# Run integration tests only
cargo test -p integration-tests
```

All tests use Soroban's built-in testutils — no live network required.

---

## Project structure

```
Educhain-Academy/
├── contracts/
│   ├── identity-oracle/
│   │   └── src/lib.rs          # DID anchor + VC hash registry
│   ├── credit-oracle/
│   │   └── src/lib.rs          # Skill Score computation + completion history
│   ├── revocation-registry/
│   │   └── src/lib.rs          # VC status list
│   └── tests/
│       └── src/integration_test.rs  # Cross-contract integration tests
├── packages/
│   └── sdk/
│       └── src/index.ts        # TypeScript SDK
├── docs/
│   ├── architecture.md         # Full component breakdown
│   ├── did-spec.md             # DID method specification
│   └── scoring-spec.md         # Scoring formula + worked examples
├── scripts/
│   └── deploy.sh               # Testnet deployment script
├── Cargo.toml                  # Workspace root
├── pnpm-workspace.yaml
├── CONTRIBUTING.md
└── LICENSE                     # Apache-2.0
```

---

## TypeScript SDK

The `@educhain-academy/sdk` package provides a typed client for interacting with all three contracts from a TypeScript application.

```typescript
import { EduchainAcademySDK } from "@educhain-academy/sdk";

const sdk = new EduchainAcademySDK({
  identityOracleId: "CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  creditOracleId: "CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  revocationRegistryId:
    "CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  networkPassphrase: "Test SDF Network ; September 2015",
  rpcUrl: "https://soroban-testnet.stellar.org",
});

// Read a Skill Score (read-only, no fees)
const score = await sdk.getScore("G...");
console.log(score.score); // e.g. 612
```

### SDK status

| Method                           | Status         |
| -------------------------------- | -------------- |
| `getScore(address)`              | ✅ Implemented |
| `isVerified(address)`            | 🚧 Open        |
| `anchorDID(keypair, cid)`        | 🚧 Open        |
| `issueVC(issuer, subject, hash)` | 🚧 Open        |
| `verifyVC(subject, hash)`        | ✅ Implemented |
| `revokeVC(issuer, hash)`         | 📋 Planned     |

---

## Component status

| Component               | Status         | Notes                                |
| ----------------------- | -------------- | ------------------------------------ |
| identity-oracle         | ✅ Complete    | All functions implemented and tested |
| credit-oracle           | ✅ Complete    | Scoring formula live on testnet      |
| revocation-registry     | ✅ Complete    | Batch revocation supported           |
| TypeScript SDK          | 🚧 In progress | `getScore` done, rest open           |
| CLI tool                | 📋 Planned     |                                      |
| Cross-contract vc_count | 📋 Planned     |                                      |
| ZK proof layer          | 📋 Research    |                                      |
| Governance contract     | 📋 Planned     |                                      |

---

## Roadmap

**Phase 1 — Foundation (current)**
Three core contracts deployed on testnet. TypeScript SDK for score reading. Passing CI.

**Phase 2 — SDK & tooling (contributors)**
Full TypeScript SDK with DID creation, VC issuance, and revocation. CLI tool for developers.

**Phase 3 — Cross-contract integration**
credit-oracle reads `vc_count` directly from identity-oracle via cross-contract call. Score freshness enforcement.

**Phase 4 — Privacy layer**
ZK proof circuit for selective score disclosure — prove "Skill Score > 650" without revealing the exact number or underlying credentials.

**Phase 5 — Governance**
DAO contract for scoring weight upgrades. Token-weighted voting. Timelock on changes.

**Phase 6 — Mainnet**
Security audit. Mainnet deployment. Issuer onboarding program.

---

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for setup and guidelines.

### How to contribute

1. Browse [open issues](https://github.com/Primex-tech-labs/Educhain-Academy/issues) — look for `good first issue` to start
2. Comment on the issue to signal you're working on it
3. Fork the repo and create a branch: `git checkout -b feat/your-feature`
4. Write your code with tests — `cargo test --workspace` must pass
5. Open a pull request referencing the issue number

### Issue complexity levels

| Label     | Points  | Typical scope                                               |
| --------- | ------- | ----------------------------------------------------------- |
| `trivial` | 100 pts | Single function, clear spec, minimal context needed         |
| `medium`  | 150 pts | Multiple functions or cross-package work                    |
| `high`    | 200 pts | New contract feature, ZK work, or architecture-level change |

### Development requirements

- `cargo clippy --workspace -- -D warnings` must pass with zero warnings
- Every public contract function must have a `///` doc comment
- New functions require at least one test
- No `unwrap()` in contract logic — use `expect("descriptive message")`
- Conventional commit messages: `feat:`, `fix:`, `test:`, `docs:`, `chore:`

Full setup and guidelines: [CONTRIBUTING.md](CONTRIBUTING.md)

### Resources

- [Stellar Developer Docs](https://developers.stellar.org)
- [Soroban Smart Contracts](https://soroban.stellar.org)
- [W3C DID Specification](https://www.w3.org/TR/did-core/)
- [W3C Verifiable Credentials](https://www.w3.org/TR/vc-data-model/)
- [Stellar Laboratory](https://laboratory.stellar.org)
- [Stellar Expert (Testnet Explorer)](https://stellar.expert/explorer/testnet)
- [Project Architecture](docs/architecture.md)
- [Scoring Specification](docs/scoring-spec.md)
- [DID Method Specification](docs/did-spec.md)

---

## License

Apache License 2.0 — see [LICENSE](LICENSE) for full text.
