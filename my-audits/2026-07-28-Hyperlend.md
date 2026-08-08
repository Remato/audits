## `HyperlendPairDeployer::globalPause` cannot pause any pair and reports success

**Protocol:** Hyperlend (HyperEVM, chain 999)
**Contract:** `0xD5B33d3c6e750A51fd4E90dbf4AFa2586E33d02c`
**Type:** broken access control on an emergency control, with silent failure
**Severity:** Medium
**Status:** Fixed

---

## Summary
`globalPause` is the protocol-wide emergency stop for isolated pairs. It authorizes the caller as the circuit breaker, then calls `pause()` on each pair in the supplied list. 
At that inner call, `msg.sender` is the **deployer contract**, not the circuit breaker. `HyperlendPairAccessControl._requireProtocolOrOwner()` accepts only `circuitBreakerAddress`, `owner()` and `timelockAddress`, the deployer is none of the 3, so every `pause()` reverts. The bare `catch {}` swallows every revert, so the transaction **succeeds**, returns an array of `address(0)`, and emits nothing.

The operator receives a successful transaction receipt for an emergency action that paused nothing. This is deterministic. It fails for every pair, on every call, under every configuration reachable at deploy time

---

## Root cause

The deployer gates the caller correctly:

```solidity
    // HyperlendPairDeployer::globalPause
    if (msg.sender != circuitBreakerAddress) revert CircuitBreakerOnly();
    ...
    try IHyperlendPair(_pairAddress).pause() {
        _updatedAddresses[i] = _addresses[i];
@>  } catch {}
```

But the pair authorises against a set the deployer is not in:

```solidity
// HyperlendPairAccessControl::_requireProtocolOrOwner
if (
    msg.sender != circuitBreakerAddress &&
    msg.sender != owner() &&
    msg.sender != timelockAddress
) {
    revert OnlyProtocolOrOwner();
}
```

`msg.sender` inside `pause()` is `address(deployer)`. The pair's `circuitBreakerAddress` comes from the immutables the deployer itself encoded, it is the circuit-breaker EOA/multisig, never the deployer contract

### The dead variable

`HyperlendPairAccessControl` stores the deployer and exposes it:

```solidity
address public immutable DEPLOYER_ADDRESS;   // declared
...
DEPLOYER_ADDRESS = msg.sender;               // assigned in the constructor
```

It is also exported through `IHyperlendPair`. A repository-wide search returns exactly 3 occurrences: the declaration, the assignment, and the interface getter.
**It is read by zero authorization checks anywhere in the package.** An immutable that is captured, stored and exported but never consulted is the signature of a permission that was intended to exist

### No configuration rescues it

The only value that would satisfy the pair's check is for the pair's `circuitBreakerAddress` to equal the deployer own address. But that same variable gates `globalPause` itself (`msg.sender != circuitBreakerAddress`), so the deployer contract would then have to be the caller, and nothing in the codebase ever calls `globalPause`. Both branches fail

---

## Why the silence is the dangerous part

`catch {}` has no body. There is no counter, no event, no partial-failure signal, and no revert when zero pairs are paused. The return value is a fresh `address[](n)` left at its zero default.

An operator running incident response sees a green transaction and reasonably concludes the markets are frozen. They are not. The failure is discovered by observing that the attack is still progressing, which is the worst possible moment to learn that the emergency control is inoperative

---

## Impact and likelihood

**Likelihood - High.** It fails 100% of the time on every live pair

**Impact - Medium.** Stated honestly: the emergency capability is **not** lost. The circuit breaker can still pause each pair individually by calling `pause()` on it directly. What is lost is the batch operation, plus, far more importantly, the truthfulness of its result. The realistic damage is time-to-detection during an incident: minutes spent believing markets are frozen while they are live, on a system whose entire emergency posture rests on this control

---

## Proof of concept

<details>
<summary>Full test file (click to expand)</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import {Test, console2} from "forge-std/Test.sol";
import {IHyperlendPair, IHyperlendPairDeployer, IHyperlendPairRegistry} from "../src/Interfaces.sol";

