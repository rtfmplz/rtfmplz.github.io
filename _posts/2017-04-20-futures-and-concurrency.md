---
layout: default
title: Futures and Concurrency in Scala
category: post
author: Jae
---

> Programming in Scala by Martin Odersky 를 개인적인 공부 목적으로 정리한 포스트 입니다.

#Futures and Concurrency

> http://docs.scala-lang.org/overviews/core/futures.html
> http://www.scala-lang.org/files/archive/api/current/
> http://www.scala-lang.org/files/archive/api/current/scala/concurrent/ExecutionContext.html
> http://www.scala-lang.org/files/archive/api/current/scala/concurrent/Future.html

멀티 코어 프로세서의 확산으로 인해 동시성에 대한 관심이 높아졌다.  
Java는 공유 메모리 및 잠금을 기반으로하는 동시성 프로그래밍을 지원한다. 하지만, Java의 공유 메모리와 장금을 사용해서 잘 돌아가는 동시성 프로그램을 작성하기란 매우 어렵다.

* Why??
	* race condition
	* deadlock

스칼라의 표준 라이브러리는 "asynchronous transformations of immutable state"를 통해서 이러한 어려움을 피할 수있는 대안을 제공한다.

* What??
	* Future !!

Java 또한 `Future` 를 제공한다.

* java Future API

```java
// Attempts to cancel execution of this task.
boolean cancel(boolean mayInterruptIfRunning)

// Waits if necessary for the computation to complete, and then retrieves its result.
V get()

// Waits if necessary for at most the given time for the computation to complete, and then retrieves its result, if available.
V get(long timeout, TimeUnit unit)

// Returns true if this task was cancelled before it completed normally.
boolean isCancelled()

// Returns true if this task completed.
boolean isDone()
```

* Java Future의 예제

```java
 interface ArchiveSearcher { String search(String target); }
 class App {
   ExecutorService executor = ...
   ArchiveSearcher searcher = ...
   void showSearch(final String target) throws InterruptedException {
     Future<String> future = executor.submit(new Callable<String>() {
       public String call() {
           return searcher.search(target);
       }});
     displayOtherThings(); // do other things while searching
     try {
       displayText(future.get()); // use future
     } catch (ExecutionException ex) { cleanup(); return; }
   }
 }
```

* Java Future
	* Java의 `Future` 또한 비동기 계산의 결과를 나타내지만 블로킹 `get` 메소드를 통해서만 결과에 액세스 가능
	* `isDone`를 호출 해서 get를 호출하기 전에 Java `Future`가 완료 되었는지 확인 가능
	* `Future`의 결과를 사용하는 계산은 Java `Future`가 완료 할 때까지 대기

* Scala Future
	* `Future`가 완료되든 아니든 상관없이 변환을 명시 할 수 있다.
	* 각 변환은 함수에 의해 변환 된 원래 `Future`의 비동기 결과를 나타내는 새로운 `Future`를 생성
	* 계산을 수행하는 스레드는 암시 적으로 제공된 `ExecutionContext`에 의해 결정
	* 비동기식 계산을 공유 메모리 및 잠금에 대해 추론 할 필요없이 불변 값의 일련의 변환으로 설명

> ExecutionContext == Threadpool Manager??
> 
> * http://hamait.tistory.com/768
> * https://www.scala-lang.org/api/current/scala/concurrent/ExecutionContext.html

## Trouble in paradise

Java 에서는 Critical Section 을 만들기위해서 `synchronized` 키워드를 사용한다.

* `synchronized` 키워드의 사용
    * synchronized method
	* synchronized block

* 호환성을 위해 Scala는 Java의 동시성 프리미티브에 대한 액세스를 제공
	* Scala에서는 `wait`, `notify` 및 `notifyAll` 메소드를 호출 할 수 있으며 Java와 동일한 의미를 갖는다.
		* `lock`을 잡고있는 스레드를 제어한다.
	* 스칼라에는 기술적으로 `synchronized` 키워드가 없지만 `predefined` 된 `synchronized` 메서드를 가진다.

```java
var counter = 0
synchronized {
	// One thread in here at a time
	counter = counter + 1
}
```

