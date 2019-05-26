---
layout: default
title: Equality in Scala
category: post
author: Jae
comments: true
tags: [scala, PIS]
---

> Programming in Scala by Martin Odersky 를 개인적인 공부 목적으로 정리한 포스트 입니다.

# Equality in Scala

* `==`, `!=`, `equals` : Natural equality
* `eq`, `ne` : Reference equality

`==`는 override할 수 없다:

```scala
final def == (that: Any): Boolean =
  if (null eq this) {null eq that} else {this equals that}
```

## equals 구현 시 일관성이 없는 동작을 야기할 수 있는 네가지 일반적인 함정

### 함정 #1: equals 선언 시 잘못된 시그니처를 사용하는 경우
* 아래와 같이 point 클래스가 있을 때

```scala
class Point(val x: Int, val y: Int) {
  // An utterly wrong definition of equals
  def equals(other: Point): Boolean = this.x == other.x && this.y == other.y
}
```

* 단순히 비교하면 문제가 없는 것 처럼 보이지만, Collection에 넣으면 문제가 생긴다.

```scala
scala> val p1, p2 = new Point(1,2)
p1: Point = Point@62cf484e
p2: Point = Point@16942b90

scala> val q = new Point(2,3)
q: Point = Point@26ca4ab3

scala> p1 equals p2
res0: Boolean = true

scala> p1 equals q
res0: Boolean = false

scala> import scala.collection.mutable.HashSet
scala> val coll = HashSet(p1)
scaal> coll contains p2
res0: Boolean = false
```

* Point 클래스의 `equals` 메소드는 `Any.equals`를 override 한 게 아니라 overload 한 것이다.
	* Parameter가 Any가 아닌므로..
	* 그래서 `Any.equals`의 결과와 `Point.equals`의 결과가 다르다.
	* Collections에서는 `Any.equals`을 사용할 것이기 때문에 모두 오작동한다.

* override할 거면 아래처럼 해야 한다:

```scala
// A better definition, but still not perfect
override def equals(other: Any) = other match {
  case that: Point => this.x == that.x && this.y == that.y
  case _ => false
}
```

#### 추가
* `==`를 잘못된 시그니처로 정의하는 경우도 마찬가지다.
	* 다음과 같이 `def ==(other: Point):Boolean` 함수를 정의한 경우 Any에 있는 같은 이름의 메소드를 overload한 것으로 취급된다.
	* Any의 `==`는 `final def ==(other: Any):Boolean`처럼 final이라 override할 수 없다.


### 함정 #2: equals를 변경하면서 hashCode는 그대로 놔둔 경우

* Hash Bucket 구조로 된 Collection은 `equals`과 `hashCode`는 함께 바꿔야 한다:
	* HashSet은 원소들을 Hash Bucket 구조로 관리 한다.
	* 아래 처럼 HashCode가 다르면 Hash Bucket 구조의 Collection에서는 찾을 수 없다.

```scala
scala> p1.hashCode
res4: Int = 61816062

scala> p2.hashCode
res5: Int = 1707952533

scala> q.hashCode
res6: Int = 650791603

```


* hashCode() 구현시 equals가 사용하는 필드만을 사용해서 Hash를 결정하면 된다.
	* 41이라는 소수를 더하고 곱한 것은 Distribution을 좋게 하려고 하는 건데, 자세한 건 밑에서 설명한다고(실제로도 설명하지만 나는 이해가 안된다ㅜㅜ)

```scala
class Point(val x: Int, val y: Int) {
  override def hashCode = 41 * (41 + x) + y
  override def equals(other: Any) = other match {
    case that: Point => this.x == that.x && this.y == that.y
    case _ => false
  }
}
```


### 함정 #3: equals를 변경 가능한 필드의 값을 기주으로 정의한 경우

* 아래처럼 var로 선언한 후 a.x의 값을 고치면 coll 집합에서 p의 버킷을 잘못 계산한다.

```scala
val coll = HashSet[Point]()
val p = new Point(1,2)
hs += p

println( coll.contains(a) ) // true
p.x = 2
println( coll.contains(a) ) // false
```

