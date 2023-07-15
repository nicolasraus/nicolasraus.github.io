![Elevator](/assets/img/BigLevel11.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

The level gives us the following information:

> This elevator won't let you reach the top of your building. Right?
>
>Things that might help:
> - Sometimes solidity is not good at keeping promises.
> - This Elevator expects to be used from a Building.

And gives the following contract's code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

# Solution

The instructions for the level say that the elevator won't let us reach the top of the building. So, in order to win, we will probably need to get to the top floor.

If we look at the contract we see that the elevator has two variables, which are `top` and `floor`. And their initial values are `false` and `0` respectively.

Then, we will need to get `top` to be `true` to indicate that we are actually on the top of the building (the last floor). So, how can we do that?

Another tip from the instructions is that *Elevator expects to be used from a Building*. But building isn't implemented in the contract, there is only an `interface` called `Building`.

> A Solidity contract interface is a list of function definitions without implementation. In other words, an interface is a description of all functions that an object must have for it to operate. The interface enforces a defined set of properties and functions on a contract.

Then, the contract is only telling us what inputs should `isLastFloor()` receive and what output is expected. 

Also, note in the definition of the `goTo()` function, that the first thing it does, is to set the address of the `msg.sender` as a `Building` contract and then call `isLastFloor()`.

Adding that together with the level's clue, we can infer that we need to create a Building contract that implements `isLastFloor()` and calls `Elevator`'s `goTo()` function. 

Now, before coding that, let's analyze what `goTo()` does. After setting the building variable, it calls `isLastFloor()` with the input `floor` (`uint`). If this function returns `true`, then nothing happens and the `top` variable isn't changed. So, we need `isLastFloor()` to return false... But later, `isLastFloor()` is called again with the same input and the output is stored in the `top` variable, which we need to be true to pass the level.

So, if it returns true, nothing happens, but if it returns false we cannot win the level, right? We would need a function that, when called the first time, returns false, but when called the second time, returns true. Let's code that:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./Elevator.sol";

contract Solver is Building {
    Elevator elevator;
    bool public top;

    constructor(address _elevatorAddress) {
        elevator = Elevator(_elevatorAddress);
        top = false;
    }

    function isLastFloor(uint) external override returns (bool) {
        if(!top) {
            top = true;
            return false;
        } else {
            return true;
        }
    }

    function solve(uint _floor) public {
        elevator.goTo(_floor);
    }
}
```

What this contract does, is to define a variable `top` with an initial value of `false`. Let's look at the implementation of `isLastFloor()` when this function is called the first time, top is going to be false, and so it's going to return false, and set top to true. The second time the function is called, top is going to be true, and then it will return true. What we need to pass the level!!!

Now, let's deploy the `Solver` contract with the address of the `Elevator` contract as input. And then call `solve()` with any floor number as input.

If we check now the value of Elevator's top variable, it should be true...

```javascript
await contract.top()
true
```

And, indeed, it's true!! We did it!! We won the level :D

![Well done](/assets/img/ethernaut_solved.png)

# Remediation

- If possible, use the `view` and `pure` modifiers to prevent functions from modifying the state. 

# References

- [Solidity docs: Contracts](https://docs.soliditylang.org/en/develop/contracts.html#view-functions)