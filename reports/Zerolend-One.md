# Issue H-5: `LiquidationLogic@_burnCollateralTokens` does not account for liquidation fees when withdrawing collateral during liquidation leading to incorrect accounting and Pools insolvency 
Severity: High

### Summary

`LiquidationLogic@_burnCollateralTokens` does not account for liquidation fees when withdrawing collateral during liquidation leading to incorrect accounting and Pools insolvency, ultimately impacting regular flows (.e.g borrows, withdrawals, redemptions, ...) in the protocol for the different actors (.i.e Pools users, Curated Vaults and their users, NFT Positions users).

### Root Cause

[LiquidationLogic](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol)
```solidity
// ...
library LiquidationLogic {
  // ...
  function executeLiquidationCall(
    mapping(address => DataTypes.ReserveData) storage reservesData,
    mapping(uint256 => address) storage reservesList,
    mapping(address => mapping(bytes32 => DataTypes.PositionBalance)) storage balances,
    mapping(address => DataTypes.ReserveSupplies) storage totalSupplies,
    mapping(bytes32 => DataTypes.UserConfigurationMap) storage usersConfig,
    DataTypes.ExecuteLiquidationCallParams memory params
  ) external {
    // ...

    (vars.actualCollateralToLiquidate, vars.actualDebtToLiquidate, vars.liquidationProtocolFeeAmount) = // <==== Audit 
    _calculateAvailableCollateralToLiquidate(
      collateralReserve,
      vars.debtReserveCache,
      vars.actualDebtToLiquidate,
      vars.userCollateralBalance,
      vars.liquidationBonus,
      IPool(params.pool).getAssetPrice(params.collateralAsset),
      IPool(params.pool).getAssetPrice(params.debtAsset),
      IPool(params.pool).factory().liquidationProtocolFeePercentage()
    );

    // ...

    _burnCollateralTokens(
      collateralReserve, params, vars, balances[params.collateralAsset][params.position], totalSupplies[params.collateralAsset]
    ); // <==== Audit

    if (vars.liquidationProtocolFeeAmount != 0) {
      // ...

      IERC20(params.collateralAsset).safeTransfer(IPool(params.pool).factory().treasury(), vars.liquidationProtocolFeeAmount);  // <==== Audit
    }

    // ...
  }

   function _burnCollateralTokens(
    DataTypes.ReserveData storage collateralReserve,
    DataTypes.ExecuteLiquidationCallParams memory params,
    LiquidationCallLocalVars memory vars,
    DataTypes.PositionBalance storage balances,
    DataTypes.ReserveSupplies storage totalSupplies
  ) internal {
    // ...
    balances.withdrawCollateral(totalSupplies, vars.actualCollateralToLiquidate, collateralReserveCache.nextLiquidityIndex); // <==== Audit : actualCollateralToLiquidate doesn't include liquidation fees
    IERC20(params.collateralAsset).safeTransfer(msg.sender, vars.actualCollateralToLiquidate);
  }

  // ...

  function _calculateAvailableCollateralToLiquidate(
    DataTypes.ReserveData storage collateralReserve,
    DataTypes.ReserveCache memory debtReserveCache,
    uint256 debtToCover,
    uint256 userCollateralBalance,
    uint256 liquidationBonus,
    uint256 collateralPrice,
    uint256 debtAssetPrice,
    uint256 liquidationProtocolFeePercentage
  ) internal view returns (uint256, uint256, uint256) {
    // ...

    if (liquidationProtocolFeePercentage != 0) {
      vars.bonusCollateral = vars.collateralAmount - vars.collateralAmount.percentDiv(liquidationBonus);
      vars.liquidationProtocolFee = vars.bonusCollateral.percentMul(liquidationProtocolFeePercentage);
      return (vars.collateralAmount - vars.liquidationProtocolFee, vars.debtAmountNeeded, vars.liquidationProtocolFee);  // <==== Audit
    } else {
      return (vars.collateralAmount, vars.debtAmountNeeded, 0);
    }
  }
}
```

