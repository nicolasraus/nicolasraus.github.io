![King](/assets/img/BigLevel9.svg)

- [Challenge](#challenge)
- [Solution](#solution)
   
# Challenge

The level reads:

> The contract below represents a very simple game: whoever sends it an amount of ether that is larger than the current prize becomes the new king. On such an event, the overthrown king gets paid the new prize, making a bit of ether in the process! As ponzi as it gets xD
>
> Such a fun game. Your goal is to break it.
>
> When you submit the instance back to the level, the level is going to reclaim kingship. You will beat the level if you can avoid such a self proclamation.

And this is the code of the contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {

  address king;
  uint public prize;
  address public owner;

  constructor() payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    payable(king).transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address) {
    return king;
  }
}
```

# Solution

As usual, we start by reading the contract's code, in order to understand what it does. In this case, when the `King` contract receives a value greater than the prize, it sends the `msg.value` to the previous king and updates the `prize` and `king` variables.

Note that the level states that when we submit the instance, the level is going to try to reclaim the kingship. Presumably by sending some ETH.

So, how could we make the contract not to update the king? If we look at the `receive()` function, we see that the only time that the contract interacts with an external address is when it calls `payable(king).transfer(msg.value)`. Would it be possible for us to make that transaction not to work? 

Well, that wouldn't be possible for us directly interacting with the contract, as the king would be our wallet's address, and there is no way for us to not accept that funds' transfer. But, what would happen if the "king" is another contract (created by us)?

We could create a contract that sends some ETH to the `King` contract in order to claim the kingship of the contract. Then, if we submit the instance, the level is going to try to reclaim it. At that point the contract is going to send ETH (sent by the reclaimer) to the previous king, and that would be our contract. At this point we could define a `receive()` function that just reverts the transaction. Generating an unhandled exception on the `King`'s contract. Thus, winning the level :) 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solver {

    receive() external payable {
        revert();
    }

    function solve(address payable _kingAddress) public payable {
        (bool sent, ) = _kingAddress.call{value: msg.value}("");
        require(sent, "Failed to send Ether");
    }
}
```

We just need to deploy that contract and call `solve()` with the King contract's address as input, and a value greater than the actual prize.
The initial value of `prize` is:

```javascript
await contract.prize().then(v => v.toString())
'1000000000000000'
```

Now we can execute solve() with a value of, let's say, 1000000000000001 and the King's contract address as input to make our `Solver` contract the new "king". At this point, the King game is broken, and the king variable is never going to be updated again.

We only need to submit the instance, and we won the level!!

![Well done](/assets/img/ethernaut_solved.png)
