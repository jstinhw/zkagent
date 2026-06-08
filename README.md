# ZK-Agent

A trustless, privacy-preserving GitHub bounty protocol built on
[ZK-Email](https://prove.email/). Maintainers fund bounties on GitHub issues
with ERC-20 tokens, and contributors claim the reward by proving — in zero
knowledge — that their pull request was merged and the issue was closed. The
proof is derived from the DKIM-signed notification emails GitHub already sends,
so no oracle, bot, or trusted third party ever has to attest to what happened on
GitHub.

## How it works

GitHub sends DKIM-signed emails for repository activity (issue closed, PR
merged). Because the email is cryptographically signed by GitHub's mail server,
its contents can be verified without trusting the sender. ZK-Agent turns those
emails into succinct zk-SNARK proofs:

1. **Create** — A maintainer locks an ERC-20 reward against a `(repo, issueNo)`
   pair on-chain.
2. **Assign** — Anyone submits a proof generated from the "issue closed as
   completed via #PR" email, binding the bounty to the PR number that resolved
   it.
3. **Solve** — The PR author submits a proof generated from the "PR merged"
   email. The circuit binds the proof to the claimer's `toAddress` to prevent
   front-running, and the contract releases the reward (minus protocol fee).
4. **Cancel / Report** — A maintainer can cancel an unsolved bounty and reclaim
   funds. If a maintainer cancels a bounty that was in fact solved, the solver
   can submit their merge proof to `reportBounty`, which applies a time-based
   penalty (block) to the dishonest issuer.

Only the fields the protocol needs — the sender address, repository, issue
number, and PR number — are revealed from the email. The rest of the email
contents stay private.

## Architecture

The repository is a monorepo of three packages:

### `circuits/` — Circom zk-SNARK circuits

- `issue_closed.circom` — proves a GitHub "issue closed as completed via #N"
  email, revealing the repo, issue number, and resolving PR number.
- `pr_merged.circom` — proves a GitHub "PR merged" email, revealing the repo and
  PR number, and binds the proof to a claimer `to_address` (Groth16
  malleability / front-running protection).
- `regex/` — custom regex circuits that extract the `from` header, repository,
  issue number, PR number, and PR author from the raw email bytes.
- Built on `@zk-email/circuits` for DKIM/RSA verification and SHA-256
  precomputation. The `scripts/` directory contains the full Groth16 pipeline:
  input generation → witness → zkey → proof → Solidity verifier.

### `contracts/` — Solidity contracts (Hardhat)

- `ZKAgent.sol` — the core escrow. Holds bounties, handles create / assign /
  solve / cancel / report, protocol fees, and issuer penalties.
- `processors/` — `IssueProcessor` and `PRProcessor` verify a Groth16 proof,
  enforce that the email's `from` address is GitHub's, and unpack the revealed
  signals (repo, issue/PR number, claimer address) into typed values. Built on a
  `BaseProcessorV2` + mailserver key-hash adapter pattern (inspired by the
  zkp2p processor design).
- `verifiers/` — auto-generated Groth16 verifier contracts for each circuit.

### `helper/` — Input generation

- `src/generateInput.ts` — a CLI that takes a raw `.eml` GitHub notification
  email, runs DKIM verification, and produces the JSON witness input for the
  `issue_closed` or `pr_merged` circuit.

## Getting started

Each package is independent and uses Yarn.

### Circuits

```bash
cd circuits
yarn install
yarn compile        # compile both circuits to build/
yarn gen-input      # generate witness inputs from sample emails
yarn gen-wtns       # generate witnesses
yarn gen-zkey       # generate proving/verifying keys (unsafe/dev setup)
yarn gen-proof      # generate proofs
yarn gen-verifier   # export Solidity verifiers
yarn test           # run circuit + regex tests
```

### Generating circuit inputs from an email

```bash
cd helper
yarn install
npx ts-node src/generateInput.ts \
  --email=<path/to/email.eml> \
  --output=./output \
  --type=issue_closed        # or pr_merged (then also pass --to=<address>)
```

### Contracts

```bash
cd contracts
yarn install
yarn test           # run ZKAgent / processor tests
```

## Status

Research / proof-of-concept. The proving setup uses an unsafe (development)
trusted setup, and DKIM key-hash verification is stubbed out in places pending a
production DNSSEC/registry integration — see the commented `EmailVerifier` and
`isMailServerKeyHash` checks. Not audited; do not use in production.

## Tech stack

Circom 2.1.8 · snarkjs (Groth16) · `@zk-email/circuits` & `@zk-email/helpers` ·
Solidity 0.8.18 · Hardhat · OpenZeppelin · TypeScript.
