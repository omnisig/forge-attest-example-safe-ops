# forge-attest-example-safe-ops

A **reference "producer" repo** for [`forge-attest`](../forge-attest). It stands in
for a real operations repo whose Forge script builds a transaction to be submitted
to a Gnosis Safe.

The single script, [`script/BuildSafeTx.s.sol`](script/BuildSafeTx.s.sol),
deterministically emits the full set of EIP-712 `SafeTx` fields as JSON:

```bash
mkdir -p out
forge script script/BuildSafeTx.s.sol:BuildSafeTx
cat out/safe-tx.json
```

## Why this is verifiable

- **Zero dependencies** — the cheatcode interface is declared inline, so there is no
  `forge install`, no `lib/`, nothing to drift.
- **No non-determinism** — every input is a compile-time constant and the JSON is
  built by plain string concatenation. There is no timestamp, no RNG, no on-chain
  read. Running the script on any machine, with any solc, yields **byte-identical**
  output. (The emitted bytes are calldata + fixed fields; they do not depend on the
  compiler.)

Because the output is byte-stable, `forge-attest` can pin its `sha256` at a specific
commit of this repo and prove that a submitted Safe transaction is exactly what this
script produces — and was not hand-edited on the way to the Safe UI.

The transaction built here is `USDC.transfer(0x…dEaD, 1 USDC)` on Ethereum mainnet,
Safe `0x111CEEee040739fD91D29C34C33E6B3E112F2177`, nonce `42`.

> This is example/demo data. The Safe address and nonce are illustrative.

## Optional: proposing to the Safe (separate from attestation)

Building the tx and *submitting* it to the Safe are deliberately separate steps —
submission is exactly where a value could be tampered with, and it's what
`forge-attest` independently checks. This repo includes an optional path that
signs and proposes the built tx to the Safe Transaction Service:

```bash
# 1. build the canonical tx
forge script script/BuildSafeTx.s.sol:BuildSafeTx

# 2. sign the Safe tx hash and emit the Transaction Service payload
PROPOSER_PK=0x<owner-or-delegate-key> forge script script/Propose.s.sol:Propose

# 3. POST it (needs network + a valid proposer)
./propose.sh --network ethereum
```

- `Propose.s.sol` recomputes the EIP-712 Safe tx hash from `out/safe-tx.json`, signs
  it, and writes `out/proposal.json` (the exact POST body incl. `contractTransactionHash`).
- `propose.sh` submits that payload to the network's Transaction Service.

The proposer key must be a Safe **owner or registered delegate**. This path is **not**
part of the attestation/CI pipeline — it just demonstrates the real submission step
that `forge-attest` later verifies against.
