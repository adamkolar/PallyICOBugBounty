# Pally ICO contract bug bounty

  * https://www.reddit.com/r/ethdev/comments/75k46s/up_to_10_eth_for_finding_vulnerabilities_pallyco/
  * Date: 2017-10-11
  * Reward: 10 ETH

## Contracts

[Crowdsale.sol](./contracts/Crowdsale.sol)

## Submission

CalculateExcess function in the crowdsale contract contains error that allows buyer to exceed maxTokensRaised and maxPurchase
When the contract is in the Tier 4 phase, purchase is limited in order to not exceed maxTokensRaised with this code:

```javascript
     uint256 addedTokens = tokensRaised.add(amountPaid.mul(rateTier4));

     // If tokensRaised + what you paid converted to tokens is bigger than the max
     if(addedTokens > maxTokensRaised) {

        // Refund the difference
        uint256 difference = addedTokens.sub(maxTokensRaised);
        differenceWei = difference.div(rateTier4);

        amountPaid = amountPaid.sub(differenceWei);
     }
```

after this check, there's another check for the compliance with maxPurchase which limits how much ether can one buyer contribute.

```javascript
  uint256 addedBalance = crowdsaleBalances[msg.sender].add(amountPaid);

  if(addedBalance <= maxPurchase) {
     crowdsaleBalances[msg.sender] = crowdsaleBalances[msg.sender].add(amountPaid);
  } else {

     // Substracting 1000 ether in wei
     exceedingBalance = addedBalance.sub(maxPurchase);
     amountPaid = msg.value.sub(exceedingBalance);//chyba

     // Add that balance to the balances
     crowdsaleBalances[msg.sender] = crowdsaleBalances[msg.sender].add(amountPaid);
  }
```

problem is with this line
```javascript
 amountPaid = msg.value.sub(exceedingBalance);
```
by setting amountPaid to msg.value - exceedingBalance, the previous lowering of the amountPaid to comply with maxTokensRaised is effectively forgotten
This allows not only breaking of both limits, but also leads to refunding the buyer with ETH that wasn't actually subtracted from the value of his purchase
```javascript
  if(differenceWei > 0)
     msg.sender.transfer(differenceWei);

  if(exceedingBalance > 0) {

     // Return the exceeding balance to the buyer
     msg.sender.transfer(exceedingBalance);
  }
```
One proper way of implementing the desired functionality would be to first calculate which of the two limits is lower and then checking value of the purchase only against that one.
