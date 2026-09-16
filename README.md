## Parcours français

Ce dépôt est accompagné d’un parcours documentaire en français consacré à Overlay v1-core. Voir [docs/fr/](docs/fr/).

# v1-core


[![Lint Python](https://github.com/overlay-market/v1-core/actions/workflows/lint-python.yaml/badge.svg)](https://github.com/overlay-market/v1-core/actions/workflows/lint-python.yaml)
[![Lint Solidity](https://github.com/overlay-market/v1-core/actions/workflows/lint-solidity.yaml/badge.svg)](https://github.com/overlay-market/v1-core/actions/workflows/lint-solidity.yaml)
[![Tests](https://github.com/overlay-market/v1-core/actions/workflows/test-python.yaml/badge.svg)](https://github.com/overlay-market/v1-core/actions/workflows/test-python.yaml)


V1 core smart contracts


## Requirements


To run the project you need:


- Python >= 3.9.2
- [Brownie >= 1.17.2](https://github.com/eth-brownie/brownie)
- Local Ganache environment installed
- `.env` file in project root with format


```
# required environment variables
export WEB3_INFURA_PROJECT_ID=<INFURA_TOKEN>
export ARBISCAN_TOKEN=<ETHERSCAN_TOKEN>
```


```
# add Arbitrum Fork
brownie networks add Development arbitrum-main-fork name="Ganache-CLI (Aribtrum-Mainnet Fork)" host=http://127.0.0.1 cmd=ganache-cli accounts=10 evm_version=istanbul fork=arbitrum-main mnemonic=brownie port=8545
```


```
# modify network configuration to use API key
brownie networks modify arbitrum-main host="https://arbitrum-mainnet.infura.io/v3/\$WEB3_INFURA_PROJECT_ID" provider=infura
```


To generate the required tokens, see


- `ARBISCAN_TOKEN`: Creating an API key in [Arbiscan's API docs](https://docs.arbiscan.io/getting-started/viewing-api-usage-statistics)
- `WEB3_INFURA_PROJECT_ID`: Getting Started in [Infura's API docs](https://infura.io/docs)


## Diagram


![diagram](./docs/assets/diagram.svg)


## Modules


V1 core relies on three modules:


- [v1-core](#v1-core)
  - [Requirements](#requirements)
  - [Diagram](#diagram)
  - [Modules](#modules)
    - [Markets Module](#markets-module)
    - [Feeds Module](#feeds-module)
    - [OVL Module](#ovl-module)
  - [Deployment Process](#deployment-process)


### Markets Module


Traders interact directly with the market contract to take positions on a data stream. Core functions are:


- `build()`
- `unwind()`
- `liquidate()`
- `update()`


Traders transfer OVL collateral to the market contract to back a position. This collateral is held in the market contract until the trader unwinds their position when exiting the trade. OVL is the only collateral supported for V1.


The market contract tracks the current open interest for all outstanding positions on a market as well as [information about each position](./contracts/libraries/Position.sol), that is needed in order to calculate the current value of the position in OVL terms:


```
library Position {
    /// @dev immutables: notionalInitial, debtInitial, midTick, entryTick, isLong
    /// @dev mutables: liquidated, oiShares, fractionRemaining
    struct Info {
        uint96 notionalInitial; // initial notional = collateral * leverage
        uint96 debtInitial; // initial debt = notional - collateral
        int24 midTick; // midPrice = 1.0001 ** midTick at build
        int24 entryTick; // entryPrice = 1.0001 ** entryTick at build
        bool isLong; // whether long or short
        bool liquidated; // whether has been liquidated (mutable)
        uint240 oiShares; // current shares of aggregate open interest on side (mutable)
        uint16 fractionRemaining; // fraction of initial position remaining (mutable)
    }
}
```


For each market contract, there is an associated feed contract that delivers the data from the data stream. The market contract stores a pointer to the `feed` contract that it retrieves new data from, and the market uses the feed's `update()` function to retrieve the most recent price and liquidity data from the feed through a call to `IOverlayV1Feed(feed).latest()`. This call occurs every time a user interacts with the market.


All markets are implemented by the contract `OverlayV1Market.sol`, regardless of the underlying feed type.
