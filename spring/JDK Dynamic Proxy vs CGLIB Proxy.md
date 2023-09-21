# JDK Dynamic Proxy와 CGLIB Proxy

## Proxy란?
사전적 의미로 `대리`라는 의미를 가진다.

자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서
클라이언트의 요청을 받아주는 대상(대리인, 대리자)을 말한다.

`Client` -> `Proxy` -> `Target`

스프링 AOP에서의 프록시란, 원본 객체를 감싼 새로운 객체로 부가 기능을 추가할 때 사용하는 패턴
-> 중간에 부가 기능을 추가할 때 사용한다.

### Proxy는 왜 사용하는가?
1. JPA의 Lazy Loading처럼 클라이언트가 타깃에 접근하는 방법을 제어
2. 타깃에 부가적인 기능을 제공한다.
    * 코드의 실행 시간 측정 등

### Proxy Pattern이란?
특정 객체에 대한 접근을 제어할 때 사용하는 패턴이다.
* 클라이언트가 객체를 직접 호출하지 않고 프록시로 호출하여 제어권을 갖도록 하는 패턴
* 인터페이스 타입 기반의 클라이언트 요청은 프록시 객체로 먼저 들어오게 된다.
* Client가 상위 Interface를 가리키게 하고, 실제 구현체로 프록시 객체를 사용하도록 한다.
* SOLID의 OCP, SRP를 지킬 수 있다.

### Decorator Pattern이란?
타깃에 부가적인 기능을 부여해 줄 때 사용하는 패턴이다.

## JDK Dynamic Proxy
JDK Proxy라고도 부른다.
* 프록시 클래스를 직접 구현하지 않아도 된다.
* Invocation Handler를 통해 중복 코드를 제거할 수 있다.
  * InvocationHandler에서 재정의한 `InvocationHandler.invoke()`를 통해 부가기능을 추가
  * 이를 타겟에게 다시 반환하는 형태로 해결할 수 있다. 
![img.png](img.png)

### JDK Dynamic Proxy의 특징
1. JDK에서 지원하는 프록시 생성 방법
2. Java Reflection API 사용
3. **인터페이스**가 반드시 있어야 함.
  * 내부 인터페이스를 필드로 갖는다.
4. `InvocationHandler.invoke()`를 구현해야 부가기능을 구현할 수 있다.
  
### JDK Dynamic Proxy는 리플렉션 API를 사용한다.
구체적인 클래스 타입을 알지 못해도 **런타임**에 클래스의 정보에 접근할 수 있게 해주는 자바 API
하지만, Reflection은 동적일 때 해결되는 타입을 포함하므로 JVM의 Optimization(최적화)가 적용되지 않아 느림.

![img_1.png](img_1.png)
- 클라이언트의 요청이 들어오면 JDK Dynamic Proxy는 실질적인 메서드 처리를 Invocation Handler에게 위임한다.
- Invocation Handler는 부가 기능을 수행하며 다시 타겟에게 위임한다.

- InvocationHandler의 단점
  - 실제 Target 객체를 갖고 있어야 한다.(Target에 의존적이다.)
  - 같은 기능을 제공하는 InvocationHandler이더라도 의존 객체에 따라 매번 Bean을 새롭게 생성해야 한다는 단점이 있다.

## 동적 프록시
![img_2.png](img_2.png)
클라이언트가 메서드 호출 시, proxy factory bean에서 인터페이스 유무를 체크한다.
있다면, JDK Dynamic Proxy를 사용하고, 없다면 CGLIB Proxy를 사용한다.

## CGLIB(CodeGeneratorLibrary) Proxy
Spring 4.3, Springboot 1.4부터 Default로 CGLIB를 사용하게 되었다.
특징
* 상속을 통한 프록시 구현 
  * 따라서 메서드에 final을 붙이면 사용 불가
* 바이트 코드를 조작하여 프록시 생성
* MethodInterceptor를 재정의한 intercept를 구현해야 부가기능 추가 가능
* 인터페이스에도 강제로 적용할 수 있고, 이때는 클래스에도 프록시를 적용시켜야 한다.
* 메서드에 final을 붙이면 overriding이 불가능하다.
* Enhancer 의존성을 사용하며, 기본 생성자를 필요로 한다.
* 타겟의 생성자를 두번 호출한다.

성능
* 메서드 초기 호출 시, 타겟 클래스의 Bytecode를 조사하여 이후 호출 시 조작된 바이트코드를 사용하는 형태.

![img_3.png](img_3.png)

## ProxyFactoryBean
ProxyFactoryBean이란, 하나의 프록시를 Bean으로 만들어주는 객체이다.

* 스프링에서 지원하는 프록시 생성 방법이다.
* MethodInterceptor를 재정의한 invoke를 구현해줘야 부가기능이 구현된다.
* 인터페이스가 반드시 필요한 것은 아니다.
![img_4.png](img_4.png)

* ProxyFactoryBean이 Proxy를 생성하면, 부가기능은 MethodInterceptor가 처리한다.
* MethodInterceptor가 타겟을 가지지 않음으로서, 부가기능을 독립적으로 유지할 수 있게 된다.

한계점
* 결국 ProxyFactoryBean도 매번 생성해 주어야 한다.
* @Transactioanl을 사용할 때를 생각하면 우리가 코드 생성한 적이 없는데?
  * Advice, Pointcut, 자동 프록시 생성기
  * 스프링이 어떻게 AOP를 활용해서 Declarative Transaction을 구현하는가?