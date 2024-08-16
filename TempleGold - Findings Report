# TempleGold - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. `SpiceAuction.sol`: Auction tokens in `startCooldown` period will be lost when `DAOExecutor` tries to recover these tokens.](#M-01)
- ## Low Risk Findings
    - ### [L-01. `DAOExecutor` cannot remove `auctionConfig` and recover for first epoch (i.e `epoch 1`) if required](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: TempleDAO

### Dates: Jul 4th, 2024 - Jul 11th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-templegold)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 1



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. `SpiceAuction.sol`: Auction tokens in `startCooldown` period will be lost when `DAOExecutor` tries to recover these tokens.            



## Summary

The Temple team implements a `recoverToken` function on `SpiceAuction.sol` to allow `DAOExecutor` to recover auction tokens (i.e. either `spiceToken` or `templeGold`) for an epoch or auction in `startCooldown` period (i.e not active or ended). However, these auction tokens will be lost when `DAOExecutor` tries to recover these auction tokens.

## Vulnerability Details

According to the codebase or dev, to recover auction tokens in `startCooldown` period (i.e. `auctionStart` is called and `startCooldown` is still pending), the `DAOExecutor` will use the `removeAuctionConfig` function to delete the `auctionConfig`, and the `EpochInfo` for the epoch in `startCooldown` period (while decrementing the `_currentEpochId`) then use the `recoverToken` function to recover the auction tokens for the epoch.

> /// @dev use `removeAuctionConfig` for case where `auctionStart` is called and cooldown is still pending
> <https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L250>

However, the twist is the `recoverToken` function only specifies to recover auction tokens that are not in allocation (i.e. not an allocation for an auction). This is because the `recoverToken` function is also intended to recover auction tokens for users that send to the `SpiceAuction` contract by mistake therefore these tokens will not be in allocation and can easily be recovered for the users.

`uint256 maxRecoverAmount = balance - (totalAuctionTokenAllocation - _claimedAuctionTokens[token]);`
<https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L263>
<https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L234-L268>

