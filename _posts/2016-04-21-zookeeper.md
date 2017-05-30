---
layout: default
title: Zookeeper
category: post
author: Jae
---

> Hadoop 완벽가이드 3판 을 개인적인 공부 목적으로 정리한 포스트 입니다.

# Zookeeper

* Partial Failure
	* 두 노드 사이에 네트워크를 통해 메시지를 보낼 때, 네트워크의 단절로 연산이 실패했는지도 모르는 경우
* Zookeeper는 Partial Failure를 완벽히 해결 할 수는 없지만, Partial Failure를 안전하게 다루도록 도와준다.

#### 주키퍼의 특징
* 단순하다.
	* 단순한 몇 개의 핵심적인 연산을 제공하는 간소화된 하나의 파일시스템
	* 이벤트와 관련된 ordering, notification 같은 일부 추상화 제공 ?
* 다양하게 활용된다.
	* 상호조정에 필요한 다양한 데이터 구조체
	* 프로토콜 구축을 위한 풍부한 프리미티브 제공
* 고가용성을 지원한다.
* 느슨하게 연결된 상호작용을 제공한다.
	* 프로세스가 서로의 존재나 네트워크의 세부 사항을 모르더라도 discover, interact 할 수 있도록 해준다.
* 라이브러리다.
	* openSource


## 주키퍼 설치와 실행
* 단일 서버에서 독립실행 모드로 실행하는 방법이 가장 기본!

* 설치는 DOCS 문서 참조 하셔요..


* 주키퍼 서비스를 시작하기 전에 설정파일을 구성해 주어야 한다.
	* 설정파일: `/home/zookeeper/zoo.conf`

```bash
tickTime=2000 #ms
dataDir=/Users/tom/zookeeper 
clientPort=2181
```

* 서비스 실행
	* `% zkServer.sh start`
	* `% echo ruok | nc localhost 2181`

* four-letter-words
![fourLetterWords](/images/posts/hadoop-the-definitive-guide/fourLetterWords.png)


## 예제
* java api를 사용해서 인스턴스를 생성하고 주키퍼 서버에 접속, 그룹에 멤버를 등록하는 등의 예제

### 문제상황
* 임의의 서비스를 제공하는 서버 그룹이 존재, 서버 그룹의 목록을 유지해야 한다.
* 서버 그룹의 목록을 가지고 있는 서버의 장애 == 전체 시스템의 장애
	* 고 가용성으로 극복
* 특정 프로세스는 장애 서버를 제거할 책임을 가짐
	* 장애를 제거하는 프로세스가 실행중인 서버가 장애가 발생하는 경우가 문제


### 주키퍼를 이용한 그룹 멤버십
* 고가용성 파일시스템 제공
* 파일과 디렉터리에 대한 개념은 없지만, znode라는 한 노드에 대한 통합 개념을 가지고 있다.
	* znode가 디렉토리이면서 파일인 컨셉

### 그룹 생성
* 주키퍼의 java api를 이용해서 `/zoo`라는 그룹에 해당하는 znode를 생성하는 프로그램
	* 주키퍼 Java api를 사용해서 그룹 생성하는 예제
		* 객체 생성: `connect()`
			* 생성자에 대한 호출은 즉시 반환되므로 주키퍼 객체를 사용하기 전에 클라이언트가 주키퍼에 연결된 것을 보장하는것이 필요하다.
			* 클라이언트가 주키퍼로 연결되면, Watcher는 연결되었다는 의미의 이벤트와 함께 `process()` 메소드의 호출을 받는다.
			* `process()` callback에서 이벤트를 확인하여 주키퍼 클라이언트의 연결을 보장한다.
		* znode 생성: `zk.create()`

```java
public class CreateGroup implements Watcher {
	
	private static final int SESSION_TIMEOUT = 5000;
	
	private ZooKeeper zk; 
	private CountDownLatch connectedSignal = new CountDownLatch(1); 
	
	public void connect(String hosts) throws IOException, InterruptedException { 
		zk = new ZooKeeper(hosts, SESSION_TIMEOUT, this); 	connectedSignal.await(); 
	} 
	
	@Override public void process(WatchedEvent event) { 
		// Watcher interface 
		if (event.getState() == KeeperState.SyncConnected) {
			connectedSignal.countDown(); 
		} 
	} 
	
	public void create(String groupName) throws KeeperException, InterruptedException { 
		String path = "/" + groupName; 
		String createdPath = zk.create(path, null/*data*/, Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
		System.out.println("Created " + createdPath);
	}
	
	public void close() throws InterruptedException {
		zk.close(); 
	}
	
	public static void main(String[] args) throws Exception { 
		CreateGroup createGroup = new CreateGroup();
		createGroup.connect(args[0]); 
		createGroup.create(args[1]); 	
		createGroup.close(); }
}
```


