# FundMe

A Solidity crowdfunding contract built with [Foundry](https://book.getfoundry.sh/). Anyone can send ETH via `fund()`, as long as it's worth at least $5 USD at the time — enforced with a live Chainlink price feed. Only the contract owner can `withdraw()` the balance.

The project is chain-aware: deploy it locally against a mocked price feed on Anvil, or against the real Chainlink ETH/USD feed on Sepolia, with no code changes required.

## How it works

- **`src/FundMe.sol`** — the core contract. Tracks each funder's contribution, enforces the $5 minimum via `PriceConverter`, and restricts `withdraw()` to the owner.
- **`src/PriceConverter.sol`** — a library that reads a Chainlink `AggregatorV3Interface` feed and converts an ETH amount to its USD value.
- **`script/HelperConfig.s.sol`** — picks the right price feed address for whatever chain you're on: the real Chainlink feed on Sepolia (chain ID `11155111`), or a freshly deployed `MockV3Aggregator` everywhere else (e.g. local Anvil).
- **`script/DeployFundMe.s.sol`** — deploys `FundMe` wired to the feed address `HelperConfig` resolves.
- **`script/Interactions.s.sol`** — `FundFundMe` / `WithdrawFundMe`, scripts that look up your most recently deployed `FundMe` (via [Cyfrin/foundry-devops](https://github.com/Cyfrin/foundry-devops)) and call `fund()` / `withdraw()` on it.
- **`test/unit/`** — unit tests against a fresh mock feed (no network needed).
- **`test/integration/`** — exercises the `Interactions.s.sol` scripts end-to-end.

## Requirements

- [git](https://git-scm.com/)
- [Foundry](https://book.getfoundry.sh/getting-started/installation) (`forge`, `anvil`, `cast`)
- [GNU Make](https://www.gnu.org/software/make/) *(optional — a convenience wrapper around the `forge`/`cast` commands below; see the [Makefile](Makefile))*

## Getting started

```shell
git clone https://github.com/jaydenyeong/foundry-simple
cd foundry-simple
forge install
forge build
```

## Testing

```shell
# unit + integration tests against a local mock price feed
forge test

# run against a Sepolia fork, using the real Chainlink feed
forge test --fork-url sepolia -vvv
```

`sepolia` resolves via the `[rpc_endpoints]` alias in `foundry.toml`, which reads `SEPOLIA_RPC_URL` from your `.env` (see [Environment variables](#environment-variables)).

## Deploying

### Local (Anvil)

```shell
# terminal 1
make anvil

# terminal 2
make deploy
```

### Sepolia

```shell
make deploy-sepolia ARGS="--network sepolia"
```

Note both `deploy` and `deploy-sepolia` share the same underlying `NETWORK_ARGS` in the `Makefile` — it's the `ARGS="--network sepolia"` flag that actually switches the target network, not the make target's name.

### Interacting with a deployed contract

Once something is deployed and its address is recorded under `broadcast/`, you can fund or withdraw from it directly:

```shell
make fund SENDER_ADDRESS=<your address> ARGS="--network sepolia"
make withdraw SENDER_ADDRESS=<your address> ARGS="--network sepolia"
```

These use `DevOpsTools.get_most_recent_deployment` to find the latest `FundMe` on the current chain, so you don't need to pass an address manually.

## Environment variables

Create a `.env` in the project root:

| Variable | Used for |
|---|---|
| `SEPOLIA_RPC_URL` | RPC endpoint for Sepolia (the `sepolia` alias in `foundry.toml`, and `--fork-url sepolia` in tests) |
| `ETHERSCAN_API_KEY` | Contract verification on `make deploy-sepolia` |

For signing Sepolia transactions, the `Makefile` uses `--account $(ACCOUNT)`, a named Foundry keystore rather than a raw private key. Set one up with:

```shell
cast wallet import <ACCOUNT_NAME> --interactive
```

then pass `ACCOUNT=<ACCOUNT_NAME>` when invoking `make deploy-sepolia` / `make fund` / `make withdraw`.

## CI

Every push and pull request runs [`.github/workflows/test.yml`](.github/workflows/test.yml): `forge fmt --check`, `forge build --sizes`, and `forge test -vvv`.

## License

MIT, per the SPDX headers in each contract.
