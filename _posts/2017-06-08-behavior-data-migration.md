---
layout: post
title: JAE's journey migrating 1TB of RDBMS data to Hadoop using SPARK
tags: [hdfs, migration, troubleshooting]
author: Jae
comments: true
---

# JAE's journey migrating 1TB of RDBMS data to Hadoop using SPARK

> * 본 포스트는 회사에서 프로젝트를 진행중 발생한 문제들과 문제를 해결한 과정을 소개하고 있습니다.
> * 따라서 회사 자산인 DB Schema 등을 임의로 변경하여 읽는 중 다소 어색할 수 있습니다


## A long time ago in a galaxy far, far away...

20억건에 해당하는 FileData를 이관했던 경험을 바탕으로 BehaviorData를 같은 방법으로 이관하기로 했다.

데이터를 이관하는 작업은 3 ~ 4개의 Step로 나누어서 작업하였다.
각각의 Step에서 수행한 작업은 다음과 같다.

#### 1. Oracle에 저장된 행위 관련 테이블들을 Spark에 포함된 JDBC library를 이용해서 HDFS에 Parquet format으로 Dump 한다.

FileData 관련 테이블들을 Dump 받는 도중 DB session이 끊기거나, Error가 종종 발생했기 때문에 같은 방법으로 다음과 같이 Key를 기준으로 200만건 단위로 분할해서 Dump 하였다.

```bash
hdfs://data/mover/asddb/<TABLE_1>/0000000000_0002000000
hdfs://data/mover/asddb/<TABLE_1>/0002000000_0004000000
hdfs://data/mover/asddb/<TABLE_1>/0004000000_0006000000
<...>
hdfs://data/mover/asddb/<TABLE_2>/0000000000_0002000000
hdfs://data/mover/asddb/<TABLE_2>/0002000000_0004000000
<...>
```

#### 2. List 형태의 데이터 또는 Join이 필요한 데이터를 1차적으로 Denomalization 하여 HDFS에 Parquet format으로 저장한다.

BehaviorData는 각 상세 행위단위로 Normalization 되어 DB에 저장되어 있으므로 데이터를 서비스에서 이용하는 Thrift 객체 형태로 만들어주기 위해서는 Denormalization 해야 한다. 그 중 List Type의 Field나 Normalization 형태에 따라서 Join이 필요한 데이터를 이 Step에서 선처리 해준다.

###### List Type

* FileData의 PeIdd, PeIsh 등

###### Need to join

* 상세 BehaviorData는 최종적으로 하나의 테이블로 Join 되어야하기 때문에 각각의 테이블에 Normalization 되어있는 데이터들을 LINK_KEY로 Join 하였다.

```sql
DESC LINK_TABLE
    - KEY_SEQ NUMBER [PK]
    - LINK_SEQ NUMBER [FK]
DESC DATA_TABLE
    - LINK_SEQ: NUMBER [PK]
    - CTIME: DATE
    - DATA1: VARCHAR2 (512)
    - DATA2: VARCHAR2 (128)

-- 결과
JOINED_TABLE
    - KEY_SEQ NUMBER [PK]
    - CTIME: DATE
    - DATA1: VARCHAR2 (512)
    - Data2: VARCHAR2 (128)
```

Denormalization한 데이터는 Step 1에서와 같이 Parquet format으로 HDFS에 저장한다.

```bash
hdfs://data/mover/asddb/<JOINED_TABLE>/0000000_2000000
hdfs://data/mover/asddb/<JOINED_TABLE>/2000000_4000000
hdfs://data/mover/asddb/<JOINED_TABLE>/4000000_6000000
```

> 상세 BehaviorData의 Denormalization
> 
> * KEY_SEQ는 BehaviorData가 보고될 때마다 증가하지 않는다.
> * 일단 클라이언트에 의해서 행위가 보고되면 행위를 발생시킨 파일과 부모의 파일 시퀀스 및 행위를 구별할 수 있는 다른 정보들을 DB에서 검색한다. 만일 정보가 일치하는 KEY_SEQ가 있다면 KEY_SEQ를 증가시키지 않고 LINK_SEQ만 증가시킨 후 LINK_TABLE에 insert 한다. 행위에대한 상세 내용은 DATA_TABLE에 insert한다.
> * 이런 Normalization 정책 때문에 하나의 KEY_SEQ는 N개의 상세 행위를 가질 수 있다.
> * 단, 다른 Type의 상세 행위가 같은 KEY_SEQ에 동시에 존재하지는 않는다.

