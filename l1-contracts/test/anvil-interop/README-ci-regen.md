# Reproducible v31 ecosystem-upgrade calldata via CI

The **`Ecosystem Upgrade Calldata: Regenerate + Verify (v31)`** GitHub Action
(`.github/workflows/generate-ecosystem-upgrade-calldata.yaml`) generates the v31
(`v0.31.0-interopB`) ecosystem-upgrade calldata for one environment on a stock
`ubuntu-latest` runner, so the CREATE2 addresses baked into `ecosystem.toml` are
reproducible (built from source, not a dev's laptop).

It is a thin wrapper around the canonical local flow
(`l1-contracts/test/anvil-interop/regen-and-verify.sh`): fork L1 → build
contracts + `protocol_ops` → `upgrade-prepare-all` (writes `ecosystem.toml`) →
fund + fork-replay the prepare bundles → `verify-upgrade` (PUVT) → emit the
git-portable `sim-inputs/` + the ready-to-paste `transaction-simulator.json`.

## What comes out

Under `l1-contracts/upgrade-envs/v0.31.0-interopB/output/<env>/` and uploaded as
the `ecosystem-upgrade-calldata-<env>` artifact:

| File | Purpose |
|---|---|
| `ecosystem.toml` | merged addresses + governance **stage 0/1/2 calldata** (the upgrade) |
| `sim-inputs/` | normalized manifest + Camp-B `*.safe.json` bundles for the transaction simulator |
| `transaction-simulator.json` | ready-to-paste simulator input (Camp-B bundles + the PUH/gov stages) |
| `extra-verification-logs.txt` | `forge verify-contract` commands for Etherscan |

## Running it

Actions → *Ecosystem Upgrade Calldata: Regenerate + Verify (v31)* → **Run
workflow**, or:

```bash
gh workflow run generate-ecosystem-upgrade-calldata.yaml \
  --ref <branch> \
  -f environment=battlechain \
  -f verify_mode=report-only
```

The **only required input is `environment`.** Everything else is derived from
the env config + the live chain.

### Inputs

| Input | Default | Notes |
|---|---|---|
| `environment` | `battlechain` | Basename of the config pair (see below). Built-ins: `stage`, `testnet`, `mainnet`, `battlechain`. |
| `verify_mode` | `report-only` | `strict` = PUVT must pass (PUH-governed envs). `report-only` = run PUVT, print its output, but don't fail the job (legacy-gov / single-chain envs). |
| `l1_rpc_url` | *(secret)* | Optional per-run RPC override. **Visible in the run summary** — prefer the secret. |
| `deployer_address` | *(placeholder)* | Public address, impersonated on the fork. Does **not** affect CREATE2 output. |
| `fork_block` | *(tip)* | Pin to a pre-deploy block if the upgrade is already live on-chain. |
| `zksync_foundry_version` | `v0.1.5` | foundry-zksync (canonical build). |
| `zk_governance_commit` | `9b06a16` | PUVT PUH bytecode metadata (PUH-governed envs only). |
| `use_new_salt` | `false` | Rotate every CREATE2/gov salt to fresh random values before prepare. |
| `push_artifacts` | `false` | Commit the regenerated artifacts back to the dispatched branch (refreshes its PR). |
| `deploy` | `false` | Broadcast the deployer-EOA bundle to the **real** L1 (needs `DEPLOYER_PRIVATE_KEY_<net>`). Leave off for generation. |

### Secrets

- `L1_RPC_URL_MAINNET` — used by `mainnet` **and any `l1_chain_id == 1` env** (e.g. `battlechain`).
- `L1_RPC_URL_SEPOLIA` — used by `stage` / `testnet`.
- `DEPLOYER_PRIVATE_KEY_<MAINNET|SEPOLIA>` — **only** for `deploy`. Generation is keyless (the deployer is impersonated).
- `GW_RPC_URL` (optional) — Gateway RPC for PUVT GW-side checks (gateway-enabled envs only).

The L1 to fork is chosen automatically from the env's `l1_chain_id`
(`1` → mainnet, `11155111` → Sepolia) — so the workflow picks the right RPC
secret without any per-env branching.

## Adding a new ecosystem / upgrade (reusable)

No workflow edits needed. Commit a config pair named after the env:

1. `l1-contracts/upgrade-envs/permanent-values/<env>.toml` — bridgehub,
   `l1_chain_id`, `zk_token_asset_id`, `governance_kind`, the CTM list
   (`[[ctm_contracts.ctms]]`), etc.
2. `l1-contracts/upgrade-envs/v0.31.0-interopB/<env>.toml` — `era_chain_id`,
   `testnet_verifier`, `owner_address`, `bridgehub_proxy_address`, the CREATE2
   salts.
3. Only if the env needs its own local anvil port for parallel runs: add a
   `case` in `regen-and-verify.sh` (`<env>) PORT=<port> ;;`).

Then run the workflow with `environment=<env>`. Most values (old protocol
version, validator-timelock, security council, era chain id, chain-creation
params) are introspected from the live chain / `configs/genesis/`, so the config
pair stays small.

## `verify_mode` — why `report-only` exists

PUVT (`ecosystem verify-upgrade`) was written for the canonical mainnet topology:
a `ProtocolUpgradeHandler` at `bridgehub.owner()` and a registered Era chain. Some
ecosystems don't match that shape:

- **Legacy-`Governance.sol`-owned** ecosystems (e.g. **battlechain**, owner
  `0x6145cb32…`) have no PUH — PUVT's zk-governance-provenance / PUH-ownership
  checks don't apply.
- **Single-chain** ecosystems (battlechain has only chain 626, no Era chain 324)
  make PUVT's Era-diamond fee-param check compare against a non-existent chain
  (all-zero fee params).

For these, `report-only` still **runs** PUVT and prints its full output (so the
exemptions are visible and reviewable) but doesn't fail the job — the calldata
artifacts are produced by the prepare step regardless. Use `strict` for
PUH-governed envs where PUVT is expected to be green.

> **battlechain note:** battlechain is a ZKsync-OS ecosystem, so it *sidesteps*
> the Era-only `factory dep mismatch` build-determinism check (guarded by
> `!isZKsyncOS`) that affects the mainnet Era path on some runners. Its calldata
> is reproducible on a stock `ubuntu-latest` runner.
