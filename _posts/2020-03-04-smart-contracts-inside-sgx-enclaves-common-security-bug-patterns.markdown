---
layout: post
title:  "Smart Contracts Inside SGX Enclaves: Common Security Bug Patterns"
date:   2020-03-24 14:21:19 -0400
categories: blockchain
excerpt: hello
---

**This article originally appeared on the NCC Group Cryptography Services blog. Gérald Doussot and I wrote it after performing a 3-week security analysis of an SGX-based blockchain implementation. TLDR this really comes down to: don't rely on TEE security if the TEEs will run under attacker's control, either hardware-wise or software-wise.**

Running smart contracts in a Trusted Execution Environment (TEE) such as Intel Software Guard Extensions (SGX) to preserve the confidentiality of blockchain transactions is a novel and not widely understood technique. In this blog post, we point out several bug classes that we observed in confidential smart contract designs and implementations in our recent client engagements. We argue that the infrastructure underpinning the execution of confidential smart contracts is challenging to implement securely and that confidential smart contracts are prone to misuse by users.

## Why Confidential Smart Contracts

Smart contracts may process sensitive and/or personally identifiable information, including but not limited to financial and medical records or even political opinions. In fact, one could contend that many, if not most, useful smart contracts must process sensitive data – both in-flight and at-rest. In a typical blockchain implementation, this information is inherently accessible to anyone, thus severely limiting the applicability of smart contracts to real world problems that would benefit from them.

The industry is attempting to come up with solutions to protect the secrecy of transactions in smart contracts. Specifically, executing a given smart contract method should not reveal its input arguments, returned output, state storage (the smart contract’s database) and the actual method that was called. One way to achieve “confidential smart contracts” is to encrypt all data passed to, returned and processed by the contract at a given address.

Where should the cryptographic keys used to encrypt and decrypt transaction contract input data reside in this case? What processes can we entrust to execute the decrypted transactions and to not exfiltrate the data? Any node that could decrypt confidential data inside a particular transaction is not an acceptable solution, as this would clearly violate the initial confidentiality assumption these systems need to enforce. This leads us to the discussion of TEEs and Intel SGX in the next section.

## TEEs and Intel SGX