### 그룹 가입
* 그룹에 멤버를 등록하는 예제
* 각 멤버는 하나의 프로그램으로서 실행되어 그룹에 가입
	* 주키퍼 네임스페이스에 그룹을 나타내는 임시 znode를 생성하면 프로그램이 종료될 때 그룹에서 자동적으로 제거된다.
	* `zk.create(path, ... , CreateMode.EPHEMERAL)`


### 그룹 멤버 목록화
* 그룹 내의 멤버를 찾는 예제
* `zk.getChildren(pathOfZnode, FLAG)`
	* 결과로 `znode`의 자식의 목록을 내놓는다.
	* `znode`가 존재하지 않는 경우에는 `KeeperException.NoNodeException` 발생


#### 주키퍼 명령행 도구
* 주키퍼는 주키퍼 네임스페이스와 상호작용할 수 있는 실행 도구를 제공

```bash
#/zoo 라는 znode 하부에 있는 znode를 목록화
$ zkCli.sh localhost ls /zoo
```

### 그룹 삭제
* `zk.delete(path, 버전번호)`
	* 버전번호가 같은경우에만 삭제 한다.
	* 버전 번호에 관계없이  삭제하고자 할 때는 `-1`을 쓰면 된다.
* 주키퍼는 재귀적으로 삭제할 수 없기 때문에 부모 znode를 삭제하기 전에 자식 znode를 삭제해야 한다.

> NOTE. version number
> 
> * Every change to a a node will cause an increase to one of the version numbers of that node. The three version numbers are version (number of changes to the data of a znode), cversion (number of changes to the children of a znode), and aversion (number of changes to the ACL of a znode).


## 주키퍼 서비스
* 주키퍼는 고가용, 고성능 상호조정 서비스다.
* 주기퍼가 제공하는 데이터모델, 연산, 구현과 같은 서비스의 특성을 살펴본다.


### 데이터 모델
* znode
	* 데이터 저장, ACL 포함 ?
		* 상호조정을 위해 설계되었기 때문에 큰 용량의 데이터를 저장할 수 없다. (Under 1MB)
	* 데이터 접근은 원자적
		* 저장된 데이터를 일부만 읽거나 일부만 쓸 수 없다.
			* 전체를 읽지 못하거나 모든 데이터를 쓰지 못하면 일기, 쓰기 동작 자체의 실패를 의미
		* `append`연산을 지원하지 않는다.
	* znode는 경로로 참조된다.
		* 경로는 `/`로  구분된 유니코드 string
		* 유닉스의 파일 시스템 경로와 같다
		* 절대 경로여야 하므로 슬래시 문자로 시작
		* 각 경로는 대표성을 갖춘 유일한 표기법으로 표현되어 경로 재해석이 필요 없어야 한다.
			* 유닉스에서는 `/a/b` == `/a/./b` 이지만, 주키퍼에서는 사용할 수 없다.
		* 문자열 `zookeeper`는 예약어로써 경로를 표현하는 구성요소로 사용할 수 없다.
			* `/zookeeper` 하위에 관리 정보를 저장

#### 임시 znode
* znode의 타입은 생성 때 결정되며 임시 또는 영속적인 타입을 가진다.
	* `create()`의 인자로 들어감
	* 생성 때 결정되고, 이후에는 변겨오디지 않는다.
* 임시 znode의 특징
	* 클라이언트의 세션이 종료될 때 주키퍼에 의해 삭제
	* 자식을 갖지 않는다.
	* 하나의 클라이언트 세션에 속할지라도 모든 큰라이언트가 볼 수 있다. (ACL정책을 따른다.)


#### 순차 번호 타입의 znode
* 순차 znode는 주키퍼에 의해 znode 이름의 일부에 순차 번호가 부여된다.
* 순차 플래그를 설정하고 znode를 생성하면 단조 증가 카운터의 값이 znode 이름에 덧붙는다.
	* 어떻게 설정하는지는 나오지 않음
	* `/a/b-`
		* `/a/b-1`, `/a/b-2`, `/a/b-3`, `/a/b-4`, `/a/b-5` 

#### 감시 (Watches)
* znode에 어떤 변화가 있을 때 클라이언트에 관련 이벤트를 통지하는 기능
* 예제
	* 클라이언트는 하나의 znode에 감시를 설정한 후 그 znode에서 exists 연산을 호출
		* 만일 znode가 존재하지 않는다면 exists 연산은 false를 반환
	* 일정 시간후에 znode가 두 번째 클라이언트에 의해 생성된다면?
		* 설정된 감시가 작동되면서 첫 번째 클라이언트에 해당 znode의 생성을 통지

