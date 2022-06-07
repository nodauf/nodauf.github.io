---
author: Nodauf
title: Ethernaut challenges - 0. Hello Ethernaut
date: 2022-05-18
tags: ["ethernaut", "golang", "web3"]
image: /images/openzeppelin-logo.svg
categories: "ethernaut"
description: Writeup of "0. Hello Ethernaut" challenge Ethernaut
---

# The environment

**Hello Ethernaut** is not really a challenge written to show a vulnerability, but a way to introduce the concept of interact with smart contracts.

On every languages, the client need the **ABI** of the contract to interact with it. The JS client has already been configured on the browser and we can ues it to retrieve the **ABI** of the contract.

In your development console, type `contract.abi` and you will see the ABI of the contract.

It can the be copied to the file **contracts/HelloEthernaut.abi** (see [the introduction post](../ethernaut) to know the structure of the writeups)

The ABI can be converted to a go file with **abigen** command. The generated files will contains fonction to interact with the contract.

```bash
abigen --abi=./HelloEthernaut.abi --pkg=HelloEthernaut --out=../0-HelloEthernaut/0-HelloEthernaut.go
```

# Exploitation

The instruction for the challenge is to run `await contract.info()`. We will do the same in Go.

The first step is to create the client and load th newly generated file get from the ABI.

```go
package main

import (
	"log"
	"flag"
	"github.com/ethereum/go-ethereum/ethclient"
	store "ethernaut-golang/ethernautChallenges/0-HelloEthernaut"
)

func main() {
	var blockchainURL, contractAddress, privateKey string
	fork := flag.Bool("fork", true, "Should we use the parameter for the fork blockchain ? (default: true)")
	flag.Parse()
	contractAddress = "0x80513274a0FA9abF1a458bF6BEed397fd0dBcef3"
	if *fork {
		blockchainURL = "http://127.0.0.1:8545"
		privateKey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
	} else {
		blockchainURL = "https://eth-rinkeby.alchemyapi.io/v2/<API-KEY>"
		privateKey = "<private key>"
	}
	client, err := ethclient.Dial(blockchainURL)
	if err != nil {
		log.Fatal(err)
	}

	// Contract
	contractAddressHash := common.HexToAddress(contractAddress)
	instance, err := store.NewHelloEthernaut(contractAddressHash, client)
	if err != nil {
		log.Fatal(err)
	}
```

# Full code

```go
package main

import (
	"context"
	"crypto/ecdsa"
	"fmt"
	"log"
	"math/big"

	store "ethernaut-golang/ethernautChallenges/0-HelloEthernaut"

	"flag"

	"github.com/ethereum/go-ethereum/accounts/abi/bind"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/crypto"
	"github.com/ethereum/go-ethereum/ethclient"
)

func main() {
	var blockchainURL, contractAddress, privateKey string
	fork := flag.Bool("fork", true, "Should we use the parameter for the fork blockchain ? (default: true)")
	flag.Parse()
	contractAddress = "0x80513274a0FA9abF1a458bF6BEed397fd0dBcef3"
	if *fork {
		blockchainURL = "http://127.0.0.1:8545"
		privateKey = "ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
	} else {
		blockchainURL = "https://eth-rinkeby.alchemyapi.io/v2/<API-KEY>"
		privateKey = "<private key>"

	}
	client, err := ethclient.Dial(blockchainURL)
	if err != nil {
		log.Fatal(err)
	}

	// Contract
	contractAddressHex := common.HexToAddress(contractAddress)
	instance, err := store.NewHelloEthernaut(contractAddressHex, client)
	if err != nil {
		log.Fatal(err)
	}
	infoOutput, _ := instance.Info(nil)
	fmt.Println("info:", infoOutput)

	info1Output, _ := instance.Info1(nil)
	fmt.Println("info1:", info1Output)

	info2Output, _ := instance.Info2(nil, "hello")
	fmt.Println("info2:", info2Output)

	infoNumOutput, _ := instance.InfoNum(nil)
	fmt.Println("infoNum:", infoNumOutput)

	info42Output, _ := instance.Info42(nil)
	fmt.Println("info42:", info42Output)

	theMethodNameOutput, _ := instance.TheMethodName(nil)
	fmt.Println("theMethodName:", theMethodNameOutput)

	method7123949Output, _ := instance.Method7123949(nil)
	fmt.Println("method7123949:", method7123949Output)

	passwordOutput, _ := instance.Password(nil)
	fmt.Println("password field is:", passwordOutput)

	// For a write transaction on the smart contract we need to be authenticated
	auth := newTransactor(client, privateKey)
	instance.Authenticate(auth, "ethernaut0")

}

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
	fmt.Println(fromAddress)
	auth := bind.NewKeyedTransactor(privateKey)
	auth.Nonce = big.NewInt(int64(nonce))
	auth.Value = big.NewInt(0)      // in wei
	auth.GasLimit = uint64(2000000) // in units
	auth.GasPrice = gasPrice
	return auth

}
```
