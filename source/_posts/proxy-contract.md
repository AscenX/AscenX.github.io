---
title: 透明代理合约
date: 2024-03-24 01:35:13
tags:
- Solidity
- 区块链
categories:
- 学习笔记
---

最近在BSC上调用ERC20合约获取余额的时候发现获取不到，查看了发现合约不是普通的ERC20合约，在合约Read中看不到能调用的abi方法

![ERC20合约p1.png](https://raw.githubusercontent.com/AscenX/AscenX.github.io/source/images/proxy_contract_p1.png)

<!-- more -->

在合约的 `Read as Proxy`可以看到合约的abi
![ERC20合约p2.png](https://raw.githubusercontent.com/AscenX/AscenX.github.io/source/images/proxy_contract_p2.png)

上面说明了使用了 EIP-1967 透明代理的模式

### [EIP1967](https://eips.ethereum.org/EIPS/eip-1967)是什么？

是一种合约标准，用于升级合约。通过一层代理合约，去调用另一个包含逻辑的合约，达到了合约指向和逻辑分开的目的



