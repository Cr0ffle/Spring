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
- 클라이언트는 `UserService` 인터페이스 타입을 바라보게 되면서 약결합 상태로 만들수 있고 때문에 유연한 확장이 가능한 구조가 됨



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
- 또는 특별한 상황에서 타깃에 대한 접근권한을 제어하기 위해 프록시 패턴을 사용할 수 도 있음 대표적으로 Collections 클래스의 `unmodifiableCollection()` 메서드

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
- 대표적으로 자바 IO 패키지의 InputStream과 OutputStream 구현 클래스는 데코레이터 패턴이 사용된 대표적인 예

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

### 리플렉션

- 다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어줌
- 리플렉션은 자바의 코드 자체를 추상화해서 접근하도록 만든 것
- 자바의 모든 클래스는 그 클래스 자체의 구성 정보를 담은 Class 타입 오브젝트를 하나씩 갖고 있음

```java
// Method 객체를 리플렉션으로 가져오기
Method method = String.class.getMethod("length");

// invoke() 메서드를 사용해서 실행시키기
int length = method.invoke(name); // int length = name.length();
```

### 

### 다이내믹 프록시 적용하기

[![7](https://github.com/sup2is/dev-note/raw/master/spring/images/toby-spring-vol1/7.jpeg)](https://github.com/sup2is/dev-note/blob/master/spring/images/toby-spring-vol1/7.jpeg)

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

- 아래와 같은 형태로 사용이 가능함

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
- 세번째 파라미터는 부가기능과 위임 관련 코드를 담고 있는 InvocationHandler 구현 오브젝트
- `InvocationHandler` 방식의 또 한가지 장점은 타깃의 종류에 상관없이도 적용이 가능함

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
-  `TransactionHandler`와 다이내믹 프록시를 스프링 DI를 통해 사용할 수 있도록 만들어야하지만 문제가 있음
- DI의 대상이 되는 다이내믹 프록시 오브젝트는 일반적인 스프링 빈으로 등록할 방법이 없음
  - 스프링은 내부적으로 리플렉션 api를 이용해서 빈 정의에 나오는 클래스 이름을 가지고 빈 오브젝트를 생성하는데 다이내믹 프록시 오브젝트는 이런식으로 프록시 오브젝트가 생성되지 않는다는 점.
  - 클래스 자체도 내부적으로 다이내믹하게 새로 정의해서 사용하기 때문
  - 프록시 오브젝트의 클래스 정보를 미리 알아내서 스프링 빈에 정의할 방법이 없음
  - 다이내믹 프록시의 인스턴스는 `newProxyInstance()` 라는 스태틱 팩토리 메서드를 통해서만 만들 수 있음

### 다이내믹 프록시를 위한 팩토리 빈





















