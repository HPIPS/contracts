HPIPS发起在NULS上面开发Bancor交易协议的说明文档，项目又原始的https://github.com/bancorprotocol/contracts项目0.4版本分叉过来
 
HPIPS initiated the development of Bancor transaction protocol documentation on NULS, and the original https://github.com/bancor protocol/contracts project was split by version 0.4.

 # Bancor Protocol Contracts v0.4 (alpha)（BANCOR协议V0.4（Alpha））
BANCOR是一个分散的流动性网络，它为用户提供了一种简单、低成本的买卖代币的方式。Bancor的开源协议通过智能合同直接赋予令牌内置的可转换性，允许集成令牌彼此立即转换，而无需在交换中匹配买方和卖方。Bancor钱包可以直接从钱包内自动转换令牌，其价格比交易所更可预测，并且不易操纵。要立即转换NULS上发行的token，请访问[HPIPS Web](https://hpips.io/),加入[HPIPS电报组](https://t.me/HPIPScn)或阅读Bancor协议{White.](https://www.bancor.network/white.)以获得更多信息。

Bancor is a decentralized mobile network that provides users with a simple, low-cost way to buy and sell tokens. Bancor's open source protocol directly endows tokens with built-in convertibility through smart contracts, allowing integration tokens to be converted to each other immediately without matching the buyer and seller in the exchange. Bancor wallets can automatically convert tokens directly from the wallet, and their prices are more predictable than exchanges and less manipulative. To convert tokens issued on NULS immediately, visit [HPIPS Web] (https://hpips.io/), join [HPIPS Telegraph Group] (https://t.me/HPIPScn) or read the Bancor protocol {White.] (https://www.bancor.network/white.) for more information.

## Overview（概述）
在资产交换领域，Bancor协议代表了经济学中称为“双重需求重合”的经典问题的第一个技术解决方案。为了物物交换，欲望问题的巧合是通过金钱来解决的。对于货币，交易所仍然依靠劳动力，通过投标/要求订单和外部代理人之间的贸易，创造市场并提供流动性。

The Bancor protocol represents the first technological solution for the classic problem in economics known as the “Double Coincidence of Wants”, in the domain of asset exchange. For barter, the coincidence of wants problem was solved through money. For money, exchanges still rely on labor, via bid/ask orders and trade between external agents, to make markets and supply liquidity. 

通过使用智能契约，可以创建将一个或多个其他令牌保持为连接器的智能令牌。令牌可以代表现有的国家货币或其他类型的资产。通过使用连接器令牌模型和算法计算的转换率，Bancor协议创建了用于资产交换的新型生态系统，没有中央控制。这种分散的分级货币体系为具有众多实质性优势的自主分散的全球交换奠定了基础。

Through the use of smart-contracts, Smart Tokens can be created that hold one or more other tokens as connectors. Tokens may represent existing national currencies or other types of assets. By using a connector token model and algorithmically-calculated conversion rates, the Bancor Protocol creates a new type of ecosystem for asset exchange, with no central control. This decentralized hierarchical monetary system lays the foundation for an autonomous decentralized global exchange with numerous and substantial advantages.

## Warning（警告）
BANCOR是一项正在进行中的工作。使用前一定要了解风险。

Bancor is a work in progress. Make sure you understand the risks before using it.

# The Bancor Standards（Bancor标准）
BANCOR协议是使用多个合同来实现的。其中最主要的是Stand和BANCORTER。
BANCORTER负责令牌与其连接器之间的转换。
智能令牌表示一个转换器感知的符合ECR-20的令牌。

Bancor protocol is implemented using multiple contracts. The main ones are SmartToken and BancorConverter.
BancorConverter is responsible for converting between a token and its connectors.
SmartToken represents a converter aware ERC-20 compliant token.

# The Smart Token Standard（智能令牌标准）

## Motivation（动机）
这些将允许创建符合BANCOR的令牌，同时将依赖性保持在最小值。
此外，它允许拥有的合同通过赋予所有者完全控制来扩展其功能。

Those will allow creating a Bancor compliant token while keeping dependencies at a minimum.
In addition, it allows an owning contract to extend its functionality by giving the owner full control.

## Specification（规范）

### SmartToken（智能令牌）
第一和最重要的是，智能令牌令牌在ERC 20兼容。
IT工具作为搜索标准，两个标准的令牌令牌的方法和事件。

First and foremost, a Smart Token is also an ERC-20 compliant token.
As such, it implements both the standard token methods and the standard token events.

### Methods（方法）
请注意，这些方法只能由令牌所有者执行。

Note that these methods can only be executed by the token owner.

**issue**
```cs
function issue(address _to, uint256 _amount)
```
增加令牌函数。

Increases the token supply and sends the new tokens to an account.
<br>
<br>
<br>
**destroy**
```cs
function destroy(address _from, uint256 _amount)
```
从帐户中移除令牌函数。

Removes tokens from an account and decreases the token supply.
<br>
<br>
<br>
**disableTransfers**
```cs
function disableTransfers(bool _disable)
```
禁用交易功能函数。 

Disables transfer/transferFrom functionality.
<br>
<br>
<br>
### Events

**NewSmartToken**
```cs
event NewSmartToken(address _token)
```
部署智能令牌触发函数

Triggered when a smart token is deployed.
<br>
<br>
<br>
**Issuance**
```cs
event Issuance(uint256 _amount)
```
总供给增发函数。

Triggered when the total supply is increased.
<br>
<br>
<br>
**Destruction**
```cs
event Destruction(uint256 _amount)
```
总供给减少函数。

Triggered when the total supply is decreased.
<br>
<br>
<br>

# The Bancor Converter Standard（BANCOR变换器标准）

下面的部分描述BANCOR转换器可以实现的标准功能。

The following section describes standard functions a bancor converter can implement.

## Motivation（动机）

这些将允许DAPPS和钱包购买和销售令牌。

Those will allow dapps and wallets to buy and sell the token.

这里最重要的是“转换”。

The most important here is `convert`.

## Specification（规范）

### Methods（方法）

**connectorTokenCount**
```cs
function connectorTokenCount() public constant returns (uint16 count)
```
获取为令牌定义的连接器令牌的数量。

Gets the number of connector tokens defined for the token.
<br>
<br>
<br>
**connectorTokens**
```cs
function connectorTokens() public constant returns (address[] connectorTokens)
```
获取连接器令牌合约地址的数组。

Gets an array of the connector token contract addresses.
<br>
<br>
<br>
**connectors**
```cs
function connectors(address _connectorToken) public constant
```
获取连接器令牌详细信息。

Gets the connector token details.
<br>
<br>
<br>
**convert**
```cs
function convert(address _fromToken, address _toToken, uint256 _amount, uint256 _minReturn)
```
转换特定数量
只有当它返回一个大于或等于“μmin返回”的值时，才会发生转换。

converts a specific amount of _fromToken to _toToken
The conversion will only take place if it returns a value greater or equal to `_minReturn`.
<br>
<br>
<br>

### Events

**Conversion**
```cs
event Conversion(address indexed _fromToken, address indexed _toToken, address indexed _trader, uint256 _amount, uint256 _return, uint256 _currentPriceN, uint256 _currentPriceD);
```
当一个可转换令牌发生转换时触发。

Triggered when a conversion between one of the convertible tokens takes place.

## Testing（测试）
Tests are included and can be run using truffle & ganache

测试包括，可以使用松露和甘纳什运行

### Prerequisites（关联工具）
* Node.js v7.6.0+
* truffle v4.1.5+
* ganache v1.1.0+

To run the test, first start ganache and then execute the following command from the project's root folder -
* npm test

要运行测试，首先启动GANACHE，然后从项目的根文件夹NPM测试执行以下命令 

## Collaborators

* **[Yudi Levi](https://github.com/yudilevi)**
* **[Ilana Pinhas](https://github.com/ilanapi)**
* **[Or Dadosh](https://github.com/ordd)**
* **[Barak Manos](https://github.com/barakman)**
* **[Martin Holst Swende](https://github.com/holiman)**
* **[Yongjun Qian](https://github.com/qianyongjun895)**

## License

Bancor Protocol is open source and distributed under the Apache License v2.0


BANCOR协议是开源的，并在Apache许可证V2.0下进行分发。 
