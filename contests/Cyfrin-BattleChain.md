# BattleChain Confidence Pools

> 2nd place on the Cyfrin BattleChain contest - [Leaderboard](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools/results)

Confidence Pools let sponsors bootstrap third-party confidence around an active Safe Harbor agreement on BattleChain. Stakers deposit capital, sponsors top up an optional bonus, and each pool settles based on a moderator-flagged outcome or an expiry backstop. The pool acts as an on-chain confidence mechanism: stakers economically signal their belief that the in-scope contracts will survive the agreement term without being corrupted, and are rewarded from the bonus pool if they do.

## Findings

**2M / 1L / 2I**

## [M-1] Staker capital is trapped at PRODUCTION, the exit closes while the pool's own records show no risk was borne

docs/DESIGN.md §9 states the staker's bargain as a biconditional, then denies the failure outright: "a staker only forfeits the exit option once risk has actually materialized, which is exactly when they begin earning the risk premium." ... "A sponsor cannot grief stakers by keeping the agreement out of attackable mode, stakers can freely exit until risk materializes"

The code does not implement that biconditional, the exit gate reads live registry state, while the pool's record of risk is the one-way latch

[Complete Submission](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools/s/690)

## [M-2] The sponsor is their own agreement's attack moderator: they can author the CORRUPTED verdict that sweeps the pool to their own recovery address, for free and with no breach

The pool treats a registry reading of `CORRUPTED` as exogenous evidence of a breach. §1 states it outright: "Only the terminal `CORRUPTED` state is evidence of an actual breach." §5, §6 and §11 all reason from that premise

The premise is false because the party who writes that flag is the party who receives the swept funds

[Complete Submission](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools/s/692)

## [L-1] ConfidencePool::sweepUnclaimedBonus moves real bonus without finalizing the outcome, so a pre-correction sweep permanently misdirects the whitehat's bounty

On the CORRUPTED-path, resolution finality is tied to claimsStarted: `flagOutcome` may be re-flagged to correct a mistaken outcome, but only until the first value-moving claim latches `claimsStarted`. The design's stated invariant (docs/DESIGN.md §4) is "Once value has left the contract, a corrective re-flag cannot be honored without breaking balance accounting"

`sweepUnclaimedBonus` deliberately never sets `claimsStarted` (to stop a 1-wei donation from slamming the re-flag window shut), yet in the `riskWindowStart == 0` / `totalEligibleStake == 0` branch it removes the entire real bonus from the pool and zeroes `totalBonus`. Because the outcome stays re-flaggable, a moderator correction from `SURVIVED` to good-faith `CORRUPTED` re-snapshots `snapshotTotalBonus` from an already-emptied pot, so the named whitehat is paid principal only while the bonus sits at recoveryAddress

[Complete Submission](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools/s/6)

## [I-1] An empty-data UUPS upgrade installs an implementation nothing has ever `DELEGATECALL`ed, permanently disabling the factory

[Complete Submission](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools/s/225)

## [I-2] Missing `renounceOwnership` override lets the owner permanently brick pool and factory administration

[Complete Submission](https://codehawks.cyfrin.io/c/2026-07-battlechain-confidence-pools/s/411)