[PositionBalanceConfiguration](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/configuration/PositionBalanceConfiguration.sol#L85-L96)
```solidity
  function withdrawCollateral(
    DataTypes.PositionBalance storage self,
    DataTypes.ReserveSupplies storage supply,
    uint256 amount,
    uint128 index
  ) internal returns (uint256 sharesBurnt) {
    sharesBurnt = amount.rayDiv(index);
    require(sharesBurnt != 0, PoolErrorsLib.INVALID_BURN_AMOUNT);
    self.lastSupplyLiquidtyIndex = index;
    self.supplyShares -= sharesBurnt; // <==== Audit
    supply.supplyShares -= sharesBurnt; // <==== Audit
  }
```

When there are protocol liquidation fees, `_burnCollateralTokens` doesn't account for liquidation fees when withrawing the collateral, leading to the pool and actor having more supply shares than reality.

### Internal pre-conditions

Protocol liquidations fees are set.

### External pre-conditions

N/A

### Attack Path
Not an attack path per say as this happens in every liquidation when there are liquidation fees.

1. Alice supplies `tokenA`
2. Bob supplies `tokenB`
3. Alice borrows `tokenB`
4. Alice becomes liquidatable
5. Bob liquidates Alice

### Impact

* Incorrect accounting : pool and actor supply shares are higher than reality, allowing a liquidated actor to borrow more than what they should really be able to for example.
* Pools insolvency : since the liquidation fees are transferred to the treasury from the pool but not reflected on the pool and actor supply shares, the actor can withdraw them again at the expense of other actors. This leads to the other actors not being able to fully withdraw their provided collateral and potentially disrupting functionality such as Curated Vaults reallocation where the withdrawn amount cannot be controlled.

### PoC

#### Test
```solidity
  function testLiquidationWithFees() external {
    poolFactory.setLiquidationProtcolFeePercentage(0.05e4);

    _mintAndApprove(alice, tokenA, 3000 ether, address(pool));
    _mintAndApprove(bob, tokenB, 5000 ether, address(pool));

    vm.startPrank(bob);
    pool.supplySimple(address(tokenB), bob, 3000 ether, 0);
    vm.stopPrank();

    vm.startPrank(alice);
    pool.supplySimple(address(tokenA), alice, 1000 ether, 0);
    pool.borrowSimple(address(tokenB), alice, 375 ether, 0);
    vm.stopPrank();

    oracleB.updateAnswer(2.5e8);

    vm.prank(bob);
    pool.liquidateSimple(address(tokenA), address(tokenB), pos, type(uint256).max);

    assertEq(pool.getBalance(address(tokenA), alice, 0), tokenA.balanceOf(address(pool)));
  }
```

#### Results
```console
forge test --match-test testLiquidationWithFees
[⠢] Compiling...
No files changed, compilation skipped

Ran 1 test for test/forge/core/pool/PoolLiquidationTests.t.sol:PoolLiquidationTest
[FAIL. Reason: assertion failed: 17968750000000000000 != 15625000000000000000] testLiquidationWithFees() (gas: 1003975)
Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 5.25ms (1.62ms CPU time)

Ran 1 test suite in 347.96ms (5.25ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/forge/core/pool/PoolLiquidationTests.t.sol:PoolLiquidationTest
[FAIL. Reason: assertion failed: 17968750000000000000 != 15625000000000000000] testLiquidationWithFees() (gas: 1003975)

Encountered a total of 1 failing tests, 0 tests succeeded
```

### Mitigation

```diff
diff --git a/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol b/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol
index e89d626..0a48da6 100644
--- a/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol
+++ b/zerolend-one/contracts/core/pool/logic/LiquidationLogic.sol
@@ -225,7 +225,7 @@ library LiquidationLogic {
     );

     // Burn the equivalent amount of aToken, sending the underlying to the liquidator
-    balances.withdrawCollateral(totalSupplies, vars.actualCollateralToLiquidate, collateralReserveCache.nextLiquidityIndex);
+    balances.withdrawCollateral(totalSupplies, vars.actualCollateralToLiquidate + vars.liquidationProtocolFeeAmount, collateralReserveCache.nextLiquidityIndex);
     IERC20(params.collateralAsset).safeTransfer(msg.sender, vars.actualCollateralToLiquidate);
   }
```

# Issue M-14: NFTPositionManager's `repay()` and `repayETH()` are unavailable unless preceded atomically by an accounting updating operation 

Severity: Medium

### Summary

The check in `_repay()` cannot be satisfied if pool state was not already updated by some other operation in the same block. This makes any standalone `repay()` and `repayETH()` calls revert, i.e. core repaying functionality can frequently be unavailable for end users since it's not packed with anything by default in production usage

### Root Cause


Pool state wasn't updated before `previousDebtBalance` was set:

[NFTPositionManager.sol#L115-L128](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManager.sol#L115-L128)

```solidity
  /// @inheritdoc INFTPositionManager
  function repay(AssetOperationParams memory params) external {
    if (params.asset == address(0)) revert NFTErrorsLib.ZeroAddressNotAllowed();
    IERC20Upgradeable(params.asset).safeTransferFrom(msg.sender, address(this), params.amount);
>>  _repay(params);
  }

  /// @inheritdoc INFTPositionManager
  function repayETH(AssetOperationParams memory params) external payable {
    params.asset = address(weth);
    if (msg.value != params.amount) revert NFTErrorsLib.UnequalAmountNotAllowed();
    weth.deposit{value: params.amount}();
>>  _repay(params);
  }
```

[NFTPositionManagerSetters.sol#L105-L121](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L105-L121)

```solidity
  function _repay(AssetOperationParams memory params) internal nonReentrant {
    if (params.amount == 0) revert NFTErrorsLib.ZeroValueNotAllowed();
    if (params.tokenId == 0) {
      if (msg.sender != _ownerOf(_nextId - 1)) revert NFTErrorsLib.NotTokenIdOwner();
      params.tokenId = _nextId - 1;
    }

    Position memory userPosition = _positions[params.tokenId];

    IPool pool = IPool(userPosition.pool);
    IERC20 asset = IERC20(params.asset);

    asset.forceApprove(userPosition.pool, params.amount);

>>  uint256 previousDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
    DataTypes.SharesType memory repaid = pool.repay(params.asset, params.amount, params.tokenId, params.data);
>>  uint256 currentDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
```

[PoolGetters.sol#L94-L97](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/PoolGetters.sol#L94-L97)

```solidity
  function getDebt(address asset, address who, uint256 index) external view returns (uint256 debt) {
    bytes32 positionId = who.getPositionId(index);
>>  return _balances[asset][positionId].getDebtBalance(_reserves[asset].borrowIndex);
  }
```

But it was updated in `pool.repay()` before repayment workflow altered the state:

[BorrowLogic.sol#L117-L125](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/BorrowLogic.sol#L117-L125)

```solidity
  function executeRepay(
    ...
  ) external returns (DataTypes.SharesType memory payback) {
    DataTypes.ReserveCache memory cache = reserve.cache(totalSupplies);
>>  reserve.updateState(params.reserveFactor, cache);
```

[ReserveLogic.sol#L87-L95](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L87-L95)

```solidity
  function updateState(DataTypes.ReserveData storage self, uint256 _reserveFactor, DataTypes.ReserveCache memory _cache) internal {
    // If time didn't pass since last stored timestamp, skip state update
    if (self.lastUpdateTimestamp == uint40(block.timestamp)) return;

>>  _updateIndexes(self, _cache);
    _accrueToTreasury(_reserveFactor, self, _cache);

    self.lastUpdateTimestamp = uint40(block.timestamp);
  }
```

[ReserveLogic.sol#L220-L238](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/pool/logic/ReserveLogic.sol#L220-L238)

```solidity
  function _updateIndexes(DataTypes.ReserveData storage _reserve, DataTypes.ReserveCache memory _cache) internal {
    ...
    if (_cache.currDebtShares != 0) {
      uint256 cumulatedBorrowInterest = MathUtils.calculateCompoundedInterest(_cache.currBorrowRate, _cache.reserveLastUpdateTimestamp);
      _cache.nextBorrowIndex = cumulatedBorrowInterest.rayMul(_cache.currBorrowIndex).toUint128();
>>    _reserve.borrowIndex = _cache.nextBorrowIndex;
    }
```

This way the `previousDebtBalance - currentDebtBalance` consists of state change due to the passage of time since last update and state change due to repayment:

[NFTPositionManagerSetters.sol#L105-L125](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L105-L125)

```solidity
  function _repay(AssetOperationParams memory params) internal nonReentrant {
    if (params.amount == 0) revert NFTErrorsLib.ZeroValueNotAllowed();
    if (params.tokenId == 0) {
      if (msg.sender != _ownerOf(_nextId - 1)) revert NFTErrorsLib.NotTokenIdOwner();
      params.tokenId = _nextId - 1;
    }

    Position memory userPosition = _positions[params.tokenId];

    IPool pool = IPool(userPosition.pool);
    IERC20 asset = IERC20(params.asset);

    asset.forceApprove(userPosition.pool, params.amount);

    uint256 previousDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
    DataTypes.SharesType memory repaid = pool.repay(params.asset, params.amount, params.tokenId, params.data);
    uint256 currentDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);

>>  if (previousDebtBalance - currentDebtBalance != repaid.assets) {
      revert NFTErrorsLib.BalanceMisMatch();
    }
```

`previousDebtBalance` can be stale and, in general, it is `previousDebtBalance - currentDebtBalance = (actualPreviousDebtBalance - currentDebtBalance) - (actualPreviousDebtBalance - previousDebtBalance) = repaid.assets - {debt growth due to passage of time since last update} < repaid.assets`, where `actualPreviousDebtBalance` is `pool.getDebt()` result after `reserve.updateState()`, but before repayment

### Internal pre-conditions

Interest rate and debt are positive, so there is some interest accrual happens over time. This is normal (going concern) state of any pool

### External pre-conditions

No other state updating operations were run since the last block

### Attack Path

No direct attack needed in this case, a protocol malfunction causes loss to some users

### Impact

Core system functionality, `repay()` and `repayETH()`, are unavailable whenever aren't grouped with other state updating calls, which is most of the times in terms of the typical end user interactions. Since the operation is time sensitive and is typically run by end users directly, this means that there is a substantial probability that unavailability in this case leads to losses, e.g. a material share of NFTPositionManager users cannot repay in time and end up being liquidated as a direct consequence of the issue (i.e. there are other ways to meet the risk, but time and investigational effort are needed, while liquidations will not wait).

Overall probability is medium: interest accrues almost always and most operations are stand alone (cumulatively high probability) and repay is frequently enough called close to liquidation (medium probability). Overall impact is high: loss is deterministic on liquidation, is equal to liquidation penalty and can be substantial in absolute terms for big positions. The overall severity is high

### PoC

A user wants and can repay the debt that is about to be liquidated, but all the repayment transactions revert, being done straightforwardly at a stand alone basis, meanwhile the position is liquidated, bearing the corresponding penalty as net loss

### Mitigation

Consider adding direct reserve update before reading from the state, e.g.:

[NFTPositionManagerSetters.sol#L119](https://github.com/sherlock-audit/2024-06-new-scope/blob/main/zerolend-one/contracts/core/positions/NFTPositionManagerSetters.sol#L119)

```diff
+   pool.forceUpdateReserve(params.asset);
    uint256 previousDebtBalance = pool.getDebt(params.asset, address(this), params.tokenId);
```
