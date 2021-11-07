# #6 AOP

## 트랜잭션 코드의 분리

### 문제점 파악하기

- 비지니스 로직이 주인이어야할 메서드 안에 트랜잭션 코드들이 공존하는 코드

```java
    public void upgradeLevels() {

        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            List<User> users = userDao.getAll();
            for (User user : users) {
                if (canUpgradeLevel(user)) {
                    upgradeLevel(user);
                }
            }
            this.transcationManager.commit(status);
        } catch (Exception e) {
            this.transcationManager.rollback(status);
            throw e;
        }

    }
```

- 트랜잭션 경계설정의 코드와 비지니스 로직 코드간에 서로 주고받는 정보가 없기때문에 같은 메서드에 존재하지만 완벽하게 독립적인 코드임
- 역할에 따라서 분리하는게 원칙이기 때문에 코드를 분리해야함
- 메서드 분리를 통해 어느정도 분리할 수 있지만 한계가 있음. 따라서 클래스 레벨로 분리해야함



### DI 적용을 이용한 트랜잭션 분리

[![1](https://github.com/sup2is/dev-note/raw/master/spring/images/toby-spring-vol1/1.jpeg)](https://github.com/sup2is/dev-note/blob/master/spring/images/toby-spring-vol1/1.jpeg)

- 기존에 위와 같이 `UserServiceImpl` 안에 비지니스 로직, 트랜잭션 로직이 공존하던 문제를 아래와 같이 `UserServiceImpl` 클래스와 `UserServiceTx` 로 구분지을 수 있음

[![2](https://github.com/sup2is/dev-note/raw/master/spring/images/toby-spring-vol1/2.jpeg)](https://github.com/sup2is/dev-note/blob/master/spring/images/toby-spring-vol1/2.jpeg)

-  `UserServiceImpl` : 비지니스 로직이 있는 클래스 
-  `UserServiceTx` : 트랜잭션 관련 로직이 있는 클래스
-  클라이언트는 `UserService` 인터페이스 타입을 바라보게 되면서 약결합 상태로 만들수 있고 때문에 유연한 확장이 가능한 구조가 됨



`트랜잭션 코드를 제거한 UserService 구현 클래스`

```java
package me.sup2is;

public class UserServiceImpl implements UserService {
    UserDao userDao;
    MailSender mailSender;

    private void upgradeLevelsInternal() {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
    }
}
```

`비지니스 트랜잭션 처리를 담은 UserService 구현 클래스 `

```java
package me.sup2is;

public class UserServiceTx implements UserService {
    UserService userService;
    PlatformTranscationManager transcationManager;

    public void setTranscationManager(PlatformTranscationManager transcationManager) {
        this.transcationManager = transcationManager;
    }

    public void setUserService(UserService userService) {
        this.userService = userService;
    }

    public void add(User user) {
        userService.add(user);
    }

    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            userService.upgradeLevel(user);

            this.transcationManager.commit(status);
        } catch (Exception e) {
            this.transcationManager.rollback(status);
            throw e;
        }
    }
}
```

- `UserServiceTx`는 사용자 관리라는 비지니스 로직을 전혀 갖지 않고 고스란히 다른 `UserService` 구현 오브젝트에 기능을 위임하는 것이 특징

### 트랜잭션 적용을 위한 DI 설정

- 클라이언트가 `UserService` 라는 인터페이스를 통해 사용자 관리 로직을 이용하려고 할 때 `UserServiceTx` 클래스를 주입

```xml
<bean id="userService" class="springbook.service.UserServiceTx">
    <property name="userService" ref="userServiceImpl"/>
    <property name="transactionManager" ref="transactionManager"/>
</bean>
<bean id="userServiceImpl" class="springbook.service.UserServiceImpl">
    <property name="userDao" ref="userDaoJdbc"/>
    <property name="userLevelUpgradePolicy" ref="userLevelUpgradePolicy"/>
    <property name="mailSender" ref="mailSender"/>
</bean>
```



### 트랜잭션 경계설정 코드 분리의 장점

- 비지니스 로직을 담당하고 있는 `UserServiceImpl`의 코드를 작성할 때는 트랜잭션과 같은 기술적인 내용에는 전혀 신경쓰지 않아도 됨



## 다이내믹 프록시와 팩토리 빈

### 프록시와 프록시 패턴, 데코레이터 패턴

- 위에서 본 예제처럼 트랜잭션이라는 기능은 사용자 관리 비지니스 로직과는 성격이 다르기떄문에 '부가기능', '핵심기능'으로 나눌 수 있음
- 이렇게 분리된 부가기능은 부가기능 외의 나머지 모든 기능을 원래 핵심 기능을 가진 클래스로 위임해줘야함
- 클라이언트의 입장에서는 실제로 핵심기능을 사용하는것으로 알지만 실제로는 부가기능을 거쳐서 사용하도록 만들어야함
- **클라이언트가 사용하려고하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것을 대리자, 대리인과 같은 역할을 한다고 해서 프록시라고 부름**
- 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 타깃 또는 실체라고 부름



### 프록시 패턴

- 프록시 패턴의 프록시는 타깃의 기능 자체를 확장하거나 추가하지 않고 클라이언트가 타깃에 접근하는 방식을 변경해줌
- 타깃 오브젝트를 생성하기가 복잡하거나 당장 필요하지 않은 경우에는 꼭 필요한 시점까지 오브젝트를 생성하지 않는 편이 좋음 ex: hibernate lazy load entities
- 또는 특별한 상황에서 타깃에 대한 접근권한을 제어하기 위해 프록시 패턴을 사용할 수 도 있음 대표적으로 `Collections` 클래스의 `unmodifiableCollection()` 메서드

```java
    public static <T> Collection<T> unmodifiableCollection(Collection<? extends T> c) {
        return new UnmodifiableCollection<>(c);
    }
    

    static class UnmodifiableCollection<E> implements Collection<E>, Serializable {
      
      ...

        
        // 접근제어용으로 UnsupportedOperationException를 발생시키는 모습
        public boolean add(E e) {
            throw new UnsupportedOperationException();
        }
        public boolean remove(Object o) {
            throw new UnsupportedOperationException();
        }

        public boolean containsAll(Collection<?> coll) {
            return c.containsAll(coll);
        }
        public boolean addAll(Collection<? extends E> coll) {
            throw new UnsupportedOperationException();
        }
        public boolean removeAll(Collection<?> coll) {
            throw new UnsupportedOperationException();
        }
        public boolean retainAll(Collection<?> coll) {
            throw new UnsupportedOperationException();
        }
        public void clear() {
            throw new UnsupportedOperationException();
        }

      ...
    }


```





### 데코레이터 패턴

- 데코레이터 패턴은 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴
- 다이내믹하게 기능을 부가한다는 의미는 컴파일 시점, 즉 코드상에서 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져 있지 않다는 뜻
- 프록시로서 동작하는 각 데코레이터는 위임하는 대상에도 인터페이스로 접근하기 때문에 자신이 최종 타깃으로 위임하는지, 아니면 다음 단계의 데코레이터 프록시로 위임하는지는 알지 못하는 특징이 있음
- 따라서 데코레이터의 다음 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메서드를 통해 위임 대상을 외부에서 런타임 시에 주입받을 수 있도록 만들어야함
- 대표적으로 자바 IO 패키지의 `InputStream`과 `OutputStream` 구현 클래스는 데코레이터 패턴이 사용된 대표적인 예

```java
InputStream is = new BufferedInputStream(new FileInputStream("a.txt"));
```

> 주절주절 쓴 데코레이터 패턴 예제: [https://sup2is.github.io/2020/06/26/decorator-parttern.html](https://sup2is.github.io/2020/06/26/decorator-parttern.html)



#### 프록시 패턴 예제

`Hello`

```java
public interface Hello {
    String sayHello(String name); 
    String sayHi(String name); 
    String sayThankYou(String name);
}
```

`HelloTarget`

```java
public class HelloTarget implements Hello {
    @Override
    public String sayHello(String name) {
        return "Hello " + name;
    }

    @Override
    public String sayHi(String name) {
        return "Hi " + name;
    }

    @Override
    public String sayThankYou(String name) {
        return "Thank you " + name;
    }
}
```

`HelloUppercase`

```java
public class HelloUppercase implements Hello {

    private final Hello delegate;

    public HelloUppercase(Hello delegate) {
        this.delegate = delegate;
    }

    @Override
    public String sayHello(String name) {
        return delegate.sayHello(name).toUpperCase();
    }

    @Override
    public String sayHi(String name) {
        return delegate.sayHi(name).toUpperCase();
    }

    @Override
    public String sayThankYou(String name) {
        return delegate.sayThankYou(name).toUpperCase();
    }
}
```

- 프록시를 사용하면 기존 코드에 영향을 주지 않으면서 타깃의 기능을 확장하거나 접근 방법을 제어할 수 있지만 아래와 같은 단점이 있음
  - 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기가 번거로움. 부가기능이 필요 없는 메서드도 구현해서 타깃으로 위임하는 코드를 일일이 만들어줘야함
  - 부가기능 코드가 중복될 가능성이 많다는 점
- 위와 같이 일반적인 프록시의 단점을 해결해주는 것이 바로 JDK의 다이내믹 프록시



### 다이내믹 프록시 적용하기

[![7](https://github.com/sup2is/dev-note/raw/master/spring/images/toby-spring-vol1/7.jpeg)](https://github.com/sup2is/dev-note/blob/master/spring/images/toby-spring-vol1/7.jpeg)

- 다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어줌
- 다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트임
- 다이내믹 프록시 오브젝트는 타깃의 인터페이스와 같은 타입으로 만들어짐
- 클라이언트는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용할 수 있음
- 실제로 부가기능은 `InvocationHandler`를 구현한 오브젝트에 담음

```java
public interface InvocationHandler {

    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

- 타깃 인터페이스의 모든 메서드 요청이 하나의 메서드로 집중되기 때문에 중복되는 기능을 효과적으로 제공할 수 있음
- `InvocationHandler` 를 사용해서 부가기능을 추상화시키기

```java
public class Uppercasehandler implements InvocationHandler {

    private Object target;
  	//다이내믹 프록시로부터 전달받은 요청을 다시 타깃 오브젝트에 위임해야 하기 때문에 타깃 오브젝트를 주입받아둠
  
    public Uppercasehandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String result = (String) method.invoke(target, args); // <- 타깃으로 위임. 인터페이스의 메서드 호출에 모두 적용됨
        return result.toUpperCase();
    }
}
```

- 아래와 같은 형태로 적용

[![8](https://github.com/sup2is/dev-note/raw/master/spring/images/toby-spring-vol1/8.jpeg)](https://github.com/sup2is/dev-note/blob/master/spring/images/toby-spring-vol1/8.jpeg)

- 프록시 생성하기

```java
Hello proxy = (Hello) Proxy.newProxyInstance(
  getClass().getClassLoader(), //<- 동적으로 생성되는 다이내믹 프록시 클래스 로딩에 사용할 클래스 로더
  new Class[]{Hello.class}, // <- 구현할 인터페이스
  new Uppercasehandler(new HelloTarget())); // <- 부가기능과 위임 코드를 담은 InvocationHandler
```

- 첫번째 파라미터는 다이내믹 프록시가 정의되는 클래스 로더를 지정
- 두번째 파라미터는 다이내믹 프록시가 구현해야 할 인터페이스
- 세번째 파라미터는 부가기능과 위임 관련 코드를 담고 있는 `InvocationHandler` 구현 오브젝트

### 다이내믹 프록시를 이용한 트랜잭션 부가기능

- `UserServiceTx`를 다이내믹 프록시 방식으로 변경해서 트랜잭션 부가기능을 제공하는 다이내믹 프록시를 만들기

```java
public class TransactionHandler implements InvocationHandler {

    private Object target; // <- 부가기능을 제공할 타깃 오브젝트, 어떤 타입의 오브젝트도 가능함
    private PlatformTransactionManager transactionManager; // <- 트랜잭션 기능을 제공하는데 필요한 트랜잭션 매니저
    private String pattern;

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().startsWith(pattern)) {
            return invokeInTransaction(method, args);
        }
        return method.invoke(target, args);
    }

    private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        TransactionStatus transaction = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Object result = method.invoke(target, args);
            transactionManager.commit(transaction);
            return result;
        } catch (InvocationTargetException e) {
            transactionManager.rollback(transaction);
            throw e.getTargetException();
        }
    }
}
```

- `TransactionHandler`를 사용해서 다이내믹 프록시를 생성하면 타깃오브젝트에 트랜잭션 부가기능이 적용된 형태로 사용할 수 있는 이점이 있음
- `TransactionHandler`와 다이내믹 프록시를 스프링 DI를 통해 사용할 수 있도록 만들어야하지만 문제가 있음
  - DI의 대상이 되는 다이내믹 프록시 오브젝트는 일반적인 스프링 빈으로 등록할 방법이 없음
  - 스프링은 내부적으로 리플렉션 api를 이용해서 빈 정의에 나오는 클래스 이름을 가지고 빈 오브젝트를 생성하는데 다이내믹 프록시 오브젝트는 이런식으로 프록시 오브젝트가 생성되지 않는다는 점.
  - 클래스 자체도 내부적으로 다이내믹하게 새로 정의해서 사용하기 때문에 프록시 오브젝트의 클래스 정보를 미리 알아내서 스프링 빈에 정의할 방법이 없음
- 따라서 다이내믹 프록시를 등록하기 위한 팩토리 빈이 필요함

### 다이내믹 프록시를 위한 팩토리 빈

- 팩토리 빈이란 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈
- 팩토리 빈은 `FactoryBean` 이라는 인터페이스를 구현해서 만들 수 있음

```java
public interface FactoryBean<T> {
    @Nullable
    T getObject() throws Exception;
    @Nullable
    Class<?> getObjectType();
    default boolean isSingleton() {
        return true;
    }
}
```

- `FactoryBean` 인터페이스를 구현한 클래스를 스프링의 빈으로 등록하면 팩토리 빈으로 동작함
- `FactoryBean`을 등록하는 예제

```java
public class MessageFactoryBean implements FactoryBean<Message> {
    private String text;

    public void setText(String text) {
        this.text = text;
    }

    @Override
    public Message getObject() throws Exception {
        return Message.newMessage(text); // <- 실제 빈으로 사용될 오브젝트를 팩토리 메서드를 통해서 생성함
    }

    @Override
    public Class<?> getObjectType() {
        return Message.class;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }
}

//팩토리빈을 빈으로 등록하기
@Configuration
public class MyConfig {

    @Bean
    MessageFactoryBean messageFactoryBean() {
        return new MessageFactoryBean("test");
    }
}
```

- 스프링은 `FactoryBean` 인터페이스를 구현한 클래스가 빈의 클래스로 지정되면, 팩토리 빈 클래스의 오브젝트의 `getObject()` 메서드를 이용해 오브젝트를 가져오고 이를 빈 오브젝트로 사용함
- 이 팩토리빈을 사용하면 다이내믹 프록시 오브젝트를 스프링 빈으로 등록할 수 있음

### 트랜잭션 프록시 팩토리 빈

```java
public class TxProxyFactoryBean implements FactoryBean<Object> {
    Object target;
    PlatformTransactionManager transactionManager;
    String pattern;
    Class<?> serviceInterface;

    public void setTarget(Object targer) {
        this.target = targer;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    public void setServiceInterface(Class<?> serviceInterface) {
        this.serviceInterface = serviceInterface;
    }

    //FactoryBean 인터페이스 구현 메소드
    public Object getObject() throws Exception {
        TransactionHandler txHandler = new TransactionHandler();
        txHandler.setTarget(targer);
        txHandler.setTransactionManager(transactionManager);
        txHandler.setPattern(pattern);
        return Proxy.newProxyInstance(
                getClass().getCalssLoader(), new Class[]{serviceInterface},
                txHandler);
    }

    /*
     * DI 받은 인터페이스 타입에 따라 팩토리 빈이 생성하는 오브젝트 타입이 달라진다.
     * 다양한 프록시 오브젝트 생성을 위한 재사용 코드
     */
    public Class<?> getObjectType() {
        return serviceInterface;
    }

    /*
     * 싱글톤 빈이 아니라는 뜻이 아니라
     * getObject()가 매번 같은 오브젝트를 리턴하지 않는다는 의미
     */
    public boolean isSingleton() {
        return false;
    }
}

```



### 프록시 팩토리 빈 방식의 장점과 한계

- 장점: `TxProxyFactoryBean`은 코드의 수정 없이도 다양한 클래스에 적용할 수 있음
- 단점: 하나의 클래스를 대상으로 하는것은 문제가 없지만 여러 클래스를 대상으로 하는것은 불가능함 또 하나의 타깃에 여러 부가기능을을 적용하려고할때도 제한적임

```java
//팩토리빈을 빈으로 등록하기
@Configuration
public class MyConfig {

    @Bean
    AFactoryBean aFactoryBean() {
        return new AFactoryBean("A");
    }
  
    @Bean
    BFactoryBean bFactoryBean() {
        return new BFactoryBean("B");
    }
  
  ...
}


```



## 스프링의 프록시 팩토리 빈

- 스프링은 일관된 방법으로 프록시를 만들 수 있게 도와주는 추상 레이어가 있음
- 스프링의 `ProxyFactoryBean`은 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈
- `ProxyFactoryBean`은 순수하게 프록시를 생성하는 작업만을 담당하고 프록시를 통해 제공해줄 부가 기능은 별도의 빈에 둘 수 있음
- `ProxyFactoryBean`이 생성하는 프록시에서 사용할 부가기능은 `MethodInterceptor` 인터페이스를 구현해서 만듦
  - `InvocationHandler`: `invoke()` 메서드에 타깃 오브젝트에 대한 정보를 제공받지 않음
  - `MethodInterceptor`: `invoke()` 메서드에 타깃 오브젝트에 대한 정보까지도 함께 제공받음
- `ProxyFactoryBean` 은 `FactoryBean`의 구현체임 [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/framework/ProxyFactoryBean.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/framework/ProxyFactoryBean.html)
- `ProxyFactoryBean` 을 생성하는 예제

```java
public class DynamicProxyTest {
    @Test
    public void simpleProxy() {
        Hello proxiedHello = (Hello) Proxy.newProxyInstance(
                getClass().getClassLoader(),
                new Class[]{Hello.class},
                new UppercaseHandler(new HelloTarget()));
        ...
    }

    @Test
    public void proxyFactoryBean() {
        ProxyFactoryBean pfBean = new ProxyFactoryBean();
        pfBean.setTarget(new HelloTarget()); //타깃 설정
        pfBean.addAdvice(new UppercaseAdvice()); //부가기능 추가
        Hello proxiedHello = (Hello) pfBean.getObject(); //FacotryBean이므로 생성된 프록시를 가져온다.

        assertThat(ProxiedHello.sayHello("Havi"), is("Hello Havi"));
        ...
    }

    static class UppercaseAdvice implements MethodInterceptor {
        public Object invoke(MethodInvocation invocation) throws Throwable {
            String ret = (String) invocation.proceed(); //타깃을 알고 있기에 타깃 오브젝트를 전달할 필요가 없다.
            return ret.toUpperCase(); //부가기능 적용
        }
    }

    static interface Hello {
        String sayHello(String name);

        String sayHi(String name);

        String sayThankYor(String name);
    }

    static class HelloTarget implements Hello {
        public String sayHello(String name) {
            return "Hello" + name;
        }
        ...
    }
}



```



### 어드바이스: 타깃이 필요 없는 순수한 부가기능

- `ProxyFactoryBean` 을 적용한 코드와 기존의 다이내믹 프록시와 다른점
  - `InvocationHandler`를 구현했을 때와 달리 `MethodInterceptor`를 구현한 `UppercaseAdvice`에는 타깃 오브젝트가 등장하지 않음
- `MethodInterceptor` 로는 메서드 정보와 함께 타깃 오브젝트가 담긴 `MethodInvocation` 오브젝트가 전달됨
- `MethodInvocation` 은 타깃 오브젝트의 메서드를 실행할 수 있는 기능이 있기 때문에 `MethodInterceptor` 는 부가기능을 제공하는데만 집중할 수 있음
- `MethodInvocation`은 일종의 콜백 오브젝트로 동작하기 때문에 `proceed()` 메서드를 사용하면 타깃 오브젝트의 메서드를 내부적으로 실행해주는 기능이 있음
- `ProxyFactoryBean`은 작은 단위의 템플릿/콜백 구조를 응용해서 적용했기 때문에 템플릿 역할을 하는 `MethodInvocation`을 싱글톤으로 두고 공유할 수 있음
- `ProxyFactoryBean`에는 여러개의 `MethodInvocation`를 추가할 수 있는 것도 특징. 따라서 팩토리 빈을 사용했을 때의 단점인 새로운 부가기능을 추가할 때마다 프록시와 프록시 팩토리 빈도 추가해줘야 한다는 문제를 해결할 수 있음.
- `MethodInterceptor`처럼 타깃 오브젝트에 적용하는 부가기능을 담은 오브젝트를 스프링에서는 **어드바이스**라고 부름
- 경우에 따라서 CGLib이라고 하는 오픈소스 바이트코드 생성프레임워크를 이용해 프록시를 만들기도 함 (자세한 내용은 vol.2에서 ..)
- **어드바이스는 타깃 오브젝트에 종속되지 않는 순수한 부가기능을 담은 오브젝트라는 사실이 중요**

### 포인트컷: 부가기능 적용 대상 메소드 선정 방법

- `MethodInvocation`은 재사용 가능한 순수한 부가 기능 제공 코드만 남겨주기 때문에 아래와 같이 특정 메서드를 선정해서 적용 대상을 판별할 수 없음

```java
    //InvocationHandler 구현부
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().startsWith(pattern)) {
            return invokeInTransaction(method, args);
        }
        return method.invoke(target, args);
    }
```

- 스프링의 `ProxyFactoryBean` 방식은 두가지 확장 기능인 부가기능과 메서드 선정 알고리즘을 활용하는 유연한 구조를 제공함
- **스프링은 부가기능을 제공하는 오브젝트를 어드바이스라고 부르고, 메서드 선정 알고리즘을 담은 오브젝트를 포인트컷 이라고 부름**
- 프록시는 포인트컷으로부터 부가기능을 적용할 대상 메서드인지 확인받으면 `MethodInvocation` 타입의 어드바이스를 호출함
- 포인트컷 구현체 목록 [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/Pointcut.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/Pointcut.html)
- 포인트컷 구현체중 하나인 `NameMatchMethodPointcut` 사용해보기

```java
@Test
public void pointcutAdvisor() {
    ProxyFactoryBean pfBean = new ProxyFactoryBean();
    pfBean.setTarget(new HelloTarget());
    
    NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut(); //메소드이름을 비교대상으로 선정하는 포인트컷 생성
    pointcut.setMappedName("sayH*"); //이름 비교조건 설정(sayH로 시작하는 모든 메소드 선택)
    pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice())); //포인트컷과 어드바이스를 Advisor로 묶어서 한 번에 추가
    
    Hello proxiedHello = (Hello) pfBean.getObject();
    assertThat(ProxiedHello.sayHello("Havi"), is("Hello Havi"));
    assertThat(ProxiedHello.sayThankYou("Havi"), is("Thank You Havi")); //적용 안됨
}
```

- 어드바이스와 포인트컷은 모두 프록시에 DI로 주입되어 사용되기 때문에 스프링의 싱글톤 빈으로 등록이 가능함
- 재사용 가능한 기능을 만들어두고 바뀌는 부분만 외부에서 주입해서 이를 작업흐름중에 사용하도록 하는 전형적인 템플릿/콜백 구조

[![9](https://github.com/sup2is/dev-note/raw/master/spring/images/toby-spring-vol1/9.jpeg)](https://github.com/sup2is/dev-note/blob/master/spring/images/toby-spring-vol1/9.jpeg)

- 동작 방식
  - 프록시는 클라이언트로부터 요청을 받으면 먼저 포인트컷에게 부가기능을 부여할 메서드인지 확인해달라고 요청함
  - 확인 받으면 `MethodInterceptor` 타입의 어드바이스를 호출
  - Invocation 콜백을 수행하고 타깃 오브젝트의 메서드를 수행

- 어드바이스가 일종의 템플릿이 되고 타깃을 호출하는 기능을 갖고 있는 `MethodInvocation` 오브젝트가 콜백
- 프록시로부터 어드바이스와 포인트컷을 독립시키고 DI를 사용하게 한 것은 전형적인 전략 패턴 구조
- 어드바이스와 포인트컷을 묶은 오브젝트를 **어드바이저** 라고함
  - 어드바이저 = 포인트컷 (메서드 선정 알고리즘) + 어드바이스 (부가 기능)



### MethodInterceptor 적용해보기

```java
public class TransactionAdvice implements MethodInterceptor {