### 공유 데이터 및 잠금을 사용하면서 견고한 멀티 스레드 응용 프로그램을 안정적으로 구축하는 것이 매우 어렵다.

#### 공유 데이터 및 잠금을 사용하면 추론만으로 프로그램을 개발해야 한다.
* 프로그래머는 어떤 데이터를 변경하거나 접근하고자하는 매순간 그 데이터가 다른 스레드에 의해서 변경되거나 접근되어질 수 있는지를 추론해야 한다. 또한, 어떤 잠금이 걸려 있는지도 추론해야 한다.
* 각 메소드 호출시, 잠금을 유지하려고 시도 할 잠금에 대해 추론해야하며 잠금을 얻으려고 시도하는 동안 교착상태가 발생하지 않는다고 스스로 확신해야한다.
* 잠금은 컴파일시에 고정되지 않는다. 런타임에 새로운 잠금이 생성될 수 있다.

#### 테스트 측면에서도 다중 스레드 코드는 안정적이지 않다.
* 스레드가 비 결정적이기 때문에 천 번 성공적으로 테스트를 통과한 프로그램도 다른 시스템, 다른 환경에서 여전히 잘못 될 수 있다.

#### over-synchronizing
* 모든 것을 동기화하는 것은 아무것도 동기화 하지 않는 것 처럼 문제가 될 수 있다.
* 새로운 잠금 작업으로 경쟁 조건이 제거 될 수도 있지만 동시에 교착 상태가 발생할 수 있다.

#### java.util.concurrent 라이브러리
* `java.util.concurrent`에 있는 동시성 유틸리티를 사용하면 Java의 저수준 동기화 프리미티브를 사용하여 자신의 추상화를 롤링하는 것보다 오류가 발생하지 않는 멀티 스레드 프로그래밍이 가능하다.
* 그럼에도 불구하고, `java.util.concurrent` 역시 공유 데이터 및 잠금 모델을 기반으로하므로 그 모델을 사용하는 근본적인 어려움을 해결하지 못한다.


## Asynchronous execution and Trys

스칼라 `Future`는 공유 데이터 및 잠금에 대한 추론없이 동시성 프로그램을 개발할 수 있게 한다.

보통의 스칼라 메서드를 호출하면 "기다리는 동안" 계산을 수행하고 결과를 반환한다. 그러나, 리턴값이 `Future` 인 경우, 메서드는 다른 스레드에서 비동기적으로 수행되고 메서드는 즉시 결과를 반환한다.

Future에 대한 많은 작업에는 비동기 적으로 함수를 실행하기위한 전략을 제공하는 암시적 실행 컨텍스트가 필요하다.
따라서, `scala.concurrent.ExecutionContext` 같은 실행 컨텍스트를 제공하지 않고 Future.apply 팩토리 메서드를 통해 `Future`를 생성하려고 하면 컴파일 에러가 발생한다.

```scala
scala> import scala.concurrent.Future
import scala.concurrent.Future

scala> val fut = Future { Thread.sleep(10000); 21 + 21 }
<console>:11: error: Cannot find an implicit ExecutionContext.
	You might pass an (implicit ec: ExecutionContext) parameter to your method or import 	scala.concurrent.ExecutionContext.Implicits.global.
		val fut = Future { Thread.sleep(10000); 21 + 21 }
```

왜냐하면 `Future.apply()`의 구현이 아래와 같이 `ExecutionContext`를 implicit parameter로 사용하기 때문이다.

```scala
def apply[T](body: =>T)(implicit @deprecatedName('execctx) executor: ExecutionContext): Future[T] = unit.map(_ => body)
```

실행 컨텍스트를 범위내에서 import 해주면 `Future`를 생성할 수 있다.

```scala
scala> import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.ExecutionContext.Implicits.global

scala> val fut = Future { Thread.sleep(10000); 21 + 21 }
fut: scala.concurrent.Future[Int] = Future(<not completed>)
```

### Future.isCompleted & Future.value

아직 완성되지 않은 `Future`에 대해서 호출 될 때 `isCompleted`는 `false`를 반환하고 `value`는 `None`을 반환한다.

