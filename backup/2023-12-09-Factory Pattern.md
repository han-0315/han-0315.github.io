---
layout: post
title: Factory Pattern
date: 2023-12-09 21:10 +0900 
description: Factory Pattern
category: [디자인패턴, 생성패턴] 
tags: [생성패턴, 디자인패턴, 팩토리패턴] 
pin: false
math: true
mermaid: true
---
Factory Pattern 정리
<!--more-->
**[책에서 자주 언급하는 내용]**
- 구현을 바탕으로 프로그래밍한다. → `new Object()`
- 변화를 바탕으로 프로그래밍한다. → `GetObject()`


## 상황


객체를 생성하는 부분이 계속 변경될 때, 우리는 피자 클래스를 통해서 피자를 생성할 것인다. 하지만 피자의 종류는 많고, 판매하는 피자는 계속 **“변한다.”**  앞에서도 강조한 원칙은 변하는 부분과 변하지 않는 부분을 분리하는 것이다. 판매하는 피자가 변경되면 아래의 메소드는 계속 수정해야 한다. 


```java
Pizza orderPizza(string type){
	Pizza pizza;
	if(type.equals("cheese"))
		pizza=new CheesePizza();
	else if(type.equals("pepperoni"))
		pizza=new PepperoniPizza();
	else if(type.equals("clam")
		pizza=new ClamPizza();
	else if(type.equals("veggie"))
		pizza=new VeggiePizza();
}
...
```


만약, 피자가게가 여러개라면, 피자를 생성하는 클래스는 하나가 아닌 다수가 된다. 이때 여러 클래스를 수정해야한다. 이런 의존성을 없애기 위해 SimpleFactory 클래스를 도입한다.(즉, 객체를 생성하는 부분을 클래스로 분리함)


```java
public class SimpleFactory{
	public Pizza createPizza(String type){
		Pizza pizza = null;
		if(type.equals("cheese")){
			pizza = new CheesePizza();
		}else if(type.equals("pepperoni")){
			pizza = new PepperoniPizza();
		}else if(type.equals("clam")){
			pizza = new ClamPizza();
		}else if(type.equals("veggie")){
			pizza = new VeggiePizza();
		}
		return pizza;
	}
}
```


하지만 이런 SImple Factory는 디자인 패턴이 아니고, 구현 스킬정도이다.


## Factory Pattern 란?


여러 하위 클래스를 생성할 때, 각 클래스별로 의존성을 갖고 싶지 않을 때 주로 사용하는 패턴으로 **객체 생성의 책임을 클래스 또는 메서드에 위임한다.** 결과적으로 **객체 생성로직이 주요 비즈니스 로직과 분리된다.**


## When?


**문제상황: 하위 클래스를 생성하는 코드를 분리하고 싶을 때**

- 구체적인 코드 상황으로는 다른 클래스에서 `new instance_type` 명령어를 실행하고 싶지 않을 때

하위 클래스를 직접생성하면, 하위 클래스가 추가되거나, 변경, 삭제될 때마다 기존의 코드를 수정해야 하니 의존성이 생긴다. 이를 해결하기 위해, 팩토리라는 클래스를 통해 모든 생성을 위임한다.

- 주로, 팩토리 패턴으로 구현한 후(만약 더 유연해야 한다면) 추상화 패턴, 프로토타입, 빌더 패턴으로 업데이트한다.

구현을 통해 **Single Responsibility Principle을 만족한다.**


## Factory Method Pattern


### 정의


팩토리 메소드 패턴은 객체를 생성할 때 필요한 인터페이스를 클래스로 정의한다.(간단한 구성이 아니라면 abstract method를 사용한다.) 인스턴스를 만드는 것은 하위 클래스에서 결정한다.


### 상황


위의 상황에서 언급했던 것 처럼, 적용되는 클래스가 하나가 아니고, 생성하는 방식이 다르다면 위와 같은 방법으로 문제를 해결하기 어렵다. 만약 중국, 한국, 미국 지점이 있다고 가정하면 PizzaStore는 Factory로 중국 미국 한국 지점을 모두 가지고 있어야한다. 불필요한 구성이다. 그렇기에 Method Pattern을 이용한다. 

1. 상위 클래스에서 인스턴스를 생성하는 부분을 `abstrct method()`로 정의한다.

	```java
	public abstract class PizzaStore {
	 
		protected abstract Pizza createPizza(String item);
	 
		public Pizza orderPizza(String type) {
			Pizza pizza = createPizza(type);
			System.out.println("--- Making a " + pizza.getName() + " ---");
			pizza.prepare();
			pizza.bake();
			pizza.cut();
			pizza.box();
			return pizza;
		}
	}
	```

2. 하위 클래스에서 이를 오버라이딩하여, 알맞은 피자를 생성한다.

	```java
	public class NYPizzaStore extends PizzaStore {
	
		Pizza createPizza(String item) {
			if (item.equals("cheese")) {
				return new NYStyleCheesePizza();
			} else if (item.equals("veggie")) {
				return new NYStyleVeggiePizza();
			} else if (item.equals("clam")) {
				return new NYStyleClamPizza();
			} else if (item.equals("pepperoni")) {
				return new NYStylePepperoniPizza();
			} else return null;
		}
	}
	```


