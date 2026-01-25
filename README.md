# Sui-AMM-swap

The first open source AMM swap on the [Sui](https://github.com/MystenLabs).

## [Audit Report](https://movebit.xyz/file/Sui-AMM-swap-Contracts-Audit-Report.pdf)

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

3. **Verify Move.toml configuration**

   The `Move.toml` file should already have the correct dependencies:

   ```toml
   [dependencies]
   Sui = { git = "https://github.com/MystenLabs/sui.git", subdir = "crates/sui-framework/packages/sui-framework", rev = "devnet" }

   [environments]
   devnet = "a9149565"
   ```

   If the `[environments]` section is missing, add it to both `Move.toml` and `test_coins/Move.toml`.

4. **Get SUI tokens for gas (devnet)**
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
sui move test
```

### Deploying

**Deploy to devnet:**

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

```

```