    private PlatformTransactionManager transactionManager;

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        TransactionStatus transaction = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Object result = invocation.proceed();
            transactionManager.commit(transaction);
            return result;
        } catch (RuntimeException runtimeException) {
            transactionManager.rollback(transaction);
            throw runtimeException;
        }
    }
}

```

- `MethodInterceptor`를 사용해서 `InvocationHandler` 보다 재사용성이 높고 타깃 메서드가 던지는 예외도 `RuntimeException`으로 바로 받을 수 있음
- 여전히 남아있는 문제가 하나 있는데 바로 설정쪽에 번거로운 작업과 중복문제. 한번에 여러개의 클래스에 공통적인 부가기능을 한번에 제공하는 방법이 불가능함.
- 빈 후처리기를 이용해서 자동으로 프록시를 생성하게 할 수 있음



## 스프링 AOP

### 빈 후처리기를 이용한 자동 프록시 생성기

- 빈 후처리기는 이름 그대로 스프링 빈 오브젝트로 만들어지고 난 후에 빈 오브젝트를 다시 가공할 수 있게 해줌
- 빈 후처리기는 `BeanPostProcessor` 인터페이스를 구현해서 만들 수 있음  (자세한 내용은 vol.2에서 ..)

```java
public interface BeanPostProcessor {

