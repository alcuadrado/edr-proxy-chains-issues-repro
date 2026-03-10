# gas stats proxies reproduction bug

This PR introduced supports for Proxies into the gas stats module of EDR: https://github.com/NomicFoundation/edr/pull/1303

Unfortunately, there are a few issues about it. This repo shos them.

# Installation

```bash
pnpm i
```

## Adding a debug log

To show the error we need to monkey patch the Hardhat's NetorkManager.

```bash
# Linux
sed -i '103s/$/ const utils = await import("node:util"); utils.inspect.defaultOptions.depth = 100000; console.log(gasReport);/' node_modules/hardhat/dist/src/internal/builtin-plugins/network-manager/network-manager.js

# macOS
sed -i '' '103s/$/ const utils = await import("node:util"); utils.inspect.defaultOptions.depth = 100000; console.log(gasReport);/' node_modules/hardhat/dist/src/internal/builtin-plugins/network-manager/network-manager.js
```

## Running the reproduction

```bash
npx hardhat test nodejs --gas-stats
```

Please ignore the table, and focus on the new debug logs with jsons, these are the raw `GasReport` objects that EDR produces.

## Issue 1: Calls to proxied functions are not reported

In the first `node:test` test, we deploy two implementations, two proxies, and cast the proxies to each implementation interface. This is the common way to use proxies.

The function calls made to those proxies are not collected by the gas stats module.

These reports where expected, but are missing (gas numbers are made up):

```json
{
  "contracts": {
    "project/contracts/Proxies.sol:Impl1": {
      "deployments": [],
      "functions": {
        "one()": [
          {
            "gas": 21374,
            "status": 0,
            "proxyChain": [
              "project/contracts/Proxies.sol:Proxy",
              "project/contracts/Proxies.sol:Impl1"
            ]
          }
        ]
      }
    }
  }
}
```

```json
{
  "contracts": {
    "project/contracts/Proxies.sol:Impl2": {
      "deployments": [],
      "functions": {
        "one()": [
          {
            "gas": 21374,
            "status": 0,
            "proxyChain": [
              "project/contracts/Proxies.sol:Proxy",
              "project/contracts/Proxies.sol:Impl2"
            ]
          }
        ]
      }
    }
  }
}
```

## Issue 2: Calls with different chains should be considered separate

(This may be hidden by the first issue)

If you call the same implementation with different proxy chains, they should be considered separate. The second test is missing the following reports (gas numbers are made up):

```json
{
  "contracts": {
    "project/contracts/Proxies.sol:Impl1": {
      "deployments": [],
      "functions": {
        "one()": [
          {
            "gas": 21374,
            "status": 0,
            "proxyChain": [
              "project/contracts/Proxies.sol:Proxy2",
              "project/contracts/Proxies.sol:Proxy",
              "project/contracts/Proxies.sol:Impl1"
            ]
          }
        ]
      }
    }
  }
}
```

```json
{
  "contracts": {
    "project/contracts/Proxies.sol:Impl1": {
      "deployments": [],
      "functions": {
        "one()": [
          {
            "gas": 21374,
            "status": 0,
            "proxyChain": [
              "project/contracts/Proxies.sol:Proxy",
              "project/contracts/Proxies.sol:Impl1"
            ]
          }
        ]
      }
    }
  }
}
```
