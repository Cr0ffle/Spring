# #5 AOP와 LTW

- 스프링은 AspectJ라는 뛰어난 AOP 프레임워크로부터 포인트컷 표현식과 함께 애너테이션을 이용해 AOP 모듈을 개발하는 방법을 도입했다.

## 애스펙트 AOP

### 프록시 기반 AOP

- AOP는 모듈화된 부가기능과 적용 대상의 조합을 통해 여러 오브젝트에 산재해서 나타나는 공통적인 기능을 손쉽게 개발하고 관리할 수 있는 기술이다.
- 스프링은 자바 JDK 에서 지원하는 다이내믹 프록시 기술을 이용해 복잡한 빌드 과정이나 바이트코드 조작 기술 없이도 유용한 AOP를 적용할 수 있는 프록시 기반 AOP 개발 기능을 제공한다.
- 프록시 방식의 AOP는 객체지향 디자인 패턴의 데코레이터 패턴 또는 프록시 패턴을 응용해서 기존 코드에 영향을 주지 않은 채로 부가기능을 타깃 오브젝트에 제공할 수 있는 객체지향 프로그래밍 모델에서 출발한다.
- 포인트컷이라는 적용 대상 선택 기법과 자동 프록시 생성이라는 적용 기법까지 적용하면 AOP라고 부를 수 있는 효과적인 부가기능 모듈화가 가능해진다.



#### 프록시 기반 AOP 개발 스타일의 종류와 특징

- 스프링에서 AOP를 구현하는 방법
  - 스프링 AOP의 어드바이저 인터페이스를 구현하는 방법
  - AspectJ의 애스펙트라는 개념을 사용해서 애너테이션을 사용하는 방법

#### 자동 프록시 생성기와 프록시 빈

- 스프링 AOP를 사용한다면 어떤 개발 방식을 적용하든 모두 프록시 방식의 AOP가 적용된다.
- 스프링의 프록시 개념은 데코레이터 패턴에서 나온것이고, 동작원리는 JDK 다이내믹 프록시와 DI를 이용한다.
- `Client` -> `Proxy` -> `Target` 순서로 의존관계를 맺게 하면 `Proxy`가 `Client`와 `Target` 호출 과정에 끼어들어서 부가기능을 제공할 수 있게 된다.
- `Proxy`도 빈이 되고 `Target`도 빈이되면 같은 타입이라는 가정 하에 `@Autowired` 를 사용할 수 없게 되지만(컨테이너에 같은 타입이 두 개 등록되기 때문에) 스프링은 자동 프록시 생성기를 이용해서 컨테이너 초기화중에 만들어진 빈을 바꿔치기해서 프록시빈을 자동으로 등록해준다.
- 빈을 선택하는 로직은 포인트컷을 이용하면 되고 xml이나 애너테이션을 통해 이미 정의된 빈의 의존관계를 바꿔치기 하는 것은 빈 후처리기를 사용한다.
- 자동 프록시 생성기가 만들어주는 프록시는 새로운 빈으로 추가되는 것이 아니라 AOP 타깃 빈을 대체한다. 

![8](./images/toby-spring-vol2/8.png)

- 등록된 빈의 관점으로 볼때 자동 프록시 생성기 방식에서는 `Target` 오브젝트가 빈으로 직접 노출되지 않는다.
- AOP 자동 프록시 생성기방식의 특징들

`AOP 적용은 @Autowired의 타입에 의한 의존관계 설정에 문제를 일으키지 않는다.`

- 직접 타겟 오브젝트의 인터페이스를 구현한 프록시를 적용한 경우에는 클라이언트에서 `@Autowired`를 사용할 수 없지만 자동 프록시 생성 AOP 방식에는 `@Autowired`를 사용할 수 있다.

`AOP 적용은 다른 빈들이 Target 오브젝트에 직접 의존하지 못하게 한다.`

- 타겟 오브젝트가 빈으로 등록되지 않기 때문에 클라이언트에서 아래와 같이 타겟 오브젝트에 직접적인 접근이 불가능해진다. (인터페이스 방식으로 자동프록시를 생성하는 경우)