```scala
// 아직 10초가 지나지 않았다!!
scala> fut.isCompleted
res0: Boolean = false
scala> fut.value
res1: Option[scala.util.Try[Int]] = None
```

`Future`가 완료되면 `isCompleted`는 `true`를 반환하고 `value`는 `Some[Try[T]]`을 반환합니다.

```scala
// 10초 뒤...
scala> fut.isCompleted
res2: Boolean = true
scala> fut.value
res3: Option[scala.util.Try[Int]] = Some(Success(42))
```

### Try

값으로 반환되는 옵션에는 `Try`가 포함된다.
`Try`는 동기식 계산을 위해 `try`와 같은 의미로 사용된다.

> 동기 계산의 경우 `try/catch`를 사용하여 메소드를 호출하는 스레드가 메소드에 의해 발생 된 예외를 포착하고 처리하도록 할 수 있다. 그러나 비동기식 계산의 경우 계산을 시작하는 스레드는 다른 작업을 계속 진행하게 된다. 나중에 비동기 계산에 예외가 발생하면 원래 스레드는 더 이상 catch 절에서 예외를 처리 할 수 없다. 따라서 `Future`로 작업 할 때 값을 산출하지 못하고 예외를 갑자기 완료 할 가능성을 처리하기 위해서 Try를 사용한다.

```scala
scala> val fut = Future { Thread.sleep(10000); 21 / 0 }
fut: scala.concurrent.Future[Int] = ...

scala> fut.value
res4: Option[scala.util.Try[Int]] = None

// Then, after ten seconds

scala> fut.value
res5: Option[scala.util.Try[Int]] = Some(Failure(java.lang.ArithmeticException: / by zero))
```


## Working with Futures

`Future`는 다양한 메소드들을 제공한다.

### Transforming Futures with map

```scala
def map[S](f: (T) ⇒ S)(implicit executor: ExecutionContext): Future[S]
```

* `map`을 사용하면 blocking 없이 다음 계산을 `Future`에 연속적으로 매핑할 수 있다.
	* 예제에서 `Future`의 생성, `21 + 21` 그리고 `42 + 1`은 각각 다른 thread에 의해서 수행된다.

```scala
scala> val fut = Future { Thread.sleep(10000); 21 + 21 }
fut: scala.concurrent.Future[Int] = ...

scala> val result = fut.map(x => x + 1)
result: scala.concurrent.Future[Int] = ...

scala> result.value
res6: Option[scala.util.Try[Int]] = Some(Success(43))
```


### Transforming Futures with for expressions

```scala
def flatMap[S](f: (T) ⇒ Future[S])(implicit executor: ExecutionContext): Future[S] 
```

* `flatMap` 메서드를 선언하기 때문에 `for-comprehensions`을 사용하여 `Future`를 변형 할 수 있다. (... `for-comprehensions`이 flatMap으로 구현되나..?)

```scala
scala> val fut1 = Future { Thread.sleep(10000); 21 + 21 }
fut1: scala.concurrent.Future[Int] = Future(<not completed>)

scala> val fut2 = Future { Thread.sleep(10000); 23 + 23 }
fut2: scala.concurrent.Future[Int] = Future(<not completed>)

// `for-comprehensions`은 아래 flatMap과 map 조합으로 표현할 수 있다.
// fut1.flatMap(x => fut2.map(y => x + y))
scala> for {
			x <- fut1
			y <- fut2
		} yield x + y
res7: scala.concurrent.Future[Int] = Future(<not completed>)

scala> res7.value
res8: Option[scala.util.Try[Int]] = Some(Success(88))
```

* `for-comprehensions`은 해당 변환을 직렬화하기 때문에 `for-comprehensions` 앞에 `Future`를 작성하지 않으면 병렬로 실행되지 않는다.
	* 예를 들어, 위의 예제는 완료하는데 10초가 걸리지만 아래의 예제는 적어도 20초가 이상 걸린다.