#### 3. Create ProcessAction

Step 1과 2에서 생성한 Parquet 파일을 Spark을 이용해서 Join, KEY_SEQ를 PK로 가지는 하나의 테이블을 생성한다. 생성된 테이블의 각 컬럼 값을 대응하는 Thrift 객체의 Field에 잘 배치한 후 Hadoop Sequence File Format으로 HDFS에 저장한다.

> ScroogeRDD를 DataFrame으로 변경하면 Error가 발생해서 Hadoop Sequence File Format으로 저장했었다. (이후에 버그패치가 된것으로 알고있지만 확인해보지는 않았다.)

#### 4. 완성된 데이터를 필요한 경우 Cassandra에 Submit 할 수 있다.

FileData의 경우 Cassandra에 submit하는 과정이 있었지만, BehaviorData의 경우에는 HDFS에 저장하는 것으로 충분하기 때문에 해당 Step은 진행하지 않았다.


## Episode I. DB의 복수

DB Sessions 수를 제한 하기위해서 Partition 수를 5로 설정하고, 위에서 언급했듯이 종종 Error가 발생했기 때문에 각 테이블의 Key를 기준으로 200만건 단위로 분할해서 관련 테이블들을 Dump 하였다.

>  Partition 당 하나의 Task가 생성되고 각 Task는 하나의 Session을 갖는다.
> JVM하나당 하나의 Session을 갖는것은 JDBC 기본 설정인듯 하다.

FileData 이관을 진행하면 많은 테이블들을 Dump하였고 문제가 생길때마다 방어 코드들을 적용해 놓았기 때문에 쉽게 될 줄 알았다.......

#### 현상

DATA_TABLE1의 특정 SEQ 대역을 Dump 하는도중 Spark Job이 끝나지 않는 현상 발생하였다.

에러가 나는것도 아니고, Job이 끝나지 않는 현상은 처음이라 처음에는 DB는 의심하지 못하고 문제가 무엇인지 원인 파악을 하는것이 어려웠다.

* 당시 Staging Hadoop도 안정적이지 않은 상태였고, Spark에서 발생하는 Error도 다양했기 때문에 시스템만 의심하고 Hadoop, Spark 관련 Error 들만 조사하며 시간을 썼다.
* Ep.II 에서 이어서 이야기 하겠지만 이 시기에 컨설턴트가 등판해서 문제를 더 복잡하게 만들었다.

#### 분석

분석을 위해서 각 모듈을 하나하나 해체하는 작업을 진행했다.
해체는 다음과 같은 순서로 진행하였다.

1. HDFS에 저장하는 부분을제거
2. YARN을 제거, Spark을 local mode로 동작
3. Spark을 제거

위의 과정을 거치면서 모듈들을 다 제거하고나니 DB에서 응답이 오지 않는것을 확인 할 수 있었다.

DATA_TABLE의 경우 LINK_SEQ를 PK로 200만 ~ 400만 범위를 Dump하면 결과가 나오지 않았기 때문에 아래와 같이 테스트를 진행하였다..

```sql
-- 결과가 바로 나옴
select * from DATA_TABLE where 2000000 <= LINK_SEQ and LINK_SEQ < 4000000;

-- 결과가 나오지 않음!!
select * from DATA_TABLE where 2000000 <= LINK_SEQ and LINK_SEQ < 4000000 ORDER BY LINK_SEQ ASC;

-- 다른 범위는 결과가 바로 나옴
select * from DATA_TABLE where 20000000 <= LINK_SEQ and LINK_SEQ < 22000000 ORDER BY LINK_SEQ ASC;

-- index를 지정해주면 성공!!!
select /*+ index(A LINK_SEQ_INDEX_PK) */ * from DATA_TABLE A where 2000000 <= LINK_SEQ and LINK_SEQ < 4000000 ORDER BY LINK_SEQ ASC;
```

#### 해결

위의 결과를 바탕으로 DBA에게 문의를하니 index 를 제대로 타지 않아 발생하는 문제라는 답변을 받았다.

* 다음과 같은 구문을 Query에 추가해서 문제를 해결하였다.

```sql
-- 지정된 index를 강제적으로 쓰게끔 지정한다.
/*+ INDEX(table_name index_name) */
```

