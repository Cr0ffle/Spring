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



