---
layout: default
title: 익스트랙터 in Scala
category: post
author: Jae
---

> Programming in Scala by Martin Odersky 를 개인적인 공부 목적으로 정리한 포스트 입니다.

# 익스트랙터

* 지금까지 생성자를 사용한 패턴은 케이스 클래스와 관련이 있었다.
	* 예를들어 `Some(x)`는 `Some`이 케이스 클래스라서 올바른 패턴이었다.
* 케이스 클래스 없이 이런 패턴을 사용해보자
	* 익스트랙터!!

	
## 예제: 전자우편 주소 추출
* 주어진 문자열이 전자우편 주소인지 여부를 결정, 전자우편 주소인 경우라면 사용자와 도메인 부분에 접근

```scala
def isEmail(s: String): Boolean
def domain(s: String): String
def user(s: String): String

if (isEMail(s)) println(user(s) + "AT" + domain(s))
else println("not an email address")
```

* 복잡하다, 패턴매치를 사용하면 

```scala
EMail(user, domain)

s match {
	case EMail(user, domain) => println(user + "AT" + domain)
	case _ => println("not an email address")
}
```

* 문자열 `s`는 케이스 클래스가 아니다.
	* `EMail(user, domain)`에 부합하는 표현이 아니다.
	* 항상 `case _`에 매치된다.
* 익스트랙터를 사용하면 기존 타입에 새로운 패턴을 정의할 수 있다.

## 익스트랙터

```scala
object EMail {
	// 인젝션 메소드
	// 값을 만들어내는 apply: 익스트랙터 객체에 들어 있는 경우가 자주 있지만 패턴매치를 위한 필수 사항은 아니다.
	def apply(user: String, domain: String) = user + "@" + domain
	// 익스트랙터 메소드
	// 값을 매치시켜 각 부분을 나누는 것이다.
	// 주어진 문자열이 전자 우편 주소가 아닌 경우도 처리해야 하기 때문에 Option 타입을 반환한다.
	def unapply(str: String): Option[(String, String)] = {
		val parts = str split "@"
		if (parts.length == 2) Some(parts(0), parts(1)) else None
	}
}

// EMail.unapply(s)
s match { case EMail(user, domain) => ... }
```

* `None`의 경우 패턴 매치가 이뤄지지 않고 시스템은 다름 패턴을 시도하거나 `MatchError`예외를 내고 패턴 매치에 실패한다.
* `Some(u,d)`인 경우 패턴이 매치되어, unapply가 반환한 값이 각 변수에 바인딩 된다.
* `s`의 타입인 String은 unapply의 인자 타입과 꼭 부합해야 하는것은 아니다.

```scala
val x: Any = ...
x match { case EMail(user, domain) => ... }
```

* 패턴 매처가 위의 코드를 보면, `x`가 `unapply` 메소드 인자의 타입인 `String`에 부합하는지 확인 후, 부합한다면 `x`를 `String`으로 캐스팅해서 처리, 부합하지 않으면 매치가 바로 실패 한다.

* 인젝션 메소드
	* 인자를 몇 가지 받아서 어떤 집합의 원소를 만들어 낸다.
	* 결과 집합 == 전자우편 주소인 문자열의 집합
* 익스트랙션 메소드
	* 어떤 집합에 속한 원소에서 여러 부분의 값을 뽑아낸다.
	* 사용자와 도메인 부분 문자열을 뽑아낸다.
* 인젝션 메소드와 익스트랙션 메소드
	* 어떤 객체안에 인젝션 메소드의 존재 여부와 관계 없이 익스트랙션 메소드가 존재하면 익스트랙터라고 부른다.
	* 인젝션 메소드가 포함된 경우라면 인젝션 메소드와 익스트랙션 메소드는 쌍대성을 이루어야 한다.
	* 스칼라가 이를 강제로 요구하지는 않지만, 익스트랙터를 설계할 때 이를 지키는 편이 좋다.

```scala
scala> EMail.unapply(EMail.apply(user, domain))
Some(user, domain)
```

## 변수가 없거나 1개만 있는 패턴
* unapply로 N개의 변수를 바인딩하고 싶다면 N개의 원소로 된 튜플을 `Some`에 감싸서 반환하면 될 것
* 스칼라에 1튜플은 없기 때문에 변수를 하나만 바인딩해야 하는 경우는 다르게 취급
	* 원소 자체를 `Some`으로 감싸서 반환
* 아무 변수도 바인딩하지 않는 것도 가능
	* 매치 성공인 경우 true, 실패인 경우 false
	* 호출 시 빈 파라미터 목록`()`을 취한다.
		* 괄호가 없다면 `UpperCase`라는 객체와 같은지 확인한다.


## 가변 인자 익스트랙터
* 앞에서 본 전지우편 주소 익스트랙터 메소드는 모두 정해진 개수의 값을 반환했다. 때때로 이는 충분히 유연하지 못하다.

