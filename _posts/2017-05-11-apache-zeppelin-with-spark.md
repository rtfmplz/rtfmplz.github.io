---
layout: default
title: Apache zeppelin with Spark
category: post
author: Jae
---

# Apache zeppelin with Spark

> 이 포스트의 내용은 개인적인 공부 목적으로 책 또는 Blog의 내용을 참조하여 정리한 것입니다.
> 참조한 책, Blog는 포스트의 아래 명시하였습니다.

## Apache Zeppelin?

Hadoop에 이어 가장 핫한 빅데이터 분석 도구인 Spark는 Hadoop보다 빠르고 설치가 비교적 쉽다는 장점을 가지지만 Hadoop과 마찬가지로 이를 다루려면 커맨드라인(spark-shell)에서 검은 글씨와 싸워야 하는 단점이 있다. 데이터 분석(및 Migration - _-..) 업무을 하다보면, 여러가지 알고리즘과 파라메터를 바꿔가면서 결과를 반복해서 그래프나 테이블 같은 형태로 시각화하여 확인해야 하는데 커맨드라인 도구는 불편하다.

Apache Zeppelin은 Spark를 통한 데이터 분석의 불편함을 Web기반의 Notebook을 통해서 해결해보고자 만들어진 어플리케이션이다. Web기반 Notebook 스타일 환경이란 Web에 워드프로세서 처럼 아무거나 입력 가능한 하얀 화면이 뜨고 여기에 코드를 작성-실행-결과확인-코드수정을 반복하면서 원하는 결과를 만들어 낼수있는 작업환경을 말한다.

Zeppelin은 Spark 뿐만이 아닌 Livy, Cassandra, Lens, SQL 등등의 다른 데이터 분석도구나 데이터 베이스에 접근하여 쿼리하는 것을 쉽게 할수 있는 확장 기능들을 지원한다. 이러한 확장성은 Zeppelin의 Interpreter라는 플러그인 구조로 지원되는데 각 Interpreter들은 Zeppelin의 Web Interface를 통해서 입력받은 분석 코드를 local 또는 원격에서 실행할수 있다.

예를들면 Bash로 쉘 스크립트를 짜면 Zeppelin안에 탑재된 Shell Interpreter가 이를 받아 Zeppelin이 설치된 서버에서 shell script를 실행하고 그 결과를 Web Interface에 보내주는 형태이다.


## 누가 쓰면 좋은가?

* 간단하게 데이터 분석을 시작해보려는 사람
* Spark을 처음 시작하려는 사람
* DashBoard를 빠르게 만들고 싶은 사람
* 다양한 데이터 소스로부터 민첩하게 데이터를 살펴보고 싶은 사람
* 오픈소스 프로젝트에 참여해보고 싶은 사람
	* 10줄만 수정해도 contributor가 될 수 있다(고한다. ^^)
	* 실제로 써보면서 버그도 가끔 보이고 Ambari와 연동도 매끄럽지만은 않은 것 같다.
	* 쓰면서 가장 불만스러운 것은 Code 자동완성이 안된다는것이다...... 나말고 누가 좀 기여해 줬으면... :)

## Interpreter

위에서 언급한 것처럼 Zeppelin은 Interpreter라는 플러그인을 동작시키는 구조로 동작한다. 각 Interpreter들은 Web interface를 통해서 입력받는 분석 코드를 실행시키는 역할을 한다.

다양한 Interpreter를 추가 할 수 있지만, 여기서는 Spark interpreter만 다룬다.

#### repository 설정

Zeppelin 에 접속하면 아래 그림처럼 오른쪽 상단에 user가 표시되고 클릭하면 drop list를 펼칠 수 있다.

![drop-list](/images/apache-zeppelin-with-spark/drop-list.png)

Interpreter 매뉴를 선택하면 Interpreter를 추가하거나 삭제하고, 설정을 변경할 수 있는 페이지가 나온다.

