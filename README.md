# IMDEX5 — The Imdex 5

An index of the five largest tokens paired with IMD on Ethereum mainnet (chain id 1), as one
contract, `Imdex5` (`src/Imdex5.sol`). The contract's own ERC-20 is the share: name
`The Imdex 5`, symbol `IMDEX5`, 18 decimals, plain transfers, supply 0 at deployment and no cap.
A mint issues shares, a redeem burns them, nothing else mints. Anyone can call everything; there
is no owner, admin, role, setting, pause or upgrade.

Contracts only: no launch token, pool or website.

## Layout

| Path | What |
| --- | --- |
| `src/Imdex5.sol` | The index. One contract, no constructor arguments, 21.4 KB of runtime. |
| `src/OracleAttestation.sol` | The protocol's `OracleAttestation` library and `OracleAttestationConsumer`, copied whole from the oracle-consumer reference. |
| `src/Interfaces.sol` | The slices of the Uniswap v4 `PoolManager`, the launch registries and the Intake that the index calls. |
| `test/Imdex5.t.sol` | Behaviour, failure and attack tests against mocks. |
| `test/OracleConsumerConformance.t.sol` | The protocol's digest and selector vectors. |
| `test/mocks/Mocks.sol` | Mock ERC-20, a constant-product stand-in for the PoolManager, registries, Intake, an ERC-1271 signer and a re-entering hook. |
| `lib/` | OpenZeppelin Contracts 5.6.1 and forge-std 1.9.6, vendored as ordinary files. |

Build and test:

```
forge build
forge test
forge fmt --check
```

`foundry.toml` pins solc 0.8.26, optimizer 200 runs, `via_ir`, `cancun`, `bytecode_hash = "none"`.
Tests mock every outside contract by etching mocks at the mainnet addresses the contract hard-codes;
there are no fork tests and no environment reads.

## Deployment parameters

The constructor takes no arguments, calls no contract and checks no code length. Everything about
the chain is a constant:

| Constant | Value |
| --- | --- |
| IMD | `0xD34a99Bc0f67aE1bbd63C660e6d0b0dd03E263B7` |
| Uniswap v4 PoolManager | `0x000000000004444c5dc75cB358380D2e3dE08A90` |
| Registry A | `0xA47FcB220DDd1cDddA865F89029BEc1b9EcA3bf1` |
| Registry B | `0x7c9D5ee697BfF5Ac4F10E2E54d3f40c44e48fE23` |
| Intake (version 2) | `0xa43e6F75ee006411F79Ac1C84120606C2330DE82` |
| Action | `bytes32("oracle.request@oracle-1")` |
| Oracle signer | `0x5598aa9146215bc13eb26f2c692ad1461fd32982` |
| Pool key | IMD and the coin sorted, fee 12500, tick spacing 60, hooks = the launch's hook |
| EIP-712 domain | name `IdentityMD Oracle`, version `2`, this chain, this contract |

### Starting list

The constructor seeds the list below. On 2026-10-13 both registries were read through a public
RPC: each launch is recorded in exactly one registry, and every `token` and `hook` artifact equals
the brief's value, so nothing differs and nothing was substituted.

| Launch | Symbol | Token | Hook | Registry |
| --- | --- | --- | --- | --- |
| 1282 | INFER | `0x53d0eA0A685220718Dc6a0a6e4617194C6F37921` | `0x784FF9a3ac5d88A30bfff6F7f2a270161FBE6000` | B |
| 1283 | MARG | `0xfA0EfEa559bdE229031c2B0A219472316647759c` | `0xf0457dD0Cf526686190e4f057Ba19dB14D34a0cc` | A |
| 737 | ZTO | `0xd782Bdea4EF02a0Bd391eb9089470c8080f0A68e` | `0x784FF9a3ac5d88A30bfff6F7f2a270161FBE6000` | B |
| 1075 | SCAT | `0x54C68a4D0A21DBb1ad72FaA052db159d26F967EA` | `0x784FF9a3ac5d88A30bfff6F7f2a270161FBE6000` | B |
| 1175 | SC | `0x1ae93A4D9CC0b41C594D00387B6F832a88f729b7` | `0x784FF9a3ac5d88A30bfff6F7f2a270161FBE6000` | B |

### Numbers the brief left open (named constants)

| Constant | Value | Why |
| --- | --- | --- |
| `SQRT_UP_BPS` / `SQRT_DOWN_BPS` | 10100 / 9900 | A limited swap's sqrt-price limit is the spot sqrt price moved 1%, which is a price move of about 2% (2.01% up, 1.99% down). |
| `WEIGHT_DIVISOR` | 4 | The 25% weight cap. |
| `MAX_LIST` | 5 | The most launch numbers an answer may carry. |
| `MIN_AGREED` | 4 | The panel agreement `onOracleResult` requires. |

## How it works

### Accounting

Internal, never `balanceOf`:

- `held[coin]`: units of each coin the index owns for its holders.
- `idleImd`: IMD a rebalance left unspent. The holders'.
- `feeImd`: fees. Outside every mint, redeem and rebalance; it only ever pays the oracle price
  (0.5 IMD) and caller rewards (0.25 IMD). `fund(amount)` lets anyone add to it.

The IMD the contract holds always equals `idleImd + feeImd` (a test asserts this after every
operation). A coin's balance may exceed `held[coin]` by the amounts recorded as `owed`.

### Mint

- `mintFirst(imdIn, minOut[])`: only at supply 0. 0.50% of `imdIn` goes to `feeImd`; the rest
  buys each coin of the list with an equal IMD part (any wei left by the division goes to
  `idleImd`) and mints one share per IMD of the rest. Reverts if a buy reverts or returns under
  its `minOut`. `minOut` has one entry per coin of the list, in list order.
