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
