### [H-1] `ContestManager::createContest` function doesn't verify totalRewards is the sum of all rewards.

**Description:** 

totalRewards param should be equal to the sum of all rewards and there is not a check of that.

**Impact:** 
If by error owner create a contest a totalRewards amount less than the sum of all reward some claimants won't be able to claim their cut.

**Proof of Concept:**

Paste next code in the TestMyCut.sol file

```js 
    function testUserCantClaimCutDueToLackOfMatchingTotalRewardsCheck() mintAndApproveTokens public {
        address player3 = makeAddr("player3");

        vm.startPrank(user);
        rewards = [20, 50, 100];
        players = [player1, player2, player3];
        totalRewards = 100;
        contest = ContestManager(conMan).createContest(players, rewards, IERC20(ERC20Mock(weth)), totalRewards);
        ContestManager(conMan).fundContest(0);
        vm.stopPrank();

         vm.startPrank(player1);
        Pot(contest).claimCut();
        vm.stopPrank();

        vm.startPrank(player2);
        Pot(contest).claimCut();
        vm.stopPrank();

        vm.startPrank(player3);
        vm.expectRevert(); // Will revert due to panic: arithmetic underflow or overflow error
        Pot(contest).claimCut();
        vm.stopPrank();
    }
```
**Recommended Mitigation:** 

Add a conditional to check if totalRewards id equal to the sum of the rewards.

```diff
    function createContest(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards)
        public
        onlyOwner
        returns (address)
    {
+       uint256 _totalRewards = 0;
+       for (uint256 i = 0; i < rewards.length; i++) {
+           _totalRewards += rewards[i];
+       }
+       if(_totalRewards != totalRewards){
+           revert ContestManager__WrongTotalRewards();
+       }
        Pot pot = new Pot(players, rewards, token, totalRewards);
        contests.push(address(pot));
        contestToTotalRewards[address(pot)] = totalRewards;
        return address(pot);
    }
```

### [H-2] it doesn't check the returned value from token transfers

**Description:** 

In multiple places of the code miss verify The return value of external transfer/transferFrom calls and not all the token revert on fail, some tokens return false is the transfer fails.

Here are the places where it is happening: `Pot.sol::_transferReward`, `Pot.sol::closePot` and `ContestManager::token.transferFrom`

**Impact:** 

For example it could be an scenario where the transfer fails in the `Pot.sol::claimCut` function, the reward for the user is set to zero but actually the user didn't receive the funds and the transaction didn't revert.

**Proof of Concept:**

Claimer wants to claim their cut, so call `Pot.sol::claimCut` function. The rewards of the claimer are `set to zero` and the remainingRewards is reduced by the claimer reward amount. Then for any reason the transfer fails and as there isn't a success transfer verification the transaction doesn't revert.
User realizes he didn't receive their cut, so he tries again but in this case the transaction revert with a `Pot__RewardNotFound` error because now he has zero rewards.

**Recommended Mitigation:** 

Use SafeERC20, or ensure that the transfer/transferFrom return value is checked.


### [H-3] Remaining rewards should be divided by the number of claimers

**Description:** 

When `Pot.sol::closePot` function distribute the remaining rewards is dividing that amount into the number of players to distribute into the claimants.  When the number of players is greater than the number of claimants a amount will remaining in the contract

**Impact:** 

The Readme of this project claims that the remainder is distributed equally to those who claimed in time, but this is not the case because always will be a locked remaining  in the contract if there are players that didn't claim, duet o the remaining amount is divided into the number of players when it should be divided into the number of claimants.
This will happens always this scenario happens and it is not an expected behavior

The README of this project states that the remainder is distributed equally to those who claimed in time. However, but this is not the case because. A portion of the remaining balance will always be locked in the contract if some players do not claim, as the remainder is divided among the total number of players rather than just the claimants. This issue will consistently occur in such scenarios and is not the expected behavior

**Proof of Concept:**

Paste next code in the TestMyCut.sol file

