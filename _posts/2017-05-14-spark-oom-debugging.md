---
layout: default
title: Spark의 OutOfMemoryError 분석
category: post
author: Jae
---

# Spark의 OutOfMemoryError 분석

> Spark Application을 돌리다보면 FileNotFoundException 종종 보게된다. 도대체 왜! 무슨 File을 못찾겠다는 것인가.. 오늘 잃어버린 그 File을 찾아보자.

## FileNotFoundException?? OutOfMemoryError!!

FileNotFoundException 의 원인은 보통 Spark Executor 프로세스의 의도치 않은 종료(kill)이다. (뭐 간혹 프로그램을 잘못짜서 진짜로 파일이 없을수도 있겠다.. 하하..) Driver는 Job을 분할한 Task를 Executor에게 분배하고 일이 끝나면 결과를 돌려받기를 기대한다. 하지만 Executor가 죽어버려서 받아야 할 데이터(File)를 찾지 못하면 FileNotFoundException이 발생하는 것이다.

종료된 Executor 의 에러 로그를 확인하면 "java.lang.OutOfMemoryError: Requested array size exceeds VM limit" 에러가 주 원인이고, Executor 프로세스가 의도치 않게 종료되면서 FileNotFoundException 이 발생하는 것을 확인할 수 있다.

결국 우리가 진짜 해결해야 할 문제는 왜 Executor가 Task를 처리하는 도중 OutOfMemoryError를 발생시키고 종료되었는지 원인을 찾아서 해결하는 것이다.

> 언제나 느끼는 거지만 Parallel 환경의 Debugging은 어렵다..

## OutOfMemoryError in Spark

그럼 Spark에서 OutOfMemoryError 가 발생하는 원인을 생각해보자. OOM의 원인은 보통의 경우 Join, groupBy와 같은 shuffle을 발생시키는 연산들이다. 이런 연산들은 문제는 데이터의 shuffle 과정에서 특정 Key에 과도하게 많은 Value가 몰릴 수 있다는 것이다.

다시 말하면, 특정 Key가 다수의 Value를 가지는 1:N 관계를 가진다면 하나의 Task에 다수의 Record가 몰리게된다. 이는 분할되지 않고 메모리에 한번에 올라가야하는 데이터가 많다는 이야기이고, Executor의 메모리는 Spark을 구동할 때 당신이 정한 수치로 고정되어 있기 때문에 Executor(JVM)는 OOM Error를 발생시키고 운명하게 된다.

이런 문제는 Spark 클러스터 장비를 늘려도 해결되지 않으며, 각 장비의 메모리를 증설(Spark 할당 메모리 증설)로는 근본적인 해결책이 될 수 없다.


## Solution

Executor의 OutOfMemoryError 의 해결할 수 있는 방법을 소개한다.

1. Executor 당 Memory를 증가시킨다.
	* N의 크기가 별로 크지 않은 경우, Executor의 Memory를 약간 증가시키는 것으로 의외로 쉽게 문제를 해결할 수 있다.
	* 평소에 운이 좋은 사람은 이 방법으로 문제를 해결할 수도 있겠다.
	* `spark.executor.memory` 옵션을 사용해서 조정할 수 있다.

2. partition의 수를 늘린다.
	* Join시 partition 수를 명시적으로 정할 수 있는데 이때 큰값을 준다. (Spark의 default partition 값은 200 이다.)
	* Partition의 크기가 증가하면 특정 Task, 즉 Executor에 몰리는 데이터가 일부 분산되어 문제가 해결될 수 있다.
	* 하지만 특정 Key에 데이터가 몰려있는 상황이라면 근본적인 해결책은 되지 않는다.
	* 다음과 같은 방법으로 default partition 수를 변경 할 수도있다.
    ```sql
    sqlContext.sql("set spark.sql.shuffle.partitions=800")
    sqlContext.setConf("spark.sql.shuffle.partitions", "800")
    ```

3. 특정 Key가 가질 수 있는 Value의 수를 제한하는 것이다. 즉, N의 Limit를 정한다.
	* 이 방법은 문제를 해결할 수 있지만 데이터를 누락시키게 된다.
	* 다음과 같은 방법으로 rownum, 즉 N값, 을 제한할 수 있다.
	```scala
    // @see https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$
    import org.apache.spark.sql.expressions.Window
    val w = Window.partitionBy("KEY")//.orderBy("CTIME")
    val outputDF = inputDF.withColumn("ROWNUM", row_number.over(w)).filter("ROWNUM <= 1").drop("ROWNUM")
    ```

4. Input source를 분할하여 처리한다.
	* 처리해야 하는 데이터를 나눠서 처리한다.
	* 구현이 다소 복잡해지고 시간이 오래 걸릴 수 있다.

위에서 소개한 해결책들은 결국 Executor의 메모리를 늘리거나 하나의 Executor에 몰리는 데이터를 줄여주는 방법이다. **선택은 당신의 몫**


## 상세 분석 Step
1. Spark UI에서 실패한 Stage의 상세 페이지를 접근하여 Executor의 의도치 않은 종료를 확인한다.
![confirm-executor](/images/spark-oom-debugging/confirm-executor.png)
2. Spark UI의 Executors 메뉴에서 종료된 Executor error log를 확인한다.
![confirm-log](/images/spark-oom-debugging/confirm-log.png)
3. 종합 로그 내용 중 stderr full log를 확인한다.
![read-stderr](/images/spark-oom-debugging/read-stderr.png)
4. 로그의 마지막 부분부터 거슬러 올라가며 종료 원인을 확인한다.
OutOfMemoryError(Array 사이즈 증가 실패)로 인해 종료 시그널 발생, 종료 훅이 호출되어 의도치 않은 비정상 종료 과정 중 FileNotFoundException이 발생하고 종료됨을 확인 할 수 있다.
![stderr-full-log](/images/spark-oom-debugging/stderr-full-log.png)
