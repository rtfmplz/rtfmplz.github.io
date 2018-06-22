# Hyperledger Fabric Q&A

> 본 포스트의 내용은 아래의 컨퍼런스와 교육의 Q&A 시간에 나온 질문과 답변을 개인의 공부 및 정리를 목적으로 작성한 것 입니다. 사실과 다르거나 잘못된 내용이 있으면 댓글로 남겨주시면 수정하겠습니다. (사실 확인이 필요한 부분이 많습니다. 참고용으로만 사용 부탁드립니다.)
> 개인 공부 목적으로 작성한 것이기 때문에 질문의 순서가 다소 산만할 수 있습니다.
> 
> * 2018 IBM DEV DAY (https://goo.gl/8EFCqD)
> * 하이퍼레저 패브릭을 활용한 상품 거래 시스템 구현 (https://programmers.co.kr/learn/courses/4037)

```
Q. Chaincode는 endorser가 실행하고 block에 append는 commiter가 하는데 실제 변경된 데이터가 반영되는 시점은 언제인가요?

A. Commiter가 Transaction의 유효성을 검증 한 후, 블록을 append 하는 시점에 World State에 반영됩니다.
```

```
Q. 그럼 참여자 A가 Endorser에게 Transaction을 제출하고 실제 원장에 반영되기 전 다른 참여자가 데이터를 조회하는 상황이 발생하면 Consistency가 보장되지 않는 것 아닌가요?

A. Fabric 네트워크 내 참여자 간의 Consistency를 보장하기 위해서는 Application이 이를 보장할 수 있도록 구현되어야 합니다. Consistency를 보장하기위한 방법으로 EventListener를 이용할 수 있습니다. Transcation 처리 과정에서 Application은 Endorser에게 response를 받은 이후 response를 받지 않습니다. Transaction이 commit 되는 과정에서 orderer를 거치기 때문에 response를 받기가 애매하기 때문입니다. 따라서 블록에 저장이 완료가 되면 Commiter로 부터 저장이 완료된 것을 event로 받을 수 있도록 되어 있고 Application은 이벤트를 수신 함으로써 원장의 데이터가 업데이트 된 것을 인지 할 수 있습다.
```

```
Q. Transaction ordrering은 timestamp 기준으로 하나요?

A. Kafka queue에 들어간 순서로 ordering 되며, 큐에는 시간(Orderer에 들어온 시간) 순서대로 쌓이게 됩니다. 추가로 큐는 개별 채널에 따라 만들어집니다. 

C. Transaction 생성 순서가 되어야 맞는것 아닌가..?
```

```
Q. queue에 들어가는 순서대로 orderring 한다면 transaction의 순서를 보장할 수 없지 않나요?

A. 
        * 카프카는 그룹별 컨슈머별 큐를 만들기 때문에 별 문제 없음
        * 카프카 큐에는 시간 순서대로 들어가며 시간은 orderer 서버에 들어온 시간

    * 1~5번이 처리되는 도중에 데이터가 꼬일 개연성이 충분함
        * 공유 원장이라는 말을 다시 기억해야 함
        * 모든것을 블록에 쌓는것이 아니야
        * db로 처리해야 할것들은 db로 처리한 후에 원장에 해당하는 데이터만 쌓도록 설계해야 한다.
```


```
Q. Endorsing policy 통과 후, Orderer에 submit한 transaction이 commit에 실패 할 수 있나요?

A. 네트워크 문제로 Orderer까지 전달이 되지 않거나, Commiter에 의해서 validation에 실패하는 경우 commit에 실패할 수 있습다. Commit에 실패한 Transcation은 Application에 의해서 처리되어야 합니다. 특히, Commiter가 validation 실패한 경우 event 수신에 대한 timeout 등을 설정하여 처리할 수 있습니다.
```

```
Q. Membership service, orderer service 둘 중 하나라도 셧다운 된다면 전체 시스템이 동작할 수 없나요?

A. 네, 따라서 Membership service와 orderer는 HA를 보장하도록 구성해야 합니다. Production 환경에서 Kafka 브로커는 9개, Zookeeper는 5개를 두는 것이 일반적입니다.
```

```
Q. Commiter에 의한 Validation 과정에서 블록에 포함된 Transaction중 하나라도 Validation에 실패하게 되면 전체 Transaction이 다 실패하나요?

A. 네, 블록 전체가 다 revoke 됩니다.

C. 이후 복구 과정은 어떻게 되는거지?
```

```
Q. Peer들의 리스트는 membership service가 가지고 있나요?

A. Application의 입장에서 peer 리스트는 설정 파일로 관리해야 합니다. Anchor peer(org의 peer 들을 관리하는 피어) 역시 peer 정보를 가지고 있어야 합니다. Fabric Network는 계획된 네트워크이기 때문에 설정 파일에 의한 관리가 가능합니다.
```

```
Q. Application이 Transaction propose 할 때 endorsing peer가 여러개라면 모든 peer 에 다 propose 하나요?

A. Endorsement policy에 의해 반드시 서명을 받아야 하는 peer 리스트에 있는 peer 에게 모두 Proposal을 보내게 됩니다.
```

```
Q. Chaincode 없이 Couch DB에 접근이 가능한데, World State가 위변조 되었다고 의심될 때, Blockchain을 이용해서 World State 를 replay 하는 기능은 없나요?

A. 아직까지 없습니다. 다만, 제공되는 kafka image를 보면 retention policy를 영구 저장으로 설정하고 있기 떄문에 위변조가 의심되는 상황에서 kafka 데이터를 확인 할 수 있습니다. Couch DB 또한 ACL 기능을 가지고 있기 때문에 정책적으로 접근을 관리 할 수 있습니다.

C. Kafka 설정 확인 해보기
```

```
Q. Chaincode shim package 를 보면 DelState 함수는 블록체인에서 해당 내용을 삭제하는 건가요

A. Blockchain 상의 데이터는 어떤 경우에도 삭제될 수 없습니다. DelState 역시 chaincode로서 Blockchain에 Transaction으로 남게 됩니다. 다만, World State에서 key를 삭제 함으로써 원장에서 더이상 검색할 수 없도록 동작합니다. 비지니스 로직 상 검색이 불가능하게 해야 할 때 DelState를 사용할 수 있습니다.
```

```
Q. 채널 별로 leadger가 관리되지만, World State는 모든 채널이 공유하는 구조인 것 같은데, Couchdb를 여러개 띄워서 World State를 ledger 별로 구성할 수는 없나요?

A. Fabric 로드맵 상으로는 계획이 있지만, 언제 반영될지는 알 수 없습니다.
```

```
Q. Endorser에 의해 서명이 완료된 후, 악의적인 Application에 의해 내용이 변경되어 Orderer에게 submit 된다면 어떻게 검증 하나요?

A. Commiter가 Transaction에 대해서 Validation을 체크하는 과정에서 걸러지게 됩니다.
```

```
Q. Fabric Component들은 Docker image로 배포되는데 특별히 추천하는 오케스트레이션 툴이 있나요?

A. Fabric Component들은 컨테이너 기반으로 동작하기 때문에 Production 환경의 안정적 운영을 위해서는 오케스트레이션 툴이 필수적입니다. 최근 Kubernetes가 많이 선호되는 추세입니다.
```

```
Q. Fabric 문서상 1000 TPS를 이야기 하고 있는데 실제로 샘플 네트워크를 구동해보면 그렇지 않은것 처럼 보입니다.

A. 1000 TPS는 kafka Producer를 Async Type으로 구동했을 때의 수치 입니다. 비동기 producer를 사용할 경우 메시지를 일정 시간 큐에 쌓아 두었다가 한 번에 전송하므로 producer의 메시지 처리량을 향상시킬 수 있습니다. 단, 장애가 발생할 경우 전송하지 않고 쌓아 둔 메시지가 유실될 우려가 있습니다.
Fabric에서 TPS를 높이기 위해서는 SDK(Application)를 횡으로 확장, 수를 늘리고, Transaction이 Orderer에 몰리도록 설계하는게 필요합니다.
```

```
Q. SmartContract로 구현되어야 하는 내용들은 어떤것들이 될까요?

A. SmartContract는 Database의 Procedure에 대응합니다. 따라서 Business logic에 구현되어야 합니다. 추가로 원장의 데이터에 대한 접근을 관리하는 기능을 포함할 수 있습니다.
```

```
Q. Fabric 네트워크 운영 도중 Org 또는 channel의 추가 등 Fabric network의 설정을 변경할 수 있나요? 

A. Configuration 또한 Transaction으로 관리되며 Fabric 네트워크에 Configration Transaction을 submit 함으로써 설정을 변경할 수 있습니다. 자세한 내용은 아래 페이지에서 확인할 수 있습니다.

* https://hyperledger-fabric.readthedocs.io/en/release-1.1/config_update.html#updating-a-channel-configuration
```

```
Q. Peer가 Endorser로서 동작하게 하려면 어떤 설정을 해야 하나요?

A. 컨테이너의 /etc/hyperledger/fabric 경로에서 core.yaml 파일을 변경하여 설정 할 수 있습니다. 또는 Docker-compose 파일을 통해서 환경 설정을 전달할 수 있습니다.

C. 실제로 찾아보면 core.yaml 파일에 endorser on 설정이 없다. 어디에 있는건지 확인 필요.
```

```
Q. Fabric-CA는 Production 환경에서 사용할 만큼 잘 구현되어 있나요?

A. 사용해본 사람들에 의하면 External-CA를 사용하는 편을 추천합니다.
```

```
Q. Dev 모드에서 chaincode container 에서 chaincode build 하고 실행하게 되는데 cli container에서 install이 불필요 한 것 아닌가요?

A. 일반적인 경우 Chaincode install, Instantiate는 다음과 같은 동작을 합니다.

* Install : chaincode source를 peer 에 배포
* Instantiate: peer에서 chaincode를 빌드 chaincode container 실행

개발 모드에서는 chaincode container 에서 chaincode를 직접 빌드하여 실행하지만
install, instantiate를 함으로써 chaincode 이름, 버전, 채널 등을 Application에 알려주는 역할을 합니다.

C. 아래 링크에 따르면 install 절차는 없어질 것으로 예상 됨
* https://hyperledger-fabric.readthedocs.io/en/release-1.1/chaincode4ade.html#terminal-3-use-the-chaincode
```

```
Q. Transaction의 내용이 World State에 반영되기 전에 읽으려고 한다면? (Consistency)

A. 블록체인에 저장되어야 할 데이터에 대한 이해가 필요합니다. Business Logic을 처리하는 동안 빈번히 변하는 데이터들은 기존의 Database를 이용해서 관리하고, 블록체인에는 계약서와 같이 한번 결정되면 쉽게 변하지 않으면서 여러 사람이 열람해야하는 데이터(변하지 않을)를 저장하는것이 바람직합니다. 이런 경우 TPS는 큰 의미가 없어집니다.
```

```
Q. V1.0 Architecture에서 Orderer는 합의의 한 부분이고, Kafka로 선택한 경우, 시간 순서로 정렬하는 역할만 한다고 하였는데, 새로 추가되는 SBFT는 PBFT와 비슷하다고 한다면 Orderer 역할만 하는게 아닌, 합의 전반을 처리하게되는것 아닌가요?

A. Orderer로 SBFT를 사용하게 되는경우 Transaction 검증은 Peer에서 하고 Orderer 간의 합의에 의해서 Transaction을 정렬하게 할 수 있습니다.
```

```
Q. Fabric network 띄우고 chaincode install, instantiate 하면 chaincode container가 peer의 수만큼 생성되어야 하는데 하나만 생성됩니다.

A. Chaincode container는 각 peer에서 chaincode를 실행하는 시점에 생성됩니다.
Instantiate는 chaincode 별로 한번만 수행하면 되기 때문에 예제를 수행하는 과정에서Instantiate 까지만 수행하면 chaincode container가 하나만 구동되는 것 처럼 보일 수 있습니다. 이후에 다른 peer에서 query나 invoke Transaction이 실행되면  chaincode container peer와 짝을 이루는 chaincode container가 실행되게 됩니다.
```

```
Q. Extra State DB로 다른 Couch DB 이외의 다른 Nosql을 사용할 수는 없나요?

A. RDB를 사용할 수 있도록 지원될 예정입니다.
```

```
Q. 데이터가 계속 쌓이다보면 Blockchain을 Backup 해야 하는 이슈가 생길텐데 이에대한 대응책이 있나요?

A. 일단 블록체인에는 저장되는 데이터는 파일과 같은 대용량 데이터가 아닌, chaincode 호출을 위한 Transaction과 같은 용량이 작은 데이터가 들어가야 합니다. 예를 들어 음원에 대한 저작권을 관리하는 시스템이라고 한다면 음원은 외부 스토리지에 저장하고 블록체인에는 작곡가, 가격, 계약사항들이 포함될 수 있습니다. 
그럼에도 불구하고 Backup solution은 필요하기 때문에 Hyperledger Fabric은 이 부분에 대해서 문서화 하고 릴리즈할 예정에 있습니다.
```