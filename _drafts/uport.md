# UPORT: A PLATFORM FOR SELF-SOVEREIGN IDENTITY

> 본 Post는 uport white-paper를 정리한 것입니다. 상세한 내용은 원문을 참조 부탁드립니다.
>
> * http://blockchainlab.com/pdf/uPort_whitepaper_DRAFT20161020.pdf


## INTRODUCTION

### History

* 기존 온라인 인증시스템에 문제가 많음
	* centralized data를 양성해 해커의 표적이 된다.
* public/private key 암호화와 Blockchain 같은 decentralized 기술이 대안이 될 수 있음
* 공개키 암호화 기술은 생긴지 오래됐지만 사용자 접근성이 떨어져서 잘 사용되지 않았다.
	* Blockchain이 모든 것을 바꿔놨다. (역시 돈이 껴야..)
* 블록체인 지갑은 기술을 잘 알지못하면 사용하기 어렵다.
* 블록체인을 ID를 public-key와 연결시켜주는 분산 CA로 볼 수 있다. 
* Smart Contract는 key 관리를 도와줄 수 있다.

### Introduction to Uport

* uPort는 ehtereum 기반의 SSID System
* main component
	* Smart Contract
		* mobile app(pirvate key)을 분실 했을 때의 코어 로직 구현
	* Developer libraris
		* Support to develp 3rd-party app
	* mobile app
		* hold private(user) key
* key concepts
	* self-sovereign
		* Identities fully owned and controlled by creator.
	* decentralized
		* don't rely on centralized 3rd-parties
* off-chain data sotres
	* identity는 암호화되어 off-chain data sotre에 저장된다. (e.g. dorpbox, aws)

### Propose Use Cases

* End-user with uPort
	* 개인 신원, 평판, 데이터 및 디지털 자산을 소유하고 제어
	* 데이터를 상대방에게 안전하고 선택적으로 공개 
	* 암호를 사용하지 않고 디지털 서비스를 이용
	* 청구, 거래 및 문서에 디지털 서명
	* 블록 체인에서 값을 제어하고 보냄
	* 분산 된 애플리케이션 및 Smart Contract와 상호 작용
	* 메시지와 데이터를 암호화
* Enterprises with uPort
	* 기업의 정체성을 확립
	* 새로운 고객 및 직원에게 쉽게 적용
	* 개선되고 이행적인 Know-Your-Customer 프로세스를 수립
	* 직원을위한 마찰이 적은 안전한 액세스 제어 환경을 구축
	* 민감한 고객 정보를 보유하지 않음으로써 책임을 줄임
	* 준수(compliance) 증가
	* 벤더 네트워크 유지
	* 특정 사용 권한을 가진 역할 특정, 액터 - 알 수없는 ID (즉, CTO)를 설정합니다.


## TECHNICAL OVERVIEW


### Ethereum and Smart Contracts

### uPort Technical Overview

### Technical Components

**Smart Contract Components**

* The ​Proxy Contract is a minimal contract, used to forward transactions and its address is the core identifier of a uPort identity.
* The ​Controller Contract maintains access control over the Proxy contract, and allows for additional functionalities.
* The ​Recovery Quorum Contract facilitates identity recovery in case of key loss.
* The ​Registry Contract maintains cryptographic bindings between a uPort identifier and the off-chain data attributes associated with it.

**Data Components**

* Attestations are signed data records containing profile attributes and/or verifiable claims, stored off-chain.
* Selective disclosure is a mechanism for encrypting data that adds a layer of privacy to attributes and attestations.

**Developer Components**

* A ​Developer library allows for simple integration of uPort into decentralized applications or existing digital services.

**Mobile Components**

* A ​Mobile application stores the identity’s private key, which is used to control the identity and sign attestations, in the smartphone’s secure enclave.

**Server Components**

* Chasqui - messaging server
* Sensui - gas fueling server
* Infura RPC
* Infura IPFS