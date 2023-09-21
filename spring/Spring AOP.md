# Spring AOP

### AOP란?
Aspect Oriented Programming(관점 지향 프로그래밍)이며, 횡단 관심사의 분리를 허용함으로써
모듈성을 증가시키는것이 목적인 프로그래밍 패러다임이다.

여러 객체에 공통으로 적용할 수 있는 기능을 분리해서 개발자는 반복 작업을 줄이고 핵심 기능 개발에만 집중할 수 있다.

프록시를 이용하여 클라이언트가 타겟에게 날린 요청을 가로채어, 부가 기능을 수행한다.
-> 이를 통해 타겟의 코드를 변경하지 않고도 부가 기능을 쉽게 적용할 수 있다.

### AOP의 용어들
* Aspect : AOP의 기본 모듈, 부가 기능을 정의한 Advice와 해당 기능을 어디 적용할지 결정하는 PointCut을 함께 가짐.
* Advice : 타깃에게 제공할 부가 기능을 담은 모듈. 타깃이 필요없는 순수한 부가 기능
* Join Point : 프로그램 실행 내부에서 Advice가 적용될 수 있는 위치
* Pointcut : Advice에 적용할 JoinPoint를 선별하는 작업, 그 기능을 정의한 모듈
  * 예시에서는 '주로 메서드 이름이 ~로 시작되는 경우' 포인트컷을 지정하였음.

![image](https://github.com/wonjunYou/WIL/assets/59856002/809045c4-b13c-45f5-bf16-2e55cabae022)

### Spring AOP
* 만약 컴파일 시점, 클래스 로딩 시점(바이트 코드)에 공통 기능을 삽입한다면?
  * -> AspectJ 컴파일러나 AspectJ 클래스 로더 조작기 등을 도입해야 함. -> 부가적인 의존성 추가
* **런타임 시점에 프록시 객체를 생성하여 공통 기능을 삽입한다.**

스프링은 **런타임 시점**에 프록시 객체를 사용하여 공통 기능을 삽입한다.
타겟 클래스를 Bean으로 만들 때, 타겟 클래스 타입의 프록시 빈을 감싸 생성한 후에, 
해당 프록시 빈이 클래스 중간에 코드를 추가하여 삽입한다.

* 프록시는 메서드 오버라이딩 개념으로 동작하므로, 스프링 AOP는 메서드 실행 시점에만 AOP를 적용할 수 있음.
* 스프링 AOP는 스프링 컨테이너가 관리할 수 있는 빈에만 AOP를 적용할 수 있음.

### Spring AOP 동작 과정

![image](https://github.com/wonjunYou/WIL/assets/59856002/ebdd56f2-7a65-4a09-add0-47f58cf60274)

* BeanPostProcessor(빈 후처리기) : 생성된 빈 객체를 스프링 컨테이너에 등록하기 전에 조작하는 객체

1. 빈 객체를 생성한 뒤 빈 후처리기에게 생성된 빈을 전달한다.
2. 어드바이저 내의 포인트컷을 이용해 전달받은 빈이 프록시 적용 대상인지 확인한다.
3. 프록시 생성 대상 빈들을 대상으로 프록시 객체 생성
4. 프록시를 생성한 빈이라면 프록시 객체를, 아니라면 그냥 빈을 반환한다.
5. 빈 후처리기에게 전달받은 객체를 IoC 컨테이너에게 빈 등록을 한다.

### Spring AOP는 왜 Proxy 방식을 사용하는가?
Aspect 클래스 기능을 사용하려면 직접 Aspect 클래스를 호출해야 한다.
이때 Proxy가 없다면 Target 클래스 내에 부가 기능을 호출하는 로직이 포함되기 때문에
AOP를 적용하지 않았을 때와 같은 문제가 발생한다. 여러 군데에서 Aspect를 반복 호출하기 때문이다.

따라서 Spring에서는 Target 클래스나 상위 인터페이스를 상속하는 프록시 클래스를 생성하고
프록시 클래스에서 부가 기능에 관련 처리를 한다.

### AOP는 public 메서드에서만 적용 가능하다.
Spring에서 AOP를 사용하는 대표적인 예시로 `@Transactional` 이 있다. 
`@Transactional` 을 배울 때, 항상 private 메서드에는 사용 불가능하다고 나와 있다. 왜 그럴까?
Spring AOP에서는 현재 CGLIB Proxy를 사용하고, 이는 상속을 통해 프록시를 생성한다.
private 메서드는 상속이 불가능하므로, 프록시를 만들 수 없다.

그렇다면, `protected` 메서드에는 트랜잭션이 정상 동작할까?
JDK Dynamic proxy는 인터페이스 기반으로 동작한다. 인터페이스에서 `protected` 메서드는 사용할 수 없으므로, 정상 동작하지 않는다.

하지만, Java 9에서는 Interface에 `default` 메서드를 허용했는데, 그렇다면 JDK Dynamic Proxy를 이용하여 Spring AOP를 구현할 수 있지 않을까?
-> 인터페이스에서 default 메서드는 주로 인터페이스에 새로운 기능 추가시, 기존 코드와의 호환성을 위해 도입되었다. 
그러므로 비즈니스 로직을 처리하는 public 메서드가 주로 AOP의 대상이 되게 된다.
또한, interface 내 default 메서드는 오버라이딩 여부가 선택적이다. 다시 말해, 해당 인터페이스의 default 메서드가 재정의되었는지 여부를 알 수 없다.

이러한 이유 때문에 Spring에서는 일관된 AOP 적용을 위해 오직 **오직 public 메서드**에서만 허락한다.

### 같은 클래스 내에서 트랜잭션이 걸린 메서드를 호출하면 트랜잭션이 작동하지 않는다.
```java
public class Test {
    public void start() {
        do();
    }
    
    @Transactional
    public void do() {
    }
}
```

해당 코드에서는 @Transactional이 동작하지 않는다.

Spring AOP에서 Proxy의 동작 과정을 보면, 프록시를 통해 들어오는 외부 메서드 호출을 인터셉트한다.
이 때문에 `self-invocation`이라는 현상이 발생한다. 이는 프록시의 내부 빈에서 프록시를 호출했다는 뜻이다.

하지만, 내부 메서드에서 호출했을 때 AOP 적용이 완전 불가능한 것은 아니다.
1. Spring AOP 대신 컴파일 타임 위빙을 하는 AspectJ를 사용한다.
2. 자기 자신을 Bean으로 등록하여 호출 흐름을 외부에서 내부로 가져가도록 한다.

### 연관 공부 키워드
* IOC/DI컨테이너
* JDK Dynamic Proxy & CGLIB Proxy
* Decorator Pattern, Proxy Pattern
* 자동 Proxy 생성 기법
* 빈 오브젝트의 후처리 조작 기법

### Reference
* https://www.youtube.com/watch?v=hjDSKhyYK14
* https://www.youtube.com/watch?v=7BNS6wtcbY8
* https://tecoble.techcourse.co.kr/post/2022-11-07-transaction-aop-fact-and-misconception/