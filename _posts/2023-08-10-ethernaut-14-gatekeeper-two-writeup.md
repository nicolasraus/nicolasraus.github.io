![Gatekeeper One](/assets/img/BigLevel14.svg)

- [Challenge](#challenge)
- [Solution](#solution)
  - [First Gate](#first-gate)
  - [Second Gate](#second-gate)
  - [Third Gate](#third-gate)
- [References](#references)
   
# Challenge

The level reads:

> Things that might help:
> - Remember what you've learned from getting past the first gatekeeper - the first gate is the same.
> - The `assembly` keyword in the second gate allows a contract to access functionality that is not native to vanilla Solidity. See [here](http://solidity.readthedocs.io/en/v0.4.23/assembly.html) for more information. The `extcodesize` call in this gate will get the size of a contract's code at a given address - you can learn more about how and when this is set in section 7 of the [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf).
> - The `^` character in the third gate is a bitwise operation (XOR), and is used here to apply another common bitwise operation (see [here](http://solidity.readthedocs.io/en/v0.4.23/miscellaneous.html#cheatsheet)). The Coin Flip level is also a good place to start when approaching this challenge.

And gives us the following code:

```solidity
// SPDX-License-Identifier: MITpragma solidity ^0.8.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

# Solution

## First Gate

The level states that the first gate is the same as in the previous level, which requires that `msg.sender != tx.origin`, which means that the one calling `enter()` should be a contract and not us directly interacting with it.

## Second Gate

For the second gate we see that the contract requires that:

```solidity
assembly { x := extcodesize(caller()) }
require(x == 0);
```

According to the tips of the level, `extcodesize` is an assembly instruction that returns the size of the contract deployed at the input address, in this case `caller()` which is an assembly function that is equivalent to `msg.sender`, meaning our Solver contract address. The returned value should be 0 to pass this gate. But... how can the size of our code be zero?

If we look into the given [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf) we can find this note:

> Note that while the initialisation code is executing, the newly created address exists but with no intrinsic body code.

This implies that for it to return 0, we should call it from the `constructor` of our Solver contract. That's the only moment in which the size of our code will be "0", before it's fully deployed.

## Third Gate

And last, for the third gate we have this requirement:

`uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max`

Let's analyze what's the value of each part of this statement. We can add some logging to the `GatekeeperTwo` contract to print these values:

```solidity
  modifier gateThree(bytes8 _gateKey) {
    uint64 a = uint64(bytes8(keccak256(abi.encodePacked(msg.sender))));
    uint64 b = uint64(_gateKey);
    uint64 c = type(uint64).max;
    console.log("uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))): ", a);
    console.log("uint64(_gateKey): ", b);
    console.log("type(uint64).max: ", c);
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }
```

This will print something like this:

```
uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))): 5628578533785254384
uint64(_gateKey): 1229782938247303441
type(uint64).max: 18446744073709551615
```

For the first part we have the `msg.sender` (which is the Solver’s contract address) hashed and converted to uint64 (8 bytes unsigned integer).

The second part is the input of the `enter()` function converted to uint64.

And lastly we have the maximum value for `uint64` which is <img src="https://render.githubusercontent.com/render/math?math=2^{64}-1"> or `0xffffffffffffffff`. 

This gate takes the Solver’s address, XORs it with our input (`_gateKey`) and checks if all of its bits are 1. 

In our solver, we won't be able to manually craft the `_gateKey` as input ourselves (as in the previous level), because we will not know the `Solver` address until it’s deployed… So, we will need to calculate it in the constructor. Then we will need to get the ones’ complement of the solver address so when it gets XORed with the address, it will give all ones (which is the requirement of the gate three). Let's see an example:

Having <img src="https://render.githubusercontent.com/render/math?math=A = 00001111_2"> and <img src="https://render.githubusercontent.com/render/math?math=B = \neg A = 11110000_2">

Then <img src="https://render.githubusercontent.com/render/math?math=A \oplus B = 111111111_2">

To calculate the ones' complement of our contract's address we will need to do a bitwise negation. There isn’t a function in solidity to do this, but we can use the XOR to get it!

To get the one’s complement or bitwise negation we can XOR a variable with `0xFFFF`. E.g.:

<img src="https://render.githubusercontent.com/render/math?math=A \oplus \texttt{0xFFFF} = 11110000_2 = B">

Thus <img src="https://render.githubusercontent.com/render/math?math=A \oplus \neg A = A \oplus B = 11111111_2 = \texttt{0xFFFF}">

Which is what we need to solve the challenge! Now let’s construct our solver:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./GatekeeperTwo.sol";

contract Solver {
    GatekeeperTwo gatekeeperTwo;

    constructor(address _gatekeeperTwoAddress) {
        uint64 gateKey;
        gatekeeperTwo = GatekeeperTwo(_gatekeeperTwoAddress);
        gateKey = uint64(bytes8(keccak256(abi.encodePacked(address(this)))));
        gateKey = gateKey ^ 0xFFFFFFFFFFFFFFFF;
        gatekeeperTwo.enter(bytes8(gateKey));
    }
}
```

After deploying it, sending `GatekeeperTwo`'s address as input, the challenge should be solved, as it calls `enter()` in the `constructor`. We can check this by calling `await contract.entrant()` and see that it retrieves our own address! Which means that we passed the level :D!!!!

![Well done](/assets/img/ethernaut_solved.png)


# References

- [ETHEREUM: A SECURE DECENTRALISED GENERALISED TRANSACTION LEDGER](https://ethereum.github.io/yellowpaper/paper.pdf)