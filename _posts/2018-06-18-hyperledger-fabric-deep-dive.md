---
layout: default
title: Hyperledger Fabric v1.0 - Deep Dive
category: post
author: Jae
comments: true
---

# Hyperledger Fabric v1.0 - Deep Dive

> * 본 포스트의 내용은 개인의 공부 및 정리를 목적으로 작성한 것 입니다. 혹시 사실과 다르거나 잘못된 내용이 있으면 댓글로 남겨주시면 수정하겠습니다.
> * 본 포스트에서 사용된 그림들은 "Brian Behlendorf"과 "Jonathan Levi"가 만든 "Hyperledger Fabric v1.0 - Deep Dive"	Slide에서 스크랩 하였습니다. 문제가 될 경우 삭제될 수 있습니다.

## What is Hyperledger Fabric

### Hyperledger

* Open Source
* For Cross-Industry blockchain
* Hosted by The Linux Foundation

### Hyperledger Fabric

* Blockchain Framework
* Hyperledger Project 에서 처음으로 incubation 탈출
* Key technical features
	* Shared ledger
	* Chaincode (SmartContract)
	* Privacy and permissioning through membership services
		* Private Blockchain으로 합의된 사용자만 네트워크에서 Transaction을 일으키지만 비지니스 차원에서 특정 사용자만 데이터를 열람할 수 있는 기능을 지원(ACL)
	* Modular architecture and flexible hosting options
		* 이론적으로는 hyperledger를 구성하는 모든 컴포넌트는 내맘대로 만들어서 끼어넣을 수 있음
		* Membership Service Provider
			* Fabric-CA
			* External-CA
		* Orderer
			* Solo
			* kafka
			* SBFT (likely PBFT)


## Reference Architecture

![reference-architecture](/images/posts/hyperledger-fabric-deep-dive/reference-architecture.png)

* IDENTITY
	* ACL
	* Audit
* LEDGER, TRANSACTIONS
	* Distributed Ledger
* SMART CONTRACT
	* Programmable ledger (likely Procedure)
	* Run business logic
* APIs, Events, SDKs
	* Go
	* Node.js


## Architecture

### Overview of v1.0 Design Goals

* 비지니스 프로세스를 반영하기 쉽도록 개선
	* ACL
	* Endorsement
	* Channel
	* Modular
* Support rich data query of the ledger (Couch DB)
* network와 chaincode를 동적으로 upgarde 할 수 있음
* 기타 등등...

### v0.6 Architecture

![fabric-0.6](/images/posts/hyperledger-fabric-deep-dive/fabric-0.6.png)

* PBFT를 사용, 노드가 최소 4대이고 그중에 3대만 만족하면 데이터가 저장되는 구조
* Peer가 체인코드 실행, leader 관리, 합의까지 모든 것을 전담
* Membership 공인인증서를 발행하지만, 없어도 문제 없었음
* Application은 membership service를 통해서 network에 enroll
* 인증서와 Chaincode를 포함한 Transaction을 Peer에 제출

### v1.0 Architecture

![fabric-1.0](/images/posts/hyperledger-fabric-deep-dive/fabric-1.0.png)

* Peer의 역할을 분리
	* 하나의 트랜잭션을 처리하는데 7단계를 거쳐야 함
	* Consensus의 확장, 7단계를 다 거치면 하나의 Consensus가 처리되는 구조
* 단계
    1. 클라이언트가 MSP에 enroll, 인증서 발급
    2. 인증서 정보, SmartContract Method와 Argument를 실어서 endorser에 transaction proposal
        * endorser는 validation 하거나 서명을 해주는 노드들
    3. Endorser가 Transaction을 validation 후, 서명
        * Execute chaincode
    4. 검증 완료 후 보증된 response를 반환
    5. 클라이언트가 모든 endorser들에게 검증을 받았다고 판단하면 Orderer에게 Transaction을 submit
    6. Orderer는 채널, 체인코드 이름, 버전 별로 transaction을 분류 및 정렬 후 블록 생성한 후, Commiter에게 block을 deliver (batch)
    7. Commiter는 validation 후 block을 append
		* Transaction을 endorsement policy에 대해서 검증하고, RWset이 유효한지 확인
		* 검증 완료 후 원장에 Transaction이 쓰여지고 State Database에 update
	8. Commiter는 원장의 내용이 정상적으로 update 되었음을 event를 통해서 Application에 전달