* Watcher는 한번만 동작
	* 다수의 통지를 받기 위해서는 클라이언트가 해당 감시를 등록해야 한다.
	* `exists()`를 통해 감시를 설정할 수 있다.
	* 뒤에 감시 유발자 에서 자세히?


### 연산
* 주키퍼 서비스의 9개의 **기본** 연산
![Operation](/images/posts/hadoop-the-definitive-guide/Operation.png)

* update 류 연산은 버전 숫자가 일치해야 수행됨
	* delete, setData 연산은 업데이트 될 znode의 버전 숫자를 지정해야 한다.
	* 버전은 exists 호출을 통해 알아낼 수 있음
	
> NOTE.
> 
> * sync 연산은 POSIX의 fsync와 다르다.
> * 클라이언트가 자신을 최신 상태가 되도록 sync 연산을 활용


#### 다중갱신
* `multi`
	* 여러 개의 프리미티브 연산을 하나의 갱신 단위로 묶어 연산의 성공과 실패를 반환
	* 하나라도 실패하면 전체가 실패

#### API
* `java` API
	* 동기, 비동기 API 제공
* `c` API
	* 단일 스레드 라이브러리: `zookeeper_st`
		* pthread 라이브러리를 사용 할 수 없는 플랫폼
	* 다중 스레드 라이브러리: `zookeeper_mt`
		* 동기 비동기 다 지원
* 펄, 파이썬, REST 클라이언트를 위한 contrib 바인딩 제공
	* Net::Zookeeper - c api를 per에서 사용할 수 있도록 랩핑한 것

	
#### 감시 유발자
* watch를 설정할 수 있는 읽기 연산
	* exists, getChildren, getData
* watch가 유발되는 쓰기 연산
	* create, delete, setData
* 예제
	* exists 연산에 설정된 감시는 대상인 znode가 생성, 삭제, 데이터의 변경될 때 유발
	* getData 연산에 설정된 감시는 대상인 znode가 삭제, 데이터가 변경될 때 유발
	* getChildren 연산에 설정된 감시는 대상인 znode의 자식이 생성, 삭제, 또는 그 자신이 삭제 될 때 신호를 받는다.

	

#### ACL (Access Control List)
* ZooKeeper는 노드별 ACL(Access Control List)을 지원한다.
	* 클라이언트로 노드를 만들 때 다른 클라이언트 중 일부만 읽기 또는 쓰기를 할 수 있도록 설정할 수 있다. 
	* 하지만 상위 노드에 설정한 ACL이 자동으로 하위 노드로 전파되지는 않는다. 따라서 노드마다 ACL을 직접 설정해야 한다. 
		* 만약 노드 A에 ACL을 설정해 놓고 실수로 노드 A의 하위 노드 A1에 ACL을 설정하지 않았다면, 노드 A1은 누구든 접근해 쓰고 읽을 수 있다.
* ACL 인증 방법
	* digest: 클라이언트는 사용자 이름과 암호에 의해 인증된다.
	* sasl: 클라이언트는 커버로스를 사용하여 인증된다.
	* ip: 클라이언트는  ip 주소에 의해 인증된다.
* `exists` 연산은 ACL 권한에 의해 관리되지 않기 때문에 모든 클라이언트는 모든 znode에 대해 `exists`를 호출할 수 있다.


### 구현
* 독립 실행 모드
	* 단일 주키퍼 서버 존재, 간단한 구조, 테스트용
* 앙상블
	* 클러스터, 복제 모드, 실 서비스
	* 복제를 통해 고가용성을 유지
	* 앙상블 내에서 과반수 이상의 컴퓨터가 유지되는 동안에만 서비스를 제공

#### Zab
* 두단계로 동작하는 간단한 전체 순서화된 브로드캐스트 프로토콜

##### 단계1: leader 선출
* leader를 선출, 과반수의 follower 상태가 leader와 동기화 되면 단계 종료

##### 단계2: 원자적 브로드캐스트
* 모든 쓰기 요청은 leader로 보내진다.
* leader는 follower에게 업데이트 상태를 브로드캐스트 한다.
* 과반수 노드에서 변경이 완료되면 leader는 업데이트 연산을 커밋한다.
* 클라이언트는 성공, 실패 응답을 받는다.

##### leader에게 문제가 생기면?
* follower가 새로운 leader를 선출
* 예전 leader가 복구되면 follower로 시작
* leader를 한번 선출하는데 약 200ms 걸린다.