```scala
dom match { 
	case Domain("org", "acm") => println("acm.org") 
	case Domain("com", "sun", "java") => println("java.sun.com")
	//인자 목록의 나머지 원소를 시퀀스에 매치해주는 시권스 와일드카드 패턴인 _*
	case Domain("net", _*) => println("a .net domain")
	//case Domain(d, etc @ _*) => println("a ." + d + "domain") 
}
```

* 가변 길이를 처리 할 때는 `unapplySeq` 익스트랙터 메소드를 사용한다.
	* 결과 타입은 `Option[Seq[T]]`을 만족해야 한다.

```
Object Domain { 
	// 인젝선 매소드(선택적)
	def apply(parts : String* ): String =		parts.reverse.mkString(".")
	// 익스트렉더 메소드(필수)
	def unapplySeq(whole : String) : Option[Seq[String]) =		Some(whole.split("\\.").reverse)
```

* unapplySeq에서 가변 길이 부분과 고정적인 요소를 함께 반환할 수도 있다.
	* 튜플에 모든 원소를 넣되， 마지막에 가변 부분을 넣으면 된다.

```scala
// 전지우편에서 도메인 부분을 시퀀스로 확장하는 새 익스트랙터
object ExpandedEMail { 
	def unapplySeq(email: String) 
		: Option[(String, Seq[String])] = { 
			val parts = email split "@" 
			if (parts.length == 2)
				Some(parts(0), parts(1).split("\\.").reverse) 
			else None
	}
}
```


## 익스트랙터와 시퀀스 패턴
* 15장에서 리스트나 배열의 원소에 시퀀스 패턴으로 아래와 같이 접근 할 수 있다.

```scala
List() 
List(x , y , _*) 
Array(x , 0 , 0 , _)
```

* `scala.List` 동반 객체에 `unapplySeq` 정의가 있기 때문에 아래와 같은 패턴이 가능하다.

```scala
package scala 

object List { 
	def apply[T](elems: T * ) = elems.toList 
	def unapplySeq[T](x: List[T]): Option[Seq[T]] = Some(x) 
	... 
}
```

## 익스트랙터와 케이스 클래스
* 케이스 클래스
	* 닫힌 어플리케이션 코드를 작성한다면 케이스 클래스가 더 좋다.
	* 더 간결하고 빠름
	* 컴파일 시점 검사가 가능
	* 클래스 계층구조를 변경하면 리팩토링을 해야하는 단점이 있음
* 익스트랙터
	* 패턴과 그 패턴이 선택하는 객체의 내부 데이터 표현 사이에 아무런 관계가 없도록 만들 수 있다. (표현독립성)
	* 어떤 타입을 미리 알지 못하는 여러 클라이언트에게 노출할 필요가 있다면 표현 독립성을 위해 익스트랙터를 사용하는 편이 좋다.


## 정규 표현식
* 익스트랙터를 사용하면 정규표현식을 더 멋있게 사용할 수 있다.

### 정규 표현식 만들기
* 스칼리는 자바에서 정규표현식 문법을 가져왔다.
* 스칼라의 정규 표현식 클래스는 `scala.util.matching` 패키지에 들어 있다

```scala
import scala.util.matching.Regex

// 정규 표현식 값은 문자열을 Regex 생성장에 넘겨서 만들 수 있다.
val Decimal = new Regex("(-)?(\\d+)(\\.\\d * )?")
val Decimal = new Regex("""(-)?(\d+)(\.\d * )?""")
val Decimal = """(-)?(\d+)(\.\d * )?""".r
```


### 정규 표현식 검색
```scala
scala> val Decimal = """(-)?(\d+)(\.\d * )?""".r 
Decimal: scala.util.matching.Regex = (-)?(\d+)(\.\d * )?

scala> val input = "for -1.0 to 99 by 3" 
input: java.lang.String = for -1.0 to 99 by 3

// input 안에 정규 표현식과 매치되는 모든 문자열을 반환, 결과는 Iterator 타입
scala> for (s <- Decimal findAllIn input)
			println(s) 
	
-1.0 
99 
3

// input 안에 정규 표현식과 매치되는 첫 번째 부분 문자열을 검색, 결과는 Option 타입
scala> Decimal findFirstIn input 
res7: Option[String] = Some(-1.0)

// input의 맨 앞부분부터 검사해 정규 표현식과 매치할 수 있는 접두사를 반환
scala> Decimal findPrefixOf input 
res8: Option[String] = None
```

### 정규 표현식 뽑아내기
* 스칼라의 모든 정규 표현식은 익스트랙터를 정의한다.

```scala
scala> val Decimal(sign, integerpart, decimalpart) = "-1.23"

sign: String = - 
integerpart: String = 1 
decimalpart: String = .23
```


## 결론
* 어떤 타입의 내부 표현과 클라이언트가 그 타입을 보는 방식 사이에 간접 계층을 둘 수 있다.
* 스칼라 라이브러리는 익스트랙터를 아주 많이 사용한다.

