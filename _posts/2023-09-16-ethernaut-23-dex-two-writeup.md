![DexTwo](/assets/img/BigLevel23.svg)

- [Challenge](#challenge)
- [Solution](#solution)
   
# Challenge

The level states:

> This level will ask you to break DexTwo, a subtlely modified Dex contract from the previous level, in a different way.
> 
> You need to drain all balances of token1 and token2 from the DexTwo contract to succeed in this level.
> 
> You will still start with 10 tokens of token1 and 10 of token2. The DEX contract still starts with 100 of each token.
> 
> Things that might help:
> 
> - How has the swap method been modified?

And gives us the following contract's code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract DexTwo is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  function add_liquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  } 

  function getSwapAmount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
    SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableTokenTwo is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

# Solution

This challenge is pretty similar to the previous one [Dex](/2023-09-12-ethernaut-22-dex-writeup/). The level also hints that the modification was on the `swap()` function. If we compare both of them we will see that only one lines was changed (actually removed), which is the first line from the swap function of the `Dex` contract:

```solidity
require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
```

This line ensures that the callers are only exchanging between token1 and token2. What does it mean that this line isn't present anymore? Well, some malicious actor could call this function with a third token (let's call it token3) as the function takes the address of the tokens to be exchanged from the parameters.

Also, remember from the previous level that if we, at some point, have the same amount of tokens (or more) than the exchange, we could get all the other tokens. This happened in the last challenge when we did:

```javascript
contract.swap(token2, token1, 45)
```

That was when we had 65 token2 and the exchange only had 45 and 110 token1, then by exchanging 45 token2 we would get all the token1.

So, what can we do to exploit this situation in our favor?

It would be possible to create a third token with some amount of tokens, then we could transfer some of them to the exchange's liquidity pool. Now we can swap our token3 for token1, how much? We need to swap the same amount of token3 that the exchange have, so if we send 1 TKN3 to the exchange and then call `swap(token3, token1, 1)` we would be getting the 100 token1 that the exchange have. After doing that, the `DexTwo` contract would have 0 token1, 100 token2 and 2 token3 (1 that we sent at the beginning and 1 that we swapped). Now to get all of token2 we need to again exchange the same amount of token3 that the contract has to get all token2, so we call `swap(token3, token2, 2)`.

Let's solve it then!

1. We start by creating a third token, we can deploy a `SwappableToken` in [remix](https://remix.ethereum.org/) with the following input: `dexAddress, token3, TKN3, 4`.
2. Then we need to transfer 1 `TKN3` to the `DexTwo` contract: `token3.transfer(dexTwoAddress, 1)`.
3. Now, before exchanging the tokens, we need to approve `DexTwo` to transfer `token3` in our behalf (in order for swap function to successfully execute): `token3.approve(dexTwoAddress, 3)`.
4. And, lastly, we do the actual `swaps`: `dex.swap(token3Address, token1, 1)` and `dex.swap(token3Address, token2, 2)`

After doing this, we should have all token1 and token2, right? Let's verify it:


```javascript
await contract.balanceOf(token1, player).then(v=>parseInt(v))
110
await contract.balanceOf(token2, player).then(v=>parseInt(v))
110
await contract.balanceOf(token1, contract.address).then(v=>parseInt(v))
0
await contract.balanceOf(token2, contract.address).then(v=>parseInt(v))
0
```

And that's it! We've drained all balances of token1 and token2 :)

![Well done](/assets/img/ethernaut_solved.png)