```scala
scala> for {
			x <- Future { Thread.sleep(10000); 21 + 21 }
			y <- Future { Thread.sleep(10000); 23 + 23 }
		} yield x + y
res9: scala.concurrent.Future[Int] = ...

scala> res9.value
res27: Option[scala.util.Try[Int]] = None

// Will need at least 20 seconds to complete

scala> res9.value
res28: Option[scala.util.Try[Int]] = Some(Success(88))

```

### Creating the Future: Future.failed, Future.successful, Future.fromTry, and Promises

```scala
// in companion object
// def apply[T](body: ⇒ T)(implicit executor: ExecutionContext): Future[T]
def failed[T](exception: Throwable): Future[T]
def successful[T](result: T): Future[T]
def fromTry[T](result: Try[T]): Future[T]
```

* `Future` companion object의 펙토리 메소드
	* 이미 완료된 `Future`를 생성한다.
	* 팩토리 메소드는 ExecutionContext를 필요로하지 않는다. (동기적으로 실행된다.)

```scala
scala> Future.successful { 21 + 21 }
res2: scala.concurrent.Future[Int] = Future(Success(42))

scala> Future.failed(new Exception("bummer!"))
res3: scala.concurrent.Future[Nothing] = Future(Failure(java.lang.Exception: bummer!))

scala> import scala.util.{Success,Failure}
import scala.util.{Success, Failure}

scala> Future.fromTry(Success { 21 + 21 })
res4: scala.concurrent.Future[Int] = Future(Success(42))

scala> Future.fromTry(Failure(new Exception("bummer!")))
res5: scala.concurrent.Future[Nothing] = Future(Failure(java.lang.Exception: bummer!))
```

* `Future`를 생성하기 위해서는 일반적으로 `Promise`를 사용한다.

```scala
scala> val pro = Promise[Int]
pro: scala.concurrent.Promise[Int] = Future(<not completed>)

scala> val fut = pro.future
fut: scala.concurrent.Future[Int] = Future(<not completed>)

// 여기까지는 완료되지 않음
scala> fut.value
res8: Option[scala.util.Try[Int]] = None
```

* `success`, `failure`및 `complete` 메서드를 사용하여 `Promise`를 완료 할 수 있다.
	* `Promise`의 이러한 메소드는 이미 완료된 `Future`를 구성하기 위해 앞서 설명한 메소드와 유사하게 동작한다.
	* 예를 들어, success 메서드는 `Future`를 성공적으로 완료한다.

```scala
def success(value: T): Promise.this.type
def failure(cause: Throwable): Promise.this.type
def complete(result: Try[T]): Promise.this.type
```

* `success`, `failure`및 `complete` 예제

```scala
scala> val pro = Promise[Int]
pro: scala.concurrent.Promise[Int] = Future(<not completed>)

scala> val fut = pro.future
fut: scala.concurrent.Future[Int] = Future(<not completed>)

scala> pro.success(42)
res9: pro.type = Future(Success(42))

scala> fut.value
res10: Option[scala.util.Try[Int]] = Some(Success(42))

scala> pro.failure(new Exception("bummer!"))
res138: pro.type = Future(Failure(java.lang.Exception: bummer!))

scala> pro.completeWith(Future(12))
res152: pro.type = Future(<not completed>)
```


### Filtering: filter and collect

```scala
def filter(p: (T) ⇒ Boolean)(implicit executor: ExecutionContext): Future[T]
def collect[S](pf: PartialFunction[T, S])(implicit executor: ExecutionContext): Future[S]
final def withFilter(p: (T) ⇒ Boolean)(implicit executor: ExecutionContext): Future[T]
```

* `filter` 예제

```scala
scala> val fut = Future { 42 }
fut: scala.concurrent.Future[Int] = Future(<not completed>)

scala> val valid = fut.filter(res => res > 0)
valid: scala.concurrent.Future[Int] = List

scala> valid.value
res0: Option[scala.util.Try[Int]] = Some(Success(42))

scala> val invalid = fut.filter(res => res < 0)
invalid: scala.concurrent.Future[Int] = List()

scala> invalid.value
res0: Option[scala.util.Try[Int]] = Some(Failure(java.util.NoSuchElementException: Future.filter predicate is not satisfied))
```