> **Consensus redefined**
> 
> * Consensus = Transaction endorsement + Ordering + Validation
> * Transaction endorsement: 참여자들이 Transaction을 처리하는 기준
> * Ordering: 채널 별 Chaincode 별 Transaction 들을 시간순서대로 정렬
> * Validation: endorsement를 만족하는지, ordering 되어 있는지, chaincode 내용이 맞는지 등등 검증
> 
> **Transaction Endorsement**
> 
> * Endorsement는 Transaction이 Endorser에 실행 결과가 문제 없음을 서명한 결과물
> * Endorsement policy는 참여자들에 의해서 받아들여져야 하는 Transaction의 요구사항
> * Endorsement policy는 채널의 체인 코드 인스턴스화 중에 지정
> * 각각의 채널 별 체인 코드는 다른 Endorsement policy를 가질 수 있음


## Components

### Orderer

* Transaction을 정렬하는 역할
* Genesis of a network
    * Orderer를 통해 채널 생성을 하는것이 Fabric Network의 시작
	* 모든 Configuration 트랜잭션을 처리하여 네트워크 정책을 설정
* Orderer를 Kafaka로 운영할 경우
    * Kafka Broker는 3개 이상, Zookeeper도 3개, Orderring service가 별도의 Container로써 필요하기 때문에 상당히 많은 Container가 동작 (Orderer라고 박스 하나 있는게 Container 하나가 아님, Kubernetes 필수)
    * HA를 요구하는 Production 환경에서 Kafka Broker는 보통 9개이상을 권장
    * Zookeeper는 5개 (너무 많으면 Network 부하가 많아져서 좋지 않음)
    * 여러 기업이 운영하는 Blockchain 환경이라면 Orderer는 어디에..?
        * Cloud
        * peer는 원장을 저장하니까 사내에

### Peer

* Endorser
	* Transaction을 실행하고 보증
* Commiter
	* Transaction의 결과를 검증
* Endorser 와 commiter는 하나의 컨테이너이고, 컨테이너가 실행될 때 configuration으로 endorser 역할을 할지 말지 지정하게 되어 있음
* Event 발행
	* After execute chaincode
	* After append block
* Gossip Network (Protocol)

### Ledger

* Key-value store
* Default level DB
* External로 Couch DB를 사용할 수 있음
* RDB도 지원 예정
* Couch DB를 쓰는 이유
    * 기존의 key-value Store로는 조회용 데이터를 만들기 쉽지 않음
    * Couch DB는 다양한 query를 쓸 수 있는 환경을 만들어 줌
    * 원칙은 SC를 통해서 데이터를 query해야 하지만 단순한 통계, 모니터링 화면등을 위해서는 Couch db를 쓰는것도 방법이 될 수 있음

### Channels

* 내가 거래하고 싶은 peer 들의 조합
* 이해 당사자들끼리만 통신할 수 있는 통신 채널
* 하나의 peer는 여러개의 채널에 join 할 수 있다.
* ledger는 채널 별로 관리
* 하나의 채널에는 여러개의 chaincode를 deploy할 수 있다.

### MSP(membership service provider)

* 조직을 나누는 기술
* Identity의 역할
    * 인증서 발급
    * 사용자 등록
    * Fabric network component validation
* MSP ID로 조직을 구분해서 나누고 MSP ID 하나당 Fabric CA가 하나씩 존재
* 구현체는 Fabric CA를 쓸 수도 있지만, 회사 내부의 CA를 쓸 수 있음


## Fabric Network

### Bootstrapping a Network

* Orderer가 제일 처음 뜸
* Orderer가 control 할 member들을 결정
* Network에 참여 할 Peer의 수 결정
* 이 때, Peer는 Orderer 또는 다른 Peer들과 연결되지는 않음

