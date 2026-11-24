# EverestOrBust Self-Audit Log

Every defect found in `EverestOrBust.sol` during internal review, how it was fixed, and where the regression test lives. This is a self-review. It is not a substitute for an independent third-party audit, which is planned before any mainnet deployment.

## Defects (D series)

| ID | Severity | Issue | Fix |
|---|---|---|---|
| D-01 | High | `refund()` never decremented `totalRaisedNormalized`, so refunded money still counted toward the goal | Decrement on refund |
| D-02 | High | `redeemExcess()` allowed double redemption | Function removed entirely, since the goal cap makes overfunding impossible |
| D-03 | Medium | `contribute()` violated checks-effects-interactions | State updates ordered around the external call, guarded by `nonReentrant` |
| D-04 | Low | Constructor accepted zero addresses | Zero-address validation added |
| D-05 | High | One blacklisted token could trap an entire refund or withdrawal | `_trySendToken` plus `stuckBalance` and `claimStuck()` isolate each token |
| D-06 | High | `contribute()` capped only against the per-address cap, so total raised could pass the goal | Cap against `min(remainingAllowance, remainingGoal)` |
| D-07 | Medium | Fee-on-transfer handling could desync token accounting from normalized accounting | Revert on any received-versus-requested mismatch, since only USDC and USDT are accepted |
| D-08 | Low | Constructor never validated `_start` against the current time | `InvalidStartTime` error added |

D-06 to D-08 were found in an independent pre-audit review after several internal passes, which is why an external audit remains required.

## Pre-audit review findings (F series)

| ID | Severity | Finding | Status |
|---|---|---|---|
| F-01 | Medium | `contributorCount` could be inflated with dust contributions | Fixed: `MIN_CONTRIBUTION` of $1, checked on the requested amount before any clamp |
| F-02 | Low | Constructor accepted the same address for USDC and USDT | Fixed: reverts with `UnsupportedToken` |
| F-03 | Info | Avalanche USDT return-value behavior should be verified on-chain | Open: verify on Fuji and mainnet before deployment |
| F-04 | Info | Tokens sent directly to the contract have no sender record and are not refundable | Documented in NatSpec as known behavior |
| F-05 | Cosmetic | `claimStuck()` reused the `NothingToRefund` error | Fixed: dedicated `NothingToClaim` error |
| F-06 | Info | `claimStuck()` is callable by anyone, but funds always go to `to` | Documented in NatSpec as intentional |

## Deliberate design decisions

- No pause function and no admin override. Donors do not have to trust an admin key.
- No upgradability. The deployed code is the code contributors can read.
- Token decimals are fixed at 6 rather than read from the token, which removes an external call.
- Any received-versus-requested balance mismatch reverts. Only USDC and USDT are accepted.

## Test coverage

`test/EverestOrBust.t.sol` covers unit, boundary, fee-on-transfer, blacklist resilience, reentrancy, constructor validation, goal-cap interaction, and minimum-contribution cases, including a fuzz test.
