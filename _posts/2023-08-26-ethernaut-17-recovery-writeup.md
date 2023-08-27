![Recovery](/assets/img/BigLevel17.svg)

- [Challenge](#challenge)
- [Solution](#solution)
  - [Etherscan](#etherscan)
  - [Calculating the address](#calculating-the-address)
- [References](#references)
   
# Challenge

The level reads:

> A contract creator has built a very simple token factory contract. Anyone can create new tokens with ease. After deploying the first token contract, the creator sent `0.001` ether to obtain more tokens. They have since lost the contract address.
> 
> This level will be completed if you can recover (or remove) the 0.001 ether from the lost contract address.

And gives us the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value * 10;
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender] - _amount;
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```


# Solution

To win this level we need to recover the lost address of the `SimpleToken` contract that was created using the `Recovery` contract. The recovery contract is the one we are interacting with when we call `contract` from the browser's console. There are two ways to recover this address. One is by exploring the chain with etherscan. And the other one is by calculating the address using the address of the creator and the nonce used for creating the new contract.

Let's explore each way.

## Etherscan

For this method we first need to call `contract.address` to get the address of the `Recovery` contract, which is the one creating `SimpleToken` (the one we want to recover).

We then go to [sepolia.etherscan.io](https://sepolia.etherscan.io/) (or the testnet you are using for the challenges) and paste the address we got in the previous step. Then click on `Internal Transactions`.

> An Internal Transaction refers to a transfer of ETH that is carried out through a smart contract as an intermediary. When viewing an address on Etherscan, this type of transaction will be shown under the Internal Txns tab.

As there is a contract (`Recovery`) creating another contract (`SimpleToken`) we should be looking at this tab. This should look something similar to this:

![SimpleToken's contract creation](/assets/img/17_recovery_contract-creation.png)

Those are the internal transactions of the `Recovery` contract the bottom one being the creation of it. And the second one the creation of the `SimpleToken` contract. If we look at the "From" column, we can see that's the address of the `Recovery` contract.

Now let's click on that `ContractCreation` and explore the new contract (which should be the `SimpleToken` contract)

![SimpleToken internal transactions](/assets/img/17_recovery_simpletoken-intx.png)

If we look at the balance of that contract we see that it has 0.001 ETH which is what we need to recover.

Now to recover those funds, we should be calling the `destroy()` function with our address as input. For that we can use [remix](https://remix.ethereum.org/). We can copy the code of the `SimpleToken` contract and then after compiling it, use the "Load contract ad address" by inputting the address we got from the previous step. 

Lastly call `destroy()` with our address as input.

![Load contract at address in remix](/assets/img/17_recovery_remix-load-contract.png)

That should've transfered the funds from the `SimpleToken` contract into our account. We can verify that by calling `await getBalance(contract.address)` and see that is now 0 ETH (we can verify that in etherscan as well). And that's it! We solved the challenge :) we can now submit the level!

![Well done](/assets/img/ethernaut_solved.png)

## Calculating the address

There exist another way to get the lost address. For that, we first need to understand how the addresses are pre-calculated when a new contract is created. According to the Ethereum yellow paper:

> The address of the new account is defined as being the
rightmost 160 bits of the Keccak-256 hash of the RLP
encoding of the structure containing only the sender and
the account nonce.

So how can we calculate this address from the address of the creator and the nonce. I've used the code from [this reply](https://ethereum.stackexchange.com/a/47083) to get the address.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;


contract AddressByNonceCalculator {
    function computeContractAddress(address _origin, uint _nonce) external pure returns (address _address) {
        bytes memory data;
        if(_nonce == 0x00)          data = abi.encodePacked(bytes1(0xd6), bytes1(0x94), _origin, bytes1(0x80));
        else if(_nonce <= 0x7f)     data = abi.encodePacked(bytes1(0xd6), bytes1(0x94), _origin, uint8(_nonce));
        else if(_nonce <= 0xff)     data = abi.encodePacked(bytes1(0xd7), bytes1(0x94), _origin, bytes1(0x81), uint8(_nonce));
        else if(_nonce <= 0xffff)   data = abi.encodePacked(bytes1(0xd8), bytes1(0x94), _origin, bytes1(0x82), uint16(_nonce));
        else if(_nonce <= 0xffffff) data = abi.encodePacked(bytes1(0xd9), bytes1(0x94), _origin, bytes1(0x83), uint24(_nonce));
        else                        data = abi.encodePacked(bytes1(0xda), bytes1(0x94), _origin, bytes1(0x84), uint32(_nonce));
        bytes32 hash = keccak256(data);
        assembly {
            mstore(0, hash)
            _address := mload(0)
        }
    }
}
```

Now we need to deploy that contract and call `computeContractAddress(RECOVERY_ADDRESS, 0x01)` this will give us the address of the `SimpleToken` contract (the one that contains the ETH we need to recover). Note that the nonce is 0x01, that's because it's the first transaction that's being sent from the `Recovery` contract. Note that the nonce 0x00 is the one used for the creation of the contract itself.

With the returned address we now need to follow the steps from the other method, which would be loading the `SimpleToken` from the address obtained and calling `destroy()` with our address as input.

# References

- [RECURSIVE-LENGTH PREFIX (RLP) SERIALIZATION](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/)
- [How is the address of an Ethereum contract computed?](https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed)
- [Understanding an Ethereum Transaction](https://info.etherscan.com/understanding-an-ethereum-transaction/)
- [Ethereum Yellow Paper: a formal specification of Ethereum, a programmable blockchain](https://ethereum.github.io/yellowpaper/paper.pdf)