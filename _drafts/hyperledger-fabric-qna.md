* Chaincode는 endorser가 실행시키고 block에 append는 commiter가 하는데 실제 바뀐 데이터가 반영되는 시점은?
    * 7번 commiter가 블록을 append 하는 시점

* 그럼 클라이언트가 1번에서 쓰고 7번에서 반영되기 전에 다른데서 조회하면?
    * Application 개발할 때 잘 처리해 줘야한다. 뭐?//////?/????????
    * 클라이언트는 endorser에게 response를 받고 다음부터는 response를 받지 않는다. commit에 orderer를 거치기 때문에 response를 받기가 애매하기 때문에 블록에 저장이 완료가 되면 저장이 완료된 것을 event로 받을 수 있음
    * 1번부터 7번까지 atomic 하게 동작할 수 있도록 클라이언트가 관리를 해줘야 한다.

* Transaction ordrering은 timestamp로 하나?
    * Kafka queue에 들어간 순서
    * queue는 순차성을 보장못할 수 있지 않나?
        * 카프카는 그룹별 컨슈머별 큐를 만들기 때문에 별 문제 없음
        * 카프카 큐에는 시간 순서대로 들어가며 시간은 orderer 서버에 들어온 시간
    * 1~5번이 처리되는 도중에 데이터가 꼬일 개연성이 충분함
        * 공유 원장이라는 말을 다시 기억해야 함
        * 모든것을 블록에 쌓는것이 아니야
        * db로 처리해야 할것들은 db로 처리한 후에 원장에 해당하는 데이터만 쌓도록 설계해야 한다.

* 4번에 submit한 transaction이 fail 할 수 있나?
    * orderer에 들어만 가면 전송 보장을 해줘
    * 근데 orderer에 못들어 간다던지, commiter에서 validation 에 실패하면 fail할 수 있다.
    * orderer에 못들어 간건 클라이언트가 exception처리를 해줘야 하고
    * commiter가 validation실패한건 event 수신하고 있다가 클라이언트가 처리해 줘야 한다. 

* Membership service, orderer service 둘 중 하나라도 셧다운 된다면 전체 시스템이 도작할 수 없나?
    * Membership service는 클러스터링 하도록 되어있음
    * orderer는 브로커를 보통 9개 이상 나누도록 되어 있음
    * HA를 보장하도록 구성해야 함

* 7번에서 validation에서 특정 transaction에의 validation에 실패하면 블록 전체가 다 revoke 된다.

* peer들의 리스트는 membership이 가지고 있나?
    * 클라이언트 입장에서 peer 리스트는 설정 파일로 잡아 줘야 함 (특히 endorser)
    * Anchor peer (org의 peer 들을 관리하는 피어)도 peer 정보를 가지고 있어야 함
    * 계획된 네트워크이기 때문에 누군가는 설정을 할 수 있음

* 1번에서 submit proposal 할 때 peer가 여러개면 모든 peer 에 다 submit 하나?
    * Endorsement policy에 반드시 받아야 하는 peer 리스트에 있는 peer 에게 보냄

* 블록체인 tx에는 sc 호출 코드가 들어가는데, world state가 위변조 되었다고 의심될 때, world state 를 replay 해내는 기능 같은건 없나?
    * 없음
    * kafka
        * Retention 안하는 옵션을 가지고 뜸 (데이터 삭제하지 않음)
        * Worldstate를 복원해 낼 수 있음
        * Kafka 데이터도 바꿀수 있지 않나....
    * CouchDB에 직접 접근해서 위변조 가능해
        * 검증하는 로직 없어
        * 그래서 시스템 적인 보안이 필요해

* delete는 왜 필요한거야
    * Blockchain에 delete했다는 기록이 남아
    * 다만 비지니스 로직 상 검색이 불가능하게 해야 할 때

* 채널 별로 leadger가 관리되는데 worldstate는 함께쓰는 구조인것 같은데 Couchdb를 여러개 띄워서 각각 사용할 수 없나?
    * 로드맵 상으로는 봤는데.. 구현이 됐는지는 모르겠음

* endorser가 endorsing 한 tx를 client가 악의적으로 위변조 한다면?
    * 6번 단계에서 검증함
    * Validation 툴? 은 chaincode로 되어 있어

* RWset
    * 음음음..... 이게 핵심인것 같은데.....

* 튜닝
    * Fabric은 컨테이너 기반이기 때문에 Kubernetes 쓸 수 밖에 없을꺼야 그럼 튜닝 포인트가 완전히 바낀다. 그때 고민하셈
    * 1000 TPS
        * SDK수를 횡으로 확장시켰을 때 얼마나 많이 처리 가능한지 봐야해
        * Kafka sync vs async
    * 현재 버틀랙은 SDK
        * SDK를 횡으로 확장, 수를 늘리고, transaction의 수가 orderer에 몰리도록 설계하는게 필요해

* Business
    * SmartContract에 이것저것 구현 많이 해야해
        * ACL
        * Business logic

* Channel-artifact에 들어있는 파일들을 실제 product 환경에서도 관리자가 만들어내서 추가해 줘야해?
    * 응

* 그럼 중간에 org가 추가 되거나 channel이 추가되거나 하는 등의 Fabric network의 설정이 바뀌면 어떻게 해?
    * Update 명령으로 변경해

*  cryptogen으로 생성된 인증서 구조도 좀 알아야 해
    * 설명 해줘
    * cacert: 비밀키
    * Signcert: 공개키

* Private key를 복사해 데는데.. (아 이거 잘 모르겠네...)
    * orderer의 ca 를 알아야 채널을 생성할 수 있다?
    * 결국에는 관리자 조직이 전체 설정을 관리해줘야 해

* 컨테이너 안에 /etc/hyperledger/fabric에 들어가면
    * core.yaml
    * orderer.yaml