   @Nullable
   default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      return bean;
   }

   @Nullable
   default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
      return bean;
   }
}
```

- 빈 후처리기 종류중 하나인 `DefaultAdvisorAutoProxyCreator`를 사용해서 빈 후처리기 기능을 사용할 수 있음
- `DefaultAdvisorAutoProxyCreator`는 어드바이저를 이용한 자동 프록시 생성기임
- 빈 후처리기 자체를 빈으로 등록하면 빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청할 수 있음
- 빈 후처리기는 빈 오브젝트의 프로퍼티를 강제로 수정할 수도 있고 별도의 초기화 작업을 수행할 수도 있음
- 따라서 스프링이 설정을 참고해서 만든 오브젝트가 아닌 다른 오브젝트를 빈으로 등록시키는 것이 가능함

[![10](https://github.com/sup2is/dev-note/raw/master/spring/images/toby-spring-vol1/10.jpeg)](https://github.com/sup2is/dev-note/blob/master/spring/images/toby-spring-vol1/10.jpeg)

 


- 동작 방식
  - `DefaultAdvisorAutoProxyCreator` 빈 후처리기가 등록되어 있으면 스프링은 빈 오브젝트를 만들 때마다 빈 후처리기에 빈을 보내는 방식으로 동작함
  - `DefaultAdvisorAutoProxyCreator`는 빈으로 등록된 모든 어드바이저 내의 포인트컷을 이용해 전달받은 빈이 프록시 적용 대상인지 확인함
  - 프록시 적용 대상이면 그때는 내장된 프록시 생성기에게 현재 빈에 대한 프록시를 만들게 하고, 만들어진 프록시에 어드바이저를 연결해줌
  - 빈 후처리기는 프록시가 생성되면 원래 컨테이너가 전달해준 빈 오브젝트 대신 프록시 오브젝트를 컨테이너에게 돌려줌
  - 컨테이너는 최종적으로 빈 후처리기가 돌려준 오브젝트를 빈으로 등록하고 사용함
- 적용할 빈을 선정하는 로직이 추가된 포인트컷이 담긴 어드바이저를 등록하고 빈 후처리기를 사용하면 일일이 `ProxyFactoryBean` 빈을 등록하지 않아도 타깃 오브젝트에 자동으로 프록시가 적용되게 할 수 있음



### 확장된 포인트컷

- 포인트컷은 클래스필터와 메서드 매처 기능을 하는 두가지 메서드가 존재함
- 위 예제에서 사용한 `NameMatchMethodPointcut` 은 메서드 선별 기능만 가진 특별한 포인트컷

```java
public interface Pointcut {