contract GlobalPauseNoOpTest is Test {
    IHyperlendPairDeployer constant DEPLOYER =
        IHyperlendPairDeployer(0xD5B33d3c6e750A51fd4E90dbf4AFa2586E33d02c);
    IHyperlendPairRegistry constant REGISTRY =
        IHyperlendPairRegistry(0xf55AF86c9EC3a7d5fa6367c00a120E6B262f718d);

    address[] pairs;
    address circuitBreaker;

    struct PauseState {
        uint256 borrowLimit;
        uint256 depositLimit;
        bool repay;
        bool withdraw;
        bool liquidate;
        bool interest;
    }

    function setUp() public {
        vm.createSelectFork(vm.envOr("HYPEREVM_RPC", string("https://rpc.hyperliquid.xyz/evm")));
        pairs = REGISTRY.getAllPairAddresses();
        circuitBreaker = DEPLOYER.circuitBreakerAddress();
        require(pairs.length > 0, "no pairs");
    }

    function _snapshot(address p) internal view returns (PauseState memory s) {
        IHyperlendPair pair = IHyperlendPair(p);
        s = PauseState({
            borrowLimit: pair.borrowLimit(),
            depositLimit: pair.depositLimit(),
            repay: pair.isRepayPaused(),
            withdraw: pair.isWithdrawPaused(),
            liquidate: pair.isLiquidatePaused(),
            interest: pair.isInterestPaused()
        });
    }

    function test_deployerIsNotAnAuthorizedPauser() public view {
        for (uint256 i; i < pairs.length; i++) {
            IHyperlendPair pair = IHyperlendPair(pairs[i]);
            assertEq(pair.DEPLOYER_ADDRESS(), address(DEPLOYER), "deployer recorded");
            assertTrue(address(DEPLOYER) != pair.owner(), "deployer is not owner");
            assertTrue(address(DEPLOYER) != pair.circuitBreakerAddress(), "deployer is not CB");
            assertTrue(address(DEPLOYER) != pair.timelockAddress(), "deployer is not timelock");
        }
    }

    function test_globalPausePausesNothingAndReportsSuccess() public {
        PauseState[] memory before = new PauseState[](pairs.length);
        for (uint256 i; i < pairs.length; i++) before[i] = _snapshot(pairs[i]);

        vm.prank(circuitBreaker);
        address[] memory updated = DEPLOYER.globalPause(pairs);

        for (uint256 i; i < updated.length; i++) {
            assertEq(updated[i], address(0), "pair reported as paused");
        }

        for (uint256 i; i < pairs.length; i++) {
            PauseState memory a = _snapshot(pairs[i]);
            assertEq(a.borrowLimit, before[i].borrowLimit, "borrowLimit changed");
            assertEq(a.depositLimit, before[i].depositLimit, "depositLimit changed");
            assertEq(a.repay, before[i].repay, "repay changed");
            assertEq(a.withdraw, before[i].withdraw, "withdraw changed");
            assertEq(a.liquidate, before[i].liquidate, "liquidate changed");
            assertEq(a.interest, before[i].interest, "interest changed");
            console2.log("pair", pairs[i]);
            console2.log("  borrowLimit still", a.borrowLimit);
            console2.log("  isLiquidatePaused still", a.liquidate);
        }
    }

    /// Same key, direct call: the circuit breaker itself is authorized. Only the
    /// deployer-mediated path fails, so this is not a misconfigured breaker address.
    function test_directPauseByCircuitBreakerSucceeds() public {
        address p = pairs[0];
        IHyperlendPair pair = IHyperlendPair(p);

        vm.prank(pair.circuitBreakerAddress());
        pair.pause();

        assertEq(pair.borrowLimit(), 0, "borrowLimit not zeroed");
        assertTrue(pair.isLiquidatePaused(), "liquidate not paused");
        assertTrue(pair.isWithdrawPaused(), "withdraw not paused");
    }

    /// The revert that globalPause swallows.
    function test_pauseFromDeployerReverts() public {
        vm.prank(address(DEPLOYER));
        vm.expectRevert();
        IHyperlendPair(pairs[0]).pause();
    }
}
```

</details>

## Recommended fix

2 changes. The first restores the capability, the second removes the silence

**1. Authorise the deployer on the pair.**

```diff
     function _requireProtocolOrOwner() internal view {
         if (
             msg.sender != circuitBreakerAddress &&
             msg.sender != owner() &&
+            msg.sender != DEPLOYER_ADDRESS &&
             msg.sender != timelockAddress
         ) {
             revert OnlyProtocolOrOwner();
         }
     }
```

**2. Do not swallow failures.**

```diff
-            try IHyperlendPair(_pairAddress).pause() { _updatedAddresses[i] = _addresses[i]; }
-            catch {}
+            try IHyperlendPair(_pairAddress).pause() { _updatedAddresses[i] = _addresses[i]; }
+            catch { emit GlobalPauseFailed(_pairAddress); }
```

Optionally revert if zero pairs were paused. The `try/catch` itself is right, one bad pair should not block the rest, but a batch emergency control must never report success for a batch that entirely failed