```java
public class Client {
	@Autowired Target target;
}
```

- Target을 직접 참조한다면 아래와 같은 에러메시지를 확인할 수 있다.

```
The bean 'fooImpl' could not be injected because it is a JDK dynamic proxy

The bean is of type 'com.sun.proxy.$Proxy76' and implements:
	com.example.springmvcexam.service.Foo
	org.springframework.aop.SpringProxy
	org.springframework.aop.framework.Advised
	org.springframework.core.DecoratingProxy

Expected a bean of type 'com.example.springmvcexam.service.FooImpl' which implements:
	com.example.springmvcexam.service.Foo


Action:

Consider injecting the bean as one of its interfaces or forcing the use of CGLib-based proxies by setting proxyTargetClass=true on @EnableAsync and/or @EnableCaching.

Disconnected from the target VM, address: '127.0.0.1:57768', transport: 'socket'

Process finished with exit code 1
```

- 위와 같은 문제가 발생할 수 있기 때문에 가능하다면 인터페이스를 사용해서 DI 관계를 유연하게 설정해줘야한다.



#### 프록시의 종류

- 스프링에서는 클래스를 직접 참조하면서 강한 의존관계를 맺고 있더라도 cglib 라이브러리를 통해서 프록시를 적용할 수 있다.
- 타깃 클래스를 상속한 서브클래스를 만들어서 이를 프록시를 활용하는 방식이다.
- 이 방식은 final 클래스와 final 메서드는 상속이 불가능하다는 제약과 같은 클래스 타입의 빈이 두 개 만들어지기 때문에 생성자가 두번 호출된다. 추가로 cglib이라는 바이트코드 생성 라이브러리가 필요하다. 
- Spring 3.2 에서 cglib은 스프링 프로젝트에 흡수되었다. 따라서 별도의 라이브러리 설정은 필요하지 않다. [https://github.com/spring-projects/spring-framework/commit/92500ab9023ae2afd096be9c014423fcd4180c55#diff-dd90aa36a7221a652e04631ac19dcb6693a9c841e43a90d36cc3caa51121630e](https://github.com/spring-projects/spring-framework/commit/92500ab9023ae2afd096be9c014423fcd4180c55#diff-dd90aa36a7221a652e04631ac19dcb6693a9c841e43a90d36cc3caa51121630e)
- 스프링은 타깃 오브젝트에 인터페이스가 있다면 그 인터페이스를 구현한 JDK 다이내믹 프록시를 만들어주지만, 인터페이스가 없다면 cglib을 이용한 클래스 프록시를 만든다.
- 설정을 통해 강제로 프록시 클래스를 만들게 할 수도 있다.

```java
@EnableAspectJAutoProxy(proxyTargetClass = true) // proxyTargetClass 를 true로 설정
```

- 위와 같이 적용한다면 인터페이스 존재와 상관없이 클래스 프록시가 만들어진다.
- Spring Boot 2.x 기준으로 proxyTargetClass 설정은 기본값이 true로 되어있다.

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(Advice.class)
	static class AspectJAutoProxyingConfiguration {

		@Configuration(proxyBeanMethods = false)
		@EnableAspectJAutoProxy(proxyTargetClass = false)
		@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false")
		static class JdkDynamicAutoProxyConfiguration {

		}

		@Configuration(proxyBeanMethods = false)
		@EnableAspectJAutoProxy(proxyTargetClass = true)
		@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true",
				matchIfMissing = true)
		static class CglibAutoProxyConfiguration {

		}

	}
  
  ...
    
}
```

- `@ConditionalOnProperty` 을 확인해보면 `spring.aop.proxy-target-class` 값이 true인 경우 `CglibAutoProxyConfiguration` 을 빈으로 등록한다. [https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-autoconfigure/src/main/resources/META-INF/additional-spring-configuration-metadata.json](https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-autoconfigure/src/main/resources/META-INF/additional-spring-configuration-metadata.json)
- `spring.aop.proxy-target-class`을 재정의해서 설정을 변경할 수 있다.

### @AspectJ AOP

- 애스펙트는 애플리케이션의 도메인 로직을 담은 핵심 기능은 아니지만 많은 오브젝트에 걸쳐서 필요한 부가기능을 추상화해놓은 것이다.
- 스프링의 어드바이저는 하나의 포인트컷과 하나의 어드바이스로 정의된 가장 단순한 형태의 애스펙트다.
- 독립적인 빈의 조합으로 만들어지는 어드바이저와 달리 애스펙트는 다양한 조합을 갖는 포인트컷과 어드바이스를 하나의 모듈로 정의할 수 있다.

#### @AspectJ를 이용하기 위한 준비사항

- xml방식인 경우 아래와 같이 정의해두면 클래스레벨에 `@Aspect` 가 붙은 것을 모두 애스펙트로 자동 등록해준다.

```xml
<aop:aspectj-autoproxy/>
```

- 애너테이션방식인 경우 위에서 확인했던 `@EnableAspectJAutoProxy` 를 사용하면 된다.

#### @Aspect 클래스와 구성요소

- 애스펙트는 자바 클래스에 `@Aspect`라는 애너테이션을 붙여서 만든다.

```java
@Aspejct
public class SimpleMonitoringAspect {
	...
}
```

- 이 클래스를 애스펙트로 사용하려면 빈으로 등록해야한다.
- `@Aspect` 클래스에는 포인트컷과 어드바이스를 정의할 수 있다. 두가지 모두 애너테이션이 달린 메서드를 사용해 정의한다.

`포인트컷: @Pointcut`

- 메서드에 `@Pointcut`을 사용해 포인트컷 표현식을 넣어서 정의한다.
- 하나의 `@Aspect` 클래스 안에 여러 개의 포인트컷을 선언할 수도 있다.

```java
@Pointcut("execution(* hello(..))")
private void all() { }
```



`어드바이스: @Before, @AfterReturning, @AfterThrowing, @After, @Around`

- 어드바이스도 포인트컷과 마찬가지로 애너테이션이 붙은 메서드를 이용해 정의한다.
- @AspectJ에서는 다섯가지 종류의 어드바이스를 사용할 수 있다.

- 어드바이스의 애너테이션에는 이에 적용할 포인트컷을 명시해야한다.
- 애스펙트에서는 어드바이저라는 이름을 따로 지정하지 않고 어드바이스에 포인트컷을 직접 지정해줄 수 있다.

```java
@Around("all()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
	...
	Object ret = pjp.proceed();
  ...
	return ret;
}
```

- 위 방법 대신 어드바이스 애너테이션에 직접 포인트컷 표현식을 넣어서 적용하는 방법도 있다. 이런 경우엔 포인트컷 이름이 없기 때문에 다른 어드바이스에서 참조될 수 없다.

```java
@Around("execution(* hello(..))")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
	...
	Object ret = pjp.proceed();
  ...
	return ret;
}
```

- `@Aspect` 클래스에는 일반 메서드, 일반 필드도 정의할 수 있고 상속과 인터페이스 구현 또는 DI를 통한 다른 빈의 참조도 모두 가능하다.







#### 포인트컷 메서드와 애너테이션

- 포인트컷 메서드에는 구현 코드는 필요 없고 항상 void 형이어야 한다.

```java
@Pointcut("execution(* hello(..))")
private void all() { }
```

- 포인트컷은 적용할 조인 포인트를 선별하는 것이다. 여기서 조인 포인트란 어드바이스로 정의된 부가기능을 적용할 수 있는 위치다. 스프링에서는 프록시 방식의 AOP를 사용하기 떄문에 조인 포인트는 메서드 실행 지점뿐이다. 조인포인트 = 메서드
- 포인트컷 표현식은 여러 종류의 포인트컷 지시자를 이용해 정의할 수 있다.

`execution()`

- 가장 대표적인 포인트컷 지시자.
- 접근제한자, 리턴 타입, 타입, 메서드, 파라미터 타입, 예외 타입 조건을 조합해서 메서드 단위까지 선택가능한 가장 정교한 포인트컷을 만들 수 있다.

`within()`

- `within()`은 타입 패턴만을 이용해 조인 포인트 메서드를 선택한다. 
- 타깃 클래스의 타입에만 적용되며 조인 포인트는 타깃 클래스 안에서 선언된 것만 선정된다.
- 선택된 타입의 모든 메서드가 AOP의 대상이 된다.
- 패턴을 이용할 수 있기 때문에 패키지나 모듈이 잘 설계 되어 있다면 계층별 포인트컷을 만들어 사용할 수 있다.

```java
@Pointcut(within("com.example.dao..*"))
private void daoLayer() {}