	ClassFilter getClassFilter();
	MethodMatcher getMethodMatcher();
}

```

- 만약 포인트컷 선정 기능을 모두 적용한다면 먼저 프록시를 적용할 클래스인지 판단 후 적용 대상 클래스의 메서드를 확인하는 식으로 동작함
- 모든 빈에 대해 프록시 자동적용 대상을 선별해야 하는 빈 후처리기인 `DefaultAdvisorAutoProxyCreator` 는 클래스와 메서드 선정 알고리즘을 모두 갖고 있는 포인트컷이 필요함
- 결론적으로 이 두가지 조건이 모두 충족되는 타깃의 메서드에 어드바이스가 적용됨



### DefaultAdvisorAutoProxyCreator의 적용하기

`클래스 필터를 적용한 포인트컷`

```java
public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {

    public void setMappedClassName(String mappedClassName) {
        setClassFilter(new SimpleClassFilter(mappedClassName));
    }

    static class SimpleClassFilter implements ClassFilter {

        private final String mappedName;

        public SimpleClassFilter(String mappedName) {
            this.mappedName = mappedName;
        }

        @Override
        public boolean matches(Class<?> clazz) {
            return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName());
        }
    }
}
```

`DefaultAdvisorAutoProxyCreator 등록`

```xml
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"> </>
```

`포인트컷 등록`

```xml
<bean id="transactionPointcut" class="springbook.proxy.NameMatchClassMethodPointcut">
    <property name="mappedName" value="upgrade*"/>
    <property name="mappedClassName" value="*ServiceImpl"/>
