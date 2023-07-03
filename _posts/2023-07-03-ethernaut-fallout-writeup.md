![FAL1OUT](/assets/img/BigLevel2.svg)
- [Challenge](#challenge)
- [Solution](#solution)
- [References](#references)
   
# Challenge

This challenge gives us the code of the contract and says that we should take ownership of it to pass the level.
The code is:

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

# Solution

This challenge is pretty simple and straight forward. You just need to note that the constructor of the contract is misspelled, it says `Fal1out` instead of `Fallout` (the contract's name). 
Notice that the constructor changes the owner of the contract, what we need... But we shouldn't be able to call the constructor, right??!

Well, 
> before Solidity version `0.4.22`, the only way of defining a constructor was to create a function with the same name as the contract class containing it. A function meant to become a constructor becomes a normal, callable function if its name doesn't exactly match the contract name.

And that's what's happening here, the constructor function's name is slightly different from the contract's name, becoming a publicly callable function... Then we would only need to call it to take ownership of the contract:
`contract.Fal1out({value: 1})`

Note that now if we call `await contract.owner()` it returns our own address!!! We did it!

![Well done](/assets/img/ethernaut_solved.png)

To prevent these problems, version `0.4.22` introduced the `constructor` keyword to specify the constructor function of the contract. Using the same name of the contract as the constructor is now deprecated and shouldn't be used anymore. 

# References
- [SWC-118](https://swcregistry.io/docs/SWC-118)