@Pointcut(within("com.example.service..*"))
private void serviceLayer() {}

@Pointcut(within("com.example.web..*"))
private void webLayer() {}
```

`this, target`

- `this`와 `target`은 하나의 타입을 지정하는 방식이다.
- `this`는 프록시 오브젝트의 타입을 비교해서 선정하고, `target`은 타깃 오브젝트의 타입을 비교한다.

```java
public class HelloImpl implements Hello {
...
}

// 프록시 빈 오브젝트의 타입을 지정하는 것이므로 HelloImpl은 포인트컷의 대상이됨
this("com.example.Hello")
  
// 프록시 빈 오브젝트는 타깃 오브젝트(HelloImpl) 의 타입이 아니므로 포인트컷 대상이 아님
this("com.example.HelloImpl")

```



`args`

- `args` 지시자는 메서드의 파라미터 타입만을 이용해 포인트컷을 설정할 때 사용한다.
- `args` 는 주로 다른 지시자와 함께 사용해서 파라미터 타입 조건을 추가하기도 한다.

```java
//파라미터가 없는 메서드를 선별
args()

//파라미터가 하나이고 String인 메서드를 선별
args(String)

//첫번째 파라미터가 String이고 파라미터의 개수는 하나 이상인 메서드를 선별
args(String,...)

