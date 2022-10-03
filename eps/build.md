# 🤖 合约对接
## Ethereum Password Service 合约对接

### 准备工作
Node.js 建议 v16，安装 [snarkjs](https://github.com/iden3/snarkjs)，你可以不会snarkjs，照着代码写也行
```javascript
npm install -g snarkjs
```
安装 [ethers](https://docs.ethers.io/v5/getting-started/)，你必须会ethers，所有代码示例都假设你会ethers
```javascript
npm install ethers
```
[合约源码](https://github.com/ZKSAFE/all-contracts/tree/main/contracts/eps)

[测试代码](https://github.com/ZKSAFE/all-contracts/blob/main/test/EPS-test.js)

>注意：测试环境是hardhat，ethers的用法跟正式环境略有不同，以下代码都基于测试环境

不建议用户在EPS和ZKSAFE以外的地方输入密码，防止密码泄漏。所以EPS的合约面向的是合作方合约，比如ZKSAFE
<br>

### resetPassword 设置密码
初始化密码和改密码都是这个接口，先说所有跟ZK相关的接口都要用到的工具方法`getProof()`

#### 工具方法
```javascript
//util
async function getProof(pwd, address, nonce, datahash) {
    let expiration = parseInt(Date.now() / 1000 + 600)
    let chainId = (await provider.getNetwork()).chainId
    let fullhash = utils.solidityKeccak256(['uint256','uint256','uint256','uint256'], [expiration, chainId, nonce, datahash])
    fullhash = s(b(fullhash).div(8)) //fullhash必须是254位, solidityKeccak256是256位，所以要转换

    let input = [stringToHex(pwd), address, fullhash]
    let data = await snarkjs.groth16.fullProve({in:input}, "./zk/main9/circuit_js/circuit.wasm", "./zk/main9/circuit_final.zkey")

    const vKey = JSON.parse(fs.readFileSync("./zk/main9/verification_key.json"))
    const res = await snarkjs.groth16.verify(vKey, data.publicSignals, data.proof)

    if (res === true) {
        console.log("Verification OK")

        let pwdhash = data.publicSignals[0]
        let fullhash = data.publicSignals[1]
        let allhash = data.publicSignals[2]

        let proof = [
            BigNumber.from(data.proof.pi_a[0]).toHexString(),
            BigNumber.from(data.proof.pi_a[1]).toHexString(),
            BigNumber.from(data.proof.pi_b[0][1]).toHexString(),
            BigNumber.from(data.proof.pi_b[0][0]).toHexString(),
            BigNumber.from(data.proof.pi_b[1][1]).toHexString(),
            BigNumber.from(data.proof.pi_b[1][0]).toHexString(),
            BigNumber.from(data.proof.pi_c[0]).toHexString(),
            BigNumber.from(data.proof.pi_c[1]).toHexString()
        ]

        return {proof, pwdhash, address, expiration, chainId, nonce, datahash, fullhash, allhash}

    } else {
        console.log("Invalid proof")
    }
}
```

为方便起见，我们写了一个工具方法`getProof()`，封装了所有用到的ZK算法，处理了ZK里面256位转254位的坑，需要注意的是`circuit.wasm`、`circuit_final.zkey`、`verification_key.json`是固定值，可以在[ZK源码](https://github.com/ZKSAFE/all-contracts/tree/main/zk)找到

`getProof()`即图中的ZK Circuit
<br>
<div align="center"><img src="../images/eps-1.png"></div>
<br>

`getProof()`有4个参数，分别是：

* pwd：你的密码，string类型
* address：你的钱包地址，string类型
* nonce：从EPS合约获取的你的nonce值，string类型
* datahash：你想要对什么数据进行签名，这个数据的hash值，string类型

返回所有ZK算法有关的数据：

* proof：ZK-SNARK的proof，由8个uint256组成的数组
* pwdhash：EPS合约需要用到的pwdhash，uint256类型
* address：参数里的address，string类型
* expiration：签名过期时间，默认10分钟，int类型
* chainId：公链id，int类型
* nonce：参数里的nonce，string类型
* datahash：参数里的datahash，string类型
* fullhash：这个不需要传入合约，254位，string类型
* allhash：以上所有参数的hash，uint256类型
<br>



#### 初始化密码

```javascript
let pwd = 'abc123' //你的密码
let nonce = '1' //初始化密码，nonce就是1
let datahash = '0' //对于resetPassword接口，datahash固定是0
let p = await getProof(pwd, accounts[0].address, nonce, datahash)

//需要付一点手续费 :)
fee = await eps.fee()
console.log('eps fee(Ether)', utils.formatEther(fee))

let gasLimit = await eps.estimateGas.resetPassword(p.proof, 0, 0, p.proof, p.pwdhash, p.expiration, p.allhash, {value: fee})
await eps.resetPassword(p.proof, 0, 0, p.proof, p.pwdhash, p.expiration, p.allhash, {value: fee, gasLimit})
console.log('initPassword done')
```

`resetPassword()`有7个参数，分别是：

* proof1：旧密码生成proof，由8个uint256组成的数组
* expiration1：旧密码的过期时间，uint256类型
* allhash1：旧密码生成allhash，uint256类型
* proof2：新密码生成proof，由8个uint256组成的数组
* pwdhash2：新密码的pwdhash，由ZK生成，uint256类型
* expiration2：新密码的过期时间，uint256类型
* allhash2：新密码生成allhash，uint256类型

因为初始化密码没有旧密码，所以前3个旧密码相关的参数在合约里是用不到的，但是必须得传，全部传0即可，或者把新密码的`proof2`当`proof1`传也行（示例就是这么干的）

成功后，调用者的address（msg.sender）的密码就是`pwd`
<br>

#### 修改密码

```javascript
let oldpwd = 'abc123' //旧密码
let nonce = await eps.nonceOf(accounts[0].address) //当前的nonce
let datahash = '0' //对于resetPassword接口，datahash固定是0
let oldZkp = await getProof(oldpwd, accounts[0].address, s(nonce), datahash) //旧密码的proof

let newpwd = '123123' //新密码
let newZkp = await getProof(newpwd, accounts[0].address, s(nonce.add(1)/**新密码的nonce+1*/), datahash) //新密码的proof

fee = await eps.fee()
console.log('eps fee(Ether)', utils.formatEther(fee))

//need fee
await eps.resetPassword(oldZkp.proof, oldZkp.expiration, oldZkp.allhash, newZkp.proof, newZkp.pwdhash, newZkp.expiration, newZkp.allhash, {value: fee})
console.log('resetPassword done')
```

还是`resetPassword()`接口，修改密码需要用旧密码，所以要用旧密码生成前3个参数

成功后，调用者的address（msg.sender）的密码就是`newpwd`，旧密码`oldpwd`作废
<br>

### verify 校验密码
密码可以在链下校验，获取`pwdhash`在链下就可以校验；也可以上链校验，通常是配合合作方合约一起，由合作方合约调用`EPS.verify()`，密码错误就报错，不报错的话就是密码正确，且签名有效，合作方合约可以继续处理后续

不建议用户在EPS和ZKSAFE以外的地方输入密码，防止密码泄漏，所以链下校验只在EPS和ZKSAFE就行，合作方可以用链上校验的方式对接EPS

`verify()`有5个参数，分别是

* user：哪个用户的签名，address类型
* proof：密码在ZK生成的proof，由8个uint256组成的数组
* datahash：用户对什么数据进行的签名，这个就是数据的hash，uint256类型
* expiration：签名的过期时间，uint256类型
* allhash：签名在ZK生成allhash，uint256类型

合约内会用user的`pwdhash`进行密码的校验，以及把`datahash`转成254位的`fullhash`。。。总之，`getProof()`工具会处理所有ZK校验相关的参数

ZKSAFE作为合作方的合约调用EPS
```C
function withdrawERC20(
    uint[8] memory proof, //转给EPS的参数
    address tokenAddr, //提什么token
    uint amount, //提多少
    uint expiration, //转给EPS的参数
    uint allhash //转给EPS的参数
) external onlyOwner {
    uint datahash = uint(keccak256(abi.encodePacked(tokenAddr, amount))); //计算datahash
    eps.verify(owner(), proof, datahash, expiration, allhash); //密码和签名的校验

    IERC20(tokenAddr).safeTransfer(owner(), amount); //校验通过，干活！

    emit WithdrawERC20(tokenAddr, amount);
}
```
在这个示例中，用户想要把token从ZKSAFE提出来，所以需要对提什么token（`tokenAddr`）、提多少（`amount`）用密码进行签名

ZKSAFE的链下代码
```javascript
let pwd = 'abc123' //用户的密码
let nonce = s(await eps.nonceOf(accounts[0].address)) //用户当前的nonce
let tokenAddr = usdt.address //提什么token
let amount = s(m(40, 18)) //提多少
let datahash = utils.solidityKeccak256(['address', 'uint256'], [tokenAddr, amount]) //计算datahash
datahash = s(b(datahash)) //转成string类型数字
let p = await getProof(pwd, accounts[0].address, nonce, datahash) //计算ZK Proof

await safebox.withdrawERC20(p.proof, tokenAddr, amount, p.expiration, p.allhash) //调用合约，提款
console.log('withdrawERC20 done')

await print()
```

`datahash`是合作方定义的，uint256类型，通常是hash值。也有例外的，比方说签名的是address，即uint160类型，直接放`datahash`也能装得下，可以不用hash

合作方链下计算的`datahash`，和合作方合约计算的`datahash`必须一致
