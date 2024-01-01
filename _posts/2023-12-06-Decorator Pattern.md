---
layout: post
title: Decorator Pattern
date: 2023-12-06 21:10 +0900 
description: Decorator Pattern
category: [디자인패턴, 구조패턴] 
tags: [구조패턴, 디자인패턴, 데코레이터패턴] 
pin: false
math: true
mermaid: true
---
Decorator Pattern 정리
<!--more-->


## Decorator Pattern


객체의 추가요소를 동적으로 더할 수 있다. 서브클래스를 만드는 것보다, 더 유연하게 기능을 확장할 수 있다.


즉, 아래와 같은 구조를 가능하게 한다. 아메리카노 → 휘핑 추가 → 샷 추가 → 샷추가 → 설탕추가 // 와 같이 계속 중첩적으로 기능을 가질 수 있게 한다.


![Untitled.png](/assets/img/post/Decorator%20Pattern(객체%20감싸기)/1.png)


기존의 클래스를 확장하는 데, 여기서 상**속을 이용하면 정적이므로** 런타임에서 객체의 행동을 변경할 수 없고 하나의 부모만 선택해야하는 제약도 따른다. **이에 집합 관계 혹은 합성 관계를 따른다.** Decorator에서 Component를 구성하여, 기능을 위임받을 수 있다.


### 실제 코드 예시


코드예시는 [GitHub](https://github.com/bethrobson/Head-First-Design-Patterns)에서 가져왔습니다.


**[Decorater 추상 클래스]**


```java
public abstract class CondimentDecorator extends Beverage {
	Beverage beverage;
	public abstract String getDescription();
}
```


**[Component 추상 클래스]**


```java
public abstract class Beverage {
	String description = "Unknown Beverage";
  
	public String getDescription() {
		return description;
	}
 
	public abstract double cost();
}
```


**[기능 테스트]**

1. DarkRoast차에 모카샷을 2개 추가하고, 휘핑크림을 추가한다.
2. HouseBlend에 두유와 모카를 추가하고, 휘핑크림을 추가한다.

```java
public static void main(String args[]) {
		Beverage beverage = new Espresso();
		System.out.println(beverage.getDescription() 
				+ " $" + beverage.cost());
 
		Beverage beverage2 = new DarkRoast();
		beverage2 = new Mocha(beverage2);
		beverage2 = new Mocha(beverage2);
		beverage2 = new Whip(beverage2);
		System.out.println(beverage2.getDescription() 
				+ " $" + beverage2.cost());
 
		Beverage beverage3 = new HouseBlend();
		beverage3 = new Soy(beverage3);
		beverage3 = new Mocha(beverage3);
		beverage3 = new Whip(beverage3);
		System.out.println(beverage3.getDescription() 
				+ " $" + beverage3.cost());
	}
```


실행결과 아래와 같이 출력된다.


```text
Espresso $1.99
Dark Roast Coffee, Mocha, Mocha, Whip $1.49
House Blend Coffee, Soy, Mocha, Whip $1.34
```


위와 같이 진행하면, 기존의 컴포넌트(차)를 데코레이터로 감싸면서 기능을 데코레이터에게 위임할 수 있다. 즉, 동적인 확장이 가능하다. 몇개든, 언제든 데코를 추가할 수 있다.


![설계패턴 수업 중 자료](/assets/img/post/Decorator%20Pattern(객체%20감싸기)/2.png)


### 단점

- 객체를 처음에 초기화하는 데, 코드가 많이 필요하다. 위에서도 초기화하는데, 이렇게 많은 코드가 필요했다.

```java
		Beverage beverage2 = new DarkRoast();
		beverage2 = new Mocha(beverage2);
		beverage2 = new Mocha(beverage2);
		beverage2 = new Whip(beverage2);
```

- 많은 클래스들로 인해, 이해하기 복잡할 수 있다.
- 특정 형식에 의존하는 코드에 도입하면 엉망이 된다.(아래는 GPT-4 예시)

	예를 들어, **`Bird`** 클래스의 인스턴스를 요구하는 함수가 있고, **`FlyingBirdDecorator`**로 **`Bird`** 인스턴스를 데코레이트한 경우, 데코레이트된 객체는 더 이상 **`Bird`** 타입이 아니게 됩니다. 따라서 이 함수에 데코레이트된 객체를 전달하려고 하면 타입 불일치 문제가 발생할 수 있습니다.