* `withFilter`는 `for-comprehensions`에 의해서 사용된다.

```scala
scala> val valid = for (res <- fut if res > 0) yield res
valid: scala.concurrent.Future[Int] = ...

scala> valid.value
res2: Option[scala.util.Try[Int]] = Some(Success(42))

scala> val invalid = for (res <- fut if res < 0) yield res
invalid: scala.concurrent.Future[Int] = ...

scala> invalid.value
res3: Option[scala.util.Try[Int]] = Some(Failure(java.util.NoSuchElementException: Future.filter predicate is not satisfied))
```

* `collect` 메소드는 부분함수를 인자로 받는다.

```scala
val f = Future { -5 }
val g = f collect {
  case x if x < 0 => -x
}
val h = f collect {
  case x if x > 0 => x * 2
}
g foreach println // Eventually prints 5
Await.result(h, Duration.Zero) // throw a NoSuchElementException
```


### Dealing with failure: failed, fallBackTo, recover, and recoverWith

```scala
def failed: Future[Throwable]
def fallbackTo[U >: T](that: Future[U]): Future[U]
def recover[U >: T](pf: PartialFunction[Throwable, U])(implicit executor: ExecutionContext): Future[U]
def recoverWith[U >: T](pf: PartialFunction[Throwable, Future[U]])(implicit executor: ExecutionContext): Future[U] 
```

scala `Future`는 실패를 다루는 방법을 제공한다.

* `failed` 메소드는 `Failure[Throwable]`을 `Success[Throwable]`로 바꾼다. (..... 왜..?)
	* original `Future`가 성공하면 `NoSuchElementException`을 발생시킨다.

```scala
scala> val failure = Future { 42 / 0 }
failure: scala.concurrent.Future[Int] = Future(Failure(java.lang.ArithmeticException: / by zero))

scala> failure.value
res23: Option[scala.util.Try[Int]] = Some(Failure(java.lang.ArithmeticException: / by zero))

scala> val expectedFailure = failure.failed
expectedFailure: scala.concurrent.Future[Throwable] = Future(Success(java.lang.ArithmeticException: / by zero))

scala> expectedFailure.value
res25: Option[scala.util.Try[Throwable]] = Some(Success(java.lang.ArithmeticException: / by zero))
```

* `fallbackTo` 예제
	* `Future`가 실패할 경우 대신할 `Future`를 제공한다.
	* 만약 둘다 실패하면 앞전 `Future`의 실패를 리턴한다.

```scala
val f = Future { sys.error("failed") }
val g = Future { 5 }
val h = f fallbackTo g
h foreach println // Eventually prints 5
```

* `recover` 예제

```scala
Future (6 / 0) recover { case e: ArithmeticException => 0 } // result: 0
Future (6 / 0) recover { case e: NotFoundException   => 0 } // result: exception
Future (6 / 2) recover { case e: ArithmeticException => 0 } // result: 3
```

* `recoverWith` 예제

```scala
val f = Future { Int.MaxValue }
Future (6 / 0) recoverWith { case e: ArithmeticException => f } // result: Int.MaxValue
```


### Mapping both possibilities: transform

```scala
def transform[S](s: (T) ⇒ S, f: (Throwable) ⇒ Throwable)(implicit executor: ExecutionContext): Future[S] 
```

* `transform` 예제
	* `Future`의 결과가 성공이면 `s` 함수를 적용하고, 실패이면 f 함수를 적용하여 새로운 `Future`를 만든다.

```scala
scala> valid
scala.concurrent.Future[Int] = Success(88)

scala> valid.transform(x=>x+1, Exception=> throw new Exception)
res60: scala.concurrent.Future[Int] = List()

scala> res60.value
res61: Option[scala.util.Try[Int]] = Some(Success(89))
```


### Combining futures: zip, Future.fold, Future.reduce, Future.sequence, and Future.traverse

