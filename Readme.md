# Secure Ethereum Development with Sigstore

A hands on guide for signing and verifying smart contract artifacts using Sigstore. Commands work on linux/mac.

## Prerequisites

Install Node and npm

```bash
brew install node          # mac
```

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
nvm install --lts          # linux
```

Install jq for parsing JSON output

```bash
brew install jq            # mac
sudo apt install -y jq     # linux
```

Install Foundry for compiling and deploying contracts

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

Install cosign for signing

```bash
brew install cosign        # mac
```

```bash
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign     # linux
```

Get free Sepolia testnet ETH from a faucet

```
https://cloud.google.com/application/web3/faucet/ethereum/sepolia
```

Get an RPC url from Infura or Alchemy for the Ethereum network, Sepolia included

Set environment variables

```bash
export SEPOLIA_RPC_URL="your rpc url"
export PRIVATE_KEY="your demo wallet key"
export ETHERSCAN_API_KEY="your etherscan key"
```

## Build the contract

Initialize the project

```bash
mkdir devcon-sigstore-demo && cd devcon-sigstore-demo
forge init --no-git .
```

Write the contract in `src/Counter.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Counter {
    uint256 private value;
    address public owner;

    event ValueChanged(uint256 newValue, address changedBy);

    constructor(uint256 _initial) {
        value = _initial;
        owner = msg.sender;
    }

    function set(uint256 _value) external {
        value = _value;
        emit ValueChanged(_value, msg.sender);
    }

    function get() external view returns (uint256) {
        return value;
    }

    function increment() external {
        value += 1;
        emit ValueChanged(value, msg.sender);
    }
}
```

Copy this test in `test/Counter.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test} from "forge-std/Test.sol";
import {Counter} from "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;

    function setUp() public {
        counter = new Counter(0);
        counter.set(0);
    }

    function test_Increment() public {
        counter.increment();
        assertEq(counter.get(), 1);
    }

    function testFuzz_SetNumber(uint256 x) public {
        counter.set(x);
        assertEq(counter.get(), x);
    }
}
```

Compile the contract

```bash
forge build
```

## Deploy to Sepolia

Deploy the compiled contract

```bash
forge create src/Counter.sol:Counter --broadcast --rpc-url "$SEPOLIA_RPC_URL" --private-key "$PRIVATE_KEY" --constructor-args 42
```

Save the deployed address

```bash
export CONTRACT_ADDRESS="0xPASTE_DEPLOYED_ADDRESS"
```

View the contract on Etherscan

```
https://sepolia.etherscan.io/address/CONTRACT_ADDRESS_HERE
```

Verify the source code on Etherscan, optional

```bash
forge verify-contract $CONTRACT_ADDRESS src/Counter.sol:Counter \
  --chain sepolia \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  --compiler-version 0.8.35 \
  --constructor-args $(cast abi-encode "constructor(uint256)" 42)
```

Confirm the deployed value

```bash
cast call $CONTRACT_ADDRESS "get()(uint256)" --rpc-url $SEPOLIA_RPC_URL
```

> You have to call another function, but it will cost you tokens because the other functions we wrote in our Solidity file change the state on the network, which costs you a gas fee.
>
> ```bash
> cast send $CONTRACT_ADDRESS "increment()" \
>   --rpc-url "$SEPOLIA_RPC_URL" \
>   --private-key "$PRIVATE_KEY"
> ```
>
> ```bash
> cast send $CONTRACT_ADDRESS "set(uint256)" 100 \
>   --rpc-url "$SEPOLIA_RPC_URL" \
>   --private-key "$PRIVATE_KEY"
> ```

## Sign with cosign

Build a deployment metadata file

```bash
mkdir -p artifacts
cat > artifacts/deployment.json <<EOF
{
  "contract": "Counter",
  "address": "$CONTRACT_ADDRESS",
  "chain": "sepolia",
  "chainId": 11155111,
  "deployer": "$(cast wallet address --private-key $PRIVATE_KEY)",
  "abi": $(cat out/Counter.sol/Counter.json | jq '.abi'),
  "bytecodeHash": "$(cat out/Counter.sol/Counter.json | jq -r '.bytecode.object' | sha256sum | cut -d' ' -f1)"
}
EOF
```

Note, mac users replace `sha256sum` with `shasum -a 256`

Sign the file with keyless signing

```bash
cosign sign-blob artifacts/deployment.json \
  --bundle artifacts/deployment.json.bundle \
  --yes
```

Sign the raw contract source

```bash
cosign sign-blob src/Counter.sol \
  --bundle artifacts/Counter.sol.bundle \
  --yes
```

Sign the ABI separately

```bash
cat out/Counter.sol/Counter.json | jq '.abi' > artifacts/Counter.abi.json
cosign sign-blob artifacts/Counter.abi.json \
  --bundle artifacts/Counter.abi.json.bundle \
  --yes
```

Get the file hash and check it on the Sigstore transparency log

```bash
sha256sum artifacts/deployment.json
```

```
https://search.sigstore.dev/?hash=PASTE_HASH_HERE
```

## Verify

Verify the signed artifact

```bash
cosign verify-blob artifacts/deployment.json \
  --bundle artifacts/deployment.json.bundle \
  --certificate-identity-regexp ".*" \
  --certificate-oidc-issuer https://accounts.google.com
```

Use `https://github.com/login/oauth` as the issuer if you signed in with GitHub instead of Google

Tamper with a copy of the file

```bash
cp artifacts/deployment.json artifacts/deployment_tampered.json
sed -i 's/"chainId": 11155111/"chainId": 1/' artifacts/deployment_tampered.json
```

Mac users run `sed -i '' 's/"chainId": 11155111/"chainId": 1/' artifacts/deployment_tampered.json`

Verify the tampered file to see it fail

```bash
cosign verify-blob artifacts/deployment_tampered.json \
  --bundle artifacts/deployment.json.bundle \
  --certificate-identity-regexp ".*" \
  --certificate-oidc-issuer https://accounts.google.com
```

## Automate with CI/CD

Push the project to GitHub

```bash
git init
git add .
git commit -m "initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/devcon-sigstore-demo.git
git push -u origin main
```

Add the workflow file at `.github/workflows/sign-and-verify.yml`

```yaml
name: Build, Sign, Verify

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build-sign-verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1

      - name: Build contracts
        run: forge build

      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign deployment artifact
        run: |
          cosign sign-blob artifacts/deployment.json \
            --bundle artifacts/deployment.json.bundle \
            --yes

      - name: Verify signature
        run: |
          cosign verify-blob artifacts/deployment.json \
            --bundle artifacts/deployment.json.bundle \
            --certificate-identity-regexp "https://github.com/${{ github.repository }}/.*" \
            --certificate-oidc-issuer https://token.actions.githubusercontent.com

      - name: Upload signed artifacts
        uses: actions/upload-artifact@v4
        with:
          name: signed-deployment-artifacts
          path: artifacts/
```

Push the workflow and watch it run

```bash
git add .github/workflows/sign-and-verify.yml
git commit -m "Add Sigstore signing pipeline"
git push
```

```
https://github.com/YOUR_USERNAME/devcon-sigstore-demo/actions
```

## Package verification in the wild

Check provenance on a real npm package

```bash
npm view ethers --json
```

## Reset

```bash
rm -rf out artifacts cache broadcast
```
