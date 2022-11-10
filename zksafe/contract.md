# 📜 Contract Description
## ZKSAFE Contract Description

### Preparations
Required Node.js v16, install [snarkjs](https://github.com/iden3/snarkjs)
```javascript
npm install -g snarkjs
```
Install [ethers](https://docs.ethers.io/v5/getting-started/), you need to know how to use ethers, all the code examples bellow assumed you know it:
```javascript
npm install ethers
```
[Contract source code](https://github.com/ZKSAFE/all-contracts/tree/main/contracts/zkSafe)

[Testing code](https://github.com/ZKSAFE/all-contracts/blob/main/test/Safebox-withdraw.js)

>Note：The test environment is hardhat. ethers is used slightly differently than the formal environment. The following code is based on the test environment

Proof used in contract interface is the ZKSAFE Password based on ZK-SNARK, please refer to [ZKSAFE Password build](../zkpass/build.md)

<br>
<br>

### SafeboxFactory Contract
The factory contract in ZKSAFE to manage the Safebox contracts
<br>

#### createSafebox() Activate Safebox
First, you need to call createSafebox(), which will deploy your own Safebox contract, called activation

One wallet can only activate one Safebox. If you have already activated it, an error will be reported if you activate it again. Unless you transferred your Safebox ownership, then you can activate another one

The contract address of the Safebox is determined by your address and your activating times. Your Safebox address is the same regardless of which blockchain and can be imported. If you transfer your safebox ownership, the Safebox address will be different when you reactivate it
<br>

#### getSafeboxAddr() derive the Safebox address
With or without activation, you can derive the Safebox address for a particular wallet

The format of the Safebox address is the same as that of the wallet address. You can directly transfer the ERC20 or ERC721 assets into Safebox address. regardless of whether it is activated or not, and withdraw them after activated

>Note: Only native tokens (ETH), ERC20 (Token), and ERC721 (NFT) can be transferred. Other types of assets (such as ERC1155) are not supported

However, it’s suggested to activate the Safebox before transferring your assets in
<br>

#### userToSafebox[] check the activated safebox address
Check the activated safebox address of a wallet

if it’s inactivated, return 0x00
<br>

#### changeSafeboxOwner() transfer Safebox's ownership
only for Safebox contract call, and it can’t be transferred if the new owner already have one
<br>
<br>

### Safebox Contract - Normal Operation 
Every user can deploy an exclusive Safebox contract for himself/herself using SafeboxFactory
<br>

#### owner() check the owner of the Safebox
Only owner can call the Safebox contract except the ownership is obtained by Social Recovery 
<br>

#### transferOwnership() transfer Safebox
Password required, only owner can call

After the transfer, the password will also be reset, which means that the new owner owns the safebox and all the assets in it
<br>

#### withdrawETH() withdraw native token 
Password required, only owner can call

Withdrawal amount is datahash when generating proof
<br>

#### withdrawERC20() 提取ERC20代币
Password required, only owner can call

Keccak256 hash for Token address and amount is datahash when generating proof
<br>

#### withdrawERC721() 提取ERC721标准NFT
Password required, only owner can call

keccak256 hash for NFT address and tokenId is datahash when generating proof
<br>
<br>

### Safebox Contract - Social Recovery
It’s suggested to set guardian in advance in case you need it under the following

Possible situation:
* Forget private key
* Forget ZASAFE password
* ZKSAFE password invalid (0 chance under 200,000 pressure testing)
* Private key been hacked (hacker control the owner and transfer all gas out, so owner has no gas to do anything but the guardians)
<br>

#### setSocialRecover() set guardians
Password required, only owner can call

Keccak256 hash for all guardians address and valid guardians number is datahash when generating proof

If you set 5 guardians, all the 5 address will be needed in _guardians, valid gurdians number _needGuardiansNum can be 1-5, means _needGuardiansNum of 5 can multi-signing to realize social recovery. Usually multi-signing is set as 2 of 3, which means 2 of the 3 gurdians can do the social recovery
<br>

#### getSocialRecover() check gurdians
Return 3 fields:
* guardians: the preset guardians’ addresses
* needGuardiansNum: preset valid guardian number
* doneGuardians: guardian initiated multi-signing, social recovery will be done when reach the needGuardiansNum
<br>

#### transferOwnership2() transfer Safebox by gurdians
Only guardians can call

The Guardians need to specify an address to transfer, it can be any address, but need to be consistent. If not, the whole process will go over again until it’s consistent

After the Safebox is transferred, the guardian remains in effect
<br>