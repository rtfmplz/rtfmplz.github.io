---
layout: post
title: 객체를 사용한 모듈화 프로그래밍
tags: [scala, PIS, object]
author: Jae
comments: true
---

> Programming in Scala by Martin Odersky 를 개인적인 공부 목적으로 정리한 포스트 입니다.

# 객체를 사용한 모듈화 프로그래밍

스칼라는 규모의 확장성이 있는 언어이다. 프로그램의 크기와 상관없이 같은 기법을 사용하여 프로그램을 작성할 수 있다.

13장의 패키지와 접근 수식자를 사용해서 큰 프로그램을 패키지와 모듈로 구성할 수 있다.  
하지만, 패키지는 추상화할 방법을 제공하지 않기 때문에 패키지와 접근 수식자 만으로 큰 프로그램을 작성하기에는 한계가 있다.

29장에서는 스칼라의 객체지향 기능을 사용해 프로그램을 더 모듈화할 수 있는 방법을 살펴본다.

* 모듈로 사용할 수 있는 간단한 싱글톤 객체
* 트레이트와 클래스를 사용한 모듈의 추상화
* 트레이트를 사용해 여러 파일에 모듈을 분리

> 모듈?
> 
> * 잘 정의된 인터페이스를 제공하며 내부 구현은 숨기는 '더 작은 프로그램 조각'
> 

## 시스템 수준의 모듈화를 달성하기 위해서..

크기가 큰 프로그램을 모듈 형태로 작성하면 아래와 같은 이점이 생긴다.

* 시스템을 이루는 각기 다른 모듈을 따로따로 컴파일할 수 있으면 여러 팀이 독립적으로 작업할 때 도움이 된다.
* 모듈간 독립적이기 때문에 필요에 따라 개별 모듈을 구현체를 다른 것으로 바꿀수 있다.
	* 예를들어 단위 테스트에서 특정 모듈을 mock으로 대체할 수 있다.

> 단위테스트, 통합테스트, 스테이징, 배포 등 맥락에 따라서 모듈변경 할 수 있고, 각각에 필요한 설정을 사용할 수 있다.

훌륭한 시스템 수준의 모듈화를 위해서 모듈은 다음과 같은 기능을 제공해야 한다.

* 인터페이스와 구현을 잘 분리할 수 있는 모듈 구조를 제공해야 한다.
* 어떤 모듈과 인터페이스를 동일한 다른 모듈로 변경하더라도 그 모듈 인터페이스에 의존하는 다른 모듈들은 재컴파일하지 않아야 한다.
* 모듈을 서로 연결할 방법을 제공해야 한다.
	* 시스템 설정을 사용할 수 있다.

위의 기능을 잘 제공할 수 있게 해주는 방법 중 하나가 의존 관계 주입(dependency injection) 이다. 

> Dependency Injection
> 
> * 구체적인 의존 오브젝트와 이것을 사용하는 오브젝트(일반적으로 클라이언트)를 런타임 시에 연결해주는 작업

* 자바 플랫폼에서 이런 기법을 제공하는 프레임 워크
	* 스프링
	* 구글 주스

스프링에서는 기본적으로 모듈 인터페이스를 자바 인터페이스로 표현하고, 모듈 구현을 자바 클래스로 표현한다. 각 모듈 사이의 의존 관계를 기술하고, 애플리케이션으로 연결하는것은 XML 설정 파일을 통한다.

스칼라에서도 스프링처럼 시스템 수준의 모듈화를 달성하기 위해서 몇가지 도구를 가지고 있다.  

이번 장에서는 스칼라에서 객체를 모듈로 사용해 외부 프레임워크를 사용하지 않고도 대규모 프로그래밍의 모듈화를 원하는대로 달성할수 있는 방법을 소개한다.


## 조리법 애플리케이션 (싱글톤 객체 형태의 모듈)

사용자가 조리법을 관리하는 엔터프라이즈 웹 애플리케이션을 만든다고 상상해보자.  


### 요구사항

* 소프트웨어를 도메인 계층과 어플리케이션 계층으로 나누어 작성
	* 도메인 객체 (계층)
		* 비지니스 개념과 규칙을 담는다.
		* 외부 관계형 데이터베이스에 영구 저장할 상태를 캡슐화 한다.
	* 애플리케이션 (계층)
		* 사용자에게 애플리케이션이 제공할 서비스에 대한 API를 노출한다.
		* 애플리케이션은 업무를 조율하고 작업을 도메인 계층의 객체에게 위임하는 방식으로 서비스를 구현한다.
* 단위테스트를 쉽게 작성하기 위해서 각 계층에 속한 객체를 목 버전으로 바꿔치기 하고 싶다.
	* 목 버전을 사용하고 싶은 객체를 모듈로 다룬다.


### 모델링

