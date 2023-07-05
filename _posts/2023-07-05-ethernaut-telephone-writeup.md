![Telephone](/assets/img/BigLevel4.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

This challenge gives us the code of the contract and says that we have to take ownership of the contract to pass the level.

The code given is:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {

  address public owner;

  constructor() {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

# Solution

In this level, there is only one callable function which is also the one that modifies the owner of the contract. Then it is clear that we would need to use that function to take ownership of the level.

This function takes one address as argument and modifies the ownership of the contract to be that address, if and only if `tx.origin` and `msg.sender` are different... If we interact with the contract using the Javascript console we can try to send our address as argument and call the `changeOwner` function. Note that if we call `await contract.changeOwner(OUR-ADDRESS)` the owner of the contract doesn't change. And that's expected, as `tx.origin` and `msg.sender` will be the same...

So how come can we get those values to be different? Well, let's see what each of these values are:

> `tx.origin` is a global variable in Solidity which returns the address of the account that sent the transaction.

> `msg.sender` is always the address where the current (external) function call came from.

These seem pretty similar right? Let put and example to understand it. Suppose that I have an address `0x1` and call a function from a contract A which have the address `0x2` and this contract interacts with another contract at the address `0x3`. The call chain would be something like this:
- Nico (0x1) -> A (0x2) -> B (0x3)

At the contract `A` both `tx.origin` and `msg.sender` are going to be `0x1`. But what is the contract B going to see? Well for contract `B` these variables are going to be `tx.origin == 0x1` and `msg.sender == 0x2`.

Which is exactly what we want, to have a different `tx.origin` and `msg.sender`. Then, we just need to construct a contract `B` which will interact with the `Telephone` contract by sending our address as the input of `changeOwner()`. So let's do that:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "hardhat/console.sol";
import "./Telephone.sol";

contract Solver {
    Telephone telephone;

    constructor(address telephoneAddress) {
        telephone = Telephone(telephoneAddress);
    }

    function solve(address my_address) public {
        telephone.changeOwner(my_address);
    }
}
```

Now, we only need to deploy this contract sending the address of the `Telephone` contract as the input for the constructor and then call `solve()` with our address as input.

If we now go back to the level page and check for `await contract.owner()` we will see that it responds with our own address!! And that's all, we did it!! :D We just need to submit instance now.

![Well done](/assets/img/ethernaut_solved.png)

# Remediation

It is important to never use the `tx.origin` as an authentication mechanism as this could be abused if an authorized user interacts with a malicious contract. As `tx.origin` will be the address of the authorized user, but actually the one interacting with the contract is a malicious contract that will be able to impersonate the valid user.

# References

- [SWC-115](https://swcregistry.io/docs/SWC-115)
- [Smart Contract Wallets created in frontier are vulnerable to phishing attacks](https://blog.ethereum.org/2016/06/24/security-alert-smart-contract-wallets-created-in-frontier-are-vulnerable-to-phishing-attacks)