- `mint(shares, maxImdIn)`: supply > 0. Pulls `maxImdIn` first, buys exactly `held * f` of every
  coin (`f = shares / totalSupply`, rounded up), adds `idleImd * f` (rounded up) to `idleImd`,
  adds 0.50% of the buys' cost to `feeImd`, mints, and refunds the unspent IMD. Reverts if the
  total exceeds `maxImdIn` or a buy reverts or fills short. No price is read; no cap.

### Redeem

- `redeem(shares)`: burns, then pays 99.50% of `held * f` of each coin and of `idleImd * f`
  (100% when the supply reaches 0; the 0.50% stays with the remaining holders). No price, no
  trade. A transfer that reverts, returns false or targets an address without code is recorded in
  `owed[caller][coin]` instead of blocking; `claim(coin)` takes it later and can only pay once.
- `redeemToImd(shares, minImdOut)`: same burn and `f`; sells `held * f` of each coin inside the
  PoolManager's `unlock`, adds `idleImd * f`, keeps 0.50% as `feeImd`, pays the rest in IMD and
  reverts under `minImdOut`. A coin whose sale reverts (or fills short) is paid in kind.

### Oracle

- `ask()`: anyone, when no request is open, no list awaits a rebalance, and 7 days have passed
  since the last ask. With under 0.75 IMD of `feeImd` it returns without doing anything.
  Otherwise it pays the Intake 0.5 IMD through `requestWithFailure` and the caller 0.25 IMD.
- The body is built on chain by `body()`: `v` 1, the question, chain 1, window 24 hours, answer
  type `bytes32[]`, evidence `panel`, panel 5, quorum 4, `validForSeconds` 86400, `head` 5 and
  `consumer {chainId: 1, verifyingContract: this}`.
- `onOracleResult`: only from the Intake, only for the open request, only with `agreed >= 4`:
  stores `keccak256(answer)` and closes the request. Every other case returns. It moves no
  tokens and runs well under 200,000 gas (tested with a 200,000 gas call).
- `onOracleFailure`: only from the Intake, for the open request: closes it. `closeRequest()`
  lets anyone close a request open for 2 days.
- `submit(attestation, signature)`: anyone. `keccak256(answer)` must equal the stored hash and
  the signature must verify for the signer over this contract's EIP-712 digest (ECDSA, or
  ERC-1271 if the signer has code, through OpenZeppelin's `SignatureChecker`). Decodes 1 to 5
  distinct launch numbers (distinct tokens too), reads each token and hook from the registry that
  records it, reverts if either is missing, stores them as the next list and pays the caller
  0.25 IMD from `feeImd` if it can.

### Rebalance

`rebalance()`: anyone, when a next list is stored. Inside one `unlock`:

1. Sells every coin not in the next list into `idleImd`, each sale limited to a 2% price move.
2. Adopts the list. A leaver that step 1 could not fully sell (its pool is dead, or the 2% limit
   stopped the sale) stays in the holdings after the list, so it is still redeemed and minted
   proportionally and sold again by the next rebalance.
3. With four or more coins in the list, sells any holding above 25% of total value down to 25%,
   by pool spot prices read at the start of this step (`extsload` of the pool's `slot0`).
4. Spends `idleImd` on the list in equal parts; a buy stops at 25% weight or a 2% price move.

A swap that reverts is skipped (each swap is its own external self-call, `swapSelf`, so a revert
undoes only that swap). The caller gets 0.25 IMD from `feeImd` if it can pay.

## Security notes

- Every state-changing function is `nonReentrant` and emits an event. `unlockCallback` accepts
  only the PoolManager and only while one of the index's own operations holds the guard;
  `swapSelf` accepts only the contract itself. A hook or coin that re-enters hits the guard.
- IMD is pulled only from `msg.sender`; the PoolManager is paid with `transfer` from the index's
  own balance, never `transferFrom` with another payer.
- Shares are minted only against IMD actually spent: `mint` rounds every proportional amount up
  and charges the swaps' real cost plus fee; `mintFirst` mints exactly the IMD it spent.
- Redeem reads no price, makes no swap and cannot be blocked by a coin: failures become `owed`,
  and `claim` zeroes the debt before paying.
- `feeImd` leaves only to the Intake (0.5 IMD) and as 0.25 IMD rewards.
- A forged callback (wrong sender, wrong request id, thin agreement) is ignored; a forged
  attestation fails the signature; a tampered field changes the digest.

Accepted, by the brief: thin-pool price impact, a pool's own hook fee, a dead pool blocking mint
(never redeem), and an oracle that never answers. Also known and accepted: a coin that burns all
forwarded gas in `transfer` can make a `redeem` run out of gas (the 63/64 rule leaves the caller
little; retry with more gas). `submit` checks only the stored hash and the signature, as the brief
asks, not the attestation's `expiresAt`: the hash already binds the answer to the index's own
request, and the hash is cleared once a list is stored, so an attestation is taken once.

Tests passing is not an audit. Work holding other people's funds needs an independent
adversarial review before release.

## Operational responsibilities

Nothing is owned. After launch:

- Someone must `fund` at least 0.75 IMD of `feeImd` before the first `ask` can pay; mint and
  redeemToImd fees top it up afterwards.
- Someone must call `ask()` (weekly at most), `submit` the signed attestation the Intake
  delivered (it is in the `Answered` event's request and the oracle's receipt), and `rebalance()`.
  Each of the three pays its caller 0.25 IMD when `feeImd` can.
- If the oracle never calls back, anyone may `closeRequest()` after 2 days.
- The launch deploys `Imdex5` from its bytecode as built, with no arguments. No `launch.json` is
  written here; the manifest step writes it.
