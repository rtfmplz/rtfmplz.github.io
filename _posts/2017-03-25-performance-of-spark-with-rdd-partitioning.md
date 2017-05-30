---
layout: default
title: Performance of Spark with RDD partitioning
category: post
author: Jae
---

# Performance of Spark with RDD partitioning

> 이 포스트의 내용은 개인적인 공부 목적으로 책 또는 Blog의 내용을 참조하여 정리한 것입니다.
> 참조한 책, Blog는 포스트의 아래 명시하였습니다.

## 병렬화 수준과 성능

RDD는 하부의 저장 시스템에 기반하여 병렬화 수준이 결정된다. 예를 들어, HDFS로 부터 생성된 RDD는 기반하는 HDFS 파일의 각 블럭당 하나의 파티션을 가진다. (블럭의 개수는 실제 파일의 개수보다 많을 수 있다.)

이렇듯 물리적 실행 간에 RDD는 여러 개의 파티션으로 나뉘고 각 파티션은 전체 RDD 데이터의 일부를 가지고 있다. Spark는 각각의 파티션을 처리할 테스크를 하나씩 만들고, 각각의 테스크는 클러스터에 하나의 코어를 요청한다. 결국, 파티션은 Spark의 병령성을 결정하는 중요한 요소가된다.

따라서 파티션을 효율적으로 하면 병렬성이 증가하고, Worker 노드에서 bottleneck이 발생하는 것을 줄일 수 있다. 또한 Key를 기준으로 파티셔닝을 잘 해놓으면 Worker노드 사이에 데이터 이동이 줄어들기 때문에 shuffling의 코스트도 절약을 할 수 있다. (Shuffle은 Out Of Memory의 위험이 있기 때문에 최소한으로 발생하도록 하는것이 좋다.)

> Why OOM in many shuffling
>
> shuffle 을 발생시키는 대표적인 RDD의 연산은 `join`, `repartition`이 있다.  
> 특히 `join` 같은 연산은 Key 값에 의해서 데이터가 한 곳으로 몰릴수 있다.  
> 하나의 Key가 다수의 Value를 가지는 1:N 관계를 가진다면 하나의 Task에 다수의 Record가 모이게되고 그 결과 OOM Error이 발생할 수 있다.
> Spark에서는 메모리가 차면 disk로 자동 swap 시키긴 하지만 이 동작은 자주 일어나는게 아니라 한번에 일괄적으로 동작하기 때문에 OOM Error가 발생할 수 있다.

파티션의 수는 두가지 방식으로 성능에 영향을 미칠 수 있다.

* 파티션의 수가 너무 적은 경우, 스파크가 클러스터 리소스(core, memory)를 놀리게 된다.
* 파티션의 수가 너무 많은 경우, 각 파티션 오버헤드 때문에 성능 문제가 생길 수 있다.

또한, 클러스터 리소스를 고려하지 않은 파티셔닝은 아래와 같은 문제를 발생시킬 수 있다.

* 어떤 테스크는 수 밀리초 내로 끝나고, 어떤 테스크는 아무 데이터도 읽거나 쓰지 못하는 테스크도 있을 수 있다. 이는 성능에 문제를 발생시킨다.

> 예를들어, 아래 그림은 코어가 3개인 클러스터 환경에서 파티셔닝을 3으로 했을 때와 4로 했을 때 프로그램 실행 시간에 어떤 차이가 나는지 보여준다.
>
> * 파티션을 4개로 한 경우, 3개의 코어가 3개의 파티션을 처리한 후 하나 남은 파티션은 나중에 처리된다.
> * 파티션을 3개로 한 경우, 각각의 코어는 조금 더 많은 일을 하지만 잘 분배되어 있기 때문에 파티션을 4로 한 경우보다 결과적으로는 전체 일을 빠르게 끝낼 수 있다.
> ![Partition 비교](/images/posts/spark-rdd-partitioning/PartitionExample.jpg)


위와 같은 이유로 스파크에서 RDD 파티셔닝은 프로그램의 성능에 많은 영향을 미친다.
스파크에서는 RDD의 병렬화 수준을 수정할 수 있는 두가지 방법을 제공한다.

* 데이터 셔플이 필요한 연산간에 생성되는 RDD를 위한 병렬화 정도를 인자로 줄 수 있다.

```scala
def join[W](other: JavaPairRDD[K, W], numPartitions: Int): JavaPairRDD[K, (V, W)]
```

* 이미 존재하는 RDD를 재배치 할 수 있다.
	* repartition()
		* RDD를 무작위(?)로 섞어 원하는 개수의 파티션으로 다시 나눠 준다.
		* filtering등을 통해서 records의 개수가 줄어든 이후에 전체를 rebalance를 해 파티션을 균등하게 재분배 한다.
		* repartition을 하면 병렬화 수준은 증가하지만 셔플링이 발생하게 된다.
	* coalesce()
		* 셔플을 발생시키지 않고 파티션을 통합하기 때문에 피티션 수를 감소시키는 역할을 한다.
		* 파티션 개수가 줄어들기 때문에 parallelism은 감소한다.
		* HDFS, 외부시스템으로 데이터를 저장하기전에 사용한다.