스칼라는 객체를 모듈로 사용한다.  
다음은 싱글톤 객체를 이용해서 테스트를 위한 목 객체를 구현하는 방법을 소개한다.  
도메인 계층의 데이터베이스는 모든 조리법을 보관한다.  
어플리케이션 계층의 브라우저는 지정한 재료가 포함된 모든 조리법을 찾는것 같은 데이터베이스의 검색과 브라우징을 돕는다.  

아래 Food, Recipe 클래스는 데이터베이스에 저장할 엔티티를 표현한다.

```scala
package org.stairwaybook.recipe

// 간단한 Food 엔티티 클래스
abstract class Food(val name: String) {
	override def toString = name
}
```

```scala
package org.stairwaybook.recipe

// 간단한 Recipe 엔티티 클래스
class Recipe (
	val name: String,
	val ingredients: List[Food],
	val instructions: String
) {
	override def toString = name
}
```

다음은 위에서 구현한 클래스의 싱글톤 인스턴스이다.  
이런 인스턴스들은 테스트를 작성할 때 쓰 수 있다.  

```scala
package org.stairwaybook.recipe

// 테스트에 사용할 Food와 Recipe
object Apple extends Food("Apple")
object Orange extends Food("Orange")
object Cream extends Food("Cream")
object Sugar extends Food("Sugar")

object FruitSalad extand Recipe (
	"frout salad",
	List(Apple, Orange, Cream, Sugar),
	"Stir it all together"
)
```

```scala
// 목 데이터베이스 모듈
package org.stairwaybook.recipe

// 목 객체이므로 데이터베이스 모듈은 가단한 인메모리 리스트를 사용
object SimpleDatabase {
	def allFoods = List(Apple, Orange, Cream, Sugar)
	// find는 리스트에서 술어함수를 만족하는 가장 첫 원소를 반환한다.
	def foodNamed(name: String): Option[Food] = allFoods.find(_.name == name)
	def allRecipes: List[Recipe] = List(FruitSalad)
	
	// 카테고리
	case class FoodCategory(name: String, foods: List[Food])
	private var categories = List(
		FoodCategory("fruits", List(Apple, Orange)),
		FoodCategory("misc", List(Cream, Sugar))
	)
	def allCategories = categories
}
```

```scala
// 목 브라우저 모듈
object SimpleBrowser {
	/**
		food를 사용하는 Recipe를 검색
	*/
	def recipesUsing(food: Food): List[Recipe] = 
		SimpleDatabase.allRecipes.filter(recipe => 
			recipe.ingredients.contains(food))
			
	def displayCategory(category: SimpleDatabase.FoodCategory)
}
```

데이터베이스와 브라우저는 다음과 같이 사용 할 수 있다.

```bash
scala> val apple = SimpleDatabase.foodNamed("Apple").get
apple: Food = Apple
scala> SimpleBrowser.recipesUsing(apple)
res0: List[Recipe] = List(fruit salad)
```


## 추상화 (트레이트와 클래스를 사용한 모듈의 추상화)

앞의 예제에서 애플리케이션을 데이터베이스와 블라우져 모듈로 분리하였다.  
하지만, 브라우저와 데이터베이스 모듈 사이에 '단단한 연결'이 존재하기 때문에 설계가 완벽하게 모듈화 되었다고 할 수 없다.  
예를들어 다음과 같이 SimpleBrowser 모듈이 SimpleDatabase의 이름을 직접 부르기 때문에, 데이터베이스 모듈을 바꿔 끼리면 브라우저도 컴파일 해야만 한다.

```scala
SimpleDatabase.allRecipes.filter(recipe => ....)
```

예를들어 여러 조리법 데이터베이스를 지원하고, 각각의 조리법 데이터베이스마다 별도의 브라우저를 만들수 있기를 바란다면 위와 같은 구조는 많은 중복된 코드를 가진다.  
각 인스턴스마다 브라우저 코드를 재활용하면 코드의 중복을 줄일 수 있다. 각각의 브라우저는 데이터베이스를 참조하는 코드를 제외하면 모두 같은 코드를 사용할 수 있다.  


### 코드의 반복을 최소화 하기

모듈이 객체라면, 클래스가 모듈에 대한 탬플릿이다.  
따라서 모든 인스턴스의 공통점을 클래스로 기술할 수 있다.

```scala
// 추상 데이터베이스에 대한 val 값을 가진 Browser 클래스

abstract class Browser {
	val database: Database
	
	def recipesUsing(food: Food): List[Recipe] = 
		// SimbleDatabase를 local database로 변경
		database.allRecipes.filter(recipe => 
			recipe.ingredients.contains(food))
			
	def displayCategory(category: SimpleDatabase.FoodCategory)}
```

