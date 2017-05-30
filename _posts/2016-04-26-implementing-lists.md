---
layout: default
title: 리스트 구현 in Scala
category: post
author: Jae
---

> Programming in Scala by Martin Odersky 를 개인적인 공부 목적으로 정리한 포스트 입니다.

# 리스트 구현

내부를 알면 리스트 연산의 상대적인 효율성을 더 잘 이해할 수 있다.

## 리스트 클래스 개괄
* 스칼라에서는 리스트가 내장 언어 구성요소가 아니다.

```scala
package scala
abstract class List[+T] {

	final case class ::
	case object Nil
}
```

* 리스트는 추상 클래스다
	* 생성자를 호출해 리스트의 원소를 정의할 수 없다.
* 공변적이다.
	* List[Int] 타입의 값을 List[Any]에 할당할 수 있다.
* 모든 리스트 연산은 세가지 기본 메소드로 만들 수 있다.

```scala
def isEmpty: Boolean
def head: T
def tail: List[T]
```
* 이 세 메소드는 모두 List 클래스의 추상 메소드다.
* "서브객체"인 `Nil`과 "서브클래스"인 `::`에서 이 들을 정의한다

### Nil 객체
* 빈 리스트를 정의 한다.

```scala
# 싱글톤 객체 Nil의 정의
case object Nil extends List[Nothing] {
	override def isEmpty = true
	override de def head : Nothing =
		throw new NoSuchElementException( "head of empty list " )
	override def tail : List[Nothing] = 
		throw new NoSuchElementException( " [ail of empty list " )
}
```

* 공변성을 감안하면 `Nil`은 모든 `List`타입과 서로 호환
* 반환값이 `Nothing`인데 `Nothing type`을 갖는 값은 없기 때문에 Exception을 던지는것 말고는 head를 정의 할 수 없다.

### :: 클래스
* 콘즈라고 부르며 construct의 약자이다.
* 비어있지 않은 리스트를 만든다.
* 이름이 `::`인 이유
	* 중위 표기 `::`와 패턴 매치를 하기 위해서
	* x :: xs == ::(x, xs)

```scala
# 모든 케이스 클래스의 파라미터는 암시적으로 해당 클래스의 필드이다.
final case class :: [T] (head : T, tail : List[T] ) extends List[T] {
	override def isEmpty : Boolean = false
}
```


### 추가 메소드
* 다른 모든 리스트 메소드는 기본적인 메소드를 바탕으로 정의할 수 있다.

```scala
def length : Int = if (isEmpty) 0 else 1 + tail.length

def drop(n : Int) : List[T] = {
	if (isEmpty) Nil 
	else if (n <= 0) this 
	else tail.drop(n - 1)
}

def map[U] (f : T => U) : List[U] = {
	if (isEmpty) Nil 
	else f(head) :: tail.map(f)
}
```

### 리스트 구성 메소드 `::`, `:::`
* `::` 메소드의 구현에 `::`클래스가 사용된다.
* 콜론(`:`)으로 끝나기 때문에， 이 들은 오른쪽 피연산자에 바인딩된다 즉 x :: xs는 x.::(xs) 가 이니고 xs.::(x) 호출
* x는 리스트의 원소 타입이므로 임의의 타입이 될 수 있고， 그런 타입에는 `::`메소드가 있다는 보장이 없다

> 그렇다면 x의 타입은 꼭 xs의 원소 타입어야 하는가 ?

#### 리스트 원소의 타입

```scala
abstract class Fruit 
class Apple extends Fruit 
class Orange extends Fruit

// Nil이 apples를 List로 만든다.
scala> val apples = new Apple :: Nil 
apples : List[Apple] = List(Apple@585fa9)

// 결과 리스트의 원소 타입이 원래 리스트의 원소 타입과 추가할 원소의 타입의 슈퍼타입
scala> val fruits = new Orange :: apples 
fruits : List[Fruit] = List(Orange@cd6798 , Apple@585fa9)
```

![fruitList](/images/posts/programming-in-scala/fruitList.png)

* 위와 같은 유연성이 가능한건 `::`의 구현이 아래와 같기 때문이다.

```scala
def ::[U >: T](x: U): List[U] = new scala.::(x, this)
```

* `::`는 타입 파라미터 `U`를 받는 다형성 메소드
	* `U`는 리스트의 원소 타입인 `T`의 슈퍼 타입이어야 한다
	* List 클래스가 공변적이기 때문에 List 클래스의 정의의 타입에 맞추기 위함
	* 메소드 파라미터는 반공변적인 위치에 있다.

	

## ListBuffer 클래스
* 리스트에 대한 전형적인 접근 패턴은 재귀적이다.
	* 재귀 호출의 결과로 나오는 리스트 앞에 새 원소를 붙이는 방식으로 리스트를 만들어 나감

