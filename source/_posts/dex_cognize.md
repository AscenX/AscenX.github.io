---
title: DEX了解及入门
date: 2023-06-06 01:25:59
tags:
- UniSwap
- Contract
- 区块链
categories:
- 学习笔记
---

最近做的项目需要与DEX做交互，让我对DEX有了一些了解，主要是UniSwap和PancakeSwap

PancakeSwap的合约是Fork UniSwap的，概念基本一样

<!-- more -->


### 交易及合约

首先，是什么币种和什么币种交易，例如使用BNB购买USDT，那么币对就是BNB-USDT
要能进行交易，必须要有一个池子，里面有这两种币种的资金池，这两个币种可以相互兑换，造成流动性
在Uniswap叫池子(Pool)，在PancakeSwap叫流动性(Liquidity)

没有流动性就交易不了，当我们在池子创建了币对才有了流动性，创建了V2流动性之后，会生成一个新的LP Token合约，即下面说的Pair合约，合约对应的一定数量Token会转入到创建流动性的地址，某些DeFi功能就是使用了这个token进行质押获得分红

#### Pair合约

Pair 合约是 Uniswap 中的交易对合约，用于实现资产之间的交易。每个 Pair 合约都对应着两种不同的资产，比如 ETH/USDT、ETH/DAI 等。Pair 合约中包含了资产之间的价格信息、流动性提供者的信息等。

那么问题来了，用户创建流动性的时候，是谁来创建Pair合约的呢，是Facotry合约

#### Factory合约

Factory 合约负责创建新的交易对（Pair 合约）。Factory 合约中存储了所有已创建的交易对的地址。

#### 路由 Router

Router 合约是 Uniswap 中的路由合约，用于执行交易操作。当用户在 Uniswap 上进行交易时，实际上是通过 Router 合约来执行交易操作的。Router 合约中包含了一系列的交易函数，用户可以通过调用这些函数来进行资产交易。

路由的作用就是找到对应的池子，进行兑换。如果没有对应的池子，是可以跨池子交易的，例如只有BNB-USDT, USDT-BTC 两个池子，用户想要使用BNB来兑换BTC，那么路由就可以做到兑换交易，只不过这样手续费会更高

#### SmartRouter

在UniSwap 和 Pancake Swap 的路由也是在升级，现在有V2，V3都有在使用

V3相对于之前的版本，之前的Liquidity的价格可以为0到无限大，而V3中定义了一个价格范围。

V3适用于大体量的资金池子，如果池子资金量不够大，太容易被拉盘砸盘操纵价格

用户进行交易是使用V3还是V2呢，在交易设置里可以自定义选择V3还是V2，如果没有选择的话，交易是由智能路由（SmartRouter）来进行选择的
经过我的测试，无论用户选择了V3还是V2，都是走SmartRouter，SmartRouter更像是最上层的一个分发器

### 交易原理及特性

在DEX进行Swap的话，是怎么影响价格的

#### AMM (Automated Market Maker)

AMM是UniSwap的交易协议，是去中心化的。AMM的核心原理是利用流动性池来完成交易。当用户想要进行交易时，他们可以直接从流动性池中交换一种资产为另一种资产，而无需依赖买卖订单的匹配。这是通过根据资产比例来计算价格完成的。

流动池就是有两个币种或者多个币种组成的一个资金池，例如ETH-USDT流动池，假如有1ETH和1000USDT，那么1ETH的价格就约等于1000USDT，如果有用户使用USDT购买ETH，那么池子中的ETH会减少，USDT会增加，那么ETH的价格就会增加

#### Permissionless Systems

无需许可系统。任何人都可以交易，以及任何人都可以创建池子让别人交易。