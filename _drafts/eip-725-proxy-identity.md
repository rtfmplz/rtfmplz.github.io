# EIP 725: Proxy Identity 

```
Author: Fabian Vogelsteller (ERC-20 Coauthor, Vitalik Buterin)
Discussions-To: https://github.com/ethereum/EIPs/issues/725
Status: Draft
Type: Standards Track
Category: ERC
Created: 2017-10-02
```

## Simple Summary
Blockchain ID를 설정하기위한 주요 관리 및 실행을위한 프록시 계약

## Abstract
이어지는 글에서는 인간, 그룹, 오브젝트 및 기계를 위한 Unique ID에 대한 표준 기능을 설명한다.
이 ID는 action(transactions, documents, logins, access, etc), issuer로 부터 발행된 claims 및 self attested([ERC735](https://github.com/ethereum/EIPs/issues/735))에 sign 할 수 있는 key와, blockchain에 직접 행위 할 수 있는 proxy function을 가지고 있다.


## Motivation
이 표준화 된 ID 인터페이스는 Dapps, SmartContract 및 제 3 자에게 기능 XXX에서 설명한대로 2 단계를 거쳐 사람, 조직, 대상 또는 기계의 유효성을 확인할 수있게합니다. 
신뢰는 여기에서 청구권의 발행인에게 이전됩니다.

* 신원을 확인하는 가장 중요한 기능은 다음과 같습니다. XXX
* ID를 관리하는 가장 중요한 기능은 다음과 같습니다. XXX


## Definitions
* keys: EOA(Externally Owned Account)의 Public key, 또는 CA(Contract Address)
* claim issuer: 또하나의 Smart Contract 또는 EOA, Identity에 대한 claim을 발행한다. 또는, identity contract 그 자체일 수 있다.
* claim: 자세한건 [ERC735](https://github.com/ethereum/EIPs/issues/735)를 보시오.


## Specification

### Key Management
#### getKey
#### keyHasPurpose
#### getKeysByPurpose
#### addKey
#### removeKey

### Identity usage
#### execute
#### approve

### Identity verification
#### addClaim
#### removeClaim

### Events
#### KeyAdded
#### KeyRemoved
#### ExecutionRequested
#### Executed
#### Approved
#### ClaimRequested
#### ClaimAdded


## Constraints
* A claim can only be one type per type per issuer.
* claim은 issuer 당 type 당 하나의 type 이다.


## Rationale



## Implementation
* [DID resolver specification](https://github.com/WebOfTrustInfo/rebooting-the-web-of-trust-spring2018/blob/master/topics-and-advance-readings/DID-Method-erc725.md)
* [Implementation by mirceapasoi](https://github.com/mirceapasoi/erc725-735)
* [Implementation by Nick Poulden](https://github.com/OriginProtocol/identity-playground), interface at: [https://erc725.originprotocol.com/](https://erc725.originprotocol.com/)

### Solidity Interface
```java
pragma solidity ^0.4.18;

contract ERC725 {

    uint256 constant MANAGEMENT_KEY = 1;
    uint256 constant ACTION_KEY = 2;
    uint256 constant CLAIM_SIGNER_KEY = 3;
    uint256 constant ENCRYPTION_KEY = 4;

    event KeyAdded(bytes32 indexed key, uint256 indexed purpose, uint256 indexed keyType);
    event KeyRemoved(bytes32 indexed key, uint256 indexed purpose, uint256 indexed keyType);
    event ExecutionRequested(uint256 indexed executionId, address indexed to, uint256 indexed value, bytes data);
    event Executed(uint256 indexed executionId, address indexed to, uint256 indexed value, bytes data);
    event Approved(uint256 indexed executionId, bool approved);

    struct Key {
        uint256 purpose; //e.g., MANAGEMENT_KEY = 1, ACTION_KEY = 2, etc.
        uint256 keyType; // e.g. 1 = ECDSA, 2 = RSA, etc.
        bytes32 key;
    }

    function getKey(bytes32 _key) public constant returns(uint256[] purposes, uint256 keyType, bytes32 key);
    function keyHasPurpose(bytes32 _key, uint256 purpose) constant returns(bool exists);
    function getKeysByPurpose(uint256 _purpose) public constant returns(bytes32[] keys);
    function addKey(bytes32 _key, uint256 _purpose, uint256 _keyType) public returns (bool success);
    function execute(address _to, uint256 _value, bytes _data) public returns (uint256 executionId);
    function approve(uint256 _id, bool _approve) public returns (bool success);
}
```

## Additional References
* [Slides of the ERC Identity presentation](https://www.slideshare.net/FabianVogelsteller/erc-725-identity)
* [In-contract claim VS claim registry](https://github.com/ethereum/wiki/wiki/ERC-735:-Claim-Holder-Registry-vs.-in-contract)
* [Identity related reports](http://www.weboftrust.info/specs.html)
* [W3C Verifiable Claims Use Cases](https://w3c.github.io/vc-use-cases/)
* [Decentralised Identity Foundation](http://identity.foundation/)
* [Sovrin Foundation Self Sovereign Identity](https://sovrin.org/wp-content/uploads/2017/06/The-Inevitable-Rise-of-Self-Sovereign-Identity.pdf)
