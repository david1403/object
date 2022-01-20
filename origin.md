## 9. 유연한 설계

### 개방 폐쇄 원칙
> Open-Closed Principle 소프트웨어 개체는 확장에 대해 열려있어야 하고, 수정에 대해서는 닫혀있어야 한다.

* 확장에 대해 열려 있다: 어플리케이션의 요구사항이 변경될 때 이 변경에 맞게 새로운 동작을 추가해서, 어플리케이션의 기능을 확장할 수 있다. 
* 수정에 대해 닫혀 있다: 기존의 코드를 수정하지 않고도 어플리케이션의 동작을 추가하거나 변경할 수 있다

기존의 Movie, DiscountPolicy를 수정하지 않고, NoneDiscountPolicy를 추가하는 것만으로 할인 정책이 적용되지 않는 영화를 구현하는 것이 그 예이다.

이것이 가능한 이유는 추상화 때문이다. 추상화란, 핵심적인 부분만 남기고 불필요한 부분은 생략함으로써 복잡성을 극복하는 기법이다.

```java
public abstract class DiscountPolicy {
    private List<DiscountCondition> conditions = new ArrayList<>();

    public DiscountPolicy(DiscountCondition ... conditions) {
        this.conditions = Arrays.asList(conditions);
    }

    public Money calculateDiscountAmount(Screening screening) {
        for(DiscountCondition each : conditions) {
            if (each.isSatisfiedBy(screening)) {
                return getDiscountAmount(screening);
            }
        }

        return screening.getMovieFee();
    }

    abstract protected Money getDiscountAmount(Screening Screening);
}
```

* 변하지 않는 부분: 할인 여부를 판단하는 로직
* 변하는 부분: 할인된 요금을 계산하는 방법

상속을 통해 생략된 부분을 구체화함으로써 할인 정책을 확장할 수 있게 된다.
```java
public class Movie {
    ...
    private DiscountPolicy discountPolicy;

    public Movie(String title, Duration runningTime, Money fee, DiscountPolicy discountPolicy){
        ...
        this.discountPolicy = discountPolicy;
    }

    public Money calculateMovieFee(Screening screening) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```
Movie 역시 추상화한 DiscountPolicy에 대해서만 의존하고 있기 때문에 DiscountPolicy의 자식 클래스를 추가하더라도 Movie에는 영향이 없고 수정에 닫혀 있게 된다.




### 생성 사용 분리

Movie가 다음과 같이 구현이 되어있다면 어떻게 될까?

```java
public class Movie {
    ...
    private DiscountPolicy discountPolicy;

    public Movie(String title, Duration runningTime, Money fee){
        ...
        this.discountPolicy = new AmountDiscountPolicy(...); //
    }

    public Money calculateMovieFee(Screening screening) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```
할인 정책을 변경하기 위해서는 새로운 PercentDiscountPolicy 인스턴스를 생성하도록 직접 코드를 수정하는 방법밖에 없다. 이는 동작을 추가하거나 변경하기 위해 기존의 코드를 수정하도록 만들기 때문에 개방-폐쇄 원칙을 위반한다.
객체 생성을 피할 수는 없고, 어딘가에서는 필요하지만, Movie의 문제는 생성자 안에서 DiscountPolicy의 인스턴스를 생성하고 있다는 점이다.

> 생성-사용 분리 원칙: 소프트웨어 시스템은 의존성을 서로 연결하는 시작 단계와, 실행 단계를 분리해야 한다.

```java
public class Client {
    public Money getAvatarFee() {
        Movie avatar = new Movie("아바타",
                Duration.ofMinutes(120),
                Money.wons(10000),
                new AmountDiscountPolicy(
                    Money.wons(800),
                    new SequenceCondition(1),
                    new SequenceCondition(10)));
        return avatar.getFee();
    }
}
```
Client 클래스가 객체의 생성과 동시에 getFee 메시지도 함께 전송하고 있기 때문에, 할인 정책이 변경된다면 'fee를 가져오는 로직'이 변하지 않더라도, Client 코드가 수정되어야 한다. 이를 방지하기 위해서
객체의 생성만을 전담하는 Factory 클래스를 만들 수 있다.

