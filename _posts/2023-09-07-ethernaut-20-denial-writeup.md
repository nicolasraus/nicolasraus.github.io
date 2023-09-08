![Denial](/assets/img/BigLevel20.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

This level gives us the following information:

> This is a simple wallet that drips funds over time. You can withdraw the funds slowly by becoming a withdrawing partner.
> 
> If you can deny the owner from withdrawing funds when they call withdraw() (whilst the contract still has funds, and the transaction is of 1M gas or less) you will win this level.

And gives us the following contract's code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Denial {

    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] +=  amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

# Solution

Let’s start by reading and interacting with the contract to understand what it does and how.

- `receive()`: it allows deposit of funds.
- `contractBalance()`: it returns the balance of the contract.
- `setWithdrawPartner(address)`: this functions updates de variable `partner` with the input variable.
- `withdraw()`: first it calculates the amount to be withdrawn, which is a 1% of the balance of the contract and then transfers that amount first to the partner and second to the owner.

Note that the contract first transfers to the partner, which is going to be us (or our solver contract) and then to the owner. It would be possible for us to set a contract as partner which has a fallback function. That fallback function is going to be executed when the `Denial` contract executes `partner.call{value:amountToSend}("");`. Something similar to what we’ve already seen in the [Reentrancy level](/2023-07-12-ethernaut-10-reentrancy-writeup/).

To win this level, we need to deny the owner from withdrawing funds. To do this, we need to then implement a contract that has a `receive()` function. When the owner of the `Denial` contract tries to withdraw funds, the contract is going to transfer funds to the partner (which needs to be the address of our `Solver` contract). At this point, the `receive()` function from our contract is going to get executed. We need then to do something that stops the caller from continuing executing, right?

How can we do this? We could just revert right? Well, note that `Denial` is using the low-level function `call()` to transfer funds to the partner. What does this mean? 

> When a call reverts within the EVM, it doesn't revert the entire transaction. It just pushes a true/false success value onto the stack for the context calling it to decide what to do.
> 
> Solidity is meant to add safety on top of using the EVM directly, so when you call a function properly (in your example, using the interface), it validates that success value, and if it's false, it will automatically revert. When you do a low level .call in Solidity, the compiler doesn't do that automatically for you. Instead it just gives you the value of success for you to decide what to do with it.

Also note that `revert()` returns all remaining gas to the caller (in opposition to `throw()` which consumed all the remaining gas), then, if we revert, the `Denial` contract will continue its execution, transferring funds to the owner, which is what we need to avoid.

So, we need to find another way to stop the `Denial` contract from keep running after it transfers to our `Solver` contract. Another way could be to consume all the remaining gas, then the caller won't have any gas left to transfer funds to the owner. Which is what we need!!

Let's solve it then:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;


contract Solver {
    uint i;
    
    receive() external payable {
        while(true) {
            i++;
        }
    }
}
```

This is a pretty simple contract that just burns all gas when it receives ETH. 

After deploying it, we need to set the address of our `Solver` as `partner`:

```javascript
contract.setWithdrawPartner(solverAddress)
```

Now we just need to submit the level for revision, and it's done :)

![Well done](/assets/img/ethernaut_solved.png)

# Remediation

Always assume that external calls can fail or use up all the gas.

Other than using a `checks-effects-intractions` pattern to avoid reentrancy problems, one should limit the gas for external calls to avoid denial of services attacks like this one.

Also, always validate the returned status of the low-level function `call` for errors.


# References

- [Why an external call using an interface reverts if an error occurs in the called function, but why it doesn't revert if it was made using the .call()?](https://ethereum.stackexchange.com/a/144230)
- [Security Considerations](https://docs.soliditylang.org/en/latest/security-considerations.html#use-the-checks-effects-interactions-pattern)
- [SWC-113](https://swcregistry.io/docs/SWC-113/)