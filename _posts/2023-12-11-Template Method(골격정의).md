---
layout: post
title: Template Method Pattern 정리
date: 2023-12-09 21:10 +0900 
description: Template Method
category: [디자인패턴, 행동패턴] 
tags: [행동패턴, 디자인패턴, 템플릿메서드패턴] 
pin: false
math: true
mermaid: true
---
Template Method Pattern 정리
<!--more-->


### 상황


커피와 홍차를 끊이는 과정은 상당히 유사하다. 하지만, 커피는 원두를 우려내고 홍차는 잎을 우려낸다. 


나머지 방식, “물을 끊인다”, “컵에 따른다”는 일치한다. 그리고 무엇가를 우려내고, 토핑(홍차는 레몬, 커피는 설타)을 추가하는 것도 유사하다. 이런 식으로 같은 단계를 따르는 객체가 수 없이 많다면, 템플릿 메소드 패턴을 통해 중복된 코드를 제거할 수 있다.


## Template Method Pattern 이란?


**템플릿 메서드**는 **부모 클래스에서** 알고리즘의 **단계를 정의한다. 하위** 클래스들이 알고리즘의 특정 단계들을 **오버라이드(재정의)할 수 있도록** 하는 행동 디자인 패턴이다.


> 로직을 단계 별로 나눠야 하는 상황에서 적용한다.


	단계별로 나눈 로직들이 앞으로 수정될 가능성이 있을 경우 더 효율적이다.


