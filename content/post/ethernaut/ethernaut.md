---
author: Nodauf
title: Ethernaut challenges
date: 2022-05-16
tags: ["foundry", "solidity", "ethernaut", "golang", "web3"]
image: /images/openzeppelin-logo.svg
categories: "ethernaut"
description: Ethernaut is a a website containg many vulnerable smart contracts where the goal is to find and exploit a particular vulnerability
---

I am starting a series of writeup of the [Ethernaut challenges](https://ethernaut.openzeppelin.com/), this project have been initiated by OpenZeppelin team to teach basic knowledge of smart contracts auditing.

The challenges will be resolved with **Foundry**, the client, if needed will be written in **Golang** and the smart contracts will be written in **Solidity**.

The current list of challenges are:

0. Hello Ethernaut
1. Fallback
2. Fallout
3. CoinFlip
4. Telephone
5. Token
6. Delegation
7. Force
8. Vault
9. King
10. Re-Entrancy
11. Elevator
12. Privacy
13. GatekeeperOne
14. GatekeeperTwo
15. NaughtCoin
16. Preservation
17. Recovery
18. Magic Number
19. AlienCodex
20. Denial
21. Shop
22. Dex
23. Dex Two
24. PuzzleWallet
25. Motorbike

The folder structure of each challenge is:

```text
ethernautWritup
├── about
│   ├── index.md
├── ethernautChallenges
│   ├── 0-HelloEthernaut
│   │   ├── 0-HelloEthernaut.go
│   ├──1-Fallback
│   |   └── 1-Fallback.go
│   ├──2-Fallout
│   |   └── 2-Fallout.go
│   ├──...
│
└── contracts
    ├── 0-HelloEthernaut.sol # If provided
    └── HelloEthernaut.abi
```

When a smart contract is needed to resolve the challenge, foundry will be used.
The environment can be created with

```bash
forge init ethernaut
```

It will look like this

```text
ethernaut
├── lib
│   ├── ...
├── src
│   ├── challengeName.sol # Source of the challenge
│   ├──...
│
├── test
│   ├── challengeName.t.sol # Contract to resolve the challenge
│   └── ...
│
└── foundry.toml
```

To convert easily the solidity code to go code the `solidity2go.sh` script will be used:

```bash
#!/bin/bash

if [[ $1 -eq "" ]]; then
echo "The name is required"
exit
fi

mkdir ../$1
./solc --abi $1.sol -o . --overwrite  @openzeppelin/=$(pwd)/node_modules/@openzeppelin/
pkgName=$(cat $1.sol| grep contract | cut -d" " -f 2)
./abigen --abi=./$pkgName.abi --pkg=$pkgName --out=../$1/$1.go
```

In the same folder, the node_modules should be installed

```bash
npm i @openzeppelin/contracts@3.4
```

For example for the challenge 1-Fallback, the script will be called like this:

```
./solidity2go.sh 1-Fallback
```

I'm going to learn a lot during this series, so the solutions written in Go or Solidity will probably get better and better as I go through the challenges.