//첫번째 파라미터가 String이고 파라미터가 n개인 메서드를 선별
args(String, *)
```

`@target, @within`

- `@target` 지시자는 타깃 오브젝트에 특정 애너테이션이 부여된것을 선정한다.
- `@within` 도 타깃 오브젝트의 클래스에 특정 애너테이션이 부여된 것을 찾지만 선택될 조인 포인트인 메서드는 타깃 클래스에서 선언되어 있어야 한다. 슈퍼클래스의 메서드는 해당되지 않는다.

```java
@target(org.springframework.stereotype.Controller)
```

`@args`

- `args` 와 비슷하지만 파라미터 오브젝트에 지정된 애너테이션이 부여되어 있는 경우 선정 대상이 된다

```java
@args(org.springframework.web.bind.annotation.RequestParam)
```

`@annotation`

- 조인 포인트 메서드에 특정 애너테이션이 있는 것만 선정하는 지시자

```java
@annotation(org.springframework.transaction.annotation.Transactional)
```

`bean`

- 빈 이름 또는 아이디를 이용해서 선정하는 지시자
- 와일드카드를 사용할 수 있다.

```java
bean(*Service) //Service로 끝나는 모든 빈 이름을 가진 빈 오브젝트를 선정
```





----



- 조건이 복잡한 포인트컷을 만들 때는 하나의 표현식에 모든 내용을 담기보다는 의미있는 작은 단위로 분리해서 저의한 후에 이를 조합하는 것이 좋다.
- 포인트컷 표현식은 논리연산 기호를 이용해서 여러 개의 포인트컷 지시자 또는 포인트컷 자체를 조합할 수 있다. 

`&&`

- 두 개의 포인트컷 지시자를 AND 조건으로 결합한다.

```java
within(com.example.service..*) && args(java.io.Serializable)
```

- 아래와 같이 포인트컷 자체를 조합하는 조건에 추가할 수 있다.

```java
serviceLayer() && args(java.io.Serializable)
```



`||, !`

-  `||` 는 OR조건이고 `!` 는 NOT 조건이다.

```java
daoLater() || within(com.example.dao..*)
  