</bean>
```

- 어드바이저를 이용하는 자동 프록시 생성기인 `DefaultAdvisorAutoProxyCreator` 에 의해 자동으로 수집되고 프록시 대상 선정 과정에 참여하며 자동 생성된 프록시에 다이내믹하게 DI돼서 동작하는 어드바이저가 됨



### 포인트컷 표현식을 이용하는 포인트컷 적용

- 스프링은 아주 간단하고 효과적인 방법으로 포인트컷의 클래스와 메서드를 선정하는 알고리즘을 작성할 수 있는 방법을 제공함
- 표현식 언어를 사용해서 포인트컷을 작성할 수 있음 이것을 **포인트컷 표현식**이라고 부름
- 포인트컷 표현식을 지원하는 포인트컷을 적용하려면 `AspectExpressionPointcut` 클래스를 사용하면 됨
- `Pointcut` 인터페이스를 구현하는 방식은 클래스 필터와 메서드 선정을 위한 매처 두가지를 각각 제공했지만 `AspectExpressionPointcut`은 클래스와 메서드의 선정 알고리즘을 포인트컷 표현식을 이용해 한 번에 지정할 수 있게 해줌
- 포인트컷 표현식은 자바의RegEx 클래스가 지원하는 정규식처럼 간단한 문자열로 복잡한 선정조건을 쉽게 만들어낼 수 있음
- 이 포인트컷 표현식은 AspectJ라는 프레임워크에서 제공하는 것을 가져와 일부 문법을 확장해서 사용하는 것이라 AspectJ 포인트컷 표현식이라고도 부름



### 포인트컷 표현식 문법

```java
public class Target implements TargetInterface {

    @Override
    public void hello() {}

    @Override
    public void hello(String a) {}

    @Override
    public int minus(int a, int b) throws RuntimeException { return 0; }

    @Override
    public int plus(int a, int b) { return 0;}

    public void method() {}
}
```



- AspectJ 포인트컷 표현식은 포인트컷 지시자를 이용해 작성하고 가장 대표적으로 사용되는 것은 `execution()`

```
//[] 괄호는 옵션, | 는 OR 조건

execution([접근제한자 패턴] 타입패턴 [타입패턴.] 이름패턴 (타입패턴 | ".." ...) [throws 예외 패턴])

```

- 위 Target 클래스의 `minus()` 메서드의 풀 시그니처
- `public int springbook.learningtest.spring.pointcut.Target.minus(int,int) throws java.lang.RuntimeException`
  - `public` : 접근 제한자 패턴, 생략 가능
  - `int`: 리턴 값의 타입 패턴, `*` 도 가능 생략은 불가능
  - `springbook.learningtest.spring.pointcut.Target`: 패키지와 타입 이름을 포함한 클래스 타입 패턴, 생략가능. 생략하는 경우 모든 타입을 다 허용하겠다는 뜻. 패키지 이름과 클래스 또는 인터페이스 이름에 `*` 사용 가능. `..` 를 사용하면 한 번에 여러 개의 패키지를 선택할 수 있음
  - `minus` 메서드 이름 패턴. 생략 불가능. 모든 메서드를 다 선택하겠다 하면 `*` 를 사용
  - `(int,int)`: 메서드 파라미터 타입 패턴, 필수항목, 모두 다 허용하려면 `..` 을 넣고 `...` 으로 뒷부분의 파라미터 조건만 생략할 수 있음
  - `throws java.lang.RuntimeException`: 예외 이름에 대한 타입 패턴, 생략 가능



`포인트컷 적용 예제`

```java
//int 타입의 리턴 값, minus라는 메서드 이름, 두 개의 int 파라미터를 가진 모든 메서드를 선정하는 포인트컷 표현식
execution(int minus(int,int))

//리턴 타입은 상관 없이 minus라는 메서드 이름, 두 개의 int 파라미터를 가진 모든 메서드를 선정하는 포인트컷 표현식
execution(* minus(int, int))

//리턴 타입과 파라미터의 종류, 개수에 상관없이 minus라는 메서드 이름을 가진 모든 메서드를 선정하는 포인트컷 표현식
execution(* minus(..))

