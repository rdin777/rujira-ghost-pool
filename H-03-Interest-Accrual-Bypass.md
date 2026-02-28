# [H-03] Borrowers can bypass interest accrual due to precision loss

## Severity: High
## Location: DecimalScaled::decompose

## Description:
The `calculate_interest` function in ghost-vault calculates interest based on the time elapsed. It uses `DecimalScaled::decompose()` to split the result into an integer part and a fractional part.

However, `decompose()` uses `to_uint_floor()`, which rounds down to zero for any value less than 1 unit. Since `self.last_updated` is updated regardless of whether interest was accrued, an attacker can frequently trigger updates (e.g., every second).

## Impact:
Borrowers can bypass interest accrual indefinitely. For many positions, interest per second is < 1 unit, so frequent calls result in 0 debt growth while moving the timestamp forward.

## Recommended Fix:
Carry over the fractional part (dust) and only accrue when the accumulated interest exceeds 1 unit, or use higher precision for internal accounting.
