# Devnet notes

The public EIP-8141 / ERC-8421 alt-mempool devnet. It **can be reset without warning** (it was
reset on 2026-11-06 at 06:29 UTC). After a reset, rerun the scripts below; the tokens come back at
the same addresses.

## Endpoints

| Field | Value |
|-------|-------|
| Chain id | 1337 |
| Node RPC (ethrex) | `https://altmempool.forshtat.com` |
| Alt-mempool gateway (skandha bundler) | `https://altmempoolgateway.forshtat.com/rpc` |
| Client versions | `ethrex/v23.0.0-eip8141-tracer-9aa3645d…`, `skandha/v4.1.1` |
| Block time / gas limit | ~6 s / ~114.7M |
| Base fee | 7 wei (`eth_gasPrice` suggests 2 gwei) |
| Faucet | `skandha_explorer_faucet [address]` on the gateway, 1 ETH per call |
| Frame tx submission | `eth_sendFrameTransaction` on the gateway |
| Tx inspection | `skandha_explorer_getTransaction [hash]` on the gateway (admission checks, status, revert reason) |

## Team key

`.env` in the repo root (gitignored, never commit it) holds the RPC URLs and the team key.
**Ask Sainath for the `.env` file.**

| Field | Value |
|-------|-------|
| Team address | `0x66E5a5f2D5Ae9a4fe9f68a83ABc39719fe92E85f` |
| Funded | 1 ETH from the faucet on 2026-11-06 (tx `0x665907e5…47ab`) |

## Contracts on the devnet

| Contract | Address | Notes |
|----------|---------|-------|
| CREATE2 factory (EIP-7997) | `0x4e59b44847b379578588920cA78FbF26c0B4956C` | Survives resets |
| Staking registry | `0xd51c04d7a0b01124e84de152f4e56d9eecb22d2f` | min stake 0.01 ETH; staking does **not** raise the 100k verify budget |
| Test USDC (6 decimals) | `0xd1108cBf3C5E7b44D7A58d4b2cdA492a0DDF6a72` | `TestERC20`, anyone can `mint(to, amount)` |
| Test WBTC (8 decimals) | `0xb212346871a9a08494e165F5AeFe655f06A5B75F` | `TestERC20`, anyone can `mint(to, amount)` |
| Multicall3 | `0xcA11bde05977b3631167028862bE2a173976CA11` | Listed by the gateway but **not deployed** after the reset |

No public ERC-20s or paymasters exist on the devnet. The test tokens are ours.

## Scripts (`tools/devnet/`)

```bash
tools/devnet/deploy-tokens.sh   # deploy USDC + WBTC via CREATE2 (skips ones already deployed)
tools/devnet/smoke.sh           # ETH transfer, mint, transfer, approve/transferFrom, overspend revert
```

## Gas on this devnet

The devnet charges large **state-creation gas**, about 98k for creating a new account or the
first write to a storage slot (EIP-8037-style):

| Action | Gas used |
|--------|----------|
| 0.01 ETH to a new address | 204,600 |
| Token mint (first balance slot) | ~238,300 |
| Token transfer to a new holder | 140,885 |
| `transferFrom` (existing slots) | 44,860 |
| First QDay send (Nov 5 probe; includes first `nextIndex` write) | 278,341 |
| Later QDay send with 68k verify work (Nov 5 probe) | 180,421 |

At 2 gwei a QDay send costs about 0.0004–0.0006 ETH, so 1 ETH covers well over 1,000 sends.

## Frame tx probe results

Probe kit: `~/Desktop/eth-mumbai/probes` (`npx tsx probe.ts`).

**2026-11-05 (before the reset): all assumptions held.** The QDay shape
`[pre_verify commit(idx), self_verify, SENDER]` was admitted and mined. A 2,500-byte ARBITRARY sig
was read via `SIGDATACOPY`. A reused slot was rejected at admission. A foreign `commit()` couldn't
move `nextIndex`. A 100,000 gas prefix was admitted and a 101,100 one rejected. The minimum
pre_verify limit is 13,000 (it uses 12,441). This ethrex build elides signature bytes as *empty
bytes*, not their length.

**2026-11-06 (after the reset):**

| Check | Result |
|-------|--------|
| Plain txs (legacy/1559) direct to the node | ✅ all 15 smoke checks passed |
| Faucet, CREATE2 factory | ✅ |
| Frame tx admission (prefix, signature, simulation checks) | ✅ passes |
| **Frame tx inclusion via the gateway** | ⚠️ **Stalled ~20 min, then recovered.** Other frame txs were bundled in blocks 561–623, then nothing until block 863. Our QDay-shape tx `0xa35c8750…200c` was admitted at ~block 670 (07:35 UTC), stayed `New`, and was mined in **block 863** (status 1, 278,341 gas, `nextIndex` 0 → 1) |
| Smallest pre_verify limit (p2x) | ✅ still 13,000 |

**Direct to the node** (`eth_sendRawTransaction` on ethrex, `~/Desktop/eth-mumbai/probes/direct.ts`):

| Tx | Result |
|----|--------|
| EOA self-relay frame tx `[VERIFY, SENDER]` | ✅ mined in block 760, 26,102 gas |
| QDay shape `[pre_verify commit(idx), self_verify, SENDER]` | ❌ rejected: `validation-prefix simulation failed: StateWriteOutsideDeploy`. The node enforces the *public* mempool rules, which ban the `commit()` storage write in the prefix. Only the alt-mempool gateway allows it |

**Takeaway:** the gateway's bundling can stall for many minutes. If a frame tx stays `New` in
`skandha_explorer_getTransaction` for more than ~1 minute, the gateway is stalled, not our tx. The
node never sees the tx until the gateway forwards it (`eth_getTransactionByHash` on the node
returns null). For the demo, submit early and keep the recorded video as a backup. Operator:
Alex Forshtat (ERC-8421 author).

**Likely gotcha:** after a reset, a key at the same nonce produces the **same tx hash** as before
the reset. p1 had the identical hash to its Nov 5 run and was `dropped` at admission even though
every check passed. The likeliest cause is that the gateway remembers the old hash (not confirmed).
Use fresh keys after each reset.
