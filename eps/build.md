# 🤖 对接
## Ethereum Password Service 合约对接

### 准备工作
Node.js 建议 v16，安装 [snarkjs](https://github.com/iden3/snarkjs)，你可以不会snarkjs，照着代码写也行
```javascript
npm install -g snarkjs
```
安装 [ethers](https://docs.ethers.io/v5/getting-started/)，你必须会ethers，所有代码示例都假设你会ethers，无需解释
```javascript
npm install ethers
```
[合约源码](https://github.com/ZKSAFE/all-contracts/tree/main/contracts/eps)

[测试代码](https://github.com/ZKSAFE/all-contracts/blob/main/test/EPS-test.js)

>注意：测试环境是hardhat，ethers的用法跟正式环境略有不同，以下代码都基于测试环境

<br>

### resetPassword
初始化密码和改密码都是这个接口，先说所有跟ZK相关的接口都要用到的工具方法`getProof()`

#### 工具方法
```javascript
//util
async function getProof(pwd, address, nonce, datahash) {
    let expiration = parseInt(Date.now() / 1000 + 600)
    let chainId = (await provider.getNetwork()).chainId
    let fullhash = utils.solidityKeccak256(['uint256','uint256','uint256','uint256'], [expiration, chainId, nonce, datahash])
    fullhash = s(b(fullhash).div(8)) //must be 254b, not 256b

    let input = [stringToHex(pwd), address, fullhash]
    let data = await snarkjs.groth16.fullProve({in:input}, "./zk/main9/circuit_js/circuit.wasm", "./zk/main9/circuit_final.zkey")

    // console.log(JSON.stringify(data))

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

`getProof()`的参数有4个值，分别是：

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

#### 修改密码

```javascript
let oldpwd = 'abc123'
let nonce = await eps.nonceOf(accounts[0].address)
let datahash = '0'
let oldZkp = await getProof(oldpwd, accounts[0].address, s(nonce), datahash)

let newpwd = '123123'
let newZkp = await getProof(newpwd, accounts[0].address, s(nonce.add(1)), datahash)

fee = await eps.fee()
console.log('eps fee(Ether)', utils.formatEther(fee))

//need fee
await eps.resetPassword(oldZkp.proof, oldZkp.expiration, oldZkp.allhash, newZkp.proof, newZkp.pwdhash, newZkp.expiration, newZkp.allhash, {value: fee})
console.log('resetPassword done')
```