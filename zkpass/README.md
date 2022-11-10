# 👋 Introduction
## ZKSAFE Password
Have you ever thought that the administrator can change your password, and you never have your own password actually

That’s why we want to design such a cryptosystem that should satisfy:

* No downtime
* Password not be exposed
* Only you can change your own password

It can be realized in EVM smart contract but the new issue: Sandwich Attack
<br>

### Sandwich Attack
Let's say you have submitted a tx of a withdrawal with your password verification information. Supposed a hacker copied the tx when loading and processing because the tx is open, and changed withdrawal address to his own address, because no dynamic password authentication information, so the verification passed in the contract, then he submitted with higher gas, it can be processed before you, and take your money

Therefore, it also should be with
* Signing Feature 

<br>

### Signing Feature
When you submit the withdrawal tx, the tx should sign the withdrawal information by password. if the tx information is tampered, it can be checked in the contract, so as to prevent the sandwich attack

Only the private key can sign the data in traditional algorithm. We unexpectedly found that ZK-SNARK can do some circuit programming and realize the password to sign the data. After many times of questioning and algorithm adjustment, we finally made it !

<br>

### Origin
The first product created with this system is **ZKSAFE**, and it works great !

We then decided that such a great cryptography system should be expanded to support not only ZKSAFE, but also various asset management platforms, and even private key-less wallets (which would greatly reduce the barrier for users to entry Web3). And this password system is

**ZKSAFE Password (abbr.ZKPass)**


**Your own password, you deserve it !**
<br>

### Competitor MPC
ZKSAFE Password（abbr.ZKPass）does not use the MPC (private key sharding) scheme. Here we would like to give a comparison since there are many people ask this:

* MPC divides the private key into pieces (shards) and distributes it to multiple nodes
  * It with centralization risk because the node can be attacked which can lead the loss of the private keys
  * The password is useless when private key is stolen

* ZKPASS is pure algorithm to achieve the password function with no node
  * Decentralized, stores no private keys or passwords
  * Even private key is stolen, the password still works

MPC is good as the private key custody plan. And ZKPass is protocol-level, non-custody password algorithm
<br>