```scala
// List의 모든 원소의 값을 1씩 증가시키는 함수
def incAll(xs: List[Int]): List[Int] = xs match {
	case List() => List()
	case x :: xs1 => x + 1 :: incAll(xs1)
}
```

* 꼬리재귀가 아니기 때문에 각 재귀 호출마다 새로운 스택 프레임이 필요하다.
	* 크기가 큰 리스트도 처리 할 수 있는 `incAll`을 만들어 보자

> 꼬리재귀 함수
> 
> * 반환값으로 자기 자신만을 반환하는 함수
> * 결국 컴파일러에 의해 for문으로 치환된다. 

```scala
var result = List[Int]()
for (x <- xs) result = result ::: List(x + 1)
result
```

* `:::`는 처리에 첫인자(추가될 리스트)의 길이에 비례하는 시간이 걸린다.
	* 비효율 적이므로 사용하기 싫다.

```scala
import scala.collection.mutable.ListBuffer

val buf = new ListBuffer[Int] 
for (x <- xs) buf += x + 1 
buf.toList
```

* 리스트 버퍼는 추가 연산(`+=`)과 `toList`연산을 상수 시간에 처리한다.

## 실제 List 클래스
* 클래스 List를 실제로 구현하는 경우에는 대부분 재귀를 피하고, 리스트 버퍼에 루프를 수행하는 방식을 택한다.

```scala
final override def map[U](f: T => U): List[U] = {
	val b = new ListBuffer[U]
	var these = this
	while (!these.isEmpty) {
		b += f(these.head)
		these = these.tail
	}
	b.toList
}
```

* toList는 아주 적은 CPU 사이클 만에 끝나며, 리스트의 길이와는 무관한 시간이 걸린다.
	* 왜 그런지 이해하기 위해서  `::` 클래스의 실제 구현을 살펴 본다.

#### `::` 클래스의 실제 구현

```scala
// private[scala]: scala 패키지 안에서만 접근 가능, 이 패키지 밖의 클라이언트 코드는  tl을 읽거나 쓸 수 없다.
final case class ::[U](hd: U, private[scala] var tl: List[U] extends List[U] {
	def head = hd
	def tail = tl
	override def isEmpty: Boolean = false
}
```

* `tl`이 `var`로 선언되어 있다.
	* `::`에 의해 성성된 리스트는 꼬리가 변경 가능하다.
	* ListBuffer는 scala 패키지의 하위 패키지인 `scala.collection.mutable`에 들어 있기 때문에 `::`의 `tl`에 접근 가능하다.

* 리스트 버퍼의 원소는 리스트로 되어 있고 새 원소를 버퍼에 추가하는 것은 해당 리스트의 마지막 셀의 `tl`필드를 변경하는 것으로 구현해 놓았다.

```scala
package scala.collection.immutable
final class ListBuffer[T] extends Buffer[T] {
	private var start: List[T] = Nil // 버퍼에 저장된 모든 요소의 리스트
	private var last0: ::[T] = _ // 리스트의 마지막 :: 셀?
	private var exported: Boolean = false // toList를 사용해 버퍼를 리스트로 바꾼적이 있는지 표시
	...
	
	override def toList: List[T] = {
		// ListBuffer에 저장된 리스트를 복사하지 않기 때문에 효율적
		exported = !start.isEmpty
		start
	}
```

* toList가 반환한 리스트는 변경 불가능
* `last0`원소에 다른 값을 추가하면 `start`가 가리키는 리스트가 변한다.

```scala
override def += (x: T) = {
	// copy에서는 start, last0, exported를 clear하고 기존에 있던 원소들을 다시 복사해 넣는다. 왜?
	if (exported) copy()
	if (start.isEmpty) {
		last0 = new scala.::(x, Nil)
		start = last0
	} else {
		val last1 = last0
		last0 = new scala.::(x, Nil)
		// tl은  var!!
		// start가 증가하게 된다.
		last1.tl = last0
	}
}
```

* `exported`가 `true`라면 `+=` 안에서 `start`가 가리키는 리스트를 복사한다.
* 결국, 변경 불가능한 리스트의 끝을 확장하고 싶다면, 복사를 해야 한다.
* 따라서 리스트 버퍼는 원소를 점진적으로 추가한 다음 마지막에 toList를 수행하는 로직에 적절하다.

## 외부에서 볼 때는 함수형
* `List`는 외부에서는 완전히 함수적이지만 내부에서는 `ListBuffer`를 사용해 명령형으로 되어 있음
	* 순수하지 않은 연산의 효과가 미치는 범위를 제한하여 함수적 순수성을 효율적으로 달성
	* 스칼라는 공유(얕은복사)를 곳곳에 사용하는 대신 리스트를 변경하지 못하게 막는 쪽(val)으로 발전

	
## 결론
* 리스트의 많은 핵심 메소드는 재귀보다는 ListBuffer를 사용한다.