* Fabric-ca 쓸만함?
    * 써본 애들이 그러는데.. 외부꺼 쓰라고 하더라...
    * 팬타시큐리티?

* 네트워크 토폴로지 구성 예좀 줘봐
    * 전세계적으로 Poc를 600개 이상 했는데
    * 고객사들이 오픈 허락을 안해줘...

* Endorsing peer on/off 한다고했는데 설정 값 알려줘
    * /etc/hyperledger/fabric/core.yaml in peer container
        * 못찾겠다.. 파일이 바꼈나..?
    * docker-compose.yaml 에서 환경 변수 넘기기
        * override core.yaml

* Dev 모드에서 Chaincode container 에서 chaincode 실행 하는데 cli에서 install, instantiate??
    * 원래는
        * Install : chaincode source를 peer 에 배포
        * Instantiate: peer에서 chaincode를 빌드 chaincode container 실행
    * 개발 모드에서는..
        *  install, instantiate는 chaincode 이름, 버전, 채널 등을 알려주는 역할을 한다.

* Transaction 이 stub.SetState에 의해서 world state에 반영되기 전에 get하면??
    * Application을 잘 설계해야해
    * 일반 DB처럼 사용하면 안돼...
    * 1000tps 는 Kafka async 일때..
    * 블록체인에는 계약이 완료된 데이터(변하지 않을) 데이터만 저장하는게.. 그럼 tps 같은건 별로 의미가 없어져
        * 중간에 변하고 추가되는 데이터들 (물건이 어디까지 왔는지 등의 데이터)은 다른 시스템에 보관하는게 맞을것 같아..

* peer chaincode invoke/query 둘다 Invoke()를 호출해..?
    * Sdk 사용할때랑 좀 달라
    * Cli tool이 좀 구린것 인 듯….

* Docker-compose-cli.yaml, docker-compose-e2e.yaml의 차이를 모르겠어….
    * Cli는 네트워크에 포함되어 있기 때문에 msp private key update가 필요 없는거

* Example02 에서 invoke를 빠르게 하면 무시되는 경우가 있는데 tps 문제 인가?
    * Tps문제라기보다 셋팅의 문제?

* Commiter 만 떠있는 Peer에 chaincode를 배포할 필요가 있나?
    * 

* orderer를 Kafka로 선택한 경우, 시간 순서로 줄만 세운다고 했는데, 그럼 SBFT는 ??
    * Sbft에 줄세우는 기능이 포함될꺼야
    * Orderer가 줄 세울때 4/3이상이 합의하는 방식으로 줄세운다든가...?

* Fabric network 띄우고 chaincode install, instantiate 하면 chaincode container 가 4개 떠야 되는데 1개만 뜬다.
    * dev-peer0.store1.mymarket.com-marketcc-1.0
    * Instantiate는 chaincode당 한번만 하면 되서 peer0.store1.mymarket.com 에다가만 했어
    * 그래서 여기 것만 생긴것
    * 다른 peer에 대고 query나 invoke 날리면 chaincode container 만들어 냄

* Private vs Public
	* 참여자 가 다름
	* Public 블록체인은 coin 베이스
	* 수수료를 위한 계좌 관리를 해줘야 한다. (오호오호)
	* Private는 서버제공자가 명확하기 때문에 보상이 필요 없다.

* Private and Public 공존

* burrow
	* 블록체인 간 Consensus를 위한 프로젝트
	* ILP (Inter Ledger Protocol)

* 하이브리드 인프라 환경 구성
	* order를 cloud에 올려?
	* 네트워크 연결되면 되는거지 뭐...

* Peer는 성능 저하 없이 몇개까지 추가?
	* Peer의 갯수를 늘리는것에 대한 걱정은 Public 의 몫
	* 갯수를 늘리면 endorser를 늘려서 분산 처리를 할 수 있어
	* peer가 늘어나면 orderer에 부하가 집중되지 않음?
 		* 업체와 node가 1:1 맵핑해서 가지 않아.

* 패브릭 CA의 자세한 구성 방법과 패브릭과 쿠버네티스연동 그리고 마지막으로 동적으로  Peer를 추가하는 방법
	* 컨테이너를 쿠버네티스로 관리, 운영
	* docker-compose 보다 쿠버네티스가 핫

* State DB로 다른 nosql
	* IBM이 이전에 쓰고 있던 Couchdb
	* RDB 를 지원할 예정

* 월마트사
	* 레저 정보와 실제 오프라인상의 정보가 동일한지를 구분할 수 있는 방법
	* 바코드를 바꿔치기 한다던가.. 체크를 통과한 이후에 고의로 훼손한 경우
	* 퍼블릭과 프라이빗 모두 가지고 있는 문제
	* 블록체인이 위변조가 불가능하다지만, 리얼 월드와의 데이터 매칭에는 다른 매커니즘이 필요하다.

* Hyperledger 관련 IBM의 로드맵
	* 개발자, 운영자, 아키텍트
	* 클라우드 환경에서 사용하면 운영자가 필요 없어
	* 개발자는 interface 2개만 알면 되
	* 클라우드 베이스의 서비스를 하려함

* burrow 얼마나 진행상태
	* incubating

* Fabric CA로 발급한 인증서를 다른 곳에서 사용할 수 있나?
	* Public BlockChain 과엮어서?
	* Fabric CA는 membership 인증용
	* 다른 인증 용도로는 사용할 순 있겠지만 개런티는 못하겠음

* TPS 1000??
	* 안나오던데?

* Block Backup
	* 데이터가 쌓이면 스토리지 이슈가 발생한다.
	* 1.4, 1.5에 backup solution 릴리즈 예정?
	* 문서화는 되어 있고 고민하고 있어