A trusted execution environment is a processor component that provides some level of security assurances to code running within its context and to the data it processes. Intel Software Guard Extensions is one of [several](https://developer.amd.com/sev/) [implementations](https://developer.arm.com/ip-products/security-ip/trustzone) of TEE in hardware that isolates application code and data memory into private and encrypted regions. In theory, one can therefore ensure that private data cannot be divulged to unauthorized parties and that sanctioned code cannot be modified, even in the context of a compromised operating system, with [several](https://en.wikipedia.org/wiki/Trust_(social_science)) [important](https://en.wikipedia.org/wiki/Side-channel_attack) [qualifiers](https://software.intel.com/en-us/sgx/attestation-services) and [disclaimers](https://en.wikipedia.org/wiki/Vulnerability_(computing)).

A process running within the confines of an Intel SGX enclave would appear to be a good candidate to host encryption keys and to operate on smart contract data without divulging both of these to unauthorized parties. This is one approach being adopted and we briefly describe it next.

## Confidential Smart Contracts and SGX in Practice

Smart contracts are executed in a virtual machine such as the Ethereum Virtual Machine (EVM). For confidential smart contracts, the VM process is hosted in a SGX TEE on a computation node.

Confidential smart contract data is typically stored encrypted with an authenticated symmetric cipher, in a key-value database. The database does not run in SGX for several reasons including that SGX resources are constrained and that a SGX process cannot directly interface with the host file system without some kind of external proxy process. The non-SGX database process may reside on the same computation node running the SGX VM process or on another node. The contract’s input and output arguments are encrypted using an asymmetric cipher.

The cryptographic keys to encrypt and decrypt the callables, arguments, state and output live only inside TEEs enclaves. Nodes in the network responsible for executing transactions run such enclaves which decrypt the input arguments, decrypt the contract state, execute the called methods in the smart contract VM and then update, encrypt and save new contract states. Finally, the smart contract callable/method return values also need to be encrypted and in such a way that only the caller can decrypt the output.

Continuing with the aforementioned EVM example, keys and values are read from and committed to storage using the sload and sstore opcodes. These calls may be modified by the confidential smart contract EVM implementation to ensure that both the keys and values are encrypted, when at rest outside of the enclave, and decrypted when processed by the EVM in SGX.

For some background information on how EVM stores contract state, refer to this blog [post](https://medium.com/aigang-network/how-to-read-ethereum-contract-storage-44252c8af925). At a high level, contract state variables have associated indexes used to compute the location of data inside the storage. For instance, in the case of a hashmap state variable, the storage location is calculated by hashing the hashmap’s state variable index together with the key name. Once the location has been computed, EVM can fetch the requested data from storage.

A confidential smart contract implementation might attempt to hide both hashmap keys and values from anyone observing the contract state. There are multiple ways one could approach this and we describe one option below. Consider that the following hashmap entry is stored: `balance["joe"]= 1000`. In addition, suppose that the index of the balance hashmap in the contract is p. Roughly speaking:

* EVM will compute the location inside the smart contract storage for `k=joe`‘s balance as `addr = keccak256(k . p)` with . a concatenation operation
* If such a location is used as is, it may be subject to bruteforce guessing. As such, the SGX application may encrypt the address as `enc_addr = keccak256(enc(addr, nonce))` where `enc()` is an authenticated encryption.
* The SGX application encrypts the balance value “1000” as `enc_value = enc(1000, nonce)`, using authenticated encryption
* The SGX application then sends encrypted key and value `enc_addr` and `enc_value` to storage (outside of SGX) using the `sstore` bytecode.

The use of a nonce permits to randomize the ciphertext so that encrypting the same plaintext several times yields different ciphertexts.

## Security Issues With Confidential Smart Contracts

Now that we have set the premise, we can proceed with listing some common and potential issues that we have identified in confidential smart contract implementations:

**Rogue smart contract code injection on TEE nodes:** Consider a node with an SGX enclave capable of running confidential smart contracts. When a confidential transaction needs to be processed, the enclave fetches the chosen smart contract method code, runs it on decrypted arguments, producing the output and encrypts this output before returning it outside the enclave. What prevents the node owner (or an attacker who compromised the node) from feeding the SGX enclave with incorrect smart contract code for that method? An owner of such a node may be able to execute **their own rogue smart contract** on decrypted data.

What harm can such a rogue method do? Recall that the rogue smart contract code cannot trivially return decrypted data to the attacker, since the output is encrypted with keys the node owner does not have. As such, the rogue contract code needs to **signal** the values to the outside world through leaks. This could involve encoding information inside the layout of the encrypted state code, encoding information through timing or return/output ciphertext leaks, etc.

Rogue smart code injection could be prevented by giving the means for an enclave to validate that the code deployed to a given address is correct. This may be achieved through an addressing scheme where a contract’s address is determined by hashing the contract’s code, as described in Ethereum Improvement Proposal [EIP-86](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-86.md).

**Contract state ciphertext changes leak:** Nodes that run confidential smart contracts might keep track of the contract state in encrypted form outside TEE enclaves. The owner of such a node carefully compares the encrypted state before and after running a particular confidential transaction. While the attacker cannot learn anything about plaintext values directly, the **layout of the changes to the encrypted state** may represent a valuable side channel.

For instance, the contract code specification, which is public, may reveal that a contract stores an integer and a long string. The length of the mutation in the encrypted contract state will reveal what the confidential transaction modified, which in turn may reveal data about what method was called and something about the actual arguments passed to the method.

**Non-constant time smart contracts (ease of misuse):** When performing a security review of a particular confidential smart contract, non-constant code computation is the most obvious attack vector to consider. While there has been a strong focus on side-channel attacks to retrieve cryptographic material from SGX enclaves in the research community, an (other) elephant in the room is side-channel attacks on the confidential information being processed.

**Inherent non-constant nature of smart contract VM:** In addition to the smart contract code not being constant time, the underlying VM executing the contract bytecode representation is likely to leak information. For instance, one may be able to determine which functions are called through cache attacks, as memory access patterns depend on which opcodes are executed. Making a VM implementation that can hide which opcodes are executed together with other information, in the context of SGX, seems non-trivial.

**Smart contract input, state and output length leak:** Consider a user that prepares a confidential transaction by encrypting the method specification and arguments and sends it out to the network. In the case of EVM, the encoding used to specify the method and its arguments is described in the Ethereum smart contract ABI documentation.

The length of that ciphertext will be visible to participants on the network. Suppose the contract has two methods exposed, one with small arguments and another one with lengthy arguments: from the ciphertext length, it will be clear to any network node which method is being called. If the attacker is on the node itself, then monitoring the changes in the length of the smart contract state will likely be beneficial to the attacker.

**Data store, operating system file system and database process query processing access pattern side channels:** Recall that data is stored encrypted outside of SGX and that data must be accessed from the untrusted operating system. If the VM performs lookups to data regions depending on the contract code path, monitoring accesses to encrypted data at the database and OS layers may potentially allow to infer the contract state and the data being processed. [Oblivious RAM](https://dl.acm.org/doi/abs/10.1145/233551.233553) (ORAM) may mitigate some of these side channels by hiding data access patterns; research in this area is currently very active to reduce the inherent increase in resources consumption and to provide practical applications. A naive ORAM implementation access overheard is O(n), making it unsuitable for most purposes indeed. Researchers have been able to reduce this overheard to O(log (n)) for applications such as cloud storage in worse case scenarios.

**Cross-contract calls:** Does the system involve a concept of non-confidential contracts and is it possible to make a call from a confidential to non-confidential contract? In any case, a cross-contract call will reveal the code path that took place in a confidential contract, based on the knowledge of the contract code including the conditions upon which another non-confidential contract is invoked and observation of this non-confidential contract call over the network.

**Tampering with the contract state:** Does the compute node take adequate precautions to detect modifications to data? Can one swap encrypted contract state data values? How about rolling them back to a previous state?

**General TEE side channel attacks:** A large amount of the community’s effort is currently being invested into investigating side channel attacks and mitigations on TEEs. This includes attacks that leverage monitoring enclave’s memory and cache accesses modifications in various scenarios. These attacks clearly have devastating consequences in the context of confidential smart contracts; they may target the encryption keys that span over many transactions or any other confidential information such as business data. In any case, they give the attacker a view of transactions in plaintext.

## Issues with Confidential Smart Contracts, Illustrated

In this section, we will use the “Subcurrency” example smart contract taken from the Solidity [documentation](https://solidity.readthedocs.io/en/latest/introduction-to-smart-contracts.html#subcurrency-example) to illustrate how confidential smart contract execution can leak sensitive information in different parts of a system. It implements a simple currency system on top of Ethereum. Its source code is reproduced in its entirety below:

```solidity
pragma solidity >=0.5.0 <0.7.0;

contract Coin {
    // The keyword "public" makes variables
    // accessible from other contracts
    address public minter;
    mapping (address => uint) public balances;

    // Events allow clients to react to specific
    // contract changes you declare
    event Sent(address from, address to, uint amount);

    // Constructor code is only run when the contract
    // is created
    constructor() public {
        minter = msg.sender;
    }

    // Sends an amount of newly created coins to an address
    // Can only be called by the contract creator
    function mint(address receiver, uint amount) public {
        require(msg.sender == minter);
        require(amount < 1e60);
        balances[receiver] += amount;
    }

    // Sends an amount of existing coins
    // from any caller to an address
    function send(address receiver, uint amount) public {
        require(amount <= balances[msg.sender], "Insufficient balance.");
        balances[msg.sender] -= amount;
        balances[receiver] += amount;
        emit Sent(msg.sender, receiver, amount);
    }
}
```

Let’s demonstrate some ways in which this smart contract running in an SGX enclave may leak potentially sensitive data.

```solidity
 mapping (address => uint) public balances;
```

The `public` keyword modifier creates a reader function to access the balance of an account based on its address. The address and balance are encrypted in transit and at rest. A balance query for an address could be observed in the data store process/logs running outside SGX and on the file system and may be correlated with other information, e.g. the residential IP address from which the query is initiated. The query may also reveal whether the address has an existing balance at the database and file system layers.

```solidity
function mint(address receiver, uint amount)
```

Invocation of the `mint()` function may leak information based on the number and size of encrypted arguments, or even that the `mint()` function was invoked by a specific party, in transit if an attacker is in a privileged network position or in the smart contract database if an attacker has access to the smart contract database query processing layer or files, which runs outside of SGX.

In the `mint()` function:

* A map key is created in storage outside of SGX with an encrypted amount map value in statement `balances[receiver] += amount;`, upon the first invocation of `mint()` with a specific receiver argument. This may permit attackers to correlate the receiver (encrypted map id) with an individual or organization with other information available to the attacker.
* On subsequent invocations of `mint()` with a specific receiver, the associated map value amount changes. This may permit attackers to track the count of minted amounts sent to the receiver.
* Access to data in the query processing layer of the database can be correlated with other events, assisting attackers in identifying users, organizations and amounts involved in a transaction.

Note that we do not have to restrict ourselves to confidentiality attacks. In the case of usage of multiple stand-alone AE (Authenticated Encryption) ciphertexts encrypted with the same key inside a contract state, ciphertext copy/paste attacks can be a problem. Amounts might be replaced by an attacker with older values or other values encrypted in the contract. This may permit attackers to get more money than the set limit.

```solidity
function send(address receiver, uint amount)
```

As in the case of `mint()`, invocation of the `send()` function may leak information based on the number and size of encrypted parameters, or that `send()` was invoked.

In the `send()` function:

* A change in the `balances[id]` map value with map key id may indicate a specific sender or receiver or both were involved in a transaction.
* Early return in the function may indicate the amount of funds available to the sender as part of a side-channel attack.

## Conclusion

When considering the secrecy of transaction data, running confidential smart contracts inside a TEE presents a considerable attack surface to malicious parties with a presence on the host executing the contract and, to a lesser extent, to passive network observers. It should be noted that in decentralized blockchain settings, the owners of nodes are likely to be incentivized to learn the contract inputs of the transactions they are processing.

This attack surface is due to a number of key observed issues:

* Side-channels in smart contract execution.
* Side-channels in data stores, running outside of SGX (key-value databases, file system).
* Ciphertext/plaintext length leaks.

In some way, the promises of Intel SGX, when plugged in to provide confidentiality of smart contracts, are not fulfilled. These issues are not inherent to TEEs, but to the code being fed to TEEs and to the external dependencies that they necessarily interface with. Intel SGX pushes back the confidentiality responsibility to the code it runs and this code is not ready, for now. Much effort has been expended in preventing side-channels in cryptographic code; it is unclear whether such an undertaking can take place for business logic code.

