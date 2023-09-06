![Alien Codex](/assets/img/BigLevel19.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

The level reads:

> You've uncovered an Alien contract. Claim ownership to complete the level.
> 
> Things that might help
> 
> - Understanding how array storage works
> - Understanding [ABI specifications](https://solidity.readthedocs.io/en/v0.4.21/abi-spec.html)
> - Using a very `underhanded` approach

And gives us the following contract's code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function makeContact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
    codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

# Solution

As usual, we start by reading the contract, to win this level we need to take ownership of the contract. So, we need to find some way to overwrite the `owner`.

As in previous levels, we can inspect the storage like this:

```javascript
await web3.eth.getStorageAt(contract.address, 0)
'0x0000000000000000000000000bc04aa6aac163a6b3667636d798fa053d43bd11'
```

Also, if we call `await contract.owner()` we note that's the same value that's stored at the slot 0.

We continue exploring the contract. There is a modifier called `contacted` needed to call `record()`, `retract()` and `revise()`, then, we first need to set `contact` to true. Luckily for us, there is function that does that: `makeContact()`. 

Then, we call `contract.makeContact()` and now if we call `await contract.contact()` we will see that it returns true, allowing us to call the other functions of the contract.

Note what happens if we now look at the storage again:

```javascript
await web3.eth.getStorageAt(contract.address, 0)
'0x0000000000000000000000010bc04aa6aac163a6b3667636d798fa053d43bd11'
```

Why is that? Well, as we've already seen in the [Privacy](/2023-07-17-ethernaut-12-privacy-writeup/) challenge:

> Statically-sized variables (everything except mapping and dynamically-sized array types) are laid out contiguously in storage starting from position 0. Multiple items that need less than 32 bytes are packed into a single storage slot if possible.

That's why `owner` and `contact` are stored together at the slot 0.

Now that we can call `record()` let's see what happens:

- `contract.record("0x11223344")`

Should that be stored at the following slot?

```javascript
await web3.eth.getStorageAt(contract.address, 1)
'0x0000000000000000000000000000000000000000000000000000000000000001'
```

Well, as we can see, the answer is no. The variable `codex` is a dynamically-sized array of `bytes32`. And according to documentation:

> Due to their unpredictable size, mappings and dynamically-sized array types cannot be stored “in between” the state variables preceding and following them. Instead, they are considered to occupy only 32 bytes with regards to the rules above and the elements they contain are stored starting at a different storage slot that is computed using a Keccak-256 hash.
> 
> Assume the storage location of the mapping or array ends up being a slot p after applying the storage layout rules. For dynamic arrays, this slot stores the number of elements in the array (byte arrays and strings are an exception, see below). For mappings, the slot stays empty, but it is still needed to ensure that even if there are two mappings next to each other, their content ends up at different storage locations.
> 
> Array data is located starting at keccak256(p) and it is laid out in the same way as statically-sized array data would: One element after the other, potentially sharing storage slots if the elements are not longer than 16 bytes.

In other words, the slot 1 is going to store the length of `codex`, but its values are going to be stored at position `keccak256(1)`. Then to read the value that we've just stored by calling `record`, we would need to do something like this:

```javascript
await web3.eth.getStorageAt(contract.address, web3.utils.soliditySha3("1"))
'0x1122334400000000000000000000000000000000000000000000000000000000'
```

If we call record again, to get that value we need to call `await web3.eth.getStorageAt(contract.address, web3.utils.soliditySha3("1") + 1)`. To get the element in the position "2" of the array, we need to call `await web3.eth.getStorageAt(contract.address, web3.utils.soliditySha3("1") + 2)` and so on.

Now that we know what `record()` does, let's call `retract()`.

```javascript
await contract.retract()

await web3.eth.getStorageAt(contract.address, 1)
'0x0000000000000000000000000000000000000000000000000000000000000000'

await web3.eth.getStorageAt(contract.address, web3.utils.soliditySha3("1"))
'0x0000000000000000000000000000000000000000000000000000000000000000'
```

It deleted the element we stored when we called `record()`. Note that it's doing `codex.length--;`. We've seen something similar in previous challenges... What would happen if we call `retract()` again? Now that the array is empty? Well, let's try it:

```javascript
await contract.retract()

await web3.eth.getStorageAt(contract.address, 1)
'0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff'
```

It has underflowed the length of the array, making the contract think that the length of codex is $2^{256}$. Keep this in mind!

Now, let's continue reviewing the contract, what does `revise()` do? It let us write in any position of the `codex` array, pretty simple and straight forward right? We could write at any position of `codex` from 0 to $2^{256}-1$ (remember what we did in the last call of `retract()`). And also remember how the storage works for the dynamic arrays. If we call `revise(85, "0x11223344")` it will write "0x11223344" at position 85 of the array. And how was that position on the storage calculated? `keccak256(1)+85`, right? 

*Note that if we tried to write to an arbitrary position of the codex array before underflowing the length, the contract wouldn't have let us, as i > length.*

Here comes the interesting part... Would it be possible to write to a position of the codex array so that:

$keccack256(1) + X = 0x0$

Why 0x0? Well, what's stored at address 0x0 of the contract's storage? That's right! The owner!!! 

Let's do the maths:

$2^{256} = 0x10000000000000000000000000000000000000000000000000000000000000000$ 

$2^{256} - keccack256(1) = 0x4ef1d2ad89edf8c4d91132028e8195cdf30bb4b5053d4f8cd260341d4805f30a$

Or in decimal: 35707666377435648211887908874984608119992236509074197713628505308453184860938

Then let's write our address to that position of the codex array ;) 

`contract.revise("35707666377435648211887908874984608119992236509074197713628505308453184860938", web3.utils.padLeft(player, 64))`

Now if we call `await web3.eth.getStorageAt(contract.address, 0)` or `await contract.owner()` it will return our address!!! We have taken ownership of the contract, thus, winning the level!! :D

![Well done](/assets/img/ethernaut_solved.png)

# Remediation

- As a general advice, given that all data structures share the same storage (address) space, one should make sure that writes to one data structure cannot inadvertently overwrite entries of another data structure.
- Also, it is recommended to use vetted safe math libraries for arithmetic operations consistently throughout the smart contract system.

# References

- [SWC-101](https://swcregistry.io/docs/SWC-101)
- [SWC-124](https://swcregistry.io/docs/SWC-124)
- [Layout of State Variables in Storage](https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html)
- [How to locate value from Dynamic array in the Storage?](https://ethereum.stackexchange.com/questions/106526/how-to-locate-value-from-dynamic-array-in-the-storage)