* Create 버튼을 클릭하면 새로운 Interpreter를 추가할 수 있다.
* Repository 버튼을 클릭하면 다음 그림처럼 현재 설정된 Repositories를 확인하거나 추가 할 수 있다.


![repository](/images/apache-zeppelin-with-spark/repository.png)

뒤에서 Spark이 사용 할 Library를 추가 하는 방법을 소개할 텐데, Repository 설정이 제대로 되어있지 않으면 정상적으로 동작하지 않는다.

#### Spark interpreter

Spark interpreter를 추가하면 Ambari에서 다음과 같은 설정을 조정할 수 있다. 다양한 설정을 조정할 수 있지만, 중요한 Parameter만 살펴보자..

Zeppelin의 spark interpreter은, 후려치면, spark-shell(driver)과 같은 spark application을 구동시킨 후 Notebook과 연결시켜주는 역할을 하기 때문에 spark-shell을 구동할 때 기본적으로 주어야 하는 parameter를 설정해주어야 한다.

* zeppelin.executor.instances
	* Spark Executor의 개 수를 결정한다.
	* 어떤 DataNode에 몇 개의 executor가 분배되서 뜨는지는 ResourceManager(YARN)의 선택이다.
* zeppelin.executor.mem: executor 하나의 memory 양을 설정한다.
* spark.executor.cores: executor 하나가 가지는 core 수를 설정한다.

다음은 Ambari의 Zeppelin Notebook의 config 화면이다.

![ambari-config](/images/apache-zeppelin-with-spark/ambari-config.png)

> Ambari config v.s. Spark interpreter config
>
> * Ambari가 아닌 Spark interpreter 설정에서도 위에서 설명한 parameter들을 설정 할 수 있는데, 둘을 다르게 설정했을 때 어느 설정이 우선순위에서 우위를 가지는지는 잘 모르겠다..
> * 현재 사용하고 있는 환경에서 Zeppelin 버전을 업그레이드 해야 했는데, 업그레이드 하면서 Ambari와 interpreter의 설정을 다르게 했더니 executor.instances 설정은 interpreter의 것을, executor.memory 설정은 ambari의 것을 따른다..
> * 버그인것 같다... contributor가 될 수 있다.. :)

Zeppelin의 Spark interpreter를 통해서 spark 을 실행시키면 Zeppelin Notebook Server가 설치된 서버에서 `ps -ef | grep zeppelin` 명령을 통해서 다음과 같이 `ZeppelinServer` 라는 이름의 Spark application이 실행된것을 확인할 수 있다.

```bash
zeppelin   4704      1  0  5월08 ?      00:04:27 /usr/java/jdk1.8.0_102/bin/java -Dhdp.version=2.5.0.0-1245 -Dspark.executor.memory=12g -Dspark.executor.instances=10 -Dspark.yarn.queue=default -Dfile.encoding=UTF-8 -Xms1024m -Xmx1024m -XX:MaxPermSize=512m -Dlog4j.configuration=file:///usr/hdp/current/zeppelin-server/conf/log4j.properties -Dzeppelin.log.file=/var/log/zeppelin/zeppelin-zeppelin-hdp-m1.asd.ahnlab.com.log -cp ::/usr/hdp/current/zeppelin-server/lib/interpreter/*:/usr/hdp/current/zeppelin-server/lib/*:/usr/hdp/current/zeppelin-server/*::/usr/hdp/current/zeppelin-server/conf org.apache.zeppelin.server.ZeppelinServer
```

#### Spark interpreter에 dependencies 추가하기

위에서 설정한 repository로 부터 필요한 library를 가져올 수 있다.

###### Interpreter 자체에 dependency 추가하기
* interpreter > spark > Dependencies > groupId:artifactId:version or local file path 추가

![dependencies](/images/apache-zeppelin-with-spark/dependencies.png)

