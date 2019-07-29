---
layout: post
tags: [hyperledger-fabric, Consensus, BitCoin]
author: Jae
comments: true
---

# Consensus 측면에서 본 Hyperledger-Fabric v.s. BitCoin v.s. CoinStack

> 블록체인에 대해서 막 공부하기 시작 할 무렵 여기저기서 정보를 모으고 나름 고민하고 정리해서 적었던 글인데.. 시간이 지나고보니 조잡하고 깊게 조사지 못한 내용도 있다.. 하지만 그 시절의 나는 이렇게 생각했구나 하고 남겨놓고 싶은 마음에 포스팅을 한다.

끝까지 읽으면 해결될지도 모르는 궁금증들

- Private Blockchain(Hyperledger Fabric)이 BlockChain이 맞냐? (그냥 DB 쓰는거랑 뭐가 달라)
- Hyperledger Fabric V1.0 이후에 추가된 Kafka는 Messaging System일 뿐인데 뭔놈의 합의를 한다는거냐?
- Hyperledger Fabric MultiChannel은 도대체 뭐냐? (주 내용과 좀 동떨어 지지만.. Kafka 와 연관이 있는듯 하여..)
- 그래서 Hyperledger Fabric 이냐 Coinstack 이냐 대체 무엇을 써야하는거냐??

## 블록 체인: 내용 변경이 불가능한 공유 원장

- 내용
  - Transaction(거래내역): 생성자가 개인키로 Sign
- 변경 불가능
  - Chain of Hash
- 공유 원장
  - Sync: 합의가 필요

## 합의 알고리즘

- P2P에서 하나의 블록 체인을 유지하는 기술
  - 누가 정한 블록의 체인을 인정할 것인가?

### 경쟁방법

- 경쟁을 통해 하나의 블록체인을 유지하는 방법
  - 경쟁을 하게 되므로 항상 Fork가 발생, 확정성(Finalization)이 없음
  - 종류
    - PoW (Proof of Work)
      - 채굴 능력 == 계산 능력
      - 블록 생성 후 경쟁을 통해서 하나의 블록 체인을 유지
      - 여러 체인들 중 가장 긴 체인을 주 체인으로 합의
      - 블록 체인을 유지하기위해서는 블록을 생성하는 노드간의 경쟁이 필수적이기 때문에 블록 생성에 대한 보상이 필요
      - 내가 가장 많은 보상을 얻기 위해서는 내가 만든 블록이 확정되어야 하고, 내가 만든 블록이 확정되기 위해서는 내가 만든 블록이 가장 긴 체인에 속해있어야 하고, 내가 만든 블록이 가장 긴 체인에 속하기 위해서는 남들보다 더 빨리 계산해서 많은 블록을 만들어 내야 함
      - 경쟁을 통해 가장 긴 체인을 만들어낸 채굴자가 가장 많은 보상을 가지게 되기 때문에, 손해를 보지 않기 위해서 하는 행동이 하나의 블록체인을 유지하게하는 원리
    - PoS (Proof of Stake)
      - 채굴 능력 == 지분(말뚝)
      - fork가 발생할 경우 지분(코인)의 양에 따라서 투표권이 주어지고, 대주주들의 투표 결과를 주 체인으로 인정
      - 코인을 많이 가진 사람들은 시스템이 붕괴 되기를 원치 않으므로 올바른 투표를 할 것
      - Nothing at stake 문제가 발생할 수 있음
        - PoS는 블록이 생성될 때 지분에 비례해서 보상이 지급되기 때문에, fork가 발생했을 때 모든 chain에 투표를 해서 어느 쪽이 선택되던 보상을 받으려는 움직임
        - Ethereum-casper에서는 배팅 개념을 도입해서 잘못 고르면 손해가 나도록 개선

### 비경쟁방법