### 함정 #4: equals를 동치 관계로 정의하지 않는 경우

* scala.Any의 equals 메소드의 계약은 null이 아닌 객체에 대해 equals가 동치 관계여야 한다고 명시
	* 반사성(reflexive): null이 아닌 값 x에 대해 x.equals(x)는 true를 반환해야 한다.
	* 대칭성(symmetric): null이 아닌 값 x,y에 대해 x.equals(y)가 true를 반환한다면, y.equals(x)도 true를 반환해야 한다.
	* 추이성(transitive): null이 아닌 값 x,y,z에 대해 x.equals(y)가 true이고, y.equals(z)가 true이면, x.equals(z)도 true를 반환해야 한다.
	* 일관성(consistent): null이 아닌 값 x,y에 대해 x.equals(y)를 여러 번 호출해도 x나 y객체에 있는 정보가 변경되지 않는 이상 계속 true나 false중 한 값을 일관되게 반환해야 한다.
	* null이 아닌 값 x에 대해 x.equals(null)은 false를 반환해야 한다.

#### 서브클래스를 고려해야 하는 경우 문제가 복잡해진다..
* Point의 서브클래스로 Color타입의 필드 color를 추가한 ColoredPoint가 있다고 하자..
	* 새로운 equals는 원래의 정의에 제약을 더 가한 것이기 때문에 hashCode의 계약이 여전히 성립한다.
	* 만약 색이 있는 두 점이 같다면 그 두 점은 좌표도 같아야 한다.
```scala
object Color extends Enumeration {
	val red, green, blue = Value
}
class ColoredPoint(x:Int, y:Int, color:Color.Value) extends Point(x,y) {
	override def equals(other: Any) = other match {
		case that: ColoredPoint => 
			this.color == that.color && super.equals(that)
		case _ => false
	}
}
```

* 점과 색이 있는 점을 섞어서 사용하는 순간 equals의 계약이 망가진다.
	* p는 ColoredPoint가 아니기 때문에 equals 정의는 대칭이 아니게 된다.
	* `cp equals p`는 `_ => flase`를 타게 된다.

```scala
scala> val p = new Point(1,2)
scala> val cp = new ColoredPoint(1,2,Color.red)
scala> p equals cp
res0: Boolean = true
scala> cp equals p
res0: Boolean = false
```

##### equals의 정의를 어떻게 바꾸면 대칭이 될까?: 관계를 엄격하게 만들거나 느슨하게 만들기
* 관계를 느슨하게 만들기
	* 두 객체 x,y의 동일성을 비교할 때 x에 y를 비교하거나, y에 x를 비교해 둘중 하나 이상이 true이면 성리가헥 하는 것이다.
	* `case that: Point => that equals this`를 추가 한다.
	* 이렇게 하면 원하는 대로 equals를 대칭적으로 만들 수 있지만........ 추이성이 깨지고 만다.
		* 아래 추이성이 깨졌음을 보여주는 예제
	* 관계를 느슨하게 만드는것은 답이 아닌것 같다...

```scala
scala> val redp = new ColoredPoint(1,2,Color.red)
scala> val bulep = new ColoredPoint(1,2,Color.bule)

scala> redp == p
res0: Boolean = ture
scala> p == bulep
res0: Boolean = ture
scala> redp == bulep
res0: Boolean = false
```

* 관계를 엄격하게 만들기
	* 클래스가 다른 객체는 아예 서로 다른 것으로 간주하기
	* 실행 시점 클래스가 같은지 `.getClass`를 양 객체에 호출해서 반환되는 값을 비교
	* 색이 있는 점이 그냥 단순한 점과 같아지는 일은 결코 없다.
	* Point를 상속한 이름없는 클래스에 대해서도 equals를 사용 할 수 없다.
		* pAnon은 좌표 (1,2)를 가리키는 그냥 단순한 점이기 때문에 관계를 엄격하게 만드는것도 해답이 아닌것 같다..
```scala
scala> val pAnon = new Point(1,1) { override val y = 2}
```