//리턴 타입, 파라미터, 메서드 이름에 상관없이 모든 메서드 조건을 다 허용하는 포인트컷 표현식
execution(* *(..))
```



### 포인트컷 표현식을 이용하는 포인트컷 적용

- `execution()` 외에도 다양한 문법과 활용 방법이 많음
- `bean()` 은 스프링 빈의 이름으로 비교함
- `annotation()`은 애너테이션 이름으로 비교함
- 포인트컷 표현식을 사용하면 로직이 짧은 문자열에 담기기 때문에 클래스나 코드를 추가할 필요가 없어서 코드와 설정이 단순해짐
- 하지만 문자열로 된 표현식이므로 런타임 시점까지 문법의 검증이나 기능이 확인되지 않는다는 단점이 있음
- 장점만큼 단점도 명확하기 때문에 충분히 학습하고 다양한 테스트로 검증한 표현식을 사용할 것



### 타입 패턴과 클래스 이름 패턴

- 포인트컷 표현식의 클래스 이름에 적용되는 패턴은 클래스 이름 패턴이 아니라 타입 패턴임
- `TestUserService` 클래스의 슈퍼타입이 `UserServiceImpl`, 구현 인터페이스가 `UserService`라면 `execution(* *..ServiceImpl.upgrade*(..))` 로 된 포인트컷 표현식을 사용하더라도 `TestUserService`가 타입 패턴의 조건을 충족할 수 있음.
- `TestUserService` 가 `UserServiceImpl` 타입으로 일치하기 때문에



## 지금까지 본 AOP 간단하게 정리하기

### 트랜잭션 서비스 추상화

- 트랜잭션코드와 비지니스코드가 섞이면 변경되는 포인트가 많아지고 관리가 힘들어지기 때문에 추상적인 작업 내용은 유지한 채로 구현 방법을 자유롭게 바꿀 수 있도록 서비스 추상화 기법을 적용해야함
- 트랜잭션 추상화란 결국 인터페이스와 DI를 통해 무엇을 하는지는 남기고, 그것을 어떻게 하는지를 분리한 것.
- 어떻게 할지(부가기능)는 더 이상 비지니스 로직 코드에는 영향을 주지 않고 독립적으로 변경할 수 있게 됨

### 프록시와 데코레이터 패턴

- 트랜잭션을 어떻게 다룰 것인가는 추상화를 통해 코드에서 제거했지만 여전히 비즈니스 로직 코드에는 트랜잭션을 적용하는 사실이 드러나있음 (메서드 분리의 한계)
- 데코레이터 패턴을 적용해서 비지니스 로직을 담은 클래스의 코드에는 전혀 영향을 주지 않으면서 트랜잭션이라는 부가기능을 자유롭게 부여할 수 있는 구조를 만들었음 
- 트랜잭션을 처리하는 코드는 일종의 데코레이터에 담겨서, 클라이언트와 비즈니스 로직을 담은 타깃 클래스 사이에 존재하도록해서 프록시 역할을 하는 트랜잭션 데코레이터가 타깃에 접근하는 구조가 됨
- 비즈니스 로직 코드는 트랜잭션과 같은 성격이 다른 코드로부터 자유로워졌고 독립적으로 로직을 검증하는 고립된 단위 테스트를 만들 수 있게 됨

### 다이내믹 프록시

- 프록시를 이용해서 비지니스 로직 코드에서 트랜잭션 코드는 모두 제거할 수 있었지만 비지니스 로직 인터페이스의 모든 메서드마다 트랜잭션 기능을 부여하는 코드를 넣어 프록시 클래스를 만드는 작업이 오히려 번거로웠음
- 프록시 클래스 없이도 프록시 오브젝트를 런타임 시에 만들어주는 jdk 다이내믹 프록시 기술을 적용해서 프록시 클래스 코드 작성의 부담도 덜고 기능 부여 코드가 여기저기 중복돼어 나타나는 문제점도 일부 해결할 수 있음

### 프록시 팩토리 빈

- 다이내믹 프록시 방식의 단점은 동일한 기능의 프록시를 여러 오브젝트에 적용할 경우 오브젝트 단위로 중복이 일어나는 문제
- 위 문제를 해결하기 위해 스프링 프록시 팩토리 빈을 사용해서 내부적으로 템플릿/콜백 패턴을 통해 어드바이스와 포인트컷이 프록시에서 분리되는 형태로 구현했음. 여러 프록시에서 공유해서 사용할 수 있게됨

### 자동 프록시 생성 방법과 포인트컷

- 트랜잭션 적용 대상이 되는 빈마다 일일이 프록시 팩토리 빈을 설정해줘야한다는 부담감이 남아있었음
- 위 문제를 해결하기 위해 스프링 컨테이너의 빈 생성 후처리 기법을 활용해 컨테이너 초기화 시점에서 자동으로 프록시를 만들어주는 방법을 도입했음
- 프록시를 적용할 대상을 일일이 지정하지 않고 패턴을 이용해 자동으로 선정할 수 있도록, 클래스를 선정하는 기능을 담은 확장된 포인트컷을 사용
- 트랜잭션 부가기능을 어디에 적용하는지에 대해 정보를 포인트컷이라는 독립적인 정보로 완전히 분리할 수 있음
- 최종적으로 포인트컷 표현식이라는 좀 더 편리하고 깔끔한 방법을 사용해서 간단한 설정으로 적용대상을 선택하도록 구성

### 부가기능의 모듈화

- 관심사가 같은 코드를 분리해 한데 모으는 것은 소프트웨어 개발의 가장 기본이 되는 원칙
- 트랜잭션 같은 부가기능은 핵심 기능과 같은 방식으로 모듈화하기가 매우 힘듦
- DI, 데코레이터 패턴, 다이내믹 프록시, 오브젝트 생성 후처리, 자동 프록시 생성, 포인트컷과 같은 기법은 이런 문제를 해결하기 위해 적용한 대표적인 방법



### AOP: 에스펙트 지향 프로그래밍

- 애스펙트란 그 자체로 애플리케이션의 핵심기능을 담고 있지는 않지만 애플리케이션을 구성하는 중요한 한가지 요소이고, 핵심기능에 부가되어 의미를 갖는 특별한 모듈을 가르킴
- 애스펙트는 부가될 기능을 정의한 코드인 어드바이스와, 어드바이스를 어디에 적용할지를 결정하는 포인트컷을 함께 갖고 있음.
- 애플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 애스펙트라는 독특한 모듈로 만들어서 설계하고 개발하는 방법을 **애스펙트 지향 프로그래밍(Aspect Oriented Programming)** 또는 AOP라고 부름 
- AOP는 OOP를 돕는 보조적인 기술이지 OOP를 완전히 대체하는 새로운 개념은 아님
- AOP는 애플리케이션을 다양한 측면에서 독립적으로 모델링하고, 설계하고 개발할 수 있도록 만들어주는 것

### 프록시를 이용한 AOP

- 독립적으로 개발한 부가기능 모듈을 다양한 타깃 오브젝트의 메서드에 다이내믹하게 적용해주기 위해 가장 중요한 역할을 맡고 있는게 바로 프록시
- 스프링 AOP는 기본적으로 다이내믹 프록시 방식을 사용함. 하지만 이 방식은 interface 타입을 확장한 구현체에만 적용이 가능하기 때문에 슈퍼타입이 없는 경우에는 CGLib 방식을 사용함
- EnableAspectJAutoProxy의 `proxyTargetClass` 로 기본 설정값을 변경할 수 있음

`EnableAspectJAutoProxy`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {

	/**
	 * Indicate whether subclass-based (CGLIB) proxies are to be created as opposed
	 * to standard Java interface-based proxies. The default is {@code false}.
	 */
	boolean proxyTargetClass() default false;

	/**
	 * Indicate that the proxy should be exposed by the AOP framework as a {@code ThreadLocal}
	 * for retrieval via the {@link org.springframework.aop.framework.AopContext} class.
	 * Off by default, i.e. no guarantees that {@code AopContext} access will work.
	 * @since 4.3.1
	 */
	boolean exposeProxy() default false;

}
```

