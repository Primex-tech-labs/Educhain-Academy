# Roadmap

This roadmap maps the Educhain-Academy vision to concrete, reviewable work. Each
phase is broken into issues labeled with `complexity: trivial`, `complexity:
medium`, or `complexity: high` so contributors can pick work that matches their
experience.

## Phase 1 — Foundation (in progress)

Three Soroban contracts deployed to testnet, a score read path in the TypeScript
SDK, and CI that builds and tests everything.

- [x] `identity-oracle`: DID anchoring + verifiable-credential anchoring
- [x] `credit-oracle`: Skill Score computation
- [x] `revocation-registry`: single and batch credential revocation
- [x] Testnet deployment script
- [ ] `get_active_vc_count` on `identity-oracle` (#see issues)
- [ ] SDK `isVerified()` read method

## Phase 2 — SDK and tooling

- [ ] SDK `anchorDID()` (sign + submit)
- [ ] SDK `issueVC()` and `revokeVC()`
- [ ] Learner CLI: create DID, issue a credential, read a Skill Score
- [ ] Example app that renders a shareable, verifiable learner profile

## Phase 3 — Cross-contract correctness

- [ ] `credit-oracle` reads `get_active_vc_count` directly from `identity-oracle`
- [ ] Score-freshness enforcement (`last_updated` checks in the SDK)
- [ ] Revocation propagation tests across all three contracts

## Phase 4 — Privacy

- [ ] Research spike: ZK selective disclosure ("Skill Score > 650" without revealing the number)
- [ ] Design an off-chain credential holder wallet

## Phase 5 — Governance

- [ ] DAO contract for scoring-weight upgrades with a timelock
- [ ] Issuer onboarding + public registry of trusted issuers

## Phase 6 — Mainnet readiness

- [ ] External security audit
- [ ] Mainnet deployment and monitoring
- [ ] Published issuer onboarding guide for universities and bootcamps

## How to get involved

1. Read [CONTRIBUTING.md](CONTRIBUTING.md).
2. Browse issues labeled [`good first issue`](https://github.com/Primex-tech-labs/Educhain-Academy/labels/good%20first%20issue).
3. Comment on the issue you want and wait to be assigned before opening a PR.
