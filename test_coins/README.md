# Test Coins

Test token package for the Sui AMM swap. Provides mintable coins (USDT, XBTC, BTC, ETH, etc.) and a faucet for devnet testing.

**Prerequisite:** Deploy the main swap contract first. `test_coins` depends on `swap` and needs the swap package address in `Move.toml`.

---

## Setup

### 1. Deploy Swap First

From the repo root:

```bash
sui client test-publish --gas-budget 10000000 --build-env devnet
```

Save the **package ID** from the output. You need it for the next step.

### 2. Update test_coins/Move.toml

Edit `test_coins/Move.toml` and set the swap package address:

```toml
[dependencies]
swap = { local = "../", published-at = "YOUR_SWAP_PACKAGE_ID" }
```

Replace `YOUR_SWAP_PACKAGE_ID` with the package ID from step 1.

### 3. Deploy Test Coins

```bash
cd test_coins
sui client test-publish --gas-budget 10000000 --build-env devnet
cd ..
```

Save the **package ID** and **Faucet object ID** from the output.

---

## Available Coins

| Type | Description    |
| ---- | -------------- |
| USDT | Stablecoin     |
| XBTC | Test BTC       |
| BTC  | Bitcoin        |
| ETH  | Ethereum       |
| BNB  | Binance Coin   |
| WBTC | Wrapped BTC    |
| USDC | USD Coin       |
| DAI  | Dai stablecoin |

**ONE_COIN** = `100000000` (1 token with 8 decimals). `force_claim` amount `10` = 10 × ONE_COIN.

---

## Commands Reference

### Variables (set after deployment)

```bash
# From test_coins deployment output
package=0x...   # Your test_coins package ID
faucet=0x...    # Faucet object ID

# Coin type args (use your package ID)
USDT="$package::coins::USDT"
XBTC="$package::coins::XBTC"
BTC="$package::coins::BTC"
ETH="$package::coins::ETH"
BNB="$package::coins::BNB"
USDC="$package::coins::USDC"
DAI="$package::coins::DAI"

# Swap Global (from swap deployment)
swap_global=0x...
```

### Faucet — User

```bash
# Claim 1 unit of token (anyone)
sui client call --gas-budget 10000000 \
  --package $package \
  --module faucet \
  --function claim \
  --args $faucet \
  --type-args $USDT
```

### Faucet — Admin (creator or added admin)

```bash
# Add admin
sui client call --gas-budget 10000000 \
  --package $package \
  --module faucet \
  --function add_admin \
  --args $faucet $NEW_ADMIN_ADDRESS

# Remove admin
sui client call --gas-budget 10000000 \
  --package $package \
  --module faucet \
  --function remove_admin \
  --args $faucet $ADMIN_ADDRESS_TO_REMOVE

# Force claim (mint custom amount; 10 = 10 × ONE_COIN)
sui client call --gas-budget 10000000 \
  --package $package \
  --module faucet \
  --function force_claim \
  --args $faucet 10 \
  --type-args $XBTC

# Add new coin supply (requires TreasuryCap<T>)
sui client call --gas-budget 10000000 \
  --package $package \
  --module faucet \
  --function add_supply \
  --args $faucet $TREASURY_CAP \
  --type-args $COIN_TYPE

# Bootstrap liquidity (mint + add liquidity for predefined pools)
# Requires higher gas budget
sui client call --gas-budget 100000000 \
  --package $package \
  --module faucet \
  --function force_add_liquidity \
  --args $faucet $swap_global
```

### Predefined Pools (from force_add_liquidity)

- BTC–ETH, BTC–USDT, ETH–USDT
- USDC–USDT, BNB–USDT, BNB–USDC
- DAI–USDC, BTC–DAI, DAI–ETH

---

## Quick Start Example

```bash
# 1. Deploy swap (from repo root)
sui client test-publish --gas-budget 10000000 --build-env devnet
# Save package ID → update test_coins/Move.toml

# 2. Deploy test coins
cd test_coins
sui client test-publish --gas-budget 10000000 --build-env devnet
# Save package, faucet IDs

# 3. Set variables
package=0x5f205364e20114075512028e2c3976bbaaa5f482
faucet=0x37019bd3aa332a1ee442c76c6ceaf9390a6e99de
swap_global=0x28ae932ee07d4a0881e4bd24f630fe7b0d18a332
USDT="$package::coins::USDT"
XBTC="$package::coins::XBTC"

# 4. Claim test tokens
sui client call --gas-budget 10000000 --package $package --module faucet \
  --function claim --args $faucet --type-args $USDT

sui client call --gas-budget 10000000 --package $package --module faucet \
  --function claim --args $faucet --type-args $XBTC

# 5. Bootstrap liquidity (admin only)
sui client call --gas-budget 100000000 --package $package --module faucet \
  --function force_add_liquidity --args $faucet $swap_global
```

---

## Build & Test

```bash
# Build (from test_coins/)
sui move build --environment devnet

# Run tests (from repo root; test_coins tests run with swap)
sui move test --environment testnet
```

---

See [../DOCUMENTATION.md](../DOCUMENTATION.md) for full technical documentation.