![https://refactoring.guru/ko/design-patterns/template-method](/assets/img/post/Template%20Method(골격정의)/1.png)
[출처: https://refactoring.guru/ko/design-patterns]

1. **구상 클래스들**은 모든 단계들을 **오버라이드**할 수 있지만, 골격(단계)를 나타내는 `templateMethod()` 함수는 **오버라이드 할 수 없다.** Java에서는 **수정이 불가능하게 final** 키워드를 추가한다.
2. 각 단계들은 외부는 막고, 자식들만 활용할 수 있도록 protected로 선언한다

### 예시 코드


책에서는 설명한 커피와 홍차를 예시로 설명한다. 커피, 홍차 모두 카페인 음료라는 것에서 공통점이 있다.


```java
public abstract class CaffeineBeverage {
  
	final void prepareRecipe() {
		boilWater();
		brew();
		pourInCup();
		addCondiments();
	}
 
	abstract void brew();
  
	abstract void addCondiments();
 
	void boilWater() {
		System.out.println("Boiling water");
	}
  
	void pourInCup() {
		System.out.println("Pouring into cup");
	}
}
```


**[커피 클래스]**


```java
public class Coffee extends CaffeineBeverage {
	public void brew() {
		System.out.println("Dripping Coffee through filter");
	}
	public void addCondiments() {
		System.out.println("Adding Sugar and Milk");
	}
}
```


**[홍차 클래스]**


```java
public class Tea extends CaffeineBeverage {
	public void brew() {
		System.out.println("Steeping the tea");
	}
	public void addCondiments() {
		System.out.println("Adding Lemon");
	}
}
```


### Hook(훅)


만약, Type별로 선택적으로 단계를 수행해야 한다면, 예를 들면 커피인 경우 이번엔 우유를 필수적으로 추가해야한다고 해보자. 그러면 만약 CaffeineBeverage의 타입이 Coffee라면 단계를 나타내는 메서드에 Hook을 만들어놔야 한다. 왜냐하면 단계를 나타내는 메서드는 final로 하위클래스에서 오버라이딩할 수 없다. 그렇기에 오버라이딩 가능하게 열어놔야 하는데, 그게 Hook method이다. 


아래의 코드를 확인하면 `prepareRecipe()` 오버라이딩할 수 없지만, `customerWantsCondiments()` 은 언제든 오버라이딩할 수 있다. 그렇기에 만약, 홍차라면 `customerWantsCondiments()`를 오버라이딩하여 false를 반환하게 하면 단계를 수정할 수 있다.


```java
final void prepareRecipe() {
		boilWater();
		brew();
		pourInCup();
		if (customerWantsCondiments()) {
			addCondiments();
		}
	}
... 
	boolean customerWantsCondiments() {
		return true;
	}
```


### 할리우드 원칙


“우리한테 연락하지 마세요, 알아서 연락줄게요” 라는 원칙으로 고수준의 요소가 저수준의 요소에게 해야하는 원칙이다. 즉, 저수준의 요소가 고수준의 요소를 연락(호출)하지 않고 고수준이 필요할 때 저수준을 호출한다.


Template method Pattern 또한, 이를 준수하고 있다. 상위 요소(CaffeineBeverage)는 필요할 때(오버라이딩)만 저수준의 메서드를 호출하고, 반대로 저수준은 상위 요소를 호출하는 일이 없다.


여기서 저수준의 요소가 반드시 고수준의 요소를 호출하면 안되는 것이 아니다. 다만, **순환 의존성**이 생기는 일은 방지해야한다.


### Java 프레임워크 예시


자바의 ArrayList에서는 sort함수가 있다. sort는 template method 패턴을 사용하고 있다. 구체적인 단계를 정의해두고, 하위 클래스에서 단지 비교하는 법만(CompareTo) 정의해주면 된다. 아래의 코드는 “서브 클래스에서 하나의 단계를 오버라이딩한다”라는 Template method Pattern 정의에 정확하게 부합하지 않는다. 여기선 ArrayList가 아닌 Comparable라는 인터페이스를 통해 연결되어있다. 


> 책에서는 자바 개발자가, 배열은 서브클래스를 만들 수 없다라는 제약조건에 의해 다음과 같은 방식을 선택했을거라고 한다.


```java
public class Duck implements Comparable<Duck> {
	String name;
	int weight;
  
	public Duck(String name, int weight) {
		this.name = name;
		this.weight = weight;
	}
 
	public String toString() {
		return name + " weighs " + weight;
	}
  
	public int compareTo(Duck otherDuck) {
 
  
		if (this.weight < otherDuck.weight) {
			return -1;
		} else if (this.weight == otherDuck.weight) {
			return 0;
		} else { // this.weight > otherDuck.weight
			return 1;
		}
	}
}
```


**[테스트 코드]**


```java
public static void main(String[] args) {
		Duck[] ducks = { 
						new Duck("Daffy", 8), 
						new Duck("Dewey", 2),
						new Duck("Howard", 7),
						new Duck("Louie", 2),
						new Duck("Donald", 10), 
						new Duck("Huey", 2)
		 };

		System.out.println("Before sorting:");
		display(ducks);

		Arrays.sort(ducks);
 
		System.out.println("\nAfter sorting:");
		display(ducks);
	}
```


**[결과]**


```text
Before sorting:
Daffy weighs 8
Dewey weighs 2
Howard weighs 7
Louie weighs 2
Donald weighs 10
Huey weighs 2

After sorting:
Dewey weighs 2
Louie weighs 2
Huey weighs 2
Howard weighs 7
Daffy weighs 8
Donald weighs 10
```


### 다른 패턴과 비교


**[Factory Method Pattern]**


팩토리 메서드 패턴은 특화된 Template Method Pattern으로 **기본단계에서 객체를 생성하고 리턴하는 Pattern**이다.


**[Strategy Pattern]**


전략 패턴은 모든 알고리즘이 다 명시되어있는 반면, 템플릿 메서드 패턴은 단계가 명시되어있고 일부 단계는 하위 클래스에서 오버라이딩할 수 있다.


### ****abstract와 Interface의 차이는?**

- abstract : 부모의 기능을 자식에서 확장시켜나가고 싶을 때 → 상속(단 한개의 클래스만 상속가능)
- interface : 해당 클래스가 가진 함수의 기능을 활용하고 싶을 때 → 구현(여러 인터페이스도 구현가능)

> Java에서는 다중 상속이 안된다. 상황에 맞게 활용하자!


[**팩토리 메서드**](https://refactoring.guru/ko/design-patterns/factory-method)는 [**템플릿 메서드**](https://refactoring.guru/ko/design-patterns/template-method)의 특수화라고 생각할 수 있다.