> Interpreter의 설정을 변경하는 경우 Spark Interpreter를 restart 해주어야 한다.  
> Interpreter를 재시작 하면 위에서 살펴본 `ZeppelinServer` 가 dependencies에 명시된 library를 물고 새로 시작된다.
>
> * 주의: 다른 사람과 함께 Zeppelin을 공유하는 경우 내가 사용 할 library를 추가한 후 restart 해버리면 동료에게 어택당할 수 있다. (항상 누가 사용중인지 살피자..)

###### 해당 notebook에서만 사용하도록 추가하기
* notebook paragraph에서 아래와 같은 코드로 추가할 수 있다.

```scala
// local file path는 zeppelin이 설치된 hdp-m1.asd.ahnlab.com의 local path
%dep
z.load("groupId:artifactId:version or local file path")
```

> Notebooek에서 직접 library를 Load 하면 Interpreter를 재시작 하지 않아도 되는지 잘 모르겠다. (동적으로 Library를 Link 한다는 건가.. 자 모르겠다...)

## 그 밖에 쓸만한 기능

#### Data visualization
spark SQL을 통해서 얻은 결과 (다른 언어들도 가능하다고 함)를 시각화 할 수 있다.

![visualization](/images/apache-zeppelin-with-spark/visualization.png)

시각화에서 끝나지 않고 코드를 가려주는 Report 뷰 모드와 Scheduler를 사용하면 Dashboard를 빠르게 만들 수 있다.

* 코드와 차트가 한곳에서 관리되므로 손쉽게 페이지를 만들고 유지, 관리를 할 수 있다.
* Scheduler를 내장하고 있기 때문에 매일, 매시간 업데이트 되어야 하는 Dashboard, Batch 작업을 관리하기 용이하다.

#### Dynamic forms
입력 형태를 동적으로 생성할 수 있다.
* 특정 달, 일의 데이터를 보고자 할 때 유용하다.

![dynamicform](/images/apache-zeppelin-with-spark/dynamicform.png)

## Troubleshooting

##### zeppelin은 zeppelin user로 동작하기 때문에 hdfs내 권한이 없는 폴더에 write를 할 수 없다.

해결 방법은 다음과 같다.

* zeppelin user를 쓰기 작업을 할 폴더의 소유 그룹에 포함시킨다.
	* id에 그룹 추가하기: `usermod -a -G <group> zeppelin`
* 쓰기 작업을 할 폴더의 소유 그룹을 zeppelin 이 포함된 그룹으로 변경한다.
	* hdfs 파일, 폴더의 그룹 변경: `hadoop fs -chgrp -R <GROUP> <hdfs://nn1.example.com/file1>`

> hdfs는 umask 설정이 최초 022 설정되어 있기 때문에 파일, 폴더 생성시 디폴트로 755 설정을 가지게 된다.  
> 생성하려느느 최종 파일이 755권한을 가지는것은 문제가 없지만, 폴더가 755 권한을 가질 경우 해당 그룹에 포함된 zeppelin user는 폴더에 파일을 생성할 수 없다.
>
>	* hdfs의 `fs.permissions.umask-mode` 설정을 바꾸면 umask를 변경할 수 있다.


## 참고한 사이트

1. [Zeppelin Docs](http://zeppelin.apache.org/docs/0.7.1/)
2. [Apache Zeppelin 이란 무엇인가?](https://medium.com/apache-zeppelin-stories/%EC%98%A4%ED%94%88%EC%86%8C%EC%8A%A4-%EC%9D%BC%EA%B8%B0-2-apache-zeppelin-%EC%9D%B4%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80-f3a520297938#.19l8s4gvd)
3. [Apache Zepplin으로 데이터 분석하기](https://www.slideshare.net/sangwookimme/apachezeppelin)
4. [세상에서 가장쉬운 Zeppelin Notebook 만들기](https://www.slideshare.net/SooKyungChoi/zeppelin-notebookss)