- 블록에 채굴자의 디지털 서명 등을 삽입하는 방법을 통해서 블록을 구분, 하나의 블록 체인 만을 인정(합의)
- 경쟁이 없으므로 Fork가 일어나지 않아 확정성이 있음
- 종류
  - PBFT (Practical Byzantine Fault Tolerance) - 투표
    - 노드 중 하나가 리더(Primary)가 되며, 자신을 포함해서 모든 노드들에게 합의 요청, 요청에 대한 응답을 집계한 다음 다수결을 이용해 주 블록 체인을 확정
    - PoW나 PoS와는 달리 의사 결정을 먼저 한 다음 블록을 생성하기 때문에 블록체인의 분기가 발생하지 않음
    - 따라서 자연스럽게 확정성을 지원
    - PoW처럼 해답을 찾기 위한 반복적인 연산 과정이 없기 때문에 필요한 리소스 코스트가 낮으며 훨씬 더 좋은 성능으로 동작
    - 악의적인 노드가 위변조를 하려고 해도 과반수 획득을 해야 하는데, 허가된 노드만 참여하기 때문에 과반수 획득이 쉽지 않음
    - 또한 리더가 악의적인 행동을 했다고 판단이 되면 다수결을 통해 리더를 교체할 수도 있기 때문에 매우 강력한 보안을 갖고 있음
    - 하지만, 모든 노드를 알고 있어야 하고 전원의 의사 소통이 필요하기 때문에 노드가 증가할 수록 성능은 감소
    - PoW나 PoS는 수천 개의 노드를 운영할 수 있지만 PBFT는 수십 개의 노드가 한계

## 결론

경쟁 방법은 블록 생성 후 경쟁에 의해서 블록 확정, 비경쟁 방법은 투표를 통한 합의 후 블록 확정

따라서, 노드 수가 제한적이고 관리가 가능하다면, 비경쟁 합의 알고리즘을 통해서 성능은 올리고, 확정성을 가지는 블록체인을 운영할 수 있다.
(사기꾼 노드가 등장해도, 심지어 그게 리더일지라도 블록체인을 확정하고 공유하는데 문제가 없다.)

## 구현체 비교

### BitCoin

- PoW
  - 경쟁방법
  - 전세계 어디서나 누구나 블록을 생성할 수 있기 때문에 경쟁 방법 합의 알고리즘을 택할수 밖에 없음

### Ethereum

- PoS
  - 경쟁방법
  - Casper
    - Nothing at stake를 해결하고, finality를 지원하는 합의 알고리즘(개발중)

### Hyperledger Fabric

- PBFT (V0.6)
  - 비경쟁방법
  - 결제 완결성과 성능 문제를 해결한 합의 알고리즘
  - 모든 참가자를 알아야 하기 때문에 퍼블릭 블록체인에서 적용하기는 어렵다.
  - 의사 결정을 먼저 한 다음 블록을 생성하기 때문에 블록체인의 분기가 발생하지 않는다.
  - 결제 완결성을 지원하며, PoW처럼 해답을 찾기 위한 반복적인 연산 과정이 없기 때문에 필요한 리소스 코스트가 낮으며 훨씬 더 좋은 성능으로 동작한다.
  - 악의적인 사용을 하려고 해도 과반수 획득을 해야 하며, 누구나 노드로 참여할 수 있는게 아니라 허가된 노드만 참여하기 때문에 과반수 획득이 쉽지 않다.
  - 모든 노드를 알고 있어야 하고 전원의 의사 소통이 필요하기 때문에 노드가 증가할 수록 성능은 감소한다.
  - PoW나 PoS는 수천 개의 노드를 운영할 수 있지만 PBFT는 수십 개의 노드가 한계
    - Conference에서 누가 노드수 증가에 따른 성능 문제를 제기했는데 IBM에서 노드수가 많이 늘어나지 않을것이라고 답변 (어쨌든 늘어나면 곤란)