이렇게하면, 똑같은 시스템(`orderPizza` 등)를 이용하면서 동적으로 알맞은 피자를 생성할 수 있다, 추후 다른 피자가 추가된다면 해당 피자클래스를 생성하고 Factory에 추가하면된다.




### 의존성 역전


만약, 팩토리 패턴을 사용하지 않고 초기 상황처럼 PizzaStore 클래스에서 객체를 생성한다면, 여러 Pizza 종류의 의존성을 갖게된다. 각 종류의 Pizza는 구성클래스이지만, Pizza가 변한다면 PizzaStore의 코드도 변경해야 한다. 즉, PizzaStore는 여러 종류의 Pizza의 의존성을 가진다. 즉, 구현에 의존하게 된다. 


의존성 역전원칙은 “구현에 의존하지 않고, 추상화된 것에 의존한다”라는 원칙이다. 


```java
Pizza orderPizza(string type){
	Pizza pizza;
	if(type.equals("cheese"))
		pizza=new CheesePizza();
	else if(type.equals("pepperoni"))
		pizza=new PepperoniPizza();
	else if(type.equals("clam")
		pizza=new ClamPizza();
	else if(type.equals("veggie"))
		pizza=new VeggiePizza();
}
...
```


팩토리 메서드 패턴을 적용하면, 각 피자 클래스가 아닌 Pizza라는 abstract class에 의존하게 된다. 


![](/assets/img/post/Factory%20Pattern/5.png)


의존성 역전 원칙 지키는 법

1. 변수에 구상 클래스의 레퍼런스를 저장하지 않는다.(new 연산자)
2. 구상 클래스에서 유도된 클래스를 만들지 않는다.
3. 베이스 클래스에 이미 구현되어 있는 메소드를 오버라이드하지 않는다. (베이스 클래스가 제대로된 추상화되지 않기에)

위의 원칙을 지키는 Java 프로그램은 거의 없다. 다만, 앞으로 지속적인 변화가 필요하다면 패턴을 준수해야 유지보수에 편하다.


### 단점


팩토리 메소드를 이용하면 형식 안정성(type - safety)에 지장이 있을 수 있다. 위의 예에서도 string 입력되는 인자가 잘못전달되면 null을 반환하게 된다. 

- 위의 해결방법으로는 형식을 고정한다. 정적 상수 혹은 enum, 매개변수를 나타내는 특별한 객체를 통해 해결할 수 있다.

## 추상 팩토리 패턴


추상 팩토리 패턴은 구상 클래스에 의존하지 않고도 서로 연관되거나 의존적인 객체로 이루어진 제품군을 생성하는 인터페이스를 제공한다. 일련된 이라는 말에서, 팩토리 메서드와 다르다. 팩토리 메서드는 하나의 인스턴스만 생성하는 반면, 추상 팩토리 패턴으로는 일련된(연관된)인스턴스를 모두 생성할 수 있다.


### 상황


앞서 피자를 제작했을 때, 이제 각 지점마다 재료를 달리 사용한다고 한다. 종류는 아래와 같이 총 6개가 있다.


```java
  Dough dough;
	Sauce sauce;
	Veggies veggies[];
	Cheese cheese;
	Pepperoni pepperoni;
	Clams clam;
```


이럴 경우, 각 지점에서 피자를 생성할 때 팩토리를 이용하여 재료를 설정해줘야한다.


아래는 **뉴욕 피자 재료 팩토리이다.**


```java
public class NYPizzaIngredientFactory implements PizzaIngredientFactory {
 
	public Dough createDough() {
		return new ThinCrustDough();
	}
 
	public Sauce createSauce() {
		return new MarinaraSauce();
	}
 
	public Cheese createCheese() {
		return new ReggianoCheese();
	}
 
	public Veggies[] createVeggies() {
		Veggies veggies[] = { new Garlic(), new Onion(), new Mushroom(), new RedPepper() };
		return veggies;
	}
 
	public Pepperoni createPepperoni() {
		return new SlicedPepperoni();
	}

	public Clams createClam() {
		return new FreshClams();
	}
}
```


이제 피자 생성과정을 천천히 살펴보면 

1. 뉴욕 피자 가게에서 치즈 피자를 주문한다. `NYPizzaStore.orderPizza(”cheese”);`
2. orderPizza 매소드 내부에서 createPizza(”cheese”)를 호출한다.
3. 처음에 지정된 원재료 팩토리를 통해, 치즈피자가 생성된다.

	```java
	Pizza pizza = new CheesePizza(ingredientFactory);
	```

	- cheese 피자 클래스 일부

	```java
	void prepare() {
			System.out.println("Preparing " + name);
			dough = ingredientFactory.createDough();
			sauce = ingredientFactory.createSauce();
			cheese = ingredientFactory.createCheese();
		}
	```



### 단점


새로운 제품을 추가하거나, 새로운 도핑 추가와 같이 내부를 변경하면 인터페이스를 전면 수정해야 한다. 즉, 서브 클래스의 인터페이스도 전부 수정해야 한다. 그렇기에 새로운 요소가 잘 추가되지 않고, 만드는 방식이 여러개 필요한 경우 유용하다. 즉, 변화의 요소가 재료가 아닌 피자인 경우 유용하다.


## 팩토리 메서드 vs 추상 팩토리


사용방식

- 상속 및 오버라이딩, 즉 서브(하위) 클래스에서 생성을 담당
- 객체 구성(factory를 멤버변수로), 팩토리에서 생성을 담당

 