daoLater() && !within(com.example.exclude.dao..*)
```





#### 어드바이스 메서드와 애너테이션

- 어드바이스는 다섯 종류가있다. 메서드 실행 과정의 일부분에만 적용할 수 있도록 만들어졌다.

![9](./images/toby-spring-vol2/9.png)

- 어드바이스가 프록시 안에서 어느 단계에 적용되느냐에 따라 어드바이스의 종류가 달라진다.
- 어드바이스를 정의하는 메서드는 어드바이스 애너테이션과 어드바이스에 적용되는 포인트컷 그리고 메서드 코드로 구성된다.

```java
@Before("daoLayer()")
public void logDaoAccess(JoinPoint jp) {
	...
}
```



`@Around`

- `@Around`는 프록시를 통해서 타깃 오브젝트의 메서드가 호출되는 전 과정을 모두 담을 수 있는 어드바이스다.
- 인터페이스 구현 방식의 AOP에서 어드바이스를 만들 때 사용했던 `MethodInterceptor` 인터페이스와 유사하게 동작한다.
- 파라미터는 `ProceedingJoinPoint`를 받는데 이 오브젝트를 이용해 타깃 오브젝트의 메서드를 실행하고 그 결과를 받을 수 있다.
- 어드바이스중 가장 강력한 어드바이스이다. 타깃 오브젝트의 메서드를 여러번 호출하거나, 호출 파라미터를 바꿔치기하는 등등의 다양한 동작을 수행할 수 있다.

```java
@Around("myPointcut()")
public Object doNothing(ProceedingJoinPoint pjp) throws Throwable {
	Object ret = pjp.proceed();
  return ret;
}
```



`@Before`

- `@Before`는 타깃 오브젝트의 메서드가 실행되기 전에 사용되는 어드바이스다.
- `@Before`를 사용해서 타깃 오브젝트 메서드를 호출하는 방식을 제어하거나 파라미터를 변경할 수 없다.
- `@Before`는 `JoinPoint` 타입의 파라미터를 사용할 수 있다. 이 `JointPoint`는 `ProceedingJoinPoint` 의 상위 타입인데 `proceed()` 메서드가 없다.

```java
@Before("myPointcut()")
public void logJoinPoint(JoinPoint) {
  System.out.println(jp.getSignature().getDeclaringTypeName());
  System.out.println(jp.getSignature().getName());
}

```



`@AfterReturning`

- `@AfterReturning`은 타깃 오브젝트의 메서드가 실행을 마친 뒤에 실행되는 어드바이스이다.
- 예외가 발생하지 않고 정상적으로 종료한 경우에만 해당된다.
- 타깃 오브젝트의 메서드가 정상 종료된 후에 호출되기 때문에 메서드의 리턴 값을 참조할 수 있다.

```java
@AfterReturning(pointcut="myPointcut()", returning="ret")
public void logReturnValue(Object ret) {
  ...
}
```



`@AfterThrowing`

- `@AfterThrowing`은 타깃 오브젝트의 메서드를 호출했을 때 예외가 발생하면 실행되는 어드바이스다.
- throwing 앨리먼트를 이용해서 예외를 전달받을 메서드 파라미터 이름을 지정할 수 있다.

```java
@AfterThrowing(pointcut="myPointcut()", throwing="ex")
public void logDAException(DataAccessException ex) {
  ...
}
```



`@After`

- `@After`는 메서드 실행이 정상 종료됐을때, 예외가 발생했을 때 모두 실행되는 어드바이스다.
- finally 키워드와 비슷하지만 리턴값이나 예외를 직접 전달받을 수 없다



#### 파라미터 선언과 바인딩

- 어드바이스 메서드에는 `JoinPoint`, `ProceedingJoinPoint`를 기본적으로 사용할 수 있다. 또 종류에 따라 returning이나 throwing을 이용해서 선언된 리턴값 또는 예외 파라미터를 이용할 수 있다.
- 포인트컷 표현식의 타입 정보를 파라미터와 연결하는 방법도 있다.

```java
@Before("@target(com.example.foo.bar.BatchJob)")
public void beforeBatch() {...}


@Before("@target(batchJob)")
public void beforeBatch(BatchJob batchJob) {...}