```scala
// 추상 메소드가 있는 데이터베이스 클래스
abstract class Database {
	def allFoods = List[Food]
	def allRecipes: List[Recipe]
	def foodNamed(name: String): Option[Food] = allFoods.find(_.name == name)
	
	// 카테고리
	case class FoodCategory(name: String, foods: List[Food])
	def allCategories: List[FoodCategory]
}
```

```scala
// Database를 상속하는 SimpleDatabase
object SimpleDatabase extends Database {
	def allFoods = List(Apple, Orange, Cream, Sugar)
	def allRecipes: List[Recipe] = List(FruitSalad)
	
	private var categories = List(
		FoodCategory("fruits", List(Apple, Orange)),
		FoodCategory("misc", List(Cream, Sugar))
	)
	def allCategories = categories
}
```

이제, 다음과 같이 사용할 데이터베이스를 지정해서 Browser 클래스를 인스턴스화해서 구체적인 브라우저 모듈을 만들 수 있다.

```scala
object SimpleBrowser extends Browser {
	val database = SimpleDatabase
}
```

이렇게 만든 모듈은 예전과 다름없이 사용할 수 있다.  
원한다면 두번째 목 데이터베이스를 만들 수 있고 앞에서 만든 브라우저 클래스를 사용할 수 있다.

코드는 생략!  
Page. 699 리스트 29.10 참조


## 모듈을 트레이트로 분리하기

모듈이 너무 커서 한 파일에 넣기 어려울 때가 있다.  
그러면 트레이트를 사용해서 모듈을 여러 파일로 나눌 수 있다.  

예를들어, 카테고리 관련 코드는 Database 파일에서 빼서 따로 분리할 수 있다.

```scala
trate FoodCategories {
	case class FoodCategory(name: String, foods: List[Food])
	def allCategories: List[FoodCategory]
}
```

```scala
abstract class Database extends FoodCategories {
	def allFoods: List[Food]
	def allRecipes: List[Recipe]
	def foodNamed(name: String) = allFoods.find(f => f.name == name)
}
```

이런 방법으로 `allFoods`와 `allRecipes`를 `trait SimpleFoods`와 `trait SimpleRecipes`로 분리할 수 있다.

```scala
object SimpleDatabase extends Database with Simple Foods with SimpleRecipes
```

```scala
trait SimpleFoods {
	object Pear extends Food("Pear")
	
	def allFoods = List(Apple, Pear)
	def allCategories = Nil
}
```

```scala
trait SimpleRecipes {
	//this: SimpleFoods => // 컴파일러에게 알려주기 위한 방법: 셀프 타입
	
	object FruitSalad extand Recipe (
		"frout salad",
		List(Apple, Pear),
		"Stir it all together"
	)
	def allRecipes = List(FruitSalad)
}
```

Pear가 다른 트레이드에 있기 때문에 컴파일 오류난다.  
컴파일러는 SimpleFoods를 SimpleRecipes와 함께 믹스인 해야 한다는 사실을 모른다.  
이를 컴파일러에게 알려주기 위한 방법이 셀프 타입(self type)이다.

> self type
> 
> * 클래스 안에서 사용하는 모든 this의 타입이 무엇인지를 가정하는것이다. 트레이트가 섞여 들어갈 구체적 클래스를 제한한다.


## 실행 시점 링킹

메인 함수의 argment를 이용한 실행시점 프로그램의 역할 선택.  
실행 시점에 데이터베이스를 선택하여 데이터를 출력하는 예제, 동적으로 모듈 구현체를 선택한다.

```scala
object GotApples {
	def main(args: Array[String]) {
		val db: Databse = 
			if(args(0) == "student")
				StudentDatabase
			else
				SimpleDatagse
				
				...  생략 ...
	}
}
```


## 모듈 인스턴스 추적

타입이 같은데도 컴파일러가 검증에 실패하는 경우 싱글톤 타입(.type)을 사용해서 문제를 해결한다.

```scala
object GotApples {
  def main(args: Array[String]) {
     val db: Database = SimpleDatabase
     object browser extends Browser {
       val database = db //: db.type 이거 추가하여 타입 검사기에게 이둘이 같은 객체임을 알려줘야한다.
     }
    //컴파일러는 db와 browser.database가 같다는 사실을 모른다.
    for(category <- db.allCategories)
      browser.displayCategory(category)
  }
}
```

db.type 을 사용하여 이둘이 같은 객체임을 알려줘야한다. .type은 싱글톤 타입이라는 뜻이다. 싱글톤 타입을 사용해 컴파일러에게 db와 browser.database가 같은 객체임을 알려주는 방식으로 타입 오류를 없앨 수 있다.


## 결론

스칼라의 객체를 모듈로 사용하는 방법을 살펴봄.  
다른 사람이 작성한 코드에서 이런 코딩 패턴이 눈에 뜨인다면 이 챕터를 참고하세요.



