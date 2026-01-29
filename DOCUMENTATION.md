# Sui AMM Swap — Technical Documentation

## TLDR

| What             | Module                        | Notes                      |
| ---------------- | ----------------------------- | -------------------------- |
| Add liquidity    | `interface::add_liquidity`    | X + Y → LP tokens          |
| Remove liquidity | `interface::remove_liquidity` | Burn LP → X + Y back       |
| Swap             | `interface::swap`             | 0.3% fee, constant product |
| Core logic       | `implements`                  | Pool state, LP minting     |
| Pause/resume     | `controller`                  | Controller only            |
| Withdraw fees    | `beneficiary`                 | Beneficiary only           |
| Pool ordering    | `comparator`                  | X < Y by BCS               |
| Safe math        | `math`                        | mul_div, sqrt              |

**Formula:** `x * y = k`. Swap: `amount_out = (amount_in * 99.7%) * reserve_out / (reserve_in + amount_in * 99.7%)`

---

## 1. Overview

Fork of [OmniBTC Sui-AMM-swap](https://github.com/OmniBTC/Sui-AMM-swap). AMM = constant product, no order book. `test_coins` depends on `swap` (deploy swap first).

```
sources/
├── interface.move   # entrypoints
├── implements.move  # Pool, Global, LP, core logic
├── controller.move # pause/resume
├── beneficiary.move
├── event.move
├── comparator.move
├── math.move
└── implements_tests.move
```

`implements` is friend to `interface`, `controller`, `beneficiary`. `event` is friend to `interface`, `beneficiary`.

---

## 2. Concepts

**Constant product:** `x * y = k`. Swap keeps k (minus fee). Add/remove liquidity changes k proportionally.

**LP tokens:** `LP<X,Y>` = share of pool. Minted on add, burned on remove. Amount ∝ your share.

**Pool keys:** Pools ordered by `(X, Y)` where X < Y (BCS bytes). `LP-USDT-XBTC` = `LP-XBTC-USDT`.

---

## 3. Move Syntax Quick Reference

| Syntax                | Meaning                                  |
| --------------------- | ---------------------------------------- |
| `module swap::name`   | Module in package `swap`                 |
| `public entry fun`    | Callable from outside (tx entrypoint)    |
| `public(friend) fun`  | Only callable by friend modules          |
| `public fun`          | Callable by anyone (read)                |
| `&mut T`              | Mutable reference                        |
| `&T`                  | Immutable reference                      |
| `<X, Y>`              | Generic type params (coin types)         |
| `phantom X`           | Type param for identity only, not stored |
| `assert!(cond, code)` | Abort with `code` if `cond` false        |
| `ctx: &mut TxContext` | Tx context (sender, etc.)                |

**Sui types:** `Coin<T>` = fungible token (has `value`). `Balance<T>` = internal balance (no object ID). `Supply<T>` = controls minting. `UID` = object ID. `Bag` = key-value store (String → Pool).

---

## 4. Module Reference

### 4.1 interface — Entrypoints

**add_liquidity** — `(global, coin_x, coin_x_min, coin_y, coin_y_min)`. Registers pool if first time. Sends LP to sender.

**remove_liquidity** — `(global, lp_coin)`. Burns LP, sends X and Y to sender.

**swap** — `(global, coin_in, coin_out_min)`. `coin_out_min` = slippage tolerance.

**multi_add_liquidity**, **multi_remove_liquidity**, **multi_swap** — Same but take `vector<Coin>`. Merge coins first.

### 4.2 implements — Core Logic & Data

**Global** (shared config):

```move
struct Global has key {
    id: UID,
    has_paused: bool,      // emergency pause
    controller: address,   // can pause/resume
    beneficiary: address,  // can withdraw fees
    pools: Bag,            // lp_name (String) → Pool<X,Y>
}
```

**Pool** (one liquidity pool):

```move
struct Pool<phantom X, phantom Y> has store {
    global: ID,
    coin_x: Balance<X>,        // reserve X
    fee_coin_x: Balance<X>,    // accumulated fees (X)
    coin_y: Balance<Y>,       // reserve Y
    fee_coin_y: Balance<Y>,   // accumulated fees (Y)
    lp_supply: Supply<LP<X,Y>>,
    min_liquidity: Balance<LP<X,Y>>,  // locked (anti-drain)
}
```

**LP** (LP token type):

```move
struct LP<phantom X, phantom Y> has drop, store {}
```

**Key functions:**

| Function                   | Visibility | Purpose                           |
| -------------------------- | ---------- | --------------------------------- |
| `init`                     | (init)     | Create Global on publish          |
| `register_pool<X,Y>`       | friend     | Create new pool                   |
| `add_liquidity`            | friend     | Add X,Y → mint LP                 |
| `remove_liquidity`         | friend     | Burn LP → return X,Y              |
| `swap_out`                 | friend     | Swap X→Y or Y→X                   |
| `withdraw`                 | friend     | Pull fees to beneficiary          |
| `get_mut_pool`             | friend     | Borrow pool from Global           |
| `has_registered`           | friend     | Check pool exists                 |
| `get_reserves_size`        | public     | (reserve_x, reserve_y, lp_supply) |
| `get_pool_reserves`        | public     | Same, takes Global                |
| `get_amount_out`           | public     | Swap output (with fee)            |
| `calc_optimal_coin_values` | public     | Optimal X,Y for add               |
| `generate_lp_name`         | public     | e.g. "LP-USDT-XBTC"               |
| `is_order<X,Y>`            | public     | true if X < Y                     |
| `pause` / `resume`         | friend     | Emergency controls                |

### 4.3 controller

`pause`, `resume`, `modify_controller` — controller address only.

### 4.4 beneficiary

`withdraw<X,Y>` — pulls fee balances to beneficiary.

### 4.5 event

`AddedEvent`, `RemovedEvent`, `SwappedEvent`, `WithdrewEvent` — (global, lp_name, amounts).

### 4.6 comparator

`compare<T>` → Result. `is_equal`, `is_smaller_than`, `is_greater_than`. BCS lexicographic.

### 4.7 math

| Function                | Formula               | Purpose            |
| ----------------------- | --------------------- | ------------------ |
| `mul_to_u128(x, y)`     | x \* y as u128        | Avoid u64 overflow |
| `sqrt(y)`               | √y (Babylonian)       | Initial LP amount  |
| `mul_div(x, y, z)`      | x \* y / z (u64)      | Proportional math  |
| `mul_div_u128(x, y, z)` | x \* y / z (u128→u64) | Swap output        |

---

## 5. Test Coins

**coins:** USDT, XBTC, BTC, ETH, BNB, WBTC, USDC, DAI

**faucet:** `claim<T>` (anyone, 1 unit), `force_claim<T>` (admin), `force_add_liquidity` (admin, bootstraps predefined pools), `add_admin`/`remove_admin`, `add_supply<T>`

Predefined pools: BTC–ETH, BTC–USDT, ETH–USDT, USDC–USDT, BNB–USDT, BNB–USDC, DAI–USDC, BTC–DAI, DAI–ETH

---

## 6. Math & Formulas

### Constants (implements)

| Name              | Value         | Meaning                |
| ----------------- | ------------- | ---------------------- |
| FEE_MULTIPLIER    | 30            | 0.3% = 30/10000        |
| FEE_SCALE         | 10000         | Fee denominator        |
| MINIMAL_LIQUIDITY | 1000          | Locked LP (anti-drain) |
| MAX_POOL_VALUE    | U64_MAX/10000 | Max per reserve        |

### Initial LP (first liquidity)

```
initial_lp = sqrt(optimal_x * optimal_y)
minted_lp = initial_lp - MINIMAL_LIQUIDITY
```

MINIMAL_LIQUIDITY stays locked in pool.

### LP when pool exists

```
lp_x = lp_supply * optimal_x / reserve_x
lp_y = lp_supply * optimal_y / reserve_y
minted_lp = min(lp_x, lp_y)
```

### Swap output (constant product with fee)

```
coin_in_after_fee = coin_in * (FEE_SCALE - FEE_MULTIPLIER) / FEE_SCALE
                  = coin_in * 9970 / 10000
amount_out = (coin_in_after_fee * reserve_out) / (reserve_in + coin_in_after_fee)
```

### Fee split

- 0.3% total swap fee (taken from input)
- 20% of that (0.06%) → protocol via `get_fee_to_fundation` (fee_coin_x, fee_coin_y)
- 80% of that (0.24%) → stays in pool (increases reserves)

### Remove liquidity

```
coin_x_out = reserve_x * lp_burned / lp_supply
coin_y_out = reserve_y * lp_burned / lp_supply
```

---

## 7. Error Codes

**interface:** 101–105 (permissions, emergency, empty coins)

**implements:** 0–15 — zero amount, empty reserves, pool full, insufficient X/Y, divide by zero, overlimit, slippage, same coin, pool exists/not, wrong order, overflow, incorrect swap, etc.

**controller:** 201–203 (permissions, already/not paused)

**beneficiary:** 301–303 (permissions, emergency)

Full list in source: `implements.move` (ERR\_\* constants).
