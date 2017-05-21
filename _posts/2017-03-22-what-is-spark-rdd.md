---
layout: default
title: What is Spark RDD
category: post
author: Jae
---

# What is Spark RDD


> 이 포스트의 내용은 개인적인 공부 목적으로 [Mastering Apache Spark 2](https://www.gitbook.com/book/jaceklaskowski/mastering-apache-spark/details) 정리한 것입니다.

## RDD (Resilience Distributed DataSet)

RDD는 클러스터에 분산된 복원 가능한 Collection이다. 다시 말해서, 하나 이상의 Partition으로 흩어진 Record(e.g. element, row)들의 집합이다.

> RDD v.s. Collection in Scala
>
> * Scala Collection이 단일 JVM에서 계산되는반면 RDD는 Partition 된 체로 많은 JVM에서 계산된다.

RDD를 구성하는 각 알파벳에대해서 자세히 알아보자

###### Resilient - (충격・부상 등에 대해) 회복력 있는; 탄력 있는
* RDD Lineage Graph의 도움으로 fault-tolerant 하기 때문에 노드 장애로 인해 누락되거나 손상된 파티션을 재 계산할 수 있다.

###### Distributed - (널리) 분포된; 광범위한; 광역성의; 분산된 (다수의 컴퓨터로 분산 처리되는)
* 클러스터의 여러 노드에있는 데이터로 분산된다.

###### Dataset - 컴퓨터상의 데이터 처리에서 한 개의 단위로 취급하는 데이터의 집합
* 데이터 집합은 프리미티브 값 또는 Value 값을 가진 분할 된 데이터의 모음 (e.g. 튜플, 객체, 작업하는 데이터의 레코드)

즉, RDD는 여러 분산 노드에 걸쳐서 저장되는 변경이 불가능한 데이터(객체)의 집합으로 각각의 RDD는 여러개의 파티션으로 분리될 수 있다.

추가로 RDD는 다음과 같은 특성도 가진다.

###### In-Memory
* RDD 내의 데이터는 가능한 한 크고(크기) 길게(시간) 메모리에 저장된다.

###### Immutable or Read-Only
* RDD는 일단 생성되면 변경되지 않고 Transformation 연산을 통해서 새로운 RDD로 변환된다.

###### Lazy evaluated
* RDD 내의 데이터는 Action을 실행하기전에는 사용하거나, 변환할 수 없다.

> RDD 연산의 두가지 Type
>
> * transformation: **lazy operations** that return **another RDD**.
> 	* [Spark Docs - Transformations](http://spark.apache.org/docs/latest/programming-guide.html#transformations)
> * action: operations that **trigger computation** and **return values**.
> 	* [Spark Docs - Actions](http://spark.apache.org/docs/latest/programming-guide.html#actions)

###### Cacheable
* RDD 내의 데이터는 메모리 또는 디스크와 같은 영구 "저장소"에 보관할 수 있다.

###### Parallel
* 데이터를 병렬로 처리한다.

###### Typed
* RDD records는 Type을 가진다. e.g. RDD[Long], RDD[(Int, String)]

###### Partitioned
* 레코드는 분할되어 (논리적 파티션으로) 클러스터의 노드에 분산된다.

###### Location-Stickiness
* RDD는 파티션을 계산하기위해서 RDD의 배치 기본 설정을 정의 할 수 있다. (가능한 한 레코드에 가깝게)

> Preferred location (aka locality preferences or placement preferences or locality info)
>
> * RDD Records의 위치에 관한 정보
> * Spark의 DAGScheduler는 작업을 가능한 한 데이터에 가깝게 배치하기 위해 컴퓨팅 파티션을 배치하는 데 사용한다.

![distributed-and-partitioned-RDD](/images/posts/what-is-spark-rdd/distributed-and-partitioned-RDD.png)


#### 정리하면, RDD란.

* Scala의 Collection과 비교할 수 있다.
* 차이점은 일정 크기의 파티션으로 나뉘어서 클러스터에 저장된다는 것이다.
* 나뉘어진 RDD는 Partition 별로 JVM을 할당받아 원하는 계산을 수행할 수 있다.
* 분산 환경이기에 네트워크 단절과 같은 이유로 계산 과정중 일부가 실패할 수 있다.
* 따라서, Fault-tolerant 한 계산 환경이 중요하다.
* 이를 위해서 RDD는 Immutable하고, Lazy Evaluated한 특성을 가진다.
* Immutable한 데이터는 변경될 위험이 없기 때문에 언제든 같은 데이터를 다시 읽을 수 있다.
* Lazy Evaluated 하기 때문에 RDD는 Action을 만나기 전까지 Transformation을 처리하지 않고 그 과정만을 기록한다. (RDD Lineage)
* 이렇게 기록된 RDD Lineage 와 Immutable 하다는 특성을 이용해서 데이터가 유실되면 언제든 다시 만들어낼 수 있는것이다.

> DAG(Directed Acyclic Graph | 비순환 방향 그래프): RDD와 Transformations의 관계(RDD Lineage)를 Graph로 표현
> ![DAG](/images/posts/what-is-spark-rdd/DAG.png)



## 참조
1. [Mastering Apache Spark 2](https://www.gitbook.com/book/jaceklaskowski/mastering-apache-spark/details)