#####equals의 계약을 지키면서 여러 단계의 클래스 계층구조에 대해 동일성을 재정의할 방법: canEquals()

```scala
// cnaEquals를 호출하는 슈퍼클래스의 equals 메소드
class Point(val x: Int, val y: Int) {
  override def hashCode = 41 * (41 + x) + y
  override def equals(other: Any) = other match {
    case that: Point => 
		(that canEquals this) &&
		(this.x == that.x) && 
		(this.y == that.y)
    case _ => false
  }
  def canEquals(other:Any) = other.isInstanceOf[Point]
}

// cnaEquals를 호출하는 서브클래스의 equals 메소드
class ColoredPoint(val x: Int, val y: Int, color: Color.Value) extneds Point(x,y) {
  override def hashCode = 41 * super.hasCode + color.hashCode
  override def equals(other: Any) = other match {
    case that: ColoredPoint => 
		(that canEquals this) &&
		super.equals(that) && this.color == that.color
    case _ => false
  }
  override def canEquals(other:Any) = other.isInstanceOf[ColoredPoint]
}
```

* 클래스가 equals를 재정의하자마자, 이 클래스에 속하는 객체들은 더이상 다른 동일성 메소드 정의가 있는 다른 슈퍼클래스의 객체와 같지 않다는 사실을 명시해야한다.
* 슈퍼클래스의 equals 구현이 cnaEquals를 호출한다면 서브클래스를 만든 프로그래머가 해당 서브클래스의 인스턴스가 슈퍼클래스의 인스턴스와 같을 수 있는지 여부를 결정할 수 있음을 보여준다.
	* ColoredPoint는 canEquals를 오버라이드한다, 따라서 색이 있는 점은 켤로 전형적인 기존의 점과 같을 수 없다.
	* 하지만 pAnon이 참조하는 이름없는 서브클래스는 canEquals를 오버라이드 하지 않기때문에 그 인스턴스는 Point의 인스턴스와 같을 수 있다.

```scala
scala> val p = new Point(1,2)
scala> val cp = new ColoredPoint(1,2,Color.blue)
scala> val pAnon = new Point(1,1) {override val y = 2}
scala> val coll = List(p)
scala> coll contains p
res0: Boolean = true
scala> coll contains cp
res0: Boolean = false
scala> coll contains pAnon
res0: Boolean = true
```


## 파라미터화한 타입의 동일성 정의

아래와 같은 Tree를 만든다고 하면:

```scala
trait Tree[+T] {
  def elem: T
  def left: Tree[T]
  def right: Tree[T]
}

class Branch[T](
  val elem: T,
  val left: Tree[T],
  val right: Tree[T]) extends Tree[T] {

  ...

}
```

`equals`은 모든 엘리먼트가 포함되게 한다:

```scala
override def equals(other: Any) = other match {
  case that: Branch[_] => (that canEqual this) &&
    this.elem == that.elem &&
    this.left == that.left &&
    this.right == that.right
  case _ => false
}
```

Runtime에 엘리먼트 타입을 알 수 없기 때문에(Type Erasure) Pattern Matching에서 `Branch[T]`같은 엘리먼트 타입은 의미가 없다. Scala 컴파일러가 보여주는 unchecked Warning은 `Branch[_]`로 하면 사라진다.

`canEqual`은 아래처럼 만든다:

```scala
def canEqual(other: Any) = other.isInstanceOf[Branch[_]]
```

`canEqual`을 하는 이유는 타입도 비교해야 하기 때문이다. 아래와 같은 예를 보면 이해가 쉽다:

```scala
class Cranch[T](elem: T) extends Branch[T](elem, EmptyTree, EmptyTree){}
class Dranch[T](elem: T) extends Branch[T](elem, EmptyTree, EmptyTree){}

scala> val b1 = new Cranch(1)
b1: Cranch[Int] = Cranch@22c898d4

scala> val b2 = new Dranch(1)
b2: Dranch[Int] = Dranch@22c898d4

scala> b1 == b2
res0: Boolean = true
```