Therefore, the protocol assumes once an auction in `startCooldown` period has been removed using the `removeAuctionConfig` function, the exact auction token amount will be out of allocation and will be recoverable with the `recoverToken` function which is not the case because while removing the auction in `removeAuctionConfig` function the allocation of the auction was never removed from the global `_totalAuctionTokenAllocation` (since it's added in the `startAuction` function for every initialized auction).

<https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L107-L133>
<https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L171-L173>

So each time `DAOExecutor`tries to recover auction tokens for an auction, these amounts will be irrecoverable or lost. To be precise the `recoverToken` function will always revert because of this check below and these auction amounts will be irrecoverable.

`if (amount > maxRecoverAmount) { revert CommonEventsAndErrors.InvalidParam(); }`
<https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L263>

## Impact

* Auction tokens to recover are lost (i.e. irrecoverable).

## POC

Run command `forge test --match-test test_recoverToken_lost_POC` in `SpiceAuction.t.sol`

```javascript
function test_recoverToken_lost_POC() public {
        vm.startPrank(daoExecutor);

        address _spiceToken = spice.spiceToken();
        address _templeGold = address(templeGold);

        _startAuction(true, true);
        IAuctionBase.EpochInfo memory info = spice.getEpochInfo(1);

        //1. start a dummy auction

        // auction active       
        vm.warp(info.startTime);
        
        // some bids
        deal(daiToken, alice, 100 ether);
        deal(daiToken, bob, 100 ether);
        uint256 bidTokenAmount = 10 ether;
        vm.startPrank(alice);
        IERC20(daiToken).approve(address(spice), type(uint).max);
        spice.bid(bidTokenAmount);
        vm.startPrank(bob);
        IERC20(daiToken).approve(address(spice), type(uint).max);
        spice.bid(bidTokenAmount);
        
        // auction ends
        vm.warp(info.endTime);

        // everyone claims
        vm.startPrank(alice);
        spice.claim(1);
        vm.startPrank(bob);
        spice.claim(1);

        // wait period
        vm.warp(info.endTime + 2 weeks);

        //2. real issue demonstration
        _startAuction(true, true);
        IAuctionBase.EpochInfo memory _info = spice.getEpochInfo(2);

        // auction in cooldown
        vm.warp(_info.startTime - 5);

        uint256 amountToRecover = _info.totalAuctionTokenAmount;

        // remove the auction in cooldown to recover auction tokens
        vm.startPrank(daoExecutor);
        spice.removeAuctionConfig();

        // recover fails - auction tokens become irrecoverable
        vm.expectRevert(abi.encodeWithSelector(CommonEventsAndErrors.InvalidParam.selector));
        spice.recoverToken(_templeGold, treasury, amountToRecover);
    }
```

## Tools Used

Manual

## Recommendations

In the `removeAuctionConfig` function, remove the `info.totalAuctionTokenAmount` of the current epoch that will be deleted from the `_totalAuctionTokenAllocation` of the auction token.

> This is a good reference: <https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L285-L288>


# Low Risk Findings

## <a id='L-01'></a>L-01. `DAOExecutor` cannot remove `auctionConfig` and recover for first epoch (i.e `epoch 1`) if required            



## Summary

`removeAuctionConfig` function in `SpiceAuction.sol` prevents `DAOExecutor` from removing the auction configuration for first epoch (i.e `epoch id == 1`). This issue restricts `DAOExecutor` from removing the `setAuctionConfig` for first epoch (if required) and forces `startAuction` in order to remove the configuration. However, this could escalate to a situation where if the `startCooldown` is set as zero, the auction starts immediately and the configuration can never be removed. This could be significant if the `setAuctionConfig` needs to be removed due to wrong set parameters that could cause harm for the protocol.

## Vulnerability Details

The `removeAuctionConfig` function is designed to allow the `DAOExecutor` to remove the auction configuration for the current epoch or next epoch. To be precise, it only allows `auctionConfig` removal for current epoch if the auction for the current epoch is not active (and has not ended to prevent deleting old ended auctions) i.e in `startCooldown` period and `auctionConfig` removal for next epoch if `auctionConfig` for the next epoch is set.

<https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L107-L133>

While its not an ideal nature of `DAOExecutor` to provide wrong parameters, the existence of `removeAuctionConfig` function is to remove wrong `auctionConfig` parameters provided by mistake and also remove `auctionConfig` to cancel auctions in `startCooldown` period. However, the current implementation seems to have created a complication for the first epoch (i.e `epoch id == 1`) when the current epoch is zero (`if (info.startTime == 0) { revert InvalidConfigOperation(); }`), which restricts `DAOExecutor` from removing the `setAuctionConfig` for first epoch i.e if required.

<https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L112-L113>

This check is intended to ensure that the current epoch is valid (i.e cannot be zero since `epoch 0` is a redundant epoch, `info.startTime` will be zero). However, when the current epoch is 0, it becomes impossible to remove the auction configuration for first epoch (i.e `auctionConfig` for `epoch id 1` that has not started) without actually starting the auction. Therefore to remove `auctionConfig` for `epoch id == 1` (if required) that has not started, `startAuction` will need to be triggered to start the auction before `DAOExecutor` will be able to remove `auctionConfig` for `epoch id == 1` (since `epoch id == 1` will be in `startCooldown` period once initiated). However, this can escalate badly if the `startCooldown` for `epoch id == 1` is set to zero (note: `startCooldown` can be set as zero), which means the auction starts immediately, and `auctionConfig` for `epoch id == 1` can never be removed, causing a permanent block in the removal process.

Therefore, if by chance the `auctionConfig` for `epoch id == 1` needed to be removed due to wrong set parameters in `auctionConfig` e.g wrong `recipient` address, this will cause all bid tokens to be sent to a wrong `recipient` address which will cause a grave impact (note: impact will vary based on the reason why the `auctionConfig` needs to be removed). While its unexpected of the `DAOExecutor` to provide wrong parameters for `auctionConfig`, the existence of `removeAuctionConfig` function suggests the possibility and the need to cancel an auction.

## Impact

First epoch (i.e `epoch 1`) cannot be removed if required which could cause variable severe impacts (e.g loss of bid tokens to wrong `recipient` address).

## POC

Run command `forge test --match-test test_removeAuctionConfig_POC` in `SpiceAuction.t.sol`

```javascript
function test_removeAuctionConfig_POC() public {
        vm.startPrank(daoExecutor);
        // config set but auction not started
        ISpiceAuction.SpiceAuctionConfig memory _config = _getAuctionConfig();
        spice.setAuctionConfig(_config);
        // _currentEpochId = 0 cannot remove for epoch id = 1
        vm.expectRevert(abi.encodeWithSelector(ISpiceAuction.InvalidConfigOperation.selector));
        spice.removeAuctionConfig();
    }
```

## Tools Used

Manual

## Recommendations

Remove this check below as its very unlikely to remove `auctionConfig` for `epoch id == 0` because `epoch 0` is redundant. Therefore, if `current epoch = 0`, and `DAOExecutor` want to remove `auctionConfig` for `epoch id == 1`,  `removeAuctionConfig` will remove `auctionConfig` for `epoch id == 1` (the next epoch) if set.

```shell
-        if (info.startTime == 0) { revert InvalidConfigOperation(); }
```

> Please note this issue also exists in the `recover` function therefore will need to be fixed on both ends to fully mitigate this issue else auction tokens for the first epoch will be irrevocable <https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L251>



