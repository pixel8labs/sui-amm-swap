# Sui-AMM-swap

The first open source AMM swap on the [Sui](https://github.com/MystenLabs).

- **Technical docs:** [DOCUMENTATION.md](./DOCUMENTATION.md) — functions, modules, math, architecture
- **Test coins:** [test_coins/README.md](./test_coins/README.md) — faucet, minting, bootstrap liquidity
- **Audit:** [Sui-AMM-swap Contracts Audit Report](https://movebit.xyz/file/Sui-AMM-swap-Contracts-Audit-Report.pdf)

This code has been audited by MoveBit professional auditing company.
Audit report click [here](<https://github.com/OmniBTC/Sui-AMM-swap/blob/main/Sui-AMM-swap%20Contracts%20Audit%20Report%20(5).pdf>)

## Getting Started

### Prerequisites

1. **Install Sui CLI**

   ```bash
   cargo install --locked --git https://github.com/MystenLabs/sui.git --branch devnet sui
   ```

2. **Verify installation**
   ```bash
   sui --version
   ```

### Setup Steps

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd sui-amm-swap
   ```

2. **Configure Sui client for devnet**

   ```bash
   sui client switch --env devnet
   ```

3. **Check your active address**

   ```bash
   sui client active-address
   ```

   Save this address - you'll need it for getting test tokens from the faucet.

4. **Verify Move.toml configuration**

   The `Move.toml` file should already have the correct dependencies:

   ```toml
   [dependencies]
   Sui = { git = "https://github.com/MystenLabs/sui.git", subdir = "crates/sui-framework/packages/sui-framework", rev = "devnet" }

   [environments]
   devnet = "a9149565"
   testnet = "4c78adac"
   mainnet = "35834a8a"
   ```

   If the `[environments]` section is missing, add it to both `Move.toml` and `test_coins/Move.toml`.

5. **Get SUI tokens for gas (devnet)**
   ```bash
   sui client faucet
   sui client gas  # Verify you received tokens
   ```

### Building

**Build the main swap contract:**

```bash
sui client test-publish --gas-budget 10000000 --build-env devnet
```

**Or build test coins (optional):**

```bash
cd test_coins
sui client test-publish --gas-budget 10000000 --build-env devnet
cd ..
```

**Note:** If you encounter environment errors, use `test-publish` with `--build-env devnet` instead of `sui move build`.

### Testing

Run unit tests locally (no deployment needed):

```bash
# Tests must be run with testnet environment
sui move test --environment testnet
```

**Note:** Unit tests don't require network access, but Sui requires an environment to be specified. Use `testnet` for running tests.

### Deploying

**Deployment order:** Deploy the swap contract first. If you use `test_coins`, deploy that second and update `test_coins/Move.toml` with the swap package ID. See [test_coins/README.md](test_coins/README.md).

**Deploy swap to devnet:**

```bash
sui client test-publish --gas-budget 10000000 --build-env devnet
```

After successful deployment, save the output:

- `package` ID (your published package address)
- `global` object ID (the Global config object for your AMM)

**For production deployment:**

```bash
sui client publish --gas-budget 10000000
```

This will create a `Publications.toml` file with your published addresses.

## Usage Examples

After deploying, you can interact with your AMM. **Replace the example IDs below with your actual deployed package and object IDs.**

### Example: Adding Liquidity

```bash
# Set up coin type addresses (example - replace with your test coins)
XBTC="0x985c26f5edba256380648d4ad84b202094a4ade3::coins::XBTC"
USDT="0x985c26f5edba256380648d4ad84b202094a4ade3::coins::USDT"
SUI="0x2::sui::SUI"

# Replace these with your actual deployed IDs
package=0xc6f8ce30d96bb9b728e000be94e25cab1a6011d1  # Your package ID from deployment
global=0x28ae932ee07d4a0881e4bd24f630fe7b0d18a332   # Your Global object ID from deployment

# Get your coin objects
sui client objects
sui_coin=0x525c0eb0e1f4d8744ae21984de2e8a089366a557  # Replace with your SUI coin object ID
usdt_coin=0x8e81c2362ff1e7101b2ef2a0d1ff9b3c358a1ac9  # Replace with your USDT coin object ID

# Add liquidity to create a pool
sui client call --gas-budget 10000000 \
  --package=$package \
  --module=interface \
  --function=add_liquidity \
  --args $global $sui_coin 1 $usdt_coin 1 \
  --type-args $SUI $USDT

# Save the LP token ID from the output
lp_sui_usdt=0xdf622fddc8447b0c1d15f8418e010933dd5f0a6c  # Replace with your LP token ID

# Split LP token for testing
sui client split-coin --gas-budget 10000000 \
  --coin-id $lp_sui_usdt \
  --amounts 100000

lp_sui_usdt2=0x6cde2fe9277c92e196585fb12c6e3d5aaa4eab34  # Replace with your split LP token ID
```

### Example: Removing Liquidity

```bash
# Remove liquidity from the pool
sui client call --gas-budget 10000000 \
  --package=$package \
  --module=interface \
  --function=remove_liquidity \
  --args $global $lp_sui_usdt2 \
  --type-args $SUI $USDT

# Save the returned coin IDs from the output
new_usdt_coin=0xc090e45f9461e39abb0452cf3ec297a40efbfdc3  # Replace with your coin ID
new_sui_coin=0x9c8c1cc38cc61a94264911933c69a772ced07a09  # Replace with your coin ID
```

### Example: Swapping Tokens

```bash
# Swap SUI -> USDT
sui client call --gas-budget 10000000 \
  --package=$package \
  --module=interface \
  --function=swap \
  --args $global $new_sui_coin 1 \
  --type-args $SUI $USDT

out_usdt_coin=0x80076d95c8bd1d5a0f97b537669008a1a369ce12  # Replace with your output coin ID

# Swap USDT -> SUI
sui client call --gas-budget 10000000 \
  --package=$package \
  --module=interface \
  --function=swap \
  --args $global $out_usdt_coin 1 \
  --type-args $USDT $SUI

out_sui_coin=0xaa89836115e1e1a4f5fa990ebd2c7be3a5124d07  # Replace with your output coin ID
```

### Example: Adding More Liquidity

```bash
# Add more liquidity to an existing pool
sui client call --gas-budget 10000000 \
  --package=$package \
  --module=interface \
  --function=add_liquidity \
  --args $global $out_sui_coin 100 $new_usdt_coin 1000 \
  --type-args $SUI $USDT
```

---

## Commands Reference

### Utility Commands

```bash
# List your objects (coins, LP tokens, etc.)
sui client objects

# Check gas balance
sui client gas

# Get SUI for gas (devnet)
sui client faucet

# Switch network (devnet / testnet / mainnet)
sui client switch --env devnet

# Show active address
sui client active-address

# Split a coin into smaller amounts
sui client split-coin --gas-budget 10000000 --coin-id $COIN_ID --amounts 100 200 500

# Merge coins (combine multiple coins of same type)
sui client merge-coin --gas-budget 10000000 --primary-coin $COIN1 --coin-to-merge $COIN2
```

### Swap Contract — User Entrypoints

```bash
# Add liquidity (creates pool if first time)
sui client call --gas-budget 10000000 --package $PKG --module interface --function add_liquidity \
  --args $global $coin_x $coin_x_min $coin_y $coin_y_min --type-args $X $Y

# Remove liquidity
sui client call --gas-budget 10000000 --package $PKG --module interface --function remove_liquidity \
  --args $global $lp_coin --type-args $X $Y

# Swap X → Y
sui client call --gas-budget 10000000 --package $PKG --module interface --function swap \
  --args $global $coin_in $coin_out_min --type-args $X $Y

# Multi-coin variants (merge multiple coins first, then call)
# Note: multi_* functions take vector<Coin> args; use merge-coin or SDK to prepare
sui client call --gas-budget 10000000 --package $PKG --module interface --function multi_add_liquidity \
  --args $global $coins_x $coins_x_value $coin_x_min $coins_y $coins_y_value $coin_y_min --type-args $X $Y

sui client call --gas-budget 10000000 --package $PKG --module interface --function multi_remove_liquidity \
  --args $global $lp_coins --type-args $X $Y

sui client call --gas-budget 10000000 --package $PKG --module interface --function multi_swap \
  --args $global $coins_in $coins_in_value $coin_out_min --type-args $X $Y
```

### Swap Contract — Admin (Controller)

```bash
# Pause all operations (emergency)
sui client call --gas-budget 10000000 --package $PKG --module controller --function pause \
  --args $global

# Resume operations
sui client call --gas-budget 10000000 --package $PKG --module controller --function resume \
  --args $global

# Transfer controller to new address
sui client call --gas-budget 10000000 --package $PKG --module controller --function modify_controller \
  --args $global $NEW_CONTROLLER_ADDRESS
```

### Swap Contract — Admin (Beneficiary)

```bash
# Withdraw accumulated fees from pool X-Y to beneficiary
sui client call --gas-budget 10000000 --package $PKG --module beneficiary --function withdraw \
  --args $global --type-args $X $Y
```

### Build & Test

```bash
# Build main swap (no publish)
sui move build --environment devnet

# Run unit tests
sui move test --environment testnet

# Deploy swap to devnet (save package + global IDs)
sui client test-publish --gas-budget 10000000 --build-env devnet

# Deploy for production (creates Publications.toml)
sui client publish --gas-budget 10000000
```