- spring boot는 기본적으로 cglib 방식으로 프록시를 생성함 





### 바이트코드 생성과 조작을 통한 AOP

- AspectJ는 프록시를 사용하지 않는 대표적인 AOP 기술
- AspectJ는 프록시처럼 간접적인 방법이 아니라, 타깃 오브젝트를 뜯어고쳐서 부가기능을 직접 넣어주는 직접적인 방법을 사용함
- 컴파일된 타깃 클래스 파일의 자체를 수정하거나 클래스가 jvm에 로딩되는 시점을 가로채서 바이트코드를 조작하는 복잡한 방법을 사용함
- AspectJ가 프록시를 사용하지 않고 바이트코드 조작 방법을 사용하는 이유
  - 바이트코드를 조작해서 타깃 오브젝트를 수정하면 스프링과 같은 DI 컨테이너의 도움을 받아서 자동 프록시 생성 방식을 사용하지 않아도 AOP를 적용할 수 있음
  - 프록시 방식보다 훨씬 강력하고 유연한 AOP가 가능함

### AOP의 용어

- `타깃`
  - 타깃은 부가기능을 부여할 대상.
- `어드바이스`
  - 어드바이스는 타깃에게 제공할 부가기능을 담은 모듈
  - 오브젝트 레벨로 정의하기도 하지만 메서드 레벨에서 정의할 수도 있음
  - 어드바이스는 여러 종류가 있음. MethodInterceptor 터럼 메서드 호출 과정에 전반적으로 참여하지만 예외가 발생했을때만 동작하는 어드바이스처럼 메서드 호출 과정의 일부에서만 동작하는 어드바이스도 있음
- `조인 포인트`
  - 조인 포인트란 어드바이스가 적용될 수 있는 위치를 말함
  - 스프링의 프록시 AOP에서 조인포인트는 메서드의 실행 단계뿐
  - 오브젝트가 구현한 인터페이스의 모든 메서드는 조인 포인트가됨
- `포인트컷`
  - 포인트컷이란 어드바이스를 적용할 조인 포인트를 선별하는 작업 또는 그 기능을 정의한 모듈
  - 스프링 AOP의 조인 포인트는 메서드의 실행이므로 스프링의 포인트컷은 메서드를 선정하는 기능을 갖고 있음
- `프록시`
  - 프록시는 클라이언트와 타깃 사이에 투명하게 존재하면서 부가기능을 제공하는 오브젝트
  - DI를 통해 타깃 대신 클라이언트에게 주입되며, 클라이언트의 메서드 호출을 대신 받아서 타깃을 위임해주면서, 그 과정에서 부가기능을 부여함
- `어드바이저`
  - 어드바이저는 포인트컷과 어드바이스를 하나씩 갖고 있는 오브젝트
  - 어드바이저는 어떤 기능을 어디에 전달할 것인가를 알고 있는 AOP의 가장 기본이 되는 모듈
  - 어드바이저는 스프링 AOP에서만 사용되는 특별한 용어
- `애스펙트`
  - OOP의 클래스와 마찬가지로 애스펙트는 AOP의 기본 모듈임
  - 한 개 또는 그 이상의 포인트컷과 어드바이스의 조합으로 만들어짐
  - 보통 싱글톤 형태로 존재함

### AOP 네임스페이스

- 스프링의 프록시 방식 AOP를 적용하려면 최소한 네가지 빈을 등록해야함

  - 자동 프록시 생성기
    - 스프링의 DefaultAdvisorAutoProxyCreator 클래스를 빈으로 등록
    - 애플리케이션 컨텍스트가 빈 오브젝트를 생성하는 과정에 빈 후처리기로 참여함
    - 빈으로 등록된 어드바이저를 이용해서 프록시를 자동으로 생성하는 기능
  - 어드바이스
    - 부가기능을 구현한 클래스를 빈으로 등록
  - 포인트컷
    - 스프링의 `AspectExpressionPointCut`을 빈으로 등록하고 expression 프로퍼티에 포인트컷 표현식을 넣어줘야함
  - 어드바이저
    - 스프링의 `DefaultPointcutAdvisor` 클래스를 빈으로 등록해서 사용






## 트랜잭션 속성

### 트랜잭션이란?

- 트랜잭션 경계 안에서 정의되는 작업은 최소 단위의 작업
- 트랜잭션 경계 안에서 진행된 작업은 `commit()` 을 통해 모두 성공하든지 아니면 `rollback()`을 통해 모두 취소되어야함

### 트랜잭션 전파방식

- 트랜잭션 전파란 트랜잭션의 경계에서 이미 진행중인 트랜잭션이 있을 때 또는 없을 때 어떻게 동작할 것인가를 결정하는 방식
- `PROPAGATION_REQUIRED`
  - 진행중인 트랜잭션이 없으면 새로 시작하고 이미 시작된 트랜잭션이 있으면 이에 참여함
  - 가장 많이 사용되는 트랜잭션 전파 속성
  - DefaultTransactionDefinition의 트랜잭션 전파 속성은 PROPAGATION_REQUIRED
- `PROPAGATION_REQUIREDS_NEW`
  - 항상 새로운 트랜잭션을 실행함
  - 독립적인 트랜잭션이 보장되어야 하는 코드에 적용 가능
- `PROPAGATION_NOT_SUPPORTED`
  - 트랜잭션 없이 동작하도록 만듦. 진행중인 트랜잭션이 있더라도 무시함
  - 특정 메서드가 AOP 대상이 되지 않도록 하기 위해 사용될 수 있음
- 그 외 모든 트랜잭션 전파 옵션
  - [https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html)

### 트랜잭션 격리 수준

- 모든 DB 트랜잭션은 격리수준을 갖고 있어야 함
- `DefaultTransactionDefinition`에 설정된 격리수준은 `ISOLATION_DEFAULT`, 이는 DataSource에 설정된 디폴트 격리수준을 그대로 따른다는 뜻
- 기본적으로 DB나 DataSource에 설정된 디폴트 격리수준을 따르는 편이 좋지만, 특별한 작업을 수행하는 메서드의 경우는 독자적인 격리수준을 지정할 필요가 있음

### 트랜잭션 제한시간

- 트랜잭션을 수행하는 제한시간을 설정할 수 있음
- `DefaultTransactionDefinition`의 기본설정은 제한시간이 없음



### 읽기전용 트랜잭션

- 읽기전용으로 설정해두면 트랜잭션 내에서 데이터를 조작하는 시도를 막아줄 수 있음 또한 데이터 액세스 기술에 따라서 성능이 향상될 수도 있음



### TransactionIterceptor

