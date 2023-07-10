![Fallback](/assets/img/BigLevel1.svg)

- [Challenge](#challenge)
- [Solution](#solution)
   
# Challenge
This is the first challenge of the ethernaut platform (after the tutorial). It states that the conditions to win it are:
1. you claim ownership of the contract
2. you reduce its balance to 0

And then it presents us with the code of the contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {

  mapping(address => uint) public contributions;
  address public owner;

  constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

# Solution
I clicked on the "Get new instance" button to get the new instance of the contract and then I started to interact with the contract by also reading to the code, in order to understand what its functions do.

To win the challenge we need to take the ownership of the contract and withdraw its balance. Looking at the code we can see that there is a `withdraw` function, this function can only be called by the owner and it transfers the contract's balance to the owner's address. 
So, we cannot call that function because we're not the onwers of the contract.

```
  function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
  }
```

But this shouldn't be a problem because the other goal of the level is the claim the ownership of the contract. So once we achieve that. We should be able to call the withdraw fucntion.

If we continue looking at the code, we note that the variable owner is modified only one time (other than the constructor). And it's at the `receive` function.

```
  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
```

Receive is function that gets executed everytime the contracts receives ETH without data. The receive function from this contract sets the sender's address as the owner. Just what we want! But it only does that if the value is greater than 0 and the contributions made by the sender are also greater than 0.

So it just seems that we need to send the contract some ETH after sending some contribution. 
There is a function called contribute which requires that we send some amount less than 0.001 ETH and then it updates the contributions mapping. So if we call contribute sending 1 wei, it should update the contributions mapping so we can the call the receive fucntion to take the ownership of the contract!

So, let's do that:
- first we need to call contribute and send 1 wei
  ```
  await contract.contribute({value: 1})
  ```
- We can verify that if we call getContribution now, it won't be empty
  ```
  (await contract.getContribution()).toString()
  '1'
  ```
- Now we need to get the receive function executed, for that we just need to send some ETH to the contract, to transfer 1 wei to the contract we should call
  ```
  await contract.send("1")
  ```
- At this point we should already be the owners, we can verify it by calling `await contract.owner()` and voilà! that's our address!!
- We just only have to transfer the balance to our address, to do that we just need to call withdraw
  ```
  await contract.withdraw()
  ```

Then we submit it by clicking "Submit instance" and...

WE DID IT!!! 

![Well done](/assets/img/ethernaut_solved.png)