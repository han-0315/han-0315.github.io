---
layout: post
title: Strategy Pattern
date: 2023-12-06 21:00 +0900 
description: Strategy Pattern
category: [디자인패턴, 행동패턴] 
tags: [행동패턴, 디자인패턴, 전략패턴] 
pin: false
math: true
mermaid: true
---
Strategy Pattern 정리
<!--more-->


## 상황


기존의 상위 타입에 새로운 기능을 추가할 때, 하위 클래스에서는 해당 기능을 사용하지 않는 클래스들이 있을 때, **기능의 추가 혹은 삭제 등의 변화가 빈번하게 일어날 때** 전략패턴을 적용한다.


ex) Duck 클래스에 fly()기능을 추가하려한다. 하지만, 기능을 추가하면 몇몇 날지 못하는 오리들(하위 클래스)도 날아버리는 문제가 생긴다. 


## 포인트


**바뀌는 부분은 따로 분리하여 캡슐화한다.**  **구현보다는 “행동”에 초첨을 맞춰 설계한다.** 즉, 상속은 많은 장점이 있지만 여러 단점 또한 존재한다. 앞으로 코드가 변경되지 않는다면, 상관없지만 코드가 변경될 일이 많다면 상속에 묶여 개발하기 힘들어질 수 있다. 그렇기에 행동에 초점을 맞춰 인터페이스로 설계한다.


## 예시


이제, 예시 상황으로 돌아가서 문제를 해결해보자. 위의 문제는 몇몇오리는 날 수 없는데, Fly method를 duck(상위 타입)에 추가하면 문제가 발생한다는 것이다. 이 문제를 해결하기 위해 바뀌는 부분인 Fly를 분리한다. 특정 오리는 날지않고, 특정 오리는 날 수 있다. 이에 `FlyBeHavior()` 라는 인터페이스를 통해 분리하고, 이를 Duck 클래스에 구성한다.(State, 멤버변수로 둔다.) 이렇게 구성하면 Fly라는 행동에 대한 여러 종류(클래스)를 위임할 수 있다. 이제 오리는 자신에게 맞는 Fly 행동을 동적으로 고를 수 있다.


```java
public abstract class Duck {
    FlyBehavior flyBehavior;
    QuackBehavior quackBehavior;

    performFly() {
        flyBehavior.fly();
    }

    performQuack() {
        quackBehavior.quack();
    }

    abstract display();
}
```


FlyBehavior의 인터페이스를 구현하는 클래스들의 예시도 같이 확인해보자


```java
public class FlyWithWings implements FlyBehavior {
    fly() {
        System.out.println("I'm flying!");
    }
}
```


위와 같이 행동의 상위수준을 정의하고, 행동(전략)을 클래스로 구현한 뒤, 객체들은 이 행동들을 포함(State)하여 행동을 위임하는 것을 전략 패턴이라고 한다. 


## 전략 패턴이란?


알고리즘(메소드)의 상위 수준을 정의하고, 각 알고리즘은 별도의 클래스에 넣은 후 다른 객체가 행동을 위임할 수 있도록 하는 패턴이다. 예를 들면 전쟁 게임이라고 가정하면, 공격과 관련된 알고리즘의 상위 수준은 Attack 일 것이다. 칼로 공격, 저격, 화포 발사, 화살 공격 등은 각 알고리즘에 해당한다. 즉 Attack strategy이라는 인터페이스를 두고, sword, gun 등의 하위 방법을 클래스로 정의한다.


## When?


하나의 알고리즘(method)에 대해 다양한 변형이 필요할 떄, 주로 사용한다.


ex) 연산이라는 알고리즘에는 더하기, 빼기, 곱하기, …. 제곱, 등등 다양한 변형을 사용할 수 있다.


만약, 하나의 behavior(ex. 연산)이 변하지 않는다면 단순한 패턴을 사용한다. 이후 여러 변형이 필요하다면, 이를 하나의 인터페이스로 옮기고 여러 개의 변형(클래스)로 바꾼다.

1. 전략이라는 interface를 생성(ex. 연산 행위)
2. 인터페이스를 구현하는 class 생성(ex. 더하기 행위)

~를 상속하다. Aggressive : 여기에 내것이 있다.


![https://refactoring.guru/ko/design-patterns](/assets/img/post/Strategy%20Pattern/1.png)


![Untitled.png](/assets/img/post/Strategy%20Pattern/2.png)


flying behavior, quacking behavior가 전략패턴, Duck이 전략패턴으로 만들어진 클래스를 이용한다.


## 장단점


장점

- 상속을 합성으로 대체할 수 있음(강한 의존도 낮춤)
- 클라이언트 클래스를 크게 수정하지 않고도 새로운 알고리즘을 도입할 수 있음

단점

- 알고리즘이 몇개 없고, 변하지 않는다면 굳이 복잡도를 키울 필요가 없음

## 스타크래프트


테란을 기준으로 캐릭터들에 전략패턴을 적용해보자.

- 마린(지상): 지상, 하늘 공격가능
- 파이어벳(지상): 지상 공격만 가능
- 벌처(지상): 지상 공격만 가능
- 탱크(지상): 지상 공격만 가능
- 골리앗(지상): 지상, 하늘 공격가능
- 레이스(하늘): 지상, 하늘 공격가능
- Moveable(인터페이스)
	1. AtGroud
	2. Fly
- Attack(인터페이스)
	1. Ground
	2. Sky
	3. All(지상, 하늘 공격 유형이 다른 것도 여기서 커버)