### 일관성

* 주키퍼의 기본 구조를 이해하면 서비스의 일관성 보장을 이해하는데 도움이 된다.
* follower의 update연산 때문에 leader가 지연될 수 있다. << 한글판 해석 이상함..
	* 주키퍼에 알맞은 모델은 follower 서버에 클라이언트가 연결되는 구조다
	* 클라이언트가 leader에 직접 연결될 수도 있지만, 이를 임의로 선택할 수 없으며, 클라이언트는 자신이 어떤 서버에 연결된지 알지 못한다.

> NOTE. leaderServes = no
> 
> leader가 클라이언트로부터 연결을 승인하지 않기위한 주키퍼 환경 설정
> 세개 이상의 앙상블에서 추천된다.


#### 주키퍼 설계에 포함된 데이터 흐름의 일관성
* 순서의 일관성
	* 특정 클라이언트로부터의 업데이트 연산은 보내진 순서대로 적용된다.
* 원자성
	* 업데이트 연산은 성공 또는 실패이다.
* 단일 시스템 이미지
	* 클라이언트는 연결된 서버에 관계 없이 같은 시스템을 바라보는 것처럼 동작
	* 장애가 생긴 서버를 이어받는 서버는 자신이 장애가 생긴 서버와 동일한 상태가 될 때까지 클라이언트의 연결을 받지 않는다.
* 지속성
	* 서버의 장애 후에도 업데이트된 내역을 유지
* 적시성
	* 지연이 발생하면 수십 초 이상 뒤처진 정보를 보여주는 대신 서버를 중단시켜 강제로 클라이언트를 좀 더 최신 서버로 연결되도록 한다.

* 성능상의 이유로 읽기 연산은 주키퍼 서버의 메모리로부터 이루어고, 쓰기 연산 순서를 전체적으로 정렬하지 않는다.
	* 주키퍼 밖의 메커니즘을 통해 통신하는 클라이언트에 서버와 일치하지 않는 뷰를 갖게하는 결과를 초래할 수 있다.
	* `sync`연산은 클라이언트가 자신이 연결된 주키퍼 서버에게 leader와 동등한 상태가 되도록 한다.


### 세션
* 주키퍼 클라이언트는 앙상블 내 모든 서버 목록을 가지고, 목록의 서버 중 하나로 연결을 시도한다.
	* 만약 모든 서버가 이용할 수 없는 상태면 연결 실패
* 타임아웃
	* 서버가 타임아웃 기간 이내에 응답을 받지 못하면 세션이 닫힌다.
	* 세션이 닫히면 관련된 모든 임시 znode도 사라진다.
* 클라이언트는 ping을 이용해 세션이 살아있도록 유지한다.
* 페일오버
	* 다른 주키퍼 서버로의 페일오버는 주키퍼 클라이언트에 의해 자동으로 이루어진다.
	* 페일오버동안 응용프로그램이 연산을 수행하면 실패한다.
		* 응용프로그램에서 예외처리 필요

#### 시간
* 틱타임
	* 주키퍼 내의 시간에 대한 기본 단위
	* 앙상블 내의 서버는 수행하는 연산에 대한 스케쥴링을 위해 틱타임을 사용
	* 일반적인 틱타임 설정은 2초
	* 세션 타임아웃의 범위는 4초~40초 사이

##### 세션 타임아웃의 선택에 있어서 고려 사항
* 낮은 세션 타임아웃일 때는 서버 컴퓨터의 장애를 좀 더 빨리 탐지할 수 있다.
	* 너무 낮은 세션 타임아웃은 네트워크가 바쁠 때, 패킷이 늦게 도착하면 세션이 만료되는 세션을 닫게 할 수 있다.

##### 주키퍼 앙상블의 크기와 시간
* 주키퍼 앙상블의 크기가 늘어나면 세션 타임아웃도 늘어나야 한다.
* 연결 타임아웃기간, 읽기 타임아웃 기간, ping 대기 기간 모두 앙상블 내의 서버 개수에 대한 함수로 내부적으로 정의 된다.



### 상태
* 새로 생성된 주키퍼 인스턴스는 CONNECTING 상태
* 주키퍼 서버와 연결이 성립되면 CONNECTED 상태
* 클라이언트는 `Watcher` 객체를 등록함으로써 상태 변화에 대한 통지를 받을 수 있다.


![zookeeperStatus](/images/posts/hadoop-the-definitive-guide/zookeeperStatus.png)

> NOTE. Watcher의 두가지 역할