```scala
def repartition(numPartitions: Int): JavaPairRDD[K, V]
def coalesce(numPartitions: Int): JavaPairRDD[K, V]
```

## PartitionBy()

> RDD와 DataFrame의 PartitionBy Method의 구현이 서로 달라 이를 정리함

#### Partitioners for key-value PairRDD

`map`, `keyBy`를 통해 RDD를 생성할때, 파티션을 변경하지 않는다. 결과적으로 생성된 key-value로 이루어진 PairRDD는 최초 생성될 때 Partitioner를 가지고 있지 않다. 결국, 각각의 record 들은 key와는 전혀 상관없이 각 파티션에 랜덤하게 분포가 되어있기 때문에 key를 기반으로하는 연산을 할 경우 많은 셔플링이 발생하게 되고, 이는 성능 저하로 이어진다.

다른말로 하면 같은 key를 갖고 있는 records가 같은 파티션에 위치한다면, 반복적인 작업이나 key를 기반으로 하는 연산을 더 효율적으로 할 수 있다는 것이다.

![partitionBy](/images/posts/spark-rdd-partitioning/partitionBy.png)

위 그림처럼 HashPartitioner, RangePartitioner를 이용해 같은 key를 갖고 있는 records를 같은 파티션에 모이도록 할 수 있다.

partitionBy를 호출하면 shuffle이 발생하지만, 이후에 발생하는 key를 기반으로한 연산들은 효율적으로 할 수 있다.

예를 들어 최초에 파티션이 3개 생성되어 있고, 0개의 Partitioner의 환경에서의 RDD를 고려해보면, RDD는 key별로 같은 파티션에 모여있지 않다. 즉 Records는 key와 상관없이 3개의 파티션에 규칙이 없이 분포하고 있는 것이다. key를 기반으로 하는 연산을 할 예정이라면 같은 key를 갖는 records를 같은 파티션에 위치하도록 재파티셔닝을 하는게 좋다. 간단하게 hash partitioner와 함께 partitionBy의 함수를 호출하면, 최초에는 shuffle이 발생하지만, 그 이후에는 key기반의 연산을 더 효율적으로 할 수 있다.

만약에 partitioner 없이 두개의 RDD를 key로 join하면, 하나의 RDD가 다른 Worker Node로 모든 값들을 셔플링을 해야한다. 이렇게 되면 효율적이지 못한 join을 하게된다. 같은 작업을 반복한다면 연산의 수행시간이 늘어난다. 만약 join을 하기 위한 두개의 RDD가 같은 파티션에 있다면 셔플링이 일어나지 않기 때문에 Network I/O와 latency를 최소화 할 수 있다.


#### partitionBy of DataFrameWriter

partitionBy함수를 가지고 있는 Class는 아래와 같다.

* JavaNewHadoopRDD
* JavaNewHadoopRDD
* JavaPairRDD
* PairRDDFunctions
* DataFrameWriter

이 중 DataFrameWriter의 partitionBy는 나머지 RDD의 partitionBy와 그 구현이 다르다.

JavaPairRDD의 partitionBy의 Scala Document를 보면 아래와 같다. Partitioner를 사용해서 Partition 된 RDD를 반환한다. 위에서 다루어 왔던 그 PartitionBy의 내용이다.

```scala
def partitionBy(partitioner: Partitioner): JavaPairRDD[K, V]
    Return a copy of the RDD partitioned using the specified partitioner.
```

하지만 DataFrameWriter의 PartitonBy의 구현은 RDD의 PartitionBy와 그 내용이 조금 다르다.

```scala
// DataFrameWriter의 PartitonBy의 Scala Doc (translated by google)

파일 시스템의 지정된 열을 기준으로 출력을 분할합니다.
지정된 경우 출력은 Hive의 분할 스키마와 유사한 파일 시스템에 배치됩니다.
예를 들어, 연도와 월별로 데이터 집합을 분할하면 디렉토리 레이아웃은 다음과 같습니다.

년 = 2016 / 월 = 01 /
년 = 2016 / 월 = 02 /

파티셔닝은 물리적 데이터 레이아웃을 최적화하기 위해 가장 널리 사용되는 기술 중 하나입니다.
쿼리가 분할 된 열에 술어를 가질 때 불필요한 데이터 읽기를 건너 뛰는 대용량 인덱스를 제공합니다.
파티셔닝이 제대로 작동하려면 각 열의 고유 한 값의 수가 일반적으로 수만 개 미만이어야합니다.

이것은 초기에 Parquet에 적용 가능했지만 1.5+에서는 JSON, 텍스트, ORC 및 avro도 포함합니다.
```

이처럼 DataFrameWriter의 partitionBy는 파일 시스템에 어떠한 형태로 저장되는지 초점이 맞춰져 있다.
사용시 주의해야 한다.


## 참조
1. [러닝 스파크](http://www.yes24.com/24/goods/21667835?scode=032&OzSrank=1)
2. [Spark의 핵심은 무엇일까?](https://www.slideshare.net/yongho/rdd-paper-review)
3. [Spark RDD Architecture](http://ourcstory.tistory.com/147)
4. [Spark API Doc](http://spark.apache.org/docs/latest/api/scala/index.html#package)
