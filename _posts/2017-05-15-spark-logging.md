---
layout: post
title: Spark Logging 정리
tags: [spark]
author: Jae
comments: true
---

# Spark Logging 정리

Spark은 Apache Log4j 를 이용해서 Log를 남긴다.
SparkContext가 생성(Spark Application, Spark-shell 같은 Driver Program에 의해서 생성된다.) 될 때, `${SPARK_HOME}/conf/log4j.properties` 를 참조한다.

## log4j.properties 설정하기

log4j의 설정을 변경하기 위해서는 `log4j.properties`를 사용하는데, Spark에서는 `${SPARK_HOME}/conf/log4j.properties.template`을 `${SPARK_HOME}/conf/log4j.properties`로 복사한 후 자신의 시스템에 맞게 수정하여 사용한다.
`log4j.properties` 에서 사용가능한 설정들은 [Configuration with Properties](https://logging.apache.org/log4j/2.x/manual/configuration.html#Properties) 를 참고한다.
다음은 `log4j.properties`의 간단한 예제이다.

```bash
# Set everything to be logged to the console
log4j.rootCategory=INFO, console, AsdConsoleAppender
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.target=System.err
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n

# Settings to quiet third party logs that are too verbose
log4j.logger.org.eclipse.jetty=WARN
log4j.logger.org.eclipse.jetty.util.component.AbstractLifeCycle=ERROR
log4j.logger.org.apache.spark.repl.SparkIMain$exprTyper=INFO
log4j.logger.org.apache.spark.repl.SparkILoop$SparkILoopInterpreter=INFO

# Set everything to be logged to the File
log4j.appender.RollingFileAppender=org.apache.log4j.DailyRollingFileAppender
log4j.appender.RollingFileAppender.File=/var/log/spark/local.log
log4j.appender.RollingFileAppender.DatePattern='.'yyyy-MM-dd
log4j.appender.RollingFileAppender.layout=org.apache.log4j.PatternLayout
log4j.appender.RollingFileAppender.layout.ConversionPattern=[%p] %d %c %M - %m%n

# logger
log4j.rootLogger=INFO, console, RollingFileAppender
log4j.logger.MyLogger=INFO, console, RollingFileAppender
```


> ##### Local Mode v.s. YARN Mode
> 최초에 log4j.properties에는 file appender가 빠져있기 때문에 local에서 spark을 실행하면 log들이 console을 통해서 출력되고 file로는 남지 않는다. 따라서 local에서 spark을 실행 할 예정이라면 file appender 설정해 주어야한다.
>
> YARN을 이용해서 Spark을 실행시키는 경우에는 console로 출력(stdout)하는 log도 YARN이 알아서 저장해준다. (logger 설정 없이 println으로 로그를 출력할 수 있다.)
Application이 종료된 후, 아래 명령어로 yarn cluster 어디서든지 log 확인 할 수 있다.
>```bash
>yarn logs -applicationId <app ID>
>```

---

> ##### Log4j 2
> Log4j 2는 기존 Properties 파일 형식의 환경 설정을 지원하지 않으며, XML (log4j2.xml) 혹은 JSON (log4j2.json or log4j2.jsn) 파일 형식의 환경 설정만 가능하다.  
> 자세한 내용은 다음 페이지를 참고한다.
>
> * [Log4j 2 설정하기](http://dveamer.github.io/java/Log4j2.html)
> * [Log4j 2 환경설정 (설정 파일 사용 시)](http://www.egovframe.go.kr/wiki/doku.php?id=egovframework:rte3:fdl:%EC%84%A4%EC%A0%95_%ED%8C%8C%EC%9D%BC%EC%9D%84_%EC%82%AC%EC%9A%A9%ED%95%98%EB%8A%94_%EB%B0%A9%EB%B2%95)


## Spark Application에서 Logger 사용하기
Spark Application 작성시 `org.apache.log4j.LogManager`를 이용해서 logger를  사용할 수 있다.

```scala
object app {
  def main(args: Array[String]) {
    val log = LogManager.getRootLogger
    log.setLevel(Level.WARN)

    val conf = new SparkConf().setAppName("spark-app")
    val sc = new SparkContext(conf)

    log.warn("Hello World!")

    val data = sc.parallelize(1 to 100000)

    log.warn("I am done")
  }
}
```


## Spark Application에서 Logger 사용시 Serializable 문제

> 본 포스트에서는 `org.apache.log4j.Logger` 클래스를 사용하는 방법만 다룬다.  
> Spark에서의 `NotSerializableException` 문제는 추후에 분리해서 자세히 다룬다.

`org.apache.log4j.Logger` 클래스는 직렬화 할 수 없다. 이것은 Spark Application 구현시 Logger를 `Closure` 내부에서 사용할 수 없다는 것을 의미한다.

> Scala Closure
>
> * Closure 는 function 의 범위 밖에 있는 variable 을 function 에서 쓸수있는 기능
> * 참조: [Closure with mutable variable in Scala(스칼라)](http://coding-korea.blogspot.kr/2013/04/closure-with-mutable-variable-in-scala.html)

즉, 아래 예제에서 log 객체는 serializable 하지 않기 때문에 네트워크를 통해서 Spark worker 들에게 보내질 때 `Task not serializable: java.io.NotSerializableException `이 발생한다.

```scala
val log = LogManager.getRootLogger
val data = sc.parallelize(1 to 100000)

data.map {
  value =>
    log.info(value)
    value.toString
}
```

#### Solution

Spark에서 일어나는 Serialization 문제를 해결 하는 방법은 객체를 Top-level object의 변수로 만드는 것이다.
아래 간단한 예제를 제공한다.

> Top-level object는 sparkContext가 생성될 때 각 worker들에 이미 전송되어지기 때문에 프로그램 수행 중간에 serialization해서 전송하지 않는다. 따라서 `NotSerializableException`이 발생하지 않는다.

```scala
import org.apache.log4j.{Level, LogManager, PropertyConfigurator}
import org.apache.spark._
import org.apache.spark.rdd.RDD

class Mapper(n: Int) extends Serializable{
  @transient lazy val log = org.apache.log4j.LogManager.getLogger("myLogger")


  def doSomeMappingOnDataSetAndLogIt(rdd: RDD[Int]): RDD[String] =
    rdd.map{ i =>
      log.warn("mapping: " + i)
      (i + n).toString
    }
}

object Mapper {
  def apply(n: Int): Mapper = new Mapper(n)
}

object app {
  def main(args: Array[String]) {
    val log = LogManager.getRootLogger
    log.setLevel(Level.WARN)

    val conf = new SparkConf().setAppName("demo-app")
    val sc = new SparkContext(conf)

    log.warn("Hello demo")

    val data = sc.parallelize(1 to 100000)
    val mapper = Mapper(1)
    val other = mapper.doSomeMappingOnDataSetAndLogIt(data)
    other.collect()

    log.warn("I am done")
  }
}
```

## Yarn Cluster 환경에서 Log 남기기

YARN Cluster 환경에서 Executor와 Application Master는 "container" 안에서 구동된다. YARN은 Spark Application이 종료된 후에 컨테이너 로그를 처리할 수 있다.

로그 집계가 켜져 있으면 (yarn.log-aggregation-enable=true) 컨테이너 로그가 HDFS로 복사되고 로컬 시스템에서 삭제된다. 복사된 로그는 yarn logs 명령을 사용해서 클러스터 어디서든지 확인할 수 있다.

로그 집계가 켜져있는 경우 다음 두가지 방법으로 로그를 조회할 수 있다.

1. yarn logs command 활용하기
```bash
yarn logs -applicationId <app ID>
```
위의 명령을 이용해서 <app ID>와 관련된 모든 컨테이너의 로그를 모아서 출력할 수 있다.

2. Spark Web UI의 Executors tab 에서 확인하기
이 경우 Spark history server와 MapReduce history server가 실행되어야 한다.

로그 집계가 켜져 있지 않으면 로그는 YARN_APP_LOGS_DIR 아래의 각 시스템에서 로컬로 유지되며, 컨테이너에 대한 로그를 보려면 호스트로 직접 이동해서 로컬 디렉토리를 확인 해야한다.

> 기본 설정으로 `yarn.log-aggregation-enable=true` 로 설정되어 있으므로 수정하지 않는편이 좋다. :)


##### Customed log4j.properties 사용하기

다음 방법들을 통해서 YARN Cluster 환경에서 Customed log4j.properties 파일을 사용할 수 있다. 소개된 순서대로 우선순위가 높다.

1. spark-submit 실행시 `--files` option을 이용해서 customed log4j.properties 파일을 추가한다.
```bash
--files log4j-executor.properties
```
2. 사용할 파일의 경로를 extraJavaOptions 를 통해서 전달해준다. (이 경우 파일은 각 서버에 미리 위치해야 한다.)
```bash
--conf "spark.driver.extraJavaOptions=-Dlog4j.configuratio=<log4j.properties 파일의 로컬 저장 경로>"
--conf "spark.executor.extraJavaOptions=-Dlog4j.configuratio=<log4j.properties 파일의 로컬 저장 경로>"
```
3. `$SPARK_CONF_DIR/log4j.properties` file을 업데이트 하면 자동으로 적용된다.


## 요약

* Spark은 Log4j를 이용하므로 Log Format은 Log4j.properties 설정을 잘 해주면 된다.
* `import org.apache.log4j.Logger` 는 `serializable` 하지 않기 때문에 Top-level object의 변수에 객체를 담아 사용한다.
* YARN Cluster 환경에서는 Application이 종료된 후 Log를 aggregation 할 수 있다.


## TrubleShooting

log4j의 RollingFileAppender를 사용 할 때, 아래와 같이 File 경로를 지정할 수 있다.
`log4j.appender.RollingAppender.File=/home/dnadb/logs/root.log`
이때 생성되는 log 파일의 경로와 SparkContext를 생성한 User가 서로 다르면 권한 문제로 
`java.io.FileNotFoundException (Permission denied)` 가 발생한다.


## 참조

* [Spark Doc](http://spark.apache.org/docs/latest/configuration.html)
* [debugging-your-application](http://spark.apache.org/docs/latest/running-on-yarn.html#debugging-your-application)
* [how-log-apache-spark](https://mapr.com/blog/how-log-apache-spark/)
* [Scala and the '@transient lazy val' pattern](http://fdahms.com/2015/10/14/scala-and-the-transient-lazy-val-pattern/)