> * 주키퍼 상태 변화를 통지 받는데 사용 
> 	* 주키퍼 객체 생성자로 넘겨지는 Watcher
> 
> * znode의 변화를 통지받는 데 사용
> 	* 전용 Watcher 인스턴스를 전달받아 사용
> 	* 주키퍼 객체 생성자로 넘겨지는 Watcher를 공유
> 		* 읽기 연산의 플래그값을 셋팅





## 주키퍼로 응용프로그램 구현하기

### 환경 설정 서비스
* 주키퍼가 환경 설정을 위한 저장소로 동작, 참가 응용프로그램들이 설정 파일을 가져가거나 업데이트하는 방식
* 주키퍼의 감시를 사용해서 능동적인 환경 설정 서비스를 구성, 원하는 클라이언트는 설정의 변화를 통지받는다.

> 구현에 대한 설명이 나오는데 생략.. 필요할 때 보면 될 듯.. 예제의 종류만 아래 나열 한다.
> 
> * 랜덤 시간마다 주키퍼의 속성값 업데이트하는 응용 프로그램
> * 주키퍼의 속성이 업데이트 되는것을 감시하고 콘솔에 출력하는 프로그램

### 탄력적인 주키퍼 응용프로그램
* 분산 컴퓨팅 환경에서 네크워크는 신뢰할 수 없다.
	* exception을 잘 처리해야한다.

#### InterruptedException
* 연산이 인터럽트 되었을 때 발생
	* 블로킹 메소드가 연산을 취소하기 위해 `Interrupt()`를 호출하면 발생
	* 연산이 취소 된 것이므로 이후 동작을 적절히 해주면 된다.

#### KeeperException
* 주키퍼 서버가 오류를 발생시킬 때, 서버와의 통신에 문제가 있을 때 발생
	* 다양한 오류 상황을 위한 KeeperException의 하위클래스를 제공한다.
		* 예를 들어, KeeperException.NoNodeException은 존재하지 않는 znode에 대해 연산할 때 발생

* 상태 예외
	* 보통 동시에 여러 프로세스 znode를 변경 할 때 발생
* 회복 가능한 예외
	* 세션이 끈어진 예외의 경우 주키퍼가 재연결 시도
* 회복 불가능한 예외
	* 타임아웃, 인증 실패 등 어플리케이션 단에서 재시도 해주어야 하는 예외

#### 신뢰성 있는 환경 설정 서비스
* 멱등 연산에 대한 retry 시전과 아름다운 익셉션 처리로 마음의 평화를 얻을 수 있다는 그렇고 그런 뻔한 사랑 이야기

> NOTE. 멱등연산 (idempotent) ?
> 
> * read처럼 무차별 적으로 재시도 될 수 있는 연산
> * write도 예외 상황을 잘 피해가면 멱등 연산으로 만들 수 있다.



### 락 서비스
* 주키퍼 서버중 leader 서버와 follower 서버를 고르는 얘기가 아님
* 순번이 낮은 znode를 생성한 클라이언트에게 락을 준다는 개념



## 주키퍼 실 서비스
* 복제 모드로 동작
* 앙상블을 운용함에 있어 고려해야 할 사항을 다룬다.
* 주키퍼 관리자를 위한 가이드
	* http://zookeeper.apache.org/doc/current/zookeeperAdmin.html

### 탄력성과 성능
* 컴퓨터와 네트워크 장애로부터 영향을 최소화 하도록 서버 배치
* 주키퍼 전용 서버에서 단독으로 동작
* 트랜잭션 로그를 다른 디스크 드라이브에 쓰도록  `dataLogDir`에 위치를 명시
	* 모든 쓰기 연산이 leader를 거치기 때문에 쓰기 연산은 가능한 빠르게 이루어지는것이 중요
* 프로세스가 디스크로 스왑되면 성능이 떨어진다.
	* 자바의 힙 메모리 크기를 물리적 메모리 양보다 적게 설정

### 환경 설정
* 앙상블 내의 식별자 == 서버 번호
	* `dataDir` 속성에 의해 지정된 디렉터리에 `myid`라는 파일에 일반 텍스트로 저장된 1~255 사이의 숫자
	* `zoo.conf`에 앙상블 내의 모든 서버정보를 저장

```bash
tickTime=2000 
dataDir=/disk1/zookeeper 
dataLogDir=/disk2/zookeeper 
clientPort=2181 
initLimit=5 
syncLimit=2 
# 2888: leader가 follower들에 연결 하기 위한 포트
# 3888: leader선출을 위한 서버들 간의 연결을 위한 포트
server.1=zookeeper1:2888:3888 
server.2=zookeeper2:2888:3888 
server.3=zookeeper3:2888:3888
```