```



#### @AspectJ를 이용한 AOP의 학습 방법과 적용 전략

- 복잡한 문법, 포인트컷 표현식 등등으로 사용법이 복잡한편인데 AOP를 적용할때는 항상 단순하고 명료하게 접근해야 한다.
- 러닝커브가 있기때문에 팀내부에서 단계를 거쳐 점진적으로 도입하는 것도 중요하다.
- 객체지향적인 방법으로 해결할 수 있는 것을 불필요하게 AOP를 사용하지 말아야한다.
- AOP는 기본적으로 애플리케이션의 핵심 로직이 아니라 부가적인 기능을 적용할 떄 사용하는 것이다.
- AOP도 POJO기반이기 때문에 충분한 테스트를 작성해야한다.
- 이것 외에도 다양한 AOP 고급 적용기술이 있기 때문에 필요한 경우 레퍼런스를 확인하면된다.

## AspectJ와 @Configurable

### AspectJ AOP

- AspectJ는 타깃 오브젝트 자체의 코드를 바꿈으로써 애스펙트를 적용하기때문에 프록시를 사용하지 않는다.
- AspectJ가 바이트코드 조작을 필요로 하는 이유는 프록시 방식으로는 어드바이스를 적용할 수 없는 조인 포인트와 포인트컷 지시자를 지원하기 위해서이다.
- AspectJ의 조인 포인트는 메서드 실행 지점 외에도 필드 읽기와 쓰기, 스태틱 초기화, 인스턴스 생성, 인스턴스 초기화 등도 지원한다.
- 자바의 프록시 기법을 이용해서는 특정 오브젝트가 생성된 시점이나 특정 필드를 읽을 지점에 어드바이스를 추가할 방법이 없다.
- 일반적인 경우에는 메서드 실행 지점을 조인포인트로 사용해서 대부분의 요구사항을 스프링의 프록시 방식 AOP로 해결할 수 있지만 위 문제를 해결하기 위해 AspectJ AOP를 적용할 수 있다.
- AspectJ AOP는 방대한 주제이므로 자세한 설명은 생략한다...

### 빈이 아닌 오브젝트에 DI 적용하기

- 스프링 빈이 아니더라도 수정자 메서드를 가진 임의의 오브젝트에는 스프링의 API의 도움을 받아서 간단히 DI를 할 수 있지만 문제는 오브젝트가 생성되는 시점은 스프링 AOP에서 지원하는 조인포인트가 아니다.
- 스프링이 직접 제공하는 AspectJ AOP 적용 기능을 활용해서 위 문제를 해결할 수 있다.

#### DI 애스펙트

- 스프링은 AspectJ 기술로 만들어진 `DependencyInjectionAspect` 라는 애스펙트를 제공한다.
- 실제 내부는 AspectJ 언어로 만들어져 있기 때문에 복잡하지만 내부에는 아래와 같은 코드가 있다.

```
public pointcut beanConstruction(Object bean) :
	initilization(ConfigurableObject+.new(..)) && this(bean);

declare parents: @Configurable * impletments ConfigurableObject;

after(Object bean) returning :
	beanConstruction(bean) && postConstructionCondition() && inConfigurableBean() {
		configureBean(bean);
	}
```

- 결론적으로 `@Configurable`이 붙은 오브젝트가 생성되는 조인 포인트에 이 어드바이스가 실행되도록 되어 있다.
- `DependencyInjectionAspect` 애스펙트가 적용되면 `@Configurable`이 붙은 도메인 오브젝트가 어디서든 생성될 때마다 이 어드바이스가 적용되어 자동DI 작업이 일어난다. 또 빈의 초기화 메서드가 정의되어 있다면 실행된다. DI 방식은 도메인 오브젝트를 빈으로 선언해서 명시적으로 지정할 수도 있고 자동와이어링 방식도 이용할 수 있다.

#### @Configurable

- 다음과 같이 User 도메인 안에 비지니스 로직을 넣기 위해 스프링 빈을 DI받아야한다면 아래와 같이 작성할 수 있다.

```java
public class User {

    private MyService myService;