```java
public class Factory {
    public Movie createAvatarMovie() {
        return new Movie("아바타",
                ...
                new AmountDiscountPolicy(...)

    }
}

public class Client {
    private Factory factory;

    public Client(Factory factory) {
        this.factory = factory;
    }

    public Money getAvatarFee() {
        Movie avatar = factory.createAvatarMovie();
        return avatar.getFee();
    }
}
```
이와 같은 Factory 클래스는 도메인 개념을 표현한 객체가 아니라 설계자가 편의를 위해 임의로 만들어낸 가공의 객체이다. 이런 객체를 Pure Fabrication (순수한 가공물)이라고 부른다. 이러한 점 때문에 객체지향이
실제 세계를 항상 모방한다는 주장은 거짓이다.

### 의존성 주입

생성과 사용을 분리하면 Movie에는 인스턴스를 사용하는 책임만 남게 되고, 외부의 다른 객체가 Movie에게 생성된 인스턴스를 전달해야 한다는 것을 의미한다. 이처럼 사용하는 객체가 아닌 외부의 독립적인 객체가
인스턴스를 생성한 후, 이를 전달해서 의존성을 해결하는 방법을 의존성 주입이라고 부른다.

생성자 주입
```java
// 생성자 주입
public Movie(String title, Duration runningTime, Money fee, DiscountPolicy discountPolicy) {
    this.title = title;
    this.runningTime = runningTime;
    this.fee = fee;
    this.discountPolicy = discountPolicy;
}

// setter 주입
avatar.setDiscountPolicy(new AmountDiscountPolicy(...))

//메서드 주입
avatar.calculateDiscountAmount(screening, new AmountDiscountPolicy(...))
   
```
의존성 주입 이외에, 의존성을 해결하는 방법 중 하나로 Service Locator 패턴이 있다. 특정 객체가 의존성을 해결해 주는 패턴을 의미한다.
```java
public class ServiceLocator {
    private static ServiceLocator soleInstance = new ServiceLocator();
    private DiscountPolicy discountPolicy;

    public static DiscountPolicy discountPolicy() {
        return soleInstance.discountPolicy;
    }

    public static void provide(DiscountPolicy discountPolicy) {
        soleInstance.discountPolicy = discountPolicy;
    }

    private ServiceLocator() {
    }
}

public Movie(String title, Duration runningTime, Money fee) {
    this.title = title;
    this.runningTime = runningTime;
    this.fee = fee;
    this.discountPolicy = ServiceLocator.discountPolicy();
}

ServiceLocator.provide(new AmountDiscountPolicy(...));
Movie avatar = new Movie(...);
```

위 패턴의 가장 큰 단점은 의존성을 감춘다는 것이다. ServiceLocator.provide 없이 Movie 객체를 생성했다가는 discountPolicy 객체가 초기화되지 않아서 NPE가 발생할 것이다.
이런 문제는 의존성과 관련된 문제가 컴파일 시점이 아닌 런타임에 발견되기 때문에 발생한다.

이러한 코드는 단위 테스트 작성도 어렵다. 내부적으로 static 변수를 사용하기 때문에, 모든 단위테스트 케이스에서 ServiceLocator의 상태를 공유하게 되고, 각 단위테스트가 서로 고립되어야 한다는 원칙을 위배한다.

### 의존성과 패키지

패키지 단위로 컴파일과 배포를 하고자 하는 경우에 의존성에 따라서 패키징을 재조정하는 것이 좋을 때도 있다.
1. Movie / DiscountPolicy, AmountDiscountPolicy, PercentDiscountPolicy
2. Movie, DiscountPolicy / AmountDiscountPolicy, PercentDiscountPolicy

1번처럼 패키징하면 Movie가 컴파일되기 위해서 구체클래스의 변화가 없더라도 3개의 클래스 모두가 컴파일되어야 한다.
그러나 2번처럼 패키징하면, Movie를 다른 컨텍스트로부터 완전히 분리시킬 수 있다. 
