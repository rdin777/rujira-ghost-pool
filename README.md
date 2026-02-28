# Rujira Ghost Vault: Systemic DoS & Inflation Attack (H-01)

## Context
**Project:** Rujira (Ghost Vault / Ghost Credit)
**Timeline:** December 2025 - January 2026
**Status:** Confirmed / High Severity
**Tech Stack:** Rust, CosmWasm

## Vulnerability: Share Price Inflation & Revert-on-Zero
The `SharePool` implementation in `rujira-rs` is vulnerable to a classic Inflation Attack, but with a critical twist: unlike standard ERC4626 vaults that might just mint 0 shares, this contract **explicitly reverts** if the calculated shares equal zero.

### The Buggy Code (`src/share_pool.rs`):
```rust
let issuance = self.shares * Decimal::from_ratio(amount, self.size);
if issuance.floor().is_zero() {
    return Err(SharePoolError::Zero("Shares".to_string()));
}
```

### Attack Vector:
1. **Initial Setup**: Attacker joins first with 1 unit (`shares=1`, `size=1`).
2. **Inflation**: Attacker "donates" a large amount of assets via direct bank transfer, drastically increasing `self.size`.
3. **Permanent DoS**: Any subsequent deposit (`amount`) that results in `issuance < 1` will cause the contract to return `SharePoolError::Zero`. 

Since the protocol requires a successful deposit to function, the entire vault becomes unusable for standard users.

## Impact
* **Denial of Service**: All new deposits are blocked.
* **Loss of Functionality**: Integration with Ghost Credit is broken as users cannot provide collateral.

## Remediation
* Implement **Virtual Shares/Assets** (offset) to prevent the ratio from being easily manipulated.
* Burn a small amount of "Dead Shares" to the zero address during the first deposit.

  ### Detailed Attack Flow:
1. **Initial State**: A borrower has a significant debt. The protocol calculates interest based on `Δt = current_time - last_updated`.
2. **Precision Threshold**: For the specific debt size and interest rate, the interest becomes >= 1 unit (e.g., 1 uatom) only after 5 seconds of elapsed time.
3. **The Exploit**: 
   - An attacker (or a bot) calls any function that triggers `calculate_interest` (e.g., a small deposit or a dummy update) every **1-4 seconds**.
   - Because `Δt` is small, `interest_to_accrue` is calculated as `< 1.0`.
   - `DecimalScaled::decompose()` uses `.to_uint_floor()`, rounding this value down to **0**.
   - The state variable `last_updated` is set to `current_time`.
4. **Outcome**: The clock is reset, the fractional interest is discarded, and the borrower's debt stays exactly the same despite time passing.

## Step-by-Step Attack Scenario (H-01)

1. **Vulnerability Point**: The pool lacks "Dead Shares" and uses a strict `issuance.is_zero()` check that reverts.
2. **Step 1 (The Hook)**: Attacker calls `join(1)`. 
   - `self.shares` becomes 1.
   - `self.size` becomes 1.
3. **Step 2 (The Inflation)**: Attacker sends 1,000,000 ATOM ($10^{24}$ units) directly to the contract address via `BankMsg`.
   - `self.size` is now $10^{24} + 1$.
   - `self.shares` is still 1.
4. **Step 3 (The DoS)**: A legitimate user tries to deposit 10,000 ATOM ($10^{22}$ units).
   - Calculation: `issuance = (shares * amount) / size`
   - `issuance = (1 * 10^{22}) / (10^{24} + 1) = 0.01`
   - `issuance.floor()` returns **0**.
5. **Final Result**: The contract hits `return Err(SharePoolError::Zero("Shares"))`. The transaction reverts. No one can ever enter the pool again.