    public void doSomething() {
        myService.something();
    }
}
```

- User는 스프링의 빈이 아니기 때문에 new 키워드를 사용해서 직접생성하기 때문에 별도의 설정이 없으면 myService는 null을 리턴하지만 DI 애스펙트를 적용하면 필요한 빈이 주입된다.

```java
@Configurable
public class User {
	...
}
```

- xml 방식, setter 방식이 있지만 가장 간단한 애너테이션 의존 방식을 사용하면 필드주입으로 간단하게 DI를 적용할 수 있다.

```java
@Configurable
public class User {

    private MyService myService;

    public void doSomething() {
        myService.something();
    }
}
```



#### 로드타임 위버와 자바 에이전트

- 추가적으로 DI 애스펙트를 적용하기 위한 작업이 필요하다. 먼저 AspectJ AOP가 동작할 수 있는 환경설정과 DI 애스펙트 자체를 등록해서 `@Configurable` 오브젝트에 어드바이스가 적용되게 해야 한다.
- AspectJ를 사용하려면 클래스를 로딩하는 시점에 바이트코드 조작이 가능하도록 로드타임 위버를 적용해줘야 한다.
- 로드타임 위버는 jvm의 javaagent 옵션을 사용해서 jvm 레벨에 적용해야한다.

```
-javaagent:/Users/a10300/.m2/repository/org/springframework/spring-instrument/5.3.10/spring-instrument-5.3.10.jar
```



## 로드타임 위버(LTW)

- 로드타임위버는 스프링에서 여러가지 기능에 활용된다.
  - `@Configurable` 지원
  - AspectJ AOP 에서 사용
  - JPA에서 필요로하는 로드타임 위버로 사용
- 하이버네이트 JPA를 제외하면 대부분의 JPA 구현 제품은 고급 기능을 위해 로드타임 위버를 사용하도록 요구한다.
- 로드타임위버의 사용에서 두가지 문제
  - JVM레벨에 적용되기 때문에 JVM을 통해 로딩되는 모든 클래스를 다 조사하고 클래스 바이트코드 조작 대상으로 삼기 때문에 서버에 적용하기 부담스럽다.
  - JVM의 자바에이전트 옵션은 한 번에 한가지만 적용할 수 있다. AspectJ와 JPA 동시 적용 불가능
- 스프링 로드타임 위버는 JPA와 AspectJ를 위한 로드타임 위버로 동작하고 JVM 레벨의 자바에이전트를 대체할 수 있는 로드타임 위버 적용 방법을 자동으로 찾아줌으로써 해결한다.
- 스프링의 로드타임 위버는 자바에이전트 방식에 종속적이지 않다.

## 스프링 3.1의 AOP와 LTW

### AOP와 LTW를 위한 애너테이션

#### @EnableAspectJAutoProxy

- `@EnableAspectJAutoProxy` 는 @Aspect로 애스펙트를 정의할 수 있게 해주는 @AspectJ AOP 컨테이너 인프라 빈을 등록해준다.

#### @EnableLoadTimeWeaving

- `@EnableLoadTimeWeaving` 은 환경에 맞는 로드타임 위버를 등록해주는 애너테이션이다.

```java
@Configuration
@EnableSpringConfigured
@EnableLoadTimeWeaving
public class Config {

}
```



## 정리

- 스프링에서는 프록시 기반의 AOP 기술을 네 가지 접근 방법을 통해 활용할 수 있다. 각 접근 방법의 장당점을 파악하고 자신에게 맞는 방법을 선택할 수 있어야한다.
- @AspectJ는 AspectJ 스타일의 POJO 애스펙트 작성 방법이다. @AspectJ는 유연하고 강력한 기능을 가진 애스팩트를 손쉽게 만들어주지만, 복잡한 AspectJ 문법과 사용방법을 익혀야하는 부담이 있다.
- 스프링은 또한 AspectJ를 스프링 애플리케이션 내에서 간접적으로 활용할 수 있는 몇가지 방법을 제공한다. `@Configurable`은 스프링 빈이 아닌 오브젝트에 DI를 적용할 수 있게 해준다.
- 스프링은 다양한 서버환경에서 사용 가능한 편리한 로드타임 위버를 제공해준다.