### Setting up Channels, Policies, and Chaincodes

* Orderer가 올라왔으면 Orderer가 Channel을 생성 (Configuration Transaction을 사용)
* Peer가 Orderer를 통해 Channel에 join 요청
* Channel에 chaincode를 deploy (Peer에 Chaincode install)
* Chaincode를 instanciate 할 때 Endorsement policy를 함께 배포


## Sample Transaction

> 아래 소개된 Fabric Network에서 Endorsement policy(Ap)는 chaincode A를 instantiate 할 때 배포되었다고 가정
> 
> Endorsement policy: E0, E1 and E2 must sign

### Step 1/7 - Propose transaction

![transation-step-1](/images/posts/hyperledger-fabric-deep-dive/transation-step-1.png)

* Application propose transaction
	* Application이 Chaincode A에 대한 transaction을 생성하여 Endorsement Policy를 만족하기 위해서 {E0, E1, E2} 에 Transaction 제출

### Step 2/7 - Execute proposal

![transation-step-2](/images/posts/hyperledger-fabric-deep-dive/transation-step-2.png)

* Endorsers Execute Proposals
	* E0, E1, E2 각각은 제출된 Transaction을 실행
	* 이 때 원장에 update는 일어나지 않음
	* 각각의 실행은 원장으로부터 Read set을, Chaincode의 실행 결과로 Write set을 생성

### Step 3/7 - Proposal response

![transation-step-3](/images/posts/hyperledger-fabric-deep-dive/transation-step-3.png)

* Application receives responses
	* endorser는 RWSet에 서명을 한 후, Application에 반환

### Step 4/7 - Order transaction

![transation-step-4](/images/posts/hyperledger-fabric-deep-dive/transation-step-4.png)

* Application submits responses for ordering
	* 모든 endorser로 부터 sign된 response를 받았다면(Endorsement policy를 만족했다면), Application은 해당 response를 orderer에게 transaction으로 제출
	* 이 때, 다른 응용 프로그램에서 제출 한 트랜잭션과 병렬 처리됨

### Step 5/7 - Deliver transaction

![transation-step-5](/images/posts/hyperledger-fabric-deep-dive/transation-step-5.png)

* Orderer delivers to all committing peers
	* Orderer는 Application들로 부터 제출된 Transaction을 채널별, Chaincode별 분류 후 시간순서대로 정렬
	* 정렬된 Transaction의 수가 일정 수를 초과하거나 일정 시간이 지나면 블록을 생성, 모든 Committing peer에게 전달
	* Peer는 gossip protocol을 상용하여 다른 peer 에게 전파

### Step 6/7 - Validate Transaction

![transation-step-6](/images/posts/hyperledger-fabric-deep-dive/transation-step-6.png)

* Committing peers validate transactions
	* Committing peer 들은 Transaction을 endorsement policy에 대해서 검증하고, RWset이 유효한지 확인.
	* 검증이 완료되면 Transaction은 원장에 쓰여지고, WorldState를 upate

### Step 7/7 - Notify Transaction

![transation-step-7](/images/posts/hyperledger-fabric-deep-dive/transation-step-7.png)

* Committing peers notify applications
	* Application은 Peer로 부터 Chaincode 실행의 성공/실패와 원장의 update에 대한 event를 받아서 처리할 수 있다.


## Single Channel Network

![single-channel-network](/images/posts/hyperledger-fabric-deep-dive/single-channel-network.png)

* 0.6의 PBFT 와 비슷
* 모든 Peer들은 같은 채널로 연결되어 있음
* 모든 Peer는 같은 chaincode를 가지고, 같은 원장을 유지함


## Multi-Channel Network

![multi-channel-network](/images/posts/hyperledger-fabric-deep-dive/multi-channel-network.png)

* Peer 들은 여러 채널을 통해 복수의 원장을 가질 수 있음
* 클라이언트는 채널 명과 Chaincode 명을 실어서 Transaction을 제출