b1과 b2는 타입이 다른데 true라고 나온다. 아래 와 같이 `canEqual`을 다시 구현하면 된다:

```scala
class Cranch[T](elem: T) extends Branch[T](elem, EmptyTree, EmptyTree){
  override def canEqual(other: Any) = other.isInstanceOf[Cranch[_]]
}
class Dranch[T](elem: T) extends Branch[T](elem, EmptyTree, EmptyTree){
  override def canEqual(other: Any) = other.isInstanceOf[Dranch[_]]
}
```

이쯤 되면 왜 하필 `Branch[_]`로 표기하는지 궁금하다. Type Erasure로 어차피 타입이 사라지고 `Branch[T]`, `Branch`도 있는데 `Branch[_]`를 사용한다. `Branch[_]`는 Existential Type이고, 간단히 말해서 엘리먼트 타입을 모를 때 `_`로 표기한다는 것…. 같은데…. (응?!)

책에 나오는 Existenti### Equality in Scala

* `==`, `!=`, `equals` : Natural equality
* `eq`, `ne` : Reference equality

`==`는 override할 수 없다:

```scala
final def == (that: Any): Boolean =
  if (null eq this) {null eq that} else {this equals that}
```

### Writing an equality method

#### Pitfall #1: Defining equals with the wrong signature.

```scala
class Point(val x: Int, val y: Int) {
  // An utterly wrong definition of equals
  def equals(other: Point): Boolean = this.x == other.x && this.y == other.y
}
```

이 `equals` 메소드는 `Any.equals`를 override 한 게 아니라 overload되록한 것이다. 그래서 `Any.equals`의 결과와 `Point.equals`의 결과가 다르다. Collections에서는 `Any.equals`을 사용할 것이기 때문에 모두 오작동한다.

override할 거면 아래처럼 해야 한다:

```scala
// A better definition, but still not perfect
override def equals(other: Any) = other match {
  case that: Point => this.x == that.x && this.y == that.y
  case _ => false
}
```

`def ==(other: Point):Boolean` 같은 메소드를 만들어도 overload된다. 절대로 이렇게 하지 말아야 하고 `final def ==(other: Any):Boolean`은 final이라 override할 수도 없다.

#### Pitfall #2: Changing equals without also changing hashCode

Hash Bucket 구조로 된 Collection은 `equals`과 `hashCode`는 함께 바꿔야 한다:

```scala
class Point(val x: Int, val y: Int) {
  override def hashCode = 41 * (41 + x) + y
  override def equals(other: Any) = other match {
    case that: Point => this.x == that.x && this.y == that.y
    case _ => false
  }
}
```

41이라는 소수를 더하고 곱한 것은 Distribution을 좋게 하려고 하는 건데, 자세한 건 밑에서 설명한다고(실제로도 설명하지만 나는 이해가 안된다ㅜㅜ)

#### Pitfall #3: Defining equals in terms of mutable fields

아래처럼 var로 만들지 말 것:

```scala
case class A(var x: Int)

val hs = HashSet[A]()
val a = A(1)
hs += a

println( hs.contains(a) ) // true
a.x = 2
println( hs.contains(a) ) // false
```

#### Pitfall #4: Failing to define equals as an equivalence relation

'not null' 객체에 대해서 Equivalence Relation이 돼야 한다.:

* It is reflexive: for any non-null value x , the expression x.equals(x) should return true.
* It is symmetric: for any non-null values x and y , x.equals(y) should return true if and only if y.equals(x) returns true.
* It is transitive: for any non-null values x , y , and z , if x.equals(y) returns true and y.equals(z) returns true , then x.equals(z) should return true.
* It is consistent: for any non-null values x and y , multiple invocations of x.equals(y) should consistently return true or consistently return false , provided no information used in equals comparisons on
the objects is modified.
* For any non-null value x , x.equals(null) should return false.

이 원칙이 지켜지는 것으로 구현한다. 그 외는 생략….

### Defining equality for parameterize types

아래와 같은 Tree를 만든다고 하면:

```scala
trait Tree[+T] {
  def elem: T
  def left: Tree[T]
  def right: Tree[T]
}

class Branch[T](
  val elem: T,
  val left: Tree[T],
  val right: Tree[T]) extends Tree[T] {

  ...

}
```

`equals`은 모든 엘리먼트가 포함되게 한다:

```scala
override def equals(other: Any) = other match {
  case that: Branch[_] => (that canEqual this) &&
    this.elem == that.elem &&
    this.left == that.left &&
    this.right == that.right
  case _ => false
}
```

Runtime에 엘리먼트 타입을 알 수 없기 때문에(Type Erasure) Pattern Matching에서 `Branch[T]`같은 엘리먼트 타입은 의미가 없다. Scala 컴파일러가 보여주는 unchecked Warning은 `Branch[_]`로 하면 사라진다.

`canEqual`은 아래처럼 만든다:

```scala
def canEqual(other: Any) = other.isInstanceOf[Branch[_]]
```

`canEqual`을 하는 이유는 타입도 비교해야 하기 때문이다. 아래와 같은 예를 보면 이해가 쉽다:

```scala
class Cranch[T](elem: T) extends Branch[T](elem, EmptyTree, EmptyTree){}
class Dranch[T](elem: T) extends Branch[T](elem, EmptyTree, EmptyTree){}

scala> val b1 = new Cranch(1)
b1: Cranch[Int] = Cranch@22c898d4

scala> val b2 = new Dranch(1)
b2: Dranch[Int] = Dranch@22c898d4

scala> b1 == b2
res0: Boolean = true
```

b1과 b2는 타입이 다른데 true라고 나온다. 아래 와 같이 `canEqual`을 다시 구현하면 된다:

```scala
class Cranch[T](elem: T) extends Branch[T](elem, EmptyTree, EmptyTree){
  override def canEqual(other: Any) = other.isInstanceOf[Cranch[_]]
}
class Dranch[T](elem: T) extends Branch[T](elem, EmptyTree, EmptyTree){
  override def canEqual(other: Any) = other.isInstanceOf[Dranch[_]]
}
```

이쯤 되면 왜 하필 `Branch[_]`로 표기하는지 궁금하다. Type Erasure로 어차피 타입이 사라지고 `Branch[T]`, `Branch`도 있는데 `Branch[_]`를 사용한다. `Branch[_]`는 Existential Type이고, 간단히 말해서 엘리먼트 타입을 모를 때 `_`로 표기한다는 것…. 같은데…. (응?!)

책에 나오는 Existential Type의 정의:

> An existential type includes references to type variables that are unknown. For example, Array[T] forSome { type T } is an existential type. It is an array of T , where T is some completely unknown type. All that is assumed about T is that it exists at all. This assumption is weak, but it means at least that an Array[T] forSome { type T } is indeed an array and not a banana.

`hashCode`는 소수를 곱하고 더해서 만든다. 빠진 필드 없이 곱하고 더한다. Overflow시 정보 손실을 적게 하려고 소수(odd prime number)를 이용한다는데 이것도 모르겠다:

```scala
override def hashCode: Int =
  41 * (
    41 * (
      41 + elem.hashCode
    ) + left.hashCode
  ) + right.hashCode
```

### Recipes for equals and hasCode

### Conclusion
al Type의 정의:

> An existential type includes references to type variables that are unknown. For example, Array[T] forSome { type T } is an existential type. It is an array of T , where T is some completely unknown type. All that is assumed about T is that it exists at all. This assumption is weak, but it means at least that an Array[T] forSome { type T } is indeed an array and not a banana.

`hashCode`는 소수를 곱하고 더해서 만든다. 빠진 필드 없이 곱하고 더한다. Overflow시 정보 손실을 적게 하려고 소수(odd prime number)를 이용한다는데 이것도 모르겠다:

```scala
override def hashCode: Int =
  41 * (
    41 * (
      41 + elem.hashCode
    ) + left.hashCode
  ) + right.hashCode
```

### Recipes for equals and hasCode

### Conclusion
