![MagicNumber](/assets/img/BigLevel18.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [References](#references)
   
# Challenge

The level states:

> To solve this level, you only need to provide the Ethernaut with a Solver, a contract that responds to whatIsTheMeaningOfLife() with the right number.
> 
> Easy right? Well... there's a catch.
> 
> The solver's code needs to be really tiny. Really reaaaaaallly tiny. Like freakin' really really itty-bitty tiny: 10 opcodes at most.
> 
> _Hint: Perhaps its time to leave the comfort of the Solidity compiler momentarily, and build this one by hand O_o. That's right: Raw EVM bytecode._
> 
> Good luck!

And gives us the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {

  address public solver;

  constructor() {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```


# Solution

As the level stated, we need to create a contract which, when called by the `MagicNum` contract, replies with 42. But this contract has to be really, really tiny. According to the hint, we need to code it using EVM bytecode directly.

So, how can we do this? According to the Ethereum yellow paper, to create a contract we need to make a transaction to address zero (0x0000000000000000000000000000000000000000) and it should contain:

> **init**: An unlimited size byte array specifying the EVM-code for the account initialisation procedure, formally Ti.
> 
> **init** is an EVM-code fragment; it returns the body, a second fragment of code that executes each time the account receives a message call (either through a transaction or due to the internal execution of code). **init** is executed only once at account creation and gets discarded immediately thereafter.

Then, we need to craft some bytecode that returns the code of our new contract (`Solver`) which needs to be bytecode that returns always the number 42. And how do we do this? 

I've used [this blog](https://hackmd.io/@e18r/r1yM3rCCd) as guide to craft the following bytecode:

```solidity
60 0a // copy 10 bytes
60 0c // which are in code position 0c
60 00 // to memory position 0
39    // do it
60 0a // return 10 bytes
60 00 // which are in memory position 0
f3    // do it

// put 42 in memory position '00'
60 2a // push 42 into the stack
60 00 // push the address where we want to store it
52    // actually store 42 in position 00

// tell f3 the size and position of what we want to return
60 20 // 32 bytes to be returned
60 00 // from memory position 00
f3    // return it
```

The first 7 lines of code are the initialization routine. Is the code that's going to return the code of our `Solver` contract. Which are the last 6 lines of code. 

Those last 6 lines load 42 (0x2a) into position 0 and then returns the 32-byte word stored at position 0 (which is going to be 0x000000000000000000000000000000000000000000000000000000000000002a). Note that we need to return 32 bytes because we're storing it into an uint256:

| uint8 | Mnemonic | Stack Input | Stack | Output | Expression | Notes |
|---|---|---|---|---|---|---|
| 52 | MSTORE | offset | value | - | memory[offset:offset+32] = value | writes a (u)int256 to memory |
| 53 | MSTORE8 | offset | value | - | memory[offset] = value & 0xFF | writes a uint8 to memory |

_I tried solving this challenge using opcode 53 MSTORE8, by doing that we would only need to retrieve 1 byte, but it didn't work. Probably the level is expecting 32 bytes._

Now that we have our bytecode `600a600c600039600a6000f3602a60005260206000f3`. We need to deploy it into the testnet. As stated before, we just need to send it to the address zero (or null address). We can do that with web3.js:

```javascript
bytecode = '600a600c600039600a6000f3602a60005260206000f3'
txn = await web3.eth.sendTransaction({from: player, data: bytecode})
solver = txn.contractAddress
await contract.setSolver(solver)
```

If we want to verify that our code actually returns 42, we could run `await web3.eth.call({to: solver}).then(v=>parseInt(v,16))` from the console, and it should return 42!

And that would be it!! We just need to Submit the level for verification :) 

![Well done](/assets/img/ethernaut_solved.png)d

Note that we are storing the number to be retrieved at position 0, according to documentation we shouldn't be doing that because that space is reserved:

> As a dynamically sized byte array, the Memory area of the EVM call context can be read and written to in discrete 32-byte chunks. There is also the concept of “touched” and “untouched” memory, where newly written chunks of memory accrue increasing gas and storage costs. Before new memory can be utilized, the EVM needs to retain a section of utility memory that is used to keep track of subsequent memory writes.
> 
> This utility memory is comprised of four main sections, starting from the beginning of the Memory array:
> 
> - bytes `0x00-0x3f`: scratch space mainly used to store intermediate outputs from hashing operations such as `keccak256()`. Note that this section is allocated as two 32-byte slots.
> - bytes `0x40-0x5f`: space reserved for the 32-byte “free memory pointer.” This is used as a pointer to unallocated memory for use during contract execution.
> - bytes `0x60-0x7f`: the 32-byte “zero slot,” which is used as a volatile storage space for dynamic memory arrays which should not be written to.
> 
> Initially, the free memory pointer points to position `0x80` in memory, after which point additional memory can be assigned without overwriting memory that has already been allocated to other data. As such, the free memory pointer also functions as the currently allocated memory size.

But our contract isn't going to do anything other than returning 42, then there is no problem.

# References

- [Ethereum Yellow Paper: a formal specification of Ethereum, a programmable blockchain](https://ethereum.github.io/yellowpaper/paper.pdf)
- [EVM bytecode programming](https://hackmd.io/@e18r/r1yM3rCCd)
- [Ethereum Virtual Machine Internals – Part 1](https://www.netspi.com/blog/technical/blockchain-penetration-testing/ethereum-virtual-machine-internals-part-1)
- [Ethereum Virtual Machine Opcodes](https://www.ethervm.io/)