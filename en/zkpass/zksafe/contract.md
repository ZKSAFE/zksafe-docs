# 📜 合约说明
## ZKSAFE 合约说明

### 准备工作
Node.js 建议 v16，安装 [snarkjs](https://github.com/iden3/snarkjs)，你可以不会snarkjs，照着代码写也行
```javascript
npm install -g snarkjs
```
安装 [ethers](https://docs.ethers.io/v5/getting-started/)，你必须会ethers，所有代码示例都假设你会ethers
```javascript
npm install ethers
```
[合约源码](https://github.com/ZKSAFE/all-contracts/tree/main/contracts/zkSafe)

[测试代码](https://github.com/ZKSAFE/all-contracts/blob/main/test/Safebox-withdraw.js)

>注意：测试环境是hardhat，ethers的用法跟正式环境略有不同，以上代码都基于测试环境

合约接口里用到proof的地方即基于ZK-SNARK的ZKSAFE Password，参考[ZKSAFE Passwor 技术对接](../zkpass/build.md)

本章假设你熟悉Solidity，所以不详细说明调用方法
<br>
<br>

### SafeboxFactory合约
Safebox合约的工厂合约，管理每个人的Safebox
<br>

#### createSafebox() 激活保险箱
首先，你需要调用createSafebox()，该方法会部署一个你专属的Safebox合约，即激活

一个钱包只能激活一个保险箱，如果你已经激活，再次激活会报错。除非你转让了你的保险箱，那你就可以再激活一个

Safebox的合约地址由你的地址和激活的次数决定，无论在哪条链上，你的Safebox地址都是一样的，并且可以推导出来。如果你转让了你的保险箱，那么你再激活，Safebox的地址会跟之前不一样
<br>

#### getSafeboxAddr() 推导保险箱地址
不管有没有激活，都可以推导出某个钱包对应的Safebox地址

Safebox地址和钱包地址格式一样，可以直接给Safebox地址转ERC20或ERC721，无论有没有激活都可以转进去，激活后可以提取出来

>注意：仅支持提取原生代币（ETH）、ERC20（Token）、ERC721（NFT），其他类型资产（比如ERC1155）一概不支持，请不要存错

不过还是建议先激活再转币进去
<br>

#### userToSafebox[] 查看已激活的保险箱地址
查看某个钱包对应的已激活的Safebox地址

如果未激活，返回0x00
<br>

#### changeSafeboxOwner() 转让保险箱
仅限Safebox合约调用，如果对方已有保险箱，那就不能转给他
<br>
<br>

### Safebox合约-常规操作
每个用户都可以通过SafeboxFactory部署一个自己的专属Safebox合约
<br>

#### owner() 查看保险箱的拥有者
Safebox合约的拥有者，合约里除了社交恢复的所有方法都只能拥有者调用
<br>

#### transferOwnership() 转让保险箱
需要密码，仅限拥有者调用

转让之后，密码也将重置为新owner的密码，也就是说，新owner拥有了保险箱，以及里面的所有资产
<br>

#### withdrawETH() 提取原生代币
需要密码，仅限拥有者调用

生成proof的时候，需要把提取数量amount作为datahash
<br>

#### withdrawERC20() 提取ERC20代币
需要密码，仅限拥有者调用

生成proof的时候，需要把代币地址tokenAddr、提取数量amount的keccak256 hash值作为datahash
<br>

#### withdrawERC721() 提取ERC721标准NFT
需要密码，仅限拥有者调用

生成proof的时候，需要把NFT地址tokenAddr、NFT的tokenId的keccak256 hash值作为datahash
<br>
<br>

### Safebox合约-社交恢复
事先设置好守护者，出事时候的救命稻草

可能出事的情况：
* 忘记私钥
* 忘记ZKSAFE密码
* ZKSAFE密码失效（在20万次压力测试中从未发生）
* 私钥被盗（这时候owner被控制，转入gas给owner可能瞬间被划走，导致没法用owner，只能靠守护者）
<br>

#### setSocialRecover() 设置守护者
需要密码，仅限拥有者调用

生成proof的时候，需要把所有守护者的地址_guardians、有效守护者数量_needGuardiansNum的keccak256 hash值作为datahash

如果你设置了5个守护者，那么把5个地址都放进_guardians，有效守护者数量_needGuardiansNum可以是1～5，意思是5个守护者中只要_needGuardiansNum个发起多签，就可以实现社交恢复，通常多签设置为 2 of 3，意思是3个守护者中的2个发起多签就有效
<br>

#### getSocialRecover() 查看守护者
返回3个字段：
* guardians：设定的所有守护者的地址
* needGuardiansNum：设定的有效守护者数量
* doneGuardians：已发起多签的守护者，达到needGuardiansNum数量，社交恢复即有效
<br>

#### transferOwnership2() 通过守护者转让保险箱
仅限守护者调用

守护者需要指定转让给谁，可以是任意地址，但必须每个人指定的地址都一样。如果不一样，那么重新来过，直到达成共识

保险箱转让后，守护者依然有效
<br>