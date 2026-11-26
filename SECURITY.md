# Security Policy

## Supported Versions

| Version | Supported |
| ------- | --------- |
| `main`  | ✅        |
| < 0.1.0 | ❌        |

Educhain-Academy is pre-1.0 and under active development. Security fixes land on
`main`; older tags are not maintained.

## Reporting a Vulnerability

Please **do not open a public issue** for security problems.

Report privately using GitHub's [Security Advisories](https://github.com/Primex-tech-labs/Educhain-Academy/security/advisories/new)
or email **security@educhain.academy** with:

- A description of the issue and its impact
- Reproduction steps or a proof of concept
- Affected component (contract, SDK, or docs) and commit/version
- Any suggested remediation

You can expect an acknowledgement within **72 hours** and a remediation plan
within **7 days** for valid reports. We will coordinate disclosure with you and
credit reporters who wish to be acknowledged.

## Scope

In scope:

- `contracts/identity-oracle`, `contracts/credit-oracle`,
  `contracts/revocation-registry` (Soroban/Rust)
- `packages/sdk` (TypeScript)
- Deployment and CI scripts

Out of scope:

- Issues in third-party dependencies (report upstream)
- Theft of a learner's own keys without a protocol flaw
- Social engineering and physical attacks

## Handling Secrets

Never commit secrets, seed phrases, or private keys. The repo ships a secret
scanning workflow; if you accidentally commit a secret, revoke it immediately and
rotate it before opening a pull request.