```scala
def zip[U](that: Future[U]): Future[(T, U)]
def foldLeft[T, R](futures: collection.immutable.Iterable[Future[T]])(zero: R)(op: (R, T) ⇒ R)(implicit executor: ExecutionContext): Future[R]
def reduceLeft[T, R >: T](futures: collection.immutable.Iterable[Future[T]])(op: (R, T) ⇒ R)(implicit executor: ExecutionContext): Future[R]
def sequence[A, M[X] <: TraversableOnce[X]](in: M[Future[A]])(implicit cbf: CanBuildFrom[M[Future[A]], A, M[A]], executor: ExecutionContext): Future[M[A]]
def traverse[A, B, M[X] <: TraversableOnce[X]](in: M[A])(fn: (A) ⇒ Future[B])(implicit cbf: CanBuildFrom[M[A], B, M[B]], executor: ExecutionContext): Future[M[B]] 
```

### Performing side-effects: foreach, onComplete, and andThen

```scala
def foreach[U](f: (T) ⇒ U)(implicit executor: ExecutionContext): Unit 
abstract def onComplete[U](f: (Try[T]) ⇒ U)(implicit executor: ExecutionContext): Unit
def andThen[U](pf: PartialFunction[Try[T], U])(implicit executor: ExecutionContext): Future[T] 
```

`callback` 성 함수들..  

* `onSuccess`, `onFailure` 는 Deprecated 됨

* `andThen` 예제
	* 말그대로 and then ....

```scala
val f = Future { 5 }
f andThen {
  case r => sys.error("runtime exception")
} andThen {
  case Failure(t) => println(t)
  case Success(v) => println(v)
}
```

### Other methods added in 2.12: flatten, zipWith, and transformWith

다하면 아쉬우니까 숙제로..... ^^....



## Testing with Futures


### Await

`Future`의 장점은 블로킹을 피할 수 있다는 것이지만, 블로킹을 이용해서 `Future`의 결과를 수집해야 할 때가 있다.

```scala
scala> import scala.concurrent.Await
import scala.concurrent.Await

scala> import scala.concurrent.duration._
import scala.concurrent.duration._

scala> val fut = Future { Thread.sleep(10000); 21 + 21 }
fut: scala.concurrent.Future[Int] = ...

scala> val x = Await.result(fut, 15.seconds) // blocks
x: Int = 42
```

일반적으로 비동기 테스트 코드에서 블로킹이 사용된다.
* `Await.result`가 반환되면 `assert`를 이용하여 테스트를 구현할 수 있다.

```scala
scala> import org.scalatest.Matchers._
import org.scalatest.Matchers._

scala> x should be (42)
res0: org.scalatest.Assertion = Succeeded
```

`futureValue` 메소드는 `ScalaFutures`에 의해서 `Future`에 암시적으로 추가된다.
* `futureValue` 메소드는 `Future`가 완료될 때까지 block 된다.
* `Future`가 실패하면 `TestFailedException`이 던져진다.

```scala
scala> import org.scalatest.concurrent.ScalaFutures._
import org.scalatest.concurrent.ScalaFutures._

scala> val fut = Future { Thread.sleep(10000); 21 + 21 }
fut: scala.concurrent.Future[Int] = ...

scala> fut.futureValue should be (42) // futureValue blocks
res1: org.scalatest.Assertion = Succeeded
```

### 비동기 테스트

* 스칼라 테스트 3.0에서 추가됨

`Future` 안에서 실행되는 `assertion`이 완료되면 `ScalaTest`가 테스트 리포터에게 비동기방식으로 테스트의 성공 여부를 알려준다.

```scala
import org.scalatest.AsyncFunSpec
import scala.concurrent.Future

class AddSpec extends AsyncFunSpec {

	def addSoon(addends: Int * ): Future[Int] = Future { addends.sum }

	describe("addSoon") {
		it("will eventually compute a sum of passed Ints") {
			val futureSum: Future[Int] = addSoon(1, 2)
			// You can map assertions onto a Future, then return
			// the resulting Future[Assertion] to ScalaTest:
			futureSum map { sum => assert(sum == 3) }
		}
	}
}
```


## Conclusion

`Future`는 `deadlock`과 `race condidion`이 없는 동시성 프로그래밍 환경을 제공하는 훌륭한 도구이다.

* 하지만, 이것도 잘 쓰기는 어렵다......... ^^..

# 끝!