- Kafka(?)  (V1.0 ~)
  - 어짜피 비경쟁 방법이라면?
  - 굳이 노드가 합의부터 블록생성까지 다 할 이유가 없음
  - Hyperledger Fabric 시스템 전체에 합의를 녹여 냄
  - MSP(Membership Service Provider) - Fabric CA, Endorsing Policy 등을 통해서 악의적 사용자 및 잘못된 Transaction을 미리미리 걸러냄 (의사결정을 미리 함)
    - 유효한 트랜잭션인지
    - Sign은 제대로 됐는지 등등..
  - 즉, Kafka는 줄만 잘세우면 됨
  - Endorseing Peer에게 인정 받은 Transaction만 App에 의해서 Orderer에게 전달, Orderer는 내부에서 topic(channel)발행 Kafka(broker)가 줄세움
    - topic = channel
    - Value = transaction
  - Kafka(broker)는 여러 organization의 App으로부터 받은 transaction을 topic(channel)별로 정리, 블록 생성
    - Kafka 특성상 topic이 같아도 partition 간의 순서는 보장하지 않는데 이문제를 어떻게 해결했는지 모르겠음..
  - Commiter Peer(Node)는 자기가 속한 channel 의 topic만 구독
    - commiter는 그냥 이미 생성된 블록을 Append 하기만 함
    - 결국 특정 피어(Process)는 특정 channel에 관한 block만 들고 있음
    - 코드를 다 까본것은 아니지만, World state만을 위한 Docker Container를 따로 띄우지 않는걸로 봐서는 Commiter Peer에 내장되 있는듯
  - Organization 및 node들은 MSP가 인증해줄테고, Transaction은 Endorsing Peer에 의해서 검증됐기 때문에 Kafka(Broker)는 줄만 잘 세우면 됨
  - 합의 기능을 여기저기로 쪼개고 줄세우는것은 Kafka에게 맡김으로써 비경쟁 합의의 장점을 끌어올려 성능 극대화를 노림

### CoinStack (Bitcoin base)

- PoW
  - 경쟁 방법
- RAFT-Variant
  - 비경쟁 방법
  - 노드 중 Mining 권한을 가진 리더 선출 방식
  - Fork 발생 없는 합의 알고리즘 
  - 완결성 (Finalization) 지원
  - 나머지 노드들은 블록 전파

### Hyperledger Fabric v.s. CoinStack

- Hyperledger Fabric
  - 장점
    - 확장성이 좋음
    - 그룹사에서 원장에 붙고 싶으면 알아서 노드 추가하고 dApp 만들어서 붙으면 됨 (MSP에 추가는 해줘야 할 듯)
    - 글로벌 업체들이 사용을 검토하고 있기 때문에 초기 구축만 잘 해놓으면 글로벌 업체들이 이런거 한다 했을 때 따라서 바로 적용할 수 있음
    - Hyperledger Burrow
      - 다른 회사가 Etherium을 사용한다면 Burrow를 이용할 수 있음 (Fabric과의 차이점은 조사 필요)
  - 단점
    - 너무 빠르게 변하고 있어서 따라가기 어려움
    - Kafka, CouchDB, Kubernetese등 Fabric 보다 주변 것들 잘 다루는데 많은 노력이 필요
- CoinStack
  - 장점
    - 국내 개발사의 지원
  - 단점
    - 그룹사 확장시 노드 그룹을 한 곳에서 관리해야 하기 때문에 분산 원장이라고 하기 애매함.. (그냥 또 내부에서 DB를 블록체인으로 쓰는 꼴)
    - SC 호출에 내부 코인을 사용하기 때문에 Coin 관리 이슈가 있음

## Summary

3세대 Blockchain으로 대표되는 EOS 같은 permissionless(public) blockchain platform들이 효율성을 위해 블록체인의 본질인 탈중앙화를 양보하므로써 이도저도 아닌 시스템으로 가고있다는 지적들이 있다.
Fabric의 경우 permissioned blockchain platform으로 허가된 노드들 간의 탈 중앙화를 구현한다. 허가된 노드들만 네트워크에 참여하기때문에 public에 비해서 합의에 드는 비용이 상대적으로 적고 부수효과로 자연스럽게 성능이 증가하게 된다.
블록체인을 모든 사람이 참여할 수 있는 탈 중앙화로 본다면 fabric은 블록체인이 아니라고 할수도 있겠지만 fabric 역시 permissioned 환경 안에서 참여자간의 탈 중앙화를 제공한다.

## Reference

- http://snowdeer.github.io/blockchain/2018/01/18/blockchain-seminar-consensus-algorithm/
- https://steemit.com/kr/@loum/3
- http://hyperledger-fabric.readthedocs.io/en/latest/
- https://www.ddengle.com/traders_free/1436821
- https://steemit.com/kr/@jsralph/proof-of-stake-1-pow-vs-pos
- https://www.slideshare.net/YounghoonMoon1/casper-ethereum-pos
- https://steemkr.com/kr/@justfinance/1-by-jon-choi
- https://m.blog.naver.com/ehdtmdcouple/221182264880
