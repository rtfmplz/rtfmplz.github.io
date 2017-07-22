---
layout: default
title: What is Spark Streaming ?
category: post
author: Jae
comments: true
---

# What is Spark Streaming ?

> 개인 스터디 목적으로 아래 두 문서의 내용중 기초적인 내용을 정리하였다.
> 내용이 다소 부족할 수 있으므로 심화된 내용을 얻고 싶다면, References에 언급된 문서들을 참고한다.
>
> * 러닝 스파크 chaper 10. 스파크 스트리밍
> * [Spark Streaming Programming Guide](https://spark.apache.org/docs/latest/streaming-programming-guide.html)


## Keyword

* DStream (Discretized Stream)
* Time Sequence RDD
* Sliding window
* 24/7
* Checkpoint


## Overview

Spark Streaming은 Core Spark API의 확장판으로 Live data stream을 처리할 수 있다.
Spark Streaming은 Kafka, Flume, Kinesis, TCP등 다양한 경로의 Data Source들로부터 데이터를 받아서 map, reduce, join 등 high-level function을 이용한 Algorithms 처리를 할 수 있다.
이렇게 처리된 데이터는 HDFS(Filesystem), Database에 저장하거나 Console, Live dashboard로 출력할 수 있다.


## A Quick Example

[Spark Streaming Programming Guide](https://spark.apache.org/docs/latest/streaming-programming-guide.html)에 소개된 간단한 예제를 소개한다.

Spark Streaming은 Dependency를 추가함으로써 간단하게 사용할 수 있다. 다음은 SBT를 이용하는 경우이다.

```scala
libraryDependencies += "org.apache.spark" % "spark-streaming_2.11" % "2.1.1"
```

다음은 9999 포트에서 운영중인 서버에서 문자열을 받아들여 사용된 단어를 집계하는 Spark Streaming Application 이다.

```scala
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.dstream.{DStream, ReceiverInputDStream}

object SparkStreamingExample extends App {
  val conf: SparkConf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
  val ssc: StreamingContext = new StreamingContext(conf, Seconds(1))

  val lines: ReceiverInputDStream[String] = ssc.socketTextStream("localhost", 9999)
  val words: DStream[String] = lines.flatMap(_.split(" "))

  val pairs: DStream[(String, Int)] = words.map(word => (word, 1))
  val wordCounts: DStream[(String, Int)] = pairs.reduceByKey(_ + _)

  wordCounts.print()

  ssc.start()
  ssc.awaitTermination()
}
```

1. StreamingContext 생성
    * SparkContext를 생성할때와 마찬가지로 SparkConf를 입력 받는다.
    * SparkContext를 생성과의 차이점은 실시간으로 들어오는 데이터를 Batch Job으로 분할하기 위한 Interval을 입력 받는다.
    * StreamingContext는 내부적으로 SparkContext를 생성한다.
2. `socketTextStream()`을 사용하여 localhost:9999 로부터 들어오는 문자열 데이터에 기반하여 DStream을 생성한다.
3. 이후 작업은 RDD에 transformation 연산을 수행하는 것과 동일한 방법으로 수행한다.
4. 마지막으로 DStream을 출력한다.
	* `print()` 연산은 데이터를 외부 시스템에 쓴다는 점에서 RDD의 Action과 같다고 생각할 수 있다. 하지만 차이점은 Action과 다르게 이 라인에서 실제 처리가 시작되지는 않는다는 것이다.
    * `print()` 연산은 매 시간 간격마다 주기적으로 실행되며 출력을 배치단위로 생성한다.
5. `start()`에 의해서 StreamingContext가 연산을 시작하고, `awaitTermination()`으로 user에 의한 Termination이 올때까지 대기한다.
    * `start()`가 호출되면 StreamingContext는 내부의 SparkContext에 Spark Job을 스케쥴링 한다.

natcat을 이용해서 문자열을 흘려주면..

```bash
nc -lp 9999
Hello world
```

아래와 같은 결과를 볼 수 있다.

```bash
--------------------------------------------
Time: 10000304034503 ms
--------------------------------------------
(Hello,1)
(world,1)
...
```


## Batch & DStream

SparkStreaming은 "Micro-Batch" 아키텍처를 사용한다.
Micro-Batch는 스트리밍 데이터를 작은 배치 단위로 분리하고 각 배치 처리의 연속적인 흐름으로 간주한다.
새로운 배치들은 정해진 시간 간격(Interval)마다 생성된다. Interval 동안 데이터를 받아들인 후, 시간 간격의 마지막에 배치 추가가 완료된다.

#### DStream (Discretized Stream)

DStream은 외부 Input Source로 부터 생성하거나, 기존 DStream을 Transformation하여 생성할 수 있다. DStream은 연속적인 RDD의 묶음이며, 각 RDD에는 Batch 간격 동안 유입된 데이터가 들어 있다.

![rdd-in-dstream](/images/posts/what-is-spark-streaming/rdd-in-dstream.png)

DStream에 적용된 모든 연산은 RDD의 연산으로 변환된다. 아래 그림에서 처럼 `lines: DStream[String]`의 각 RDD에 `flatMap` 연산이 적용되어 `words:DStream[String]`의 RDD가 생성된다.

![streaming-dstream-ops](/images/posts/what-is-spark-streaming/streaming-dstream-ops.png)

#### Input DStream and Receivers

Input DStream은 Streaming Source로 부터 수신된 데이터를 표현하는 DStream이다. (앞의 예제에서 `lines`) File Source에 의해서 생성된 Input Stream을 제외한 모든 Input DStream은 `Receiver` object와 연결된다.
Receiver는 Input Source로부터 데이터를 수신하고 spark의 메모리에 저장하는 역할을 수행한다.

Spark Streaming은 두가지 Category의 Streaming Source를 제공한다.

* Basic Source: from file system, socket connections
* Advanced Source: from Kafka, Flume, Kinesis

> Advanced Source 를 이용하기 위해서는 Dependency를 추가해 주어야 한다. 다음은 SBT에 Kafka dependency를 추가하는 방법이다.
>
> ```scala
> libraryDependencies += "org.apache.spark" % "spark-streaming-kafka-0-10_2.11" % "2.1.1"
> ```

Input DStream을 여러개 생성하면, 한 개 이상의 Input Source로 부터 Multiple DStream을 Parallel하게 받는것도 가능하다. Spark Streaming은 각 Input Source마다 Receiver를 생성하여 각각의 Input Source로 부터 Data를 수신할 수 있다.

> Receiver와 Spark Core 수
>
> * Receiver는 독립된 Thread(Worker)에서 실행되므로 Spark Streaming을 local mode로 실행하는 경우 "local[n]"의  n은 Receiver의 갯수보다 많아야 한다. (그렇지 않으면 수신된 Data를 처리해야할 Executor가 생성되지 않아 Application이 정상 동작하지 않는다.)
> * Cluster에서 실행하는 경우에도 Receiver의 수보다 많은 수의 Core를 Spark Streaming Application에 할당해야 한다.


## Transformations on DStreams

DStream의 Transformation Operation은 Stateless와 Stateful로 나눌 수 있다.
* Stateless: 각 배치의 처리가 앞쪽의 배치들의 데이터와 상관없이 처리된다.
	* map, filter, reduceByKey 같은 일반적인 Transformation
* Stateful: 현재의 배치의 결과를 만들기 위해서 이전 배치의 데이터나 중간 결과를 이용한다.
	* 슬라이딩 윈도우와 시간별 상태 추척을 바탕으로 하는 Transformation

#### Stateless Transformation

각 DStream은 Batch 단위에 따라서 여러개의 RDD로 구성되며 Stateless Transformation은 각 RDD에 구분되어 적용된다.

> 예를 들어, reduceByKey()는 각 시간 단계(Batch, RDD)의 데이터를 병합하는 것이지 DStaream의 모든 데이터를 병합하지는 않는다.

#### Stateful Transformation

Stateful Transformation은 시간 단계 범위를 넘어서 데이터를 추적하는 DStream의 연산이다.

> Stateful Transformation은 StreamingContext의 Checkpoint를 활성화 해주어야 한다. Checkpoint의 자세한 내용은 뒤에서 다룬다.
>
> ```scala
> ssc.checkpoint("CHECK_POINT_DIRECTORY")
> ```

###### updateStateByKey()

Key-Value 쌍의 DStream에 대한 상태를 update 한다.
`updateStateByKey()`는 새 이벤트에 주어진 각 키를 업데이트 하도록 정의된 함수를 입력 받아서 새로운 Key-Value DStream을 생성한다.

예를들어 `updateStateByKey()`를 이용하면 앞에서 소개한 WordCounter를 전체 DSteam에 대해서 수행하도록 수정할 수 있다. (이전 예에서는 각 Batch별로 Count가 집계 된다.)

```scala
def updateCount(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
  Some(runningCount.getOrElse(0) + newValues.sum)
}
val runningCounts: DStream[(String, Int)] = pairs.updateStateByKey(updateCount)
runningCounts.print()
```

###### window()

Window 류 연산들은 Window duration 만큼의 Batch를 모아서 Sliding Duration 간격으로 Windowed DStream을 생성한다.

![streaming-dstream-window](/images/posts/what-is-spark-streaming/streaming-dstream-window.png)

> Window duration과 Sliding duration은 StreamContext의 배치 간격의 배수여야 한다. 그렇지 않으면 아래와 같은 Exception이 발생한다.
>
> ```scala
> Exception in thread "main" java.lang.Exception: The window duration of windowed DStream (3000 ms) must be a multiple of the slide duration of parent DStream (10000 ms)
> ```

가장 기본적인 `window()`연산을 써서 모든 다른 window 연산을 구현할 수 있지만 SparkStreaming은 편리를 위해 다수의 window 연산을 제공한다.
예를들어 `reduceByKeyAndWindow()` 연산을 이용해서 앞에서 소개한 WordCounter를 3개의 Batch에 대해서 3초마다 한번씩 연산을 반복하도록 할 수 있다. (Batch 간격은 1초)

```scala
val windowWordCounts = pairs.reduceByKeyAndWindow((a:Int, b:Int) => a+b, Seconds(3), Seconds(3))
val windowWordCounts = pairs.countByValueAndWindow(Seconds(3), Seconds(3))
windowWordCounts.print
```

inverseReduceFunc()을 인자로 가진 함수는 이미 계산된 값에 window에 추가되는 Batch의 값은 더하고 제외되는 Batch의 값은 제거하는 식으로 동작하기 때문에 더 효과적이다.

![streaming-dstream-window-inverse](/images/posts/what-is-spark-streaming/streaming-dstream-window-inverse.png)

```scala
val windowWordCounts = pairs.reduceByKeyAndWindow((a:Int, b:Int) => a+b, (a:Int, b:Int) => a-b, Seconds(3), Seconds(3))
```

#### join & transform

Join 연산은 다양한 형태로 가능하다.

###### Stateless Stream - Stateless Stream Join

```scala
val stream1: DStream[String, String] = ...
val stream2: DStream[String, String] = ...
val joinedStream = stream1.join(stream2)
```

###### Stateful Stream - Stateful Stream Join

```scala
val windowedStream1 = stream1.window(Seconds(20))
val windowedStream2 = stream2.window(Minutes(1))
val joinedStream = windowedStream1.join(windowedStream2)
```

###### Stream - RDD Join

```scala
val dataset: RDD[String, String] = ...
val windowedStream = stream.window(Seconds(20))...
val joinedStream = windowedStream.transform { rdd => rdd.join(dataset) }
```

> `transform(func)`은 `func`을 DStream의 Element(Row)가 아닌 RDD에 직접 적용한다.

## Output Operations on DStreams

Output Operation은 결과를 외부 데이터베이스에 보내거나 화면에 출력해 준다.
RDD가 Action 연산이 없으면 lazy evaluation 방식에 의해 어떤 수행도 일어나지 않는것 처럼 DStream 역시 Output Operation이 없으면 Transformation 등 데이터 처리에 필요한 어떤 연산도 수행되지 않는다.

`foreachRDD()`는 DStream의 RDD들에 임의의 연산을 수행하게 해준다. (각 RDD에 직접 접근하게 해주는 `transform()`과 유사하다.) `foreachRDD()` 내부에서 Spark이 가지고 있는 모든 Action을 사용할 수 있다.
예를 들면, 데이터를 HDFS에 Parquet으로 저장거나 TempTable로 등록해서 SQL로 검색하는 등의 동작을 수행할 수 있다.

```scala
words.foreachRDD((rdd: RDD[String], time: Time) => {
    val sqlContext = SQLContextSingleton.getInstance(rdd.sparkContext)
    import sqlContext.implicits._
    val wordsDataFrame = rdd.map(w => Record(w)).toDF()
    wordsDataFrame.write.mode(SaveMode.Append).parquet("/tmp/parquet");
    })
```

> IMPORTANT !!
>
> * `foreachRdd()`는 Output Operation으로 분류되지만 내부에 RDD의 Action이 없다면 연산이 시작되지 않는다.


```scala
words.foreachRDD { rdd =>

    val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()
    import spark.implicits._

    val wordsDataFrame = rdd.toDF("word")
    wordsDataFrame.createOrReplaceTempView("words")

    val wordCountsDataFrame =
      spark.sql("select word, count(*) as total from words group by word")
    wordCountsDataFrame.show()
}
```


## Caching / Persistence

`persist()`를 이용해서 DStream의 모든 RDD를 메모리에 저장할 수 있다.
Stateful Transformation 들은 암시적으로 RDD를 메모리에 저장하기 때문에 개발자가 `persist()`를 호출할 필요가 없다.
네트워크를 통해 데이터를 수신된 Input DStream (예: Kafka, Flume, 소켓 등)의 Default Persistence level은 fault-tolerance를 위해서 두 개의 노드에 데이터를 복제하도록 설정된다.

> RDD와 다르게 DStream의 Default Persistence level은 `MEMORY_ONLY_SER` 이다. (RDD는 `MEMORY_ONLY`를 사용한다.)


## Checkpointing

SparkStreaming Application은 24/7 운영될 수 있도록 Application Logic과 관련없는 시스템 장애에 대응하기 위해서 "Checkpoint" 기능을 제공한다.

Checkpoint에는 크게 두가지 타입이 있다.

* MetaData Checkpoint: Driver Program의 복구 하기위한 데이터를 저장한다. 주로 저장되는 데이터는 아래와 같다.
	* Application Configuration
	* DStream Operation
	* Incompleted Batches
* Data Checkpoint: Spark는 RDD Lineage를 기록함으로써 데이터 복구를 가능하게 하지만, Stateful Operation을 사용하는 경우 window size 등에 따라 복구 시간이 증가할 수 있다. 따라서, 중간 데이터를 저장해서 시간을 단축 시킬 수 있다.

결국 정리하면, Application이 다음 두 가지 요구사항을 만족해야 하는 경우 Checkpointing이 필요하다.
* Driver 의 실패에 대한 복구가 필요한 경우
* Staeful Transformation을 사용하는 경우

#### Driver Fault Tolerance

Fault Tolerance 한 Driver Program을 구현하기 위해서는 다음과 같이 checkpoint directory에서 설정을 얻어 올 수 있도록 프로그램을 작성해야 한다.

다음 예제는 최초에 프로그램이 실행될 때, StreamingContext를 생성하고, `start()`를 호출한다. 이 후 프로그램이 오류로 인해 종료된 후 다시 실행되면, checkpoint data로부터 StreamingContext를 재 생성한다.

```scala
// @see https://github.com/apache/spark/blob/master/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala

// Function to create and setup a new StreamingContext
def functionToCreateContext(): StreamingContext = {
  val ssc = new StreamingContext(...)   // new context
  ssc.checkpoint(checkpointDirectory)   // set checkpoint directory
  val lines = ssc.socketTextStream(...) // create DStreams

  // Transformation and Output Operations for your program

  ssc
}

// Get StreamingContext from checkpoint data or create a new one
val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)

// Do additional setup on context that needs to be done,
// irrespective of whether it is being started or restarted
context. ...

// Start the context
context.start()
context.awaitTermination()
```

Fault Tolerance 한 Driver Program을 구현했지만 실제로 Driver Program을 Monitoring 하고 재시작 해주는 역할은 Cluster Manager에 의해서 수행된다.

각각의 Cluster Manager가 Driver Program을 오류로 부터 재시작하기 위한 방법은 다음과 같다.
* Spark Standalone: spark-submit에 Application 제출시 `--supervise` 옵션을 추가한다.
* YARN: yarn-cluster mode로 실행 시 Driver Program이 특정 Worker Node에서 실행되며 오류로 종료되면 YARN에 의해서 자동으로 재실행 된다.
* Mesos: 자원 할당과 Job 생성을 담당하는 Marathon Framework에서 알아서 처리한다.

> Checkpoint interval
>
> Checkpoint는 안정적인 Storage에 데이터를 저장하므로써 전체 연산에 대한 비용을 절감할 수 있다. 하지만 과도한 Checkpointing은 Batch의 Processing Time을 증가시키게되어 Application의 성능을 저하시킨다.
> Application마다 Tuning이 필요한 부분이지만, 일반적으로 Checkpoint interval은 DStream의 슬라이딩 간격의 5~10배로 설정하는 것이 좋다.

## Streaming UI

Streaming UI를 통해서 Application의 상태를 확인 할 수 있다.

아래 화면을 통해서 Receiver가 Application의 전체 실행 시간동안 Active Job으로 유지되는 것을 확인할 수 있다.
StreamingContext 생성시 Batch 간격을 1초로 설정하였기 때문에 1초에 한번씩 주기적으로 Job이 생성된다.

![web-ui](/images/posts/what-is-spark-streaming/web-ui.png)

Streaming Tab에서는 Input Rate, Processing Time, Delay 등을 Monitoring 할 수 있다.

![web-ui-streaming-tab](/images/posts/what-is-spark-streaming/web-ui-streaming-tab.png)

## References
* https://spark.apache.org/docs/latest/streaming-programming-guide.html
* http://henning.kropponline.de/2015/04/26/spark-streaming-with-kafka-hbase-example/
* https://community.hortonworks.com/articles/72941/writing-parquet-on-hdfs-using-spark-streaming.html