```js
    function testRemainingRewardsShouldBeDividedByTheNumberOfClaimers() mintAndApproveTokens public {
        address player3 = makeAddr("player3");

        vm.startPrank(user);
        rewards = [20, 50, 100];
        players = [player1, player2, player3];
        totalRewards = 170;
        contest = ContestManager(conMan).createContest(players, rewards, IERC20(ERC20Mock(weth)), totalRewards);
        ContestManager(conMan).fundContest(0);
        vm.stopPrank();

        vm.startPrank(player1);
        Pot(contest).claimCut();
        vm.stopPrank();

        vm.startPrank(player2);
        Pot(contest).claimCut();
        vm.stopPrank();

        vm.warp(91 days);

        vm.startPrank(user);
        ContestManager(conMan).closeContest(contest);
        vm.stopPrank();

        uint256 finalPotBalance = IERC20(ERC20Mock(weth)).balanceOf(contest);
        console.log('finalPotBalance: ', finalPotBalance);
        // The remaining amount is 30 
        // ((100 - (100/10)) /3 ) => (100 - 10)/ 3 => 90/3 = 30 
        // So 30 for each claimer, as there are 2 claimer will remain 30 in the contract 
        assertEq(finalPotBalance, 30);
    }
```

**Recommended Mitigation:** 

```diff
    function closePot() external onlyOwner {
        if (block.timestamp - i_deployedAt < 90 days) {
            revert Pot__StillOpenForClaim();
        }
        if (remainingRewards > 0) {
            uint256 managerCut = remainingRewards / managerCutPercent;
            i_token.transfer(msg.sender, managerCut);

-           uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
+           uint256 claimantCut = (remainingRewards - managerCut) / claimants.length;
            for (uint256 i = 0; i < claimants.length; i++) {
                _transferReward(claimants[i], claimantCut);
            }
        }
    }
```

### [M-1] There is not verification if  player.length is equal than  rewards.length 

**Description:** 
Ideally the number of players should be equal to the number of rewards, but there isn't a check of this.

**Impact:** 
If by error the owner miss a player but create the contest with the rewards of that player, after the clamming period the rewards of that missing player will be distributed to the claimants.

**Proof of Concept:**

Paste next code in the TestMyCut.sol file

As you see in the assert player1 and player2 receive part of the rewards that were planned to be given to the player3
```js
   function testLackOfNumOfPlayersAndNumOfRewardsEqualityCheckEndsInLostOfFunds() mintAndApproveTokens public {
        address player3 = makeAddr("player3");

        vm.startPrank(user);
        rewards = [20, 50, 100];
        players = [player1, player2];
        totalRewards = 170;
        contest = ContestManager(conMan).createContest(players, rewards, IERC20(ERC20Mock(weth)), totalRewards);
        ContestManager(conMan).fundContest(0);
        vm.stopPrank();

        vm.startPrank(player1);
        Pot(contest).claimCut();
        vm.stopPrank();

        vm.startPrank(player2);
        Pot(contest).claimCut();
        vm.stopPrank();

        vm.warp(91 days);

        vm.startPrank(user);
        ContestManager(conMan).closeContest(contest);
        vm.stopPrank();

        uint256 finalPlayer1Balance = IERC20(ERC20Mock(weth)).balanceOf(player1);
        uint256 finalPlayer2Balance = IERC20(ERC20Mock(weth)).balanceOf(player2);

        // Player1 balance  should be 20 + (100 - (100-10) /2 ) => 20 + 90/2 => 65
        uint256 expectedFinalPlayer1Balance = rewards[0] + ((rewards[2] - (rewards[2]/ 10))/players.length);
        uint256 expectedFinalPlayer2Balance = rewards[1] + ((rewards[2] - (rewards[2]/ 10))/players.length);
        
        assertEq(finalPlayer1Balance, expectedFinalPlayer1Balance);
        assertEq(finalPlayer2Balance, expectedFinalPlayer2Balance);
    }
```

**Recommended Mitigation:** 

Add a condition that the if the number of players is the same than the number of rewards.

```diff
    function createContest(address[] memory players, uint256[] memory rewards, IERC20 token, uint256 totalRewards)
        public
        onlyOwner
        returns (address)
    {

+       if (players.length != rewards.length) {
+           revert ContestManager__RewardsAndPlayersNotMatch();
+       }

        Pot pot = new Pot(players, rewards, token, totalRewards);
        contests.push(address(pot));
        contestToTotalRewards[address(pot)] = totalRewards;
        return address(pot);
    }
```

### [S-#] There is not a function to sweep the remaining amount due the loss of precision at the division. 

**Description:** 

As Solidity doesn't support decimal number there is a change of loss precision in a division. Therefore this loss of precision can left some remaining amount locked in the contract. 
For this reason a function to sweep the remaining dust is a good idea.

**Impact:** 

It is not good to have some remaining amount stuck in the contract.

**Proof of Concept:**



**Recommended Mitigation:** 