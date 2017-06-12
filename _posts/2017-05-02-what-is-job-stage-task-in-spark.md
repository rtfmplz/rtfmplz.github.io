---
layout: default
title: What is job, stage, task in spark
category: post
author: Jae
comments: true
---

# JOB, STAGE, TASK in SPARK

> 이 포스트의 내용은 개인적인 공부 목적으로 [Mastering Apache Spark 2](https://www.gitbook.com/book/jaceklaskowski/mastering-apache-spark/details) 정리한 것입니다.


다음은 spark-shell에서 RDD를 생성하고, Transformation, Action 한 예제를 보여준다.

```scala
scala> val fruits = sc.parallelize(List("apple", "orange", "strawberry"))
fruits: org.apache.spark.rdd.RDD[String] = ParallelCollectionRDD[1467] at parallelize at <console>:12

scala> val filteredFruits = fruits.filter(_ != "strawberry")
filteredFruits: org.apache.spark.rdd.RDD[String] = MapPartitionsRDD[2] at filter at <console>:37

scala> // 원한다면 추가로 Transformation을 넣을 수 있다.

scala> filteredFruits.reduce((t1,t2) => t1 + t2)
res1: String = orangeapple
```


이번 Post에서는 위와 같은 Code가 실행되는 과정을 Job, Stage, Task 관점에서 간단하게 살펴보려고 한다.

아래 그림에서처럼 RDD의 Action은 RDD가 Job이 되는 트리거가 된다.

![RDD-to-Job](/images/posts/what-is-job-stage-task-in-spark/RDD-to-Job.png)

결국 Job이란 Partition으로 구성된 RDD로부터 (일련의 Transformation을 적용한 후) Action의 결과를 얻어내기 위해 DAGScheduler에 넘겨진 Action Item이다.

> 즉, Job이란 Action의 결과를 얻어내기 위해서 해야하는 일(Work)

Job은 Action의 결과를 얻기위해서 RDD에 Transformation들을 적용하는 일련의 순서를 표현한 RDD Lineage 포함한다. DAGScheduler는 이 RDD Lineage를 분석해서 Stage로 표현된 물리적 실행 계획을 만든다.

> DAG(Directed Acyclic Graph | 비순환 방향 그래프) Scheduler
>
> * Job을 위한 execution DAG 즉, DAG of stages 만든다.
> * 각 Task를 실행할 기본 위치를 결정한다.
> * shuffle output 실패를 Handling 한다.
>
> DAGScheduler 는 logical execution plan (i.e. RDD lineage of dependencies built using RDD transformations) 을 physical execution plan (using stages)으로 변환한다.
>
> ![dagscheduler-rdd-lineage-stage-dag](/images/posts/what-is-job-stage-task-in-spark/dagscheduler-rdd-lineage-stage-dag.png)
> DAGScheduler Transforming RDD Lineage Into Stage DAG


> * RDD Lineage는 toDebugString() method를 이용해서 출력할 수 있다.

Stage가 나뉘어지는 기준은 shuffle dependencies에 의해서 정해진다. 즉, Shuffle을 발생시키는 join, repartition과 같은 연산이 Stage가 나뉘어지는 기준이 된다.

![graph-of-stage](/images/posts/what-is-job-stage-task-in-spark/graph-of-stage.png)

> 즉, Stage란 DAGSchedualr에 의해서 생성된 물리적 실행 계획의 단계(Step)들이다.
> 하나의 Stage를 처리하기 위해서는 하나 이상의 Task가 필요하다.

마지막으로 Task는 RDD의 각 파티션을 처리하기 위한 가장 작은 개별 실행 단위다. Spark는 각각의 파티션을 처리할 Task를 하나씩 만들고, 각각의 Task는 클러스터에 하나의 코어를 요청한다. 생성된 Task는 serialized되어, 각 worker노드의 executors로 분배된다.

![spark-rdd-partitions-job-stage-tasks](/images/posts/what-is-job-stage-task-in-spark/spark-rdd-partitions-job-stage-tasks.png)

> 즉, Task란 어떤 Stage 가 처리되기 위해서 계산되어야 하는 partition에 대한 작업으로 executor에 의해 처리된다.

#### 짧게 정리하면,

* Task - 개별 파티션에 대한 연산을 처리하는 작업, JVM에 의해 실행
* Stage - Parallel Tasks의 집합
* Job - Stage로 분할된 계산들의 조합


## 출처
1. [Mastering Apache Spark 2](https://www.gitbook.com/book/jaceklaskowski/mastering-apache-spark/details)