- 스프링에는 편리하게 트랜잭션 경계 설정을 위해 `TransactionInterceptor` 가 존재함
- `TransactionInterceptor`에는 `PlatformTransactionManager`와 `Properties` 타입의 두가지 프로퍼티를 가짐
- `Properties`는 트랜잭션 속성을 정의한 프로퍼티
- 트랜잭션 속성은 `TransactionAttribute` 인터페이스로 정의됨 이 인터페이스로 트랜잭션 부가기능의 동작 방식을 모두 제어할 수 있음
- `TransactionInterceptor` 의 기본 구현체인 `DefaultTransactionAttribute` 는 런타임 예외는 롤백시키고 체크 예외를 던지는 경우에는 이것을 예외상황이라 생각하지 않고 커밋해버림

`DefaultTransactionAttribute.rollbackOn() 메서드`

```java
    public boolean rollbackOn(Throwable ex) {
        return ex instanceof RuntimeException || ex instanceof Error;
    }
```





### 포인트컷과 트랜잭션 속성의 적용 전략

- 트랜잭션 부가기능을 적용할 후보 메서드를 선정하는 작업은 포인트컷에 의해 진행됨. 그리고 어드바이스의 트랜잭션 전파 속성따라서 메서드별로 트랜잭션의 적용 방식이 결정됨 

- 쓰기 작업이 없는 단순한 조회 작업을 하더라도 모두 트랜잭션을 적용하는게 좋음. 조회의 경우에는 읽기전용으로 트랜잭션 속성을 설정해두면 그만큼 성능 향상을 가져올 수 있음. 제한시간도 지정 가능. 격리 수준에 따라 조회도 반드시 트랜잭션 안에서 진행해야할 경우도 있음

- `프록시 방식 AOP는 같은 타깃 오브젝트 내의 메서드를 호출할 때는 적용되지 않는다`

  - 타깃 오브젝트가 자기 자신의 메서드를 호출할 때는 프록시를 통한 부가기능이 적용되지 않음
  - 클라이언트가 인터페이스 타입으로 주입된 빈을 사용해야 프록시를 통한 부가기능을 사용할 수 있기 때문
  - 타깃 안에서 프록시를 적용할 수 있는 방법은 AspectJ와 같은 타깃의 바이트코드를 직접 조작하는 방식으로 AOP를 적용하면 됨

- `트랜잭션 경계설정의 일원화`

  - 트랜잭션 경계설정의 부가기능을 여러 계층에서 중구난방으로 적용하는 건 좋지 않음
  - 서비스 계층을 트랜잭션이 시작되고 종료되는 경계로 지정했다면 다른 계층이나 모듈에서 DAO에 직접 접근하는 것은 차단해야 함
  - 모든 DAO요청은 서비스 계층을 지나도록 하는게 좋음

  



## 애너테이션 트랜잭션 속성과 포인트컷

### @Transactional

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
    @AliasFor("transactionManager")
    String value() default "";

    @AliasFor("value")
    String transactionManager() default "";

    Propagation propagation() default Propagation.REQUIRED;

    Isolation isolation() default Isolation.DEFAULT;

    int timeout() default -1;

    boolean readOnly() default false;

    Class<? extends Throwable>[] rollbackFor() default {};

    String[] rollbackForClassName() default {};

    Class<? extends Throwable>[] noRollbackFor() default {};

    String[] noRollbackForClassName() default {};
}
```



- `@Transactional` 애너테이션은 메서드, 클래스, 인터페이스에 사용 가능
- `@Transactional` 애너테이션을 트랜잭션 속성정보로 사용하도록 지정하면 스프링은 `@Transactional`이 부여된 모든 오브젝트를 자동으로 타깃 오브젝트로 인식함
- 사용되는 포인트컷은 `TransactionAttributeSourcePointcut`, `TransactionAttributeSourcePointcut`은 스스로 표현식과 같은 선정 기준을 갖지 않지만 `@Transactional`이 부여된 빈 오브젝트를 모두 찾아서 포인트컷의 선정 결과로 돌려줌
- `@Transactional`은 기본적으로 트랜잭션 속성을 정의하는 것이지만, 동시에 포인트컷의 자동등록에도 사용됨



### @Transactional 대체 정책

- 스프링은 `@Transactional`을 적용할 때 4단계의 대체 정책을 이용하게 해줌.
- 메서드의 속성을 확인할 때 타깃 메서드, 타깃 클래스, 선언 메서드, 선언 타입의 순서에 따라서 `@Transactional`이 적용됐는지 차례로 확인하고 가장 먼저 발견되는 속성 정보를 사용함

```java
//1
public interface Service {
    //2
    void method();
}

//3
public class ServiceImpl impletes Service {
   //4
   public void method1()
}

```

- `@Transcational` 이 적용될 수 있는 위치는 총 4곳, 번호의 역순으로 우선순위를 가짐
- 인터페이스를 사용하는 프록시 방식의 AOP가 아니라면 `@Transactional`이 무시되기 때문에 안전하게 타깃 클래스에 `@Transactional`을 두는 방법을 권장함







## 정리

- 트랜잭션 경계설정 코드를 분리해서 별도의 클래스로 만들고 비지니스 로직 클래스와 동일한 인터페이스를 구현하면 DI의 확장 기능을 이용해 클라이언트의 변경 업싱도 깔끔하게 분리된 트랜잭션 부가기능을 만들 수 있다.
- 트랜잭션처럼 환경과 외부 리소스에 영향을 받는 코드를 분리하면 비지니스 로직에만 충실한 테스트를 만들 수 있다
- 목 오브젝트를 활용하면 의존관계 속에 있는 오브젝트도 손쉽게 고립된 테스트로 만들 수 있다.
- DI를 이용한 트랜잭션의 분리는 데코레이터 패턴과 프록시 패턴으로 이해될 수 있다.
- 번거로운 프록시 클래스 작성은 JDK의 다이내믹 프록시를 사용하면 간단하게 만들 수 있다
- 다이내믹 프록시는 스태틱 팩토리 메서드를 사용하기 때문에 빈으로 등록하기 번거롭다. 따라서 팩토리 빈으로 만들어야 한다. 스프링은 자동 프록시 생성 기술에 대한 추상화 서비스를 제공하는 프록시 팩토리 빈을 제공한다.
- 프록시 팩토리 빈의 설정이 반복되는 문제를 해결하기 위해 자동 프록시 생성기와 포인트컷을 활용할 수 있다. 자동 프록시 생성기는 부가기능이 담긴 어드바이스를 제공하는 프록시를 스프링 컨테이너 초기화 시점에 자동으로 만들어준다.
- 포인트컷은 AspectJ 포인트컷 표현식을 사용해서 작성하면 편리하다.
- AOP는 OOP만으로는 모듈화하기 힘든 부가기능을 효과적으로 모듈화하도록 도와주는 기술이다.
- 스프링은 자주 사용되는 AOP 설정과 트랜잭션 속성을 지정하는 데 사용할 수 있는 전용 태그를 제공한다.
- AOP를 이용해 트랜잭션 속성을 지정하는 방법에는 포인트컷 표현식과 메서드 이름 패턴을 이용하는 방법과 타깃에 직접 부여하는 @Transactional 애터네이션을 사용하는 방법이 있다.
- @Transactional을 이용한 트랜잭션 속성을 테스트에 적용하면 손쉽게 DB를 사용하는 코드의 테스트를 만들 수 있다.

