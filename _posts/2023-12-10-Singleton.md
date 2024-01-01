---
layout: post
title: Singleton Pattern
date: 2023-12-09 21:10 +0900 
description: Singleton Pattern
category: [디자인패턴, 생성패턴] 
tags: [생성패턴, 디자인패턴, 싱글톤패턴] 
pin: false
math: true
mermaid: true
---
Singleton Pattern 정리
<!--more-->
## 싱글톤 패턴이란?


**싱글턴**은 **클래스에 인스턴스가 하나만 있도록** 하면서 이 **인스턴스에 대한 전역 접근(액세스) 지점**을 제공하는 생성 디자인 패턴이다.


즉, 데이터 베이스와 같은 경우 하나의 인스턴스만을 가져야 한다. 이럴 때 싱글톤 패턴을 사용한다.(인스턴스가 1개만 있음을 보증)


> **[설계 패턴 수업에서]**

	- 디자인 패턴이 아닌 코딩 패턴으로 보는 사람도 있음(경계에 속함)
	- 여러 가지 이유로 인스턴스를 하나만 생성하고, 클래스를 통해 참조만 하게함

```java
public class Singleton {

    private static Singleton instance = new Singleton();
    
    private Singleton() {
        // 생성자는 외부에서 호출못하게 private 으로 지정해야 한다.
    }

    public static Singleton getInstance() {
        return instance;
    }

    public void say() {
        System.out.println("hi, there");
    }
}
```


### 병행성


***병행성 조심**


여기 아래의 방법은 간단히 getInstance 뿐만 아니라 병행성 프로그래밍에 대해 고려해볼만한 문제를 제시한다.

1. 아래와 같이 이미 클래스 내의 전역변수로 생성 → 안쓰이는 경우 자원 낭비

```java
public class Singleton {
    private static Singleton uniqueInstance = new Singleton();

    // other useful instance variables
    private Singleton() {
    }

    public static Singleton getInstance() {
        return uniqueInstance;
        // other useful methods }
    }
}
```

1. 혹은 getInstance 함수에 병행성 코드 추가 → lock이니 성능저하

```java
public class Singleton {
    private static Singleton uniqueInstance = new Singleton();

    // other useful instance variables
    private Singleton() {
    }

    public static synchronized Singleton getInstance() {
        return uniqueInstance;
        // other useful methods }
    }
}
```

1. double checked locking (java 1.5 이상에서 보장) → 성능 우수하나 + 구현이 어렵다.

```java
public static Singleton getInstance() {
		if (uniqueInstance == null) {
			synchronized (Singleton.class) {
				if (uniqueInstance == null) {
					uniqueInstance = new Singleton();
				}
			}
		}
		return uniqueInstance;
	}
```


[**Java 병행성, v1.5 ~ ]**

- **Synchronize:** 해당 블락에 대한 접근자체에서 lock을 통해 병행성 관리
- **Volatile:** 해당 변수에 대중

## 고려사항

1. class를 전부 static하게, 클래스끼리 공유가능하게 만들면 싱글톤이랑 똑같은 기능을 수행할 수 있다.

하지만, 정적 초기화 단계에서 여러 에러가 발생할 수 있으며 발생한 에러는 찾기 힘들다.

1. 클래스 로더

[**클래스로더란?]**


자바 클래스들은 컴파일때 로드되지 않고, 동적으로 필요할 때 로드된다. 클래스 로더는 JRE의 일부로써 **런타임에 클래스를 동적으로 JVM에 로드 하는 역할을 수행**하는 모듈이다. 자바의 클래스들은 자바 프로세스가 새로 초기화되면 클래스로더가 차례차례 로딩되며 작동한다.


[**발생할 수 있는 문제]**


클래스 로더가 여러 개라면, 각기 다른 싱글톤 인스턴스를 1개씩 소유할 수 있다. 이때 문제가 발생하므로, 클래스 로더를 지정하거나 이를 방지해야 한다.

1. 직렬화와 리플렉션

[https://inpa.tistory.com/entry/JAVA-☕-싱글톤-객체-깨뜨리는-방법-역직렬화-리플렉션#역직렬화로_깨지는_싱글톤](https://inpa.tistory.com/entry/JAVA-%E2%98%95-%EC%8B%B1%EA%B8%80%ED%86%A4-%EA%B0%9D%EC%B2%B4-%EA%B9%A8%EB%9C%A8%EB%A6%AC%EB%8A%94-%EB%B0%A9%EB%B2%95-%EC%97%AD%EC%A7%81%EB%A0%AC%ED%99%94-%EB%A6%AC%ED%94%8C%EB%A0%89%EC%85%98#%EC%97%AD%EC%A7%81%EB%A0%AC%ED%99%94%EB%A1%9C_%EA%B9%A8%EC%A7%80%EB%8A%94_%EC%8B%B1%EA%B8%80%ED%86%A4)에서 자세하게 확인할 수 있다.

1. loose coupling 원칙

싱글톤 방법을 사용하면, 싱글톤 클래스가 변경되면 관련된 모든 클래스를 변경해야 한다. 즉, 대부분의 클래스가 싱글톤 클래스의 내부를 알아야하며 이는 느슨한 결합 원칙에 위배된다. 

1. 전역변수와의 차이

동기화를 고려하여 생성하면, 병행성부분의 3번 혹은 2번 경우라면 전역변수와 다르게 한번이라도 사용해야 생성한다. 또한, 클래스이기에 접근제어자 등 여러 장점이 있다.


### 결과적으로?


그래서 나온 `Enum` 아래와 같이 enum으로 클래스를 지정한다. enum은 자바에서는 상수가 아닌 클래스처럼 사용할 수 있다. 또한, enum은 시작 시 한번만 초기화하기에 싱글톤 원칙을 지키며, atomic하게 처리되기에 멀티 쓰레드에서도 안전하다.


아래는 헤드퍼스트 관련 코드 예시이다. 


```java
public enum Singleton {
	UNIQUE_INSTANCE;
 
	// other useful fields here

	// other useful methods here
	public String getDescription() {
		return "I'm a thread safe Singleton!";
	}
}
```


아래와 같이 사용하면 된다.


```java
public class SingletonClient {
	public static void main(String[] args) {
		Singleton singleton = Singleton.UNIQUE_INSTANCE;
		System.out.println(singleton.getDescription());
	}
}
```

