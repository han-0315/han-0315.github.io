---
layout: post
title: Adapter Pattern
date: 2023-12-06 21:10 +0900 
description: Adapter Pattern 정리
category: [디자인패턴, 구조패턴] 
tags: [구조패턴, 디자인패턴, 어댑터패턴] 
pin: false
math: true
mermaid: true
---
Adapter Pattern 정리
<!--more-->


### 상황


## Adapter Pattern이란?


**어댑터**는 호환되지 않는 인터페이스를 가진 **객체들이 협업할 수 있도록** 하는 구조적 디자인 패턴이다.


가장 흔하게 사용하는 예시로, 영국 콘센트는 우리나라 콘세트와 다르다. 그렇기에, 만약 충전기를 콘센트에 꽂으려고 하면 들어가지 않는다. 이를 위해 Adapter가 존재한다. Adapter를 통해 콘센트 > Adapter > 충전기를 연결하여 기존 콘센트와 충전기를 변형하지 않고도 연결가능하다.


클래스 Apdater와 Object Adapter이 있지만, 다중 상속이 안되는 Java에서는 인터페이스와 객체 구성을 사용하는 Adapter 패턴만 사용할 수 있다.


**[Object Adapter]**


![Untitled.png](/assets/img/post/Adapter%20Pattern(어댑터)/1.png)


**[Class Adapter]**


![Untitled.png](/assets/img/post/Adapter%20Pattern(어댑터)/2.png)


## 코드 예시


### Object Adapter


기존의 Duck 인터페이스를 통해서도 Turkey(칠면조)를 다루고 싶을 때 아래와 같이 Adapter 패턴을 사용한다.


Adapter 클래스에서는 각각 Duck에 맞는 함수를 연결해준다.


```java
public interface Turkey {
	public void gobble();
	public void fly();
}
```


```java
public class TurkeyAdapter implements Duck {
	Turkey turkey;
 
	public TurkeyAdapter(Turkey turkey) {
		this.turkey = turkey;
	}
    
	public void quack() {
		turkey.gobble();
	}
  
	public void fly() {
		for(int i=0; i < 5; i++) {
			turkey.fly();
		}
	}
}
```


[테스트 코드]


```java
public class DuckTestDrive {
	public static void main(String[] args) {
		Duck duck = new MallardDuck();

		Turkey turkey = new WildTurkey();
		Duck turkeyAdapter = new TurkeyAdapter(turkey);

		System.out.println("The Turkey says...");
		turkey.gobble();
		turkey.fly();

		System.out.println("\nThe Duck says...");
		testDuck(duck);

		System.out.println("\nThe TurkeyAdapter says...");
		testDuck(turkeyAdapter);
		
		// Challenge
		Drone drone = new SuperDrone();
		Duck droneAdapter = new DroneAdapter(drone);
		testDuck(droneAdapter);
	}

	static void testDuck(Duck duck) {
		duck.quack();
		duck.fly();
	}
}
```


### Class adapter


하지만, 아래의 방법은 Java에서 권장하지 않는다.


```java
public class extends Turkey implementes Duck {
	public void quack() {
		gobble();
	}
	public void fly() {
		fly();
	}
}
```


### 비교(객체 어댑터 vs 클래스 어댑터)

- 객체 어댑터는 어댑티의 서브 클래스에 대해서도 어댑터 역할을 할 수 있다.
- 클래스 어댑터는 코드를 재사용할 수 있다. 하지만, 실제 사용하는 코드만 작성할 수 없다.

다중상속을 권장하지 않는 Java에서는 객체 어댑터 방식을 사용한다.


### 데코레이터 vs 어댑터 패턴


둘다 하나의 객체를 감싼다는 것에서 동일하다. 데코레이터 패턴은 Component 클래스를 구성객체를 통해 가지고 있으며 기존의 클래스를 지속적으로 감싸, 행동을 위임한다. 어댑터 패턴은 데코레이터 패턴처럼 객체를 감싸며, 인터페이스를 변환한다. (Type Change) 인터페이스를 변환한다는 차이점과 더불어 어댑터는 단순 메서드를 연결하는 반면 데코레이터는 행동과 책임을 위임한다.