> DB의 index 정책
>
> * DB는 통계 기반으로 index를 적용한다고 한다.
> * 따라서 자주 사용되지 않는 Record 범위는 index를 제대로 타지 않을 수 있다
> * 따라서 DB에 Query 할 때는 Query 실행 계획을 확인하는것이 중요하다


## Episode II. 컨설턴트의 습격

컨설턴트 Spark관련 여러가지 문제 상황을 문의하여 그 중 두가지 정도에 대해서 답변을 받았다.

* Spark 튜닝
* 1:N Join시 메모리 문제

#### Spark이 느려요!

* 1ProStaging 환경의 DataNode 스펙이 제 각각인데, 모든 데이터노드가 256GB 메모리를 가지고 있는것으로 설정 해 놓음
* Resource Manager는 설정을 보고 각각의 Node Manager에 Task를 할당하는데 128GB 메모리를 가지고 있는 DataNode에도 무리하게 Task를 할당하게 됨
* Container간 자원 확보를 위한 교착 상태에 빠짐 (근거없음..)

> 당시 Hadoop 설정에 익숙치 않아서 모든 데이터노드가 256GB 메모리를 가지는것 처럼 설정 파일을 작성하였다.
> Hadoop은 스펙이 다른 장비를 Group별로 관리할 수 있도록 지원하기 때문에 현재는 정상적으로 설정파일을 수정하였다.

#### 데이터를 Join을 하면 Spark이 죽어요!

* 상세 BehaviorData를 Join하는 과정에서 Spark Job이 오래 걸리거나 실패하는 경우가 발생했다.
* 원인 분석은 제대로 하지 못하고 Spark으로 잘 안되니 HIVE-MR을 검토하고 Hive External table 생성 등 작업을 진행하였으나 이 역시 속도가 너무 나오지 않아 중단


## Episode III. 새로운 희망

Step 2(1차 Denormalization) 작업을 진행 중 Spark Job이 오래 걸리거나 결국 실패하게되어 원인을 분석하고 있었다.
상세 BehaviorData의 형태가 1:N 이기 때문에 한번에 처리해야 하는 데이터의 양이 많아 시간이 오래걸리거나, 메모리가 부족한가라는 의심은 하였지만 정확한 원인 분석은 하지 못하고 있었다.

#### 현상

200만건 단위로 상세 BehaviorData를 Join 하였다.
200만건을 Join 하는 Job을 처리하는데 약 1시간 정도가 걸렸고, 어떤 구간은 실패하기도 하였다.

#### 분석

DB를 Dump한 Parquet 파일들은 DB Session 수를 제한하기 위해서 Partition 수를 5로 설정하였고, 그 결과 200만건당 Parquet 파일이 5개씩 생성되었다.
각 상세 행위마다 다르겠지만 예를 들어, Key가 되는 SEQ가 20억번까지 있다고하면 상세 BehaviorData 당 Parquet 파일이 5000개가 된다.
테이블 두 개를 Join한다고 하면 적게는 5000개 부터 많게는 10000개의 Parquet 파일을 사용해야 했다.

> DB Session = 5 -> partition = 5 -> 200만개당 Parquet 파일 수 = 5
> 5 * (20억/200만 == 1000) = 5000개 * 2개의 테이블 = 10000 개

Spark은 하부의 저장 시스템에 기반하여 병렬화 수준을 결정한다. 이 경우 Parquet 파일 하나당 Task가 생성되고, 각 Parquet 파일마다 Records의 수가 다르므로 Task 마다 처리되는 시간에 차이를 가지게 된다. 이것은 결국 전체 잡을 처리하는 시간을 증가시킨다.

> 자세한 내용을 [Performance of Spark with RDD partitioning](https://rtfmplz.github.io/post/2017/03/25/performance-of-spark-with-rdd-partitioning)에 포스트 하였다.

#### 해결

Parquet 파일의 개수를 줄여라!
* DB Dump로 생성된 Parquet 파일은 크기도 작고 Task 수에 직접적인 영향을 미치므로 그 수를 줄이기로 하였다.

> Parquet 파일의 적정 크기는 1G 정도가 적당하다.

* 다음 명령으로 parquet file을 merge 할 수 있다.

```bash
hadoop fs -ls -C /data/table | xargs -t -i hadoop jar parquet-tools-1.9.0.jar merge {} {}.gz.parquet
```

결과적으로 Parquet 수가 1/5로 줄었고, 하나의 상세 BehaviorData 전부를 Join하는 Spark Job을 1시간 정도에 처리할 수 있었다.


## Episode IV. 보이지않는 위험

Joined 테이블은 만들었는데, 전체 테이블을 Join하니 Job을 실행하니 Task들이 `OutOfMemoryError`를 던지기 시작했다.

> 사실 이문제는 Ep.III 에서 발생할 수도 있었는데.. 운좋게 넘어간 모양이다.

#### 현상

Step 1에서 Dump한 DB Table과 Step 2를 통해서 만들어진 Join 된 모든 데이터들을 KEY_SEQ로 Join하였다.
Spark Job이 완료되지 않고, 실패한 Task에서 `OutOfMemoryError`를 확인하였다.

#### 분석

KEY_SEQ 하나당 상세 행위가 N개인 1차 Join된 상세 BehaviorData 테이블과 KEY_SEQ 하나당 M개의 Record를 가지는 기타 테이블들을 Join 하면 1:NxM 의 형태로 Record가 증가하게 된다. (이런 테이블들이 추가로 존재한다면 1:NxMxZ 식으로 배수로 증가하게 된다.)

Join 연산은 데이터의 Locality를 높이기 위해서 데이터를 Suffle하여 Key를 기준으로 Record를 정렬한다. 즉, Task가 작업을 처리기 위해서 필요한 데이터들을 Key를 기준으로 분배, 조합해서 빠르게 처리할 수 있도록 하는것이다. 그런데 이 과정에서 특정 Key가 유독 많은 Record를 가진다면 하나의 Task에 다수의 Record가 몰리게 된다. 이것은 분할되지 않고 메모리에 한번에 올라가야하는 데이터가 많다는 것이고, Executor의 메모리는 Spark을 구동할 때 정해지기 때문에 Executor가 `OutOfMemoryError`를 발생시키는 것이다.

> Data Locality
>
> * PROCESS_LOCAL 데이터가 실행중인 코드와 동일한 JVM에 존재함
> * NODE_LOCAL 데이터가 동일한 노드에 존재함. 예를들어 데이터가 동일한 노드의 HDFS 또는 동일한 노드의 다른 JVM에 존재한다. 데이터가 프로세스간에 이동해야하므로 PROCESS_LOCAL보다 약간 느리다.
> * NO_PREF 데이터가 어느 곳에서나 똑같이 액세스되며 지역성이 없다.
> * RACK_LOCAL 데이터가 동일한 서버 랙에 존재. 데이터는 동일한 랙에있는 다른 서버에 있으므로 일반적으로 단일 스위치를 통해 네트워크를 통해 전송된다.
> * ANY 데이터가 동일한 랙에 있지 않고 네트워크의 다른 곳에 존재.

#### 해결

이런 형태의 문제를 해결하기 위해서는 몇가지 방법이 있을 수 있다.

###### Executor 당 Memory를 증가시킨다.

N의 크기가 별로 크지 않은 경우, Executor의 Memory를 증가시키는 것으로 문제를 해결할 수도 있다.

> Executor의 Memory는 `spark.executor.memory` 옵션을 사용해서 변경할 수 있다.

###### Partition의 수를 늘린다.

Partition의 수를 증가시키면 Record를 많이 가진 Key가 하나 이상의 Partition으로 분할되는 효과를 기대해 볼 수 있다.

> Default Partition 수는 `spark.sql.shuffle.partitions` 옵션을 사용해서 변경할 수 있다.

###### 특정 Key가 가질 수 있는 Record의 수를 제한한다.

데이터를 버린다. :)

상세 BehaviorData가 1:N의 관계를 가지는 것은 이미 알고 있던 사실이었고, 일정상의 문제로 마지막 방법을 사용하기로 하였고 Record의 수를 1로 제한하였다. (rownum=1)

* Spark에서 다음과 같은 방법으로 Rownum=1을 구현할 수 있다.

```scala
import org.apache.spark.sql.expressions.Window

val w = Window.partitionBy("KEY")//.orderBy("TIME")

val DataFrame = sqlContext.read.parquet("/data/mover/asddb/table1")
.withColumn("ROWNUM", row_number.over(w)).filter("ROWNUM <= 1").drop("ROWNUM")
```

또는

```scala
sqlContext.read.parquet("/data/mover/asddb/table1").dropDuplicates(Seq("KEY_SEQ"))
```


## Episode V. Parquet의 역습

FileData를 이관 할 때는 Denormalization 한 데이터를 Hadoop Sequence File로 저장했었지만, BehaviorData는 Parquet으로 저장하는 것으로 요구사항이 변경되었다.
Hadoop Sequence File 로 저장할 때는 아무 문제 없던것이 Parquet으로 저장하니 Error가 발생했다.

> PairRDD의 `saveAsNewAPIHadoopFile()`를 이용해서 ScroogeRDD를 Parquet으로 저장 할 수 있다.

#### 현상

"Connects to website" 행위를 `saveAsNewAPIHadoopFile()`를 이용해서 Parquet으로 저장하는 과정에서 Error가 발생했다.

```scala
Caused by: java.lang.NoSuchMethodError: org.apache.parquet.io.api.Binary.fromReusedByteArray([BII)Lorg/apache/parquet/io/api/Binary;
	at org.apache.parquet.thrift.ParquetWriteProtocol.writeBinaryToRecordConsumer(ParquetWriteProtocol.java:648)
<중략>
```

#### 분석

Error 내용을 보면 `writeBinaryToRecordConsumer()`에서 `fromReusedByteArray()`를 호출는 도중 `NoSuchMethodError`가 발생했음을 알 수 있다.
`writeBinaryToRecordConsumer()`가 호출되는 이유는 Thrift 객체인 NetworkObj의 특정 Field가 Binary Type이기 때문이다.

[GrepCode](http://grepcode.com) 에서 `writeBinaryToRecordConsumer()` 함수를 검색해보면 `parquet-thrift-1.7.0`에서는 `Binary.fromByteArray()`를 호출하도록 구현되어 있지만 `parquet-thrift-1.8.0`에서 `fromReusedByteArray()`를 사용하도록 변경된 것을 확인할 수 있다.
Spark-shell 구동시 `parquet-thrift-1.8.0`을 dependency에 추가하였기 때문에 `writeBinaryToRecordConsumer()`에서 `fromReusedByteArray()`를 호출하였지만 `fromReusedByteArray()`의 구현이 존재하는 `parquet-column` Library는 Spark에 기본적으로 포함된 1.7.0 버전이 Load 되어 `fromReusedByteArray()`를 찾을 수 없다는 Error가 발생한 것이다.

> NoSuchMethoError in Spark
>
> * Scala(JAVA) 프로그램의 경우 버전이 다른 Library의 jar가 classpath에 한 개 이상 존재하는 경우 먼저 Load 되는 Library를 사용한다.
> * 이런 문제를 해결하기 위해서 SBT에서는 Library를 Management 할 수 있는 방법을 제공한다.
> * 하지만 Spark-shell의 경우 구동 시 사용자가 자신이 사용할 Library를 추가할 수 있고, 이것이 미리 추가된 Library와 충돌 할 수 있다.

#### 해결

NetworkObj의 문제가되는 필드를 String Type으로 변경하여 `writeBinaryToRecordConsumer()`를 호출하지 않도록 하였다

> Spark 2.1.1에서는..
>
> Post를 작성하면서 [Spark Configuration](https://spark.apache.org/docs/latest/configuration.html)을 찾아보니 `spark.driver.userClassPathFirst`과 같은 설정이 실험적으로 추가되었다고 한다.
> 이 설정은 드라이버에 클래스를 로드 할 때 Spark 자체 jar보다 사용자가 추가 한 jar를 우선 적용할지 여부를 결정한다. Default 값은 false이며 클러스터모드에서만 동작한다.


## Episode VI. 깨어난 포스

긴 시간 데이터 이관 프로젝트를 진행하면서 힘들었지만 힘든만큼 배운 것이 많았다.

* 큰 이슈는 하루에 해결할 수 있는 단위로 나눌 것 (Divide And Conquer)
* Document는 꼼꼼하게 읽어 볼 것
* 끊임없이 질문할 것
* "이게 안돼요"가 아닌, "이게 필요해요" 라고 말할 것
* 협업할 때는 상대방도 개발자임을 고려할 것

끝으로 도중에 포기하지 않고 끝까지 완주 할 수 있도록 멘탈 케어에 힘써주신 팀 동료들과 선배 개발자들께 감사의 인사를 전한다.
다음 프로젝트로 무엇을 하게될지 아직 모르겠지만(나는 시키는건 다하는 개발자이므로..ㅋㅋ), 이번 프로젝트가 나의 개발 커리어에 많은 영향을 줄 것 같다.
앞으로 무엇을 하게되든..

#### May the Force be with you..
