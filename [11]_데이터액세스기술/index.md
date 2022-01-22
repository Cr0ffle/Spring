# 02. 데이터 액세스 기술

### DAO 패턴

- 데이터 액세스 계층은 DAO 패턴이라 불리는 방식으로 분리하는 것이 원칙이다
- DAO 패턴의 장점 : DAO를 이용하는 서비스 계층의 코드를 기술이나 환경에 종속되지 않은 순수한 POJO로 개발할 수 있다

### DAO의 장점을 누리기 위해 지켜야 할 사항들

- 인터페이스의 DI
    - DAO는 인터페이스를 이용해 접근하고 DI되도록 해야 한다
    - DAO 인터페이스에는 구체적인 데이터 액세스 기술과 관련된 어떤 API나 정보도 노출하지 않는다
    - 특정 데이터 액세스 기술에서만 의미있는 DAO 메소드의 이름은 피한다
    (예 : JPA의 persist, merge같은 네이밍은 피하고, add나 update같은 이름을 선택한다)
- 예외처리
    - 데이터 액세스 중에 발생하는 예외는 대부분 복구할 수 없다
    - 따라서 DAO밖으로 던져질 때는 런타임 예외여야 한다
    - DAO 메소드 선언부에 throws SQLException과 같은 내부 기술을 드러내는 예외를 직접 노출해선 안된다
    - 스프링은 특정 기술이나 DB의 종류에 상관없이 일관된 의미를 갖는 데이터 예외 추상화를 제공하고,
    각 기술과 DB에서 발생하는 예외를 스프링의 데이터 예외로 변환해주는 변환 서비스를 제공한다

### DataSource

- 보통 미리 정해진 개수만큼 DB 커넥션을 풀에 준비해두고, 어플리케이션이 요청할 때마다 풀에서 꺼내 하나씩 할당해주고 다시 돌려받아서 풀에 넣는 식의 풀링 기법을 이용한다
- DBCP
    - DataBase Connection Pool
    - DB와 커넥션을 맺고 있는 객체를 관리하는 역할을 한다
    
    ![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled.png)
    
    - 컨테이너가 실행되면서 DB와 미리 connection을 맺어 pool에 저장해두었다가,
    클라이언트 요청이 오면 connection을 빌려주고,
    처리가 끝나면 다시 connection을 반납받아 pool에 저장한다
- 스프링에서 사용되는 주요 DataSource
    - 테스트를 위한 DataSource
        - SimpleDriverDataSource
        - SingleConnectionDataSource
    - 오픈소스 또는 상용 DB 커넥션 풀
        - 웹 어플리케이션 서버로 상용제품을 사용한다면 보통 제조사에서 제공하는 커넥션 풀 구현체를 사용한다
        - 그 외의 오픈소스 라이브라리로는 아파치의 Commons DBCP, Tomcat-JDBC, HikariCP 등이 있다

### JDBC

- JDBC는 모든 자바의 데이터 액세스 기술의 근간이 된다
- 최신 ORM 기술도 내부적으로는 DB와의 연동을 위해 JDBC를 이용한다
    
    ![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled%201.png)
    
- 그만큼 안정적이고 유연한 기술이지만, 로우레벨 기술로 취급된다
- 스프링 JDBC를 이용하면 다음과 같은 작업을 템플릿이나 스프링 JDBC가 제공하는 오브젝트에게 맡길 수 있다
    - Connection 열기/닫기
        - Connection과 관련된 모든 작업은 스프링 JDBC가 필요한 시점에서 알아서 진행해준다
        - 진행 중에 예외가 발생했을 때도 문제없이 열린 모든 Connection 오브젝트를 닫아준다
    - Statement 준비/닫기
        - SQL 정보가 담긴 Statement/PreparedStatement를 생성하고 필요한 준비작업을 해준다
        - 스프링 JDBC가 Statement를 준비하는 동안에 필요한 정보(파라미터바인딩에 사용할 정보)를 준비해도는 건 개발자의 책임이다
    - Statement 실행
        - SQL이 담긴 Statement를 실행한다
    - ResultSet 루프
        - ResultSet에 담긴 쿼리 실행 결과가 한건 이상이라면 루프를 돌면서 각각의 로우를 처리해준다
        - ResultSet 각 로우의 내용을 어떻게 오브젝트에 담을 것인지는 루프 안에서 실행되는 콜백으로 만들어 템플릿에 제공해주면 된다
    - 에러처리와 변환
        - JDBC 작업 중 발생하는 모든 예외는 스프링 JDBC의 예외 변환기가 처리해준다
        - 체크 예외인 SQLException을 런타임 예외인 DataAccessException 타입으로 변환해준다
    - 트랜잭션 처리
        - 스프링 JDBC는 트랜잭션 동기화 기법을 이용해 선언적 트랜잭션 기능과 맞물려서 돌아간다
        - 트랜잭션을 시작한 후에 스프링 JDBC 작업을 요청하면 진행중인 트랜잭션에 참여하고, 트랜잭션이 없는 채로 호출된다면, 새로운 트랜잭션을 만들어 사용한다

### JPA

- JPA는 Java Persistent API의 약자
- 영속성 관리와 ORM을 위한 표준 기술
- ORM
    - Object Relational Mapping, 객체-관계 매핑
    - The Object-Relational Impedance Mismatch
        - Granularity(세분성)
            - 경우에 따라 데이터베이스에 있는 해당 테이블 수보다 더 많은 클래스를 가진 객체 모델을 가질 수 있다
            - 데이터베이스에서는 하나의 테이블이어도, 객체로는 두 개의 클래스로 나눌 수 있다
        - Inheritance(상속)
            - RDBMS는 상속의 개념이 없다
        - Identity(일치)
            - RDBMS는 ‘sameness’라는 하나의 개념을 정확히 정의하고, 자바에서는 객체 식별(identity)과 동일성(equality)을 모두 정의한다
            - RDBMS에서는 PK가 같으면 서로 동일한 record로 정의한다
            하지만 자바에서는 주솟값이 같거나 내용이 같은 경우를 구분하여 정의한다
        - Associations(연관성)
            - 객체지향 언어는 객체 참조(reference)를 사용하는 연관성을 나타내고,
            RDBMS는 연관성을 ‘외래키’로 나타낸다
        - Navigation(탐색/순회)
            - Java에서는 하나의 연결에서 다른 연결로 이동하면서 탐색/순회한다. (그래프 형태)
            - RDBMS에서는 일반적으로 SQL 쿼리 수를 최소화하고 JOIN을 통해 여러 엔터티를 로드하고 원하는 대상 엔터티를 select한다
    - 객체와 관계형 데이터베이스의 데이터를 자동으로 매핑해준다
        - 객체 지향 프로그래밍은 클래스를 사용하고, 관계형 데이터베이스는 테이블을 사용한다
        → 객체 모델과 관계형 모델 간에 불일치
        - ORM을 통해 객체 간의 관계를 바탕으로 SQL을 자동으로 생성하여 불일치를 해결한다
    - 데이터베이스 데이터 ← 매핑→ Object 필드
        - 객체를 통해 간접적으로 데이터베이스 데이터를 다룬다
    - 장점
        - 비즈니스 로직에 더 집중할 수 있다
            - 객체 지향적인 코드로 인해 더 직관적이고 비즈니스 로직에 더 집중할 수 있게 도와준다
            - ORM을 이용하면 SQL Query가 아닌 직관적인 메서드로 데이터를 조작할 수 있어 개발자가 객체 모델로 프로그래밍하는 데 집중할 수 있도록 도와준다
            - SQL의 절차적이고 순차적인 접근이 아닌 객체 지향적인 접근으로 인해 생산성이 증가한다
        - DBMS에 대한 종속성이 줄어든다
            - 객체 간의 관계를 바탕으로 SQL을 자동으로 생성하기 때문에 RDBMS의 데이터 구조와 Java의 객체지향 모델 사이의 간격을 좁힐 수 있다
            - 대부분 ORM 기술은 DB에 종속적이지 않다
            - 프로그래머는 Object에 집중함으로 극단적으로 DBMS를 교체하는 거대한 작업에도 비교적 적은 리스크와 시간이 소요된다
        - 객체모델과 관계형 모델의 불일치를 해소한다
            - ORM을 사용하는 개발자는 모든 데이터를 오브젝트 관점으로만 본다
            - 오브젝트와 RDB 사이에 존재하는 개념과 접근 방법, 성격의 차이 때문에 요구되는 불편한 작업을 제거해준다
            → 자바 개발자가 오브젝트를 가지고 정보를 다루면, ORM 프레임워크가 이를 RDB에 적절한 형태로 변환해주거나 그 반대로 RDB에 저장되어 있는 정보를 자바 오브젝트가 다루기 쉬운 형태로 변환해준다
    - 단점
        - 사용하기는 편하지만 설계는 매우 신중하게 해야한다
        - 프로젝트의 복잡성이 커질경우 난이도 또한 올라갈 수 있다
        - 잘못 구현된 경우에 속도 저하 및 심각할 경우 일관성이 무너지는 문제점이 생길 수 있다
        - 특정 DBMS의 고유 기능을 이용하기 어렵다
        - 프로시저가 많은 시스템에선 ORM의 객체 지향적인 장점을 활용하기 어렵다

### EntityManagerFactory

- JPA 퍼시스턴스 컨텍스트에 접근하고 엔티티 인스턴스를 관리하려면 JPA의 핵심 인터페이스인 EntityManager를 구현한 오브젝트가 필요하다
- 어떤 방식을 사용하든 반드시 EntityManagerFactory를 빈으로 등록해야 한다
- 스프링에서 EntityManagerFactory를 빈으로 등록하는 방법
- LocalEntityManagerFactoryBean
    - JPA 스펙의 JavaSE 기동방식을 이용해 EntityManagerFactory를 생성한다
    - 단점
        - 스프링의 빈으로 등록한 DataSource를 사용할 수 없다
        - 스프링이 제공하는 바이트코드 위빙 기법도 적용할 수 없다
        - JPA 프레임워크가 직접 제공하는 확장 기능을 사용할 수는 있다. 하지만, 스프링에서 제어할 수는 없다
- Java EE 5 서버가 제공하는 EntityManagerFactory
    - JNDI를 통해 서버가 제공하는 EntityManager와 EntityManagerFactory를 제공받을 수 있다
    - 또 서버의 JTA를 이용해 트랜잭션 관리 기능을 활용할 수 있다
    - 스프링에서는 DAO에서 사용할 수 있도록 EntityManagerFactory를 JNDI 검색을 통한 빈 등록 기능을 이용할 수 있다
    - 하지만 단지 컨테이너가 관리하는 EntityManager 방식을 사용하기 위해서라면 굳이 이 방법을 사용할 필요는 없다
- LocalContainerEntityManagerFactoryBean
    - 스프링이 직접 제공하는 컨테이너 관리 EntityManager를 위한 EntityManagerFactory를 만들어준다
    - 이 방법을 이용하면 JavaEE 서버에 배치하지 않아도 컨테이너에서 동작하는 JPA의 기능을 활용할 수 있다
    - 또한, 스프링이 제공하는 일관성 있는 데이터 액세스 기술의 접근 방법을 적용할 수 있다
    - 스프링의 JPA 확장 기능도 활용할 수 있다
    - 프로퍼티
        - persistenceUnitName
            - 하나 이상의 퍼시스턴스 유닛이 정의될 수 있다
        - jpaProperties, jpaPropertyMap
            - 예 : 바이트코드 위빙을 사용하지 않겠다고 지정할 수 있음
        - jpaVendorAdapter
            - 구현 벤더별로 다른 설정 프로퍼티를 사용해야 하는 경우가 많다
            (각 JPA 구현 제품별로 다양한 확장 기능과 자체적인 설정 프로퍼티를 가지고 있다)
            - JPA 구현 벤더별로 다르게 지정되는 포로퍼티나 설정을 JpaVenderAdapter를 이용하면 스프링이 정의한 표준 프로퍼티를 이용해 지정할 수 있다
            각 JPA 벤더별로 만들어진 전용 어댑터를 쓰기만 하면 된다

### EntityManager

- JPA의 핵심 프로그래밍 인터페이스는 EntityManager다
- EntityManager 오브젝트를 가져올 수 있으면 JPA의 모든 기능을 이용할 수 있다
- JPA DAO에서 EntityManager를 사용하는 방법
    - JpaTemplate
        - 스프링의 여타 데이터 액세스 기술에서 사용되는 것과 비슷한 스타일의 프로그래밍 기법을 제공해줄 뿐.
        - JPA API 자체를 사용하는 데는 별로 익숙하지 않고 스프링의 템플릿 방식의 접근이 편하게 느껴진다면 JpaTemplate을 사용할 수 있다
        - JPA API보다 나은점 : JPA의 예외를 스프링의 데이터 액세스 예외 추상화 클래스 계층인 DataAccessException의 예외로 변환해준다
    - 어플리케이션 관리 EntityManager
        - 컨테이너 대신 어플리케이션 코드가 관리하는 EntityManager를 이용한다
        - 컨테이너가 관리하지 않는 EntityManager이므로 트랜잭션은 직접 시작하고 종료해야 한다
        
        ```java
        public class MemberDao {
        		@PersistenceUnit EntityManagerFactory emf;
        }
        ```
        
        - EntityManagerFactory를 직접 이용하는 방법은 실제로 자주 쓰이지 않는다
        - 매번 코드에 의해 EntityManager가 생성되고 트랜잭션도 코드에 의해 관리되므로 매우 번거롭다
        - EntityManagerFactory는, EntityManagerFactory 인터페이스에서 제공하는 QueryBuilder, Metamodel, PersistenceUnitUtil을 가져오거나 캐시나 프로퍼티 정보를 참조할 필요가 있을 때 사용하는 것이 바람직하다
    - 컨테이너 관리 EntityManager
        - 가장 대표적인 방법
        - 컨테이너가 제공하는 EntityManager를 직접 제공받아서 사용한다
        - DAO가 컨테이너로부터 EntityManager를 직접 주입받으려면 JPA의 @PersistenceContext어노테이션을 사용해야 한다
        - EntityManager는 스프링의 빈으로 등록되지 않는다. 빈으로 등록한 것은 LocalContainerEntityManagerFactoryBean
        - 따라서 @Autowired같은 스프링의 DI 방법으로는 EntityManager를 주입받을 수 없다
        - 아래처럼 LocalContainerEntityManagerFactoryBean에 의해 등록된 EntityManagerFactory로부터 컨테이너가 관리하는 EntityManager를 직접 DAO에 주입받아서 사용한다
        
        ```java
        public class MemberDao {
        		@PersistencContext EntityManager em;
        }
        ```
        

### JPA 예외 변환

- JPA의 표준 예외
    - JPA의 표준 예외들은 javax.persistence.PersistenceException의 자식 클래스이다
    그리고 이 클래스는 RuntimeException의 자식이다
    → 즉, JPA의 예외는 런타임 예외를 발생시킨다
    - 런타임 예외를 발생시키기 때문에 불필요한 try/catch 블록이나 throws 선언이 필요 없다
    - 크게 2가지로 나뉜다
        - 트랜잭션 롤백을 표시하는 예외
            - 심각한 예외이기 때문에 복구하면 안된다
            - 강제로 커밋해도 트랜잭션이 커밋되지 않고 대신 RollbackException 예외가 발생한다
            
            ![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled%202.png)
            
        - 트랜잭션 롤백을 표시하지 않는 예외
            - 심각한 예외가 아니기 때문에 개발자가 판단해서 트랜잭션을 커밋/롤백한다
            
            ![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled%203.png)
            
- JPA API를 사용하는 방법은 자동으로 예외 변환이 일어나지 않는다
- JPA 예외 변환
    - 서비스 계층에서 JPA 예외를 직접 사용하게 되면 JPA라는 기술에 의존하게 된다
    - 스프링 AOP를 이용하면, JpaTemplate을 사용하지 않고 JPA API를 직접 이용하는 경우에도 JPA 예외를 스프링의 DataAccessException 예외로 전환시킬 수는 있다
    그렇지만, JPA의 예외 자체가 분류하기 힘든 예외로 던져지기 때문에 이를 적절히 해석해서 변환하기가 어렵다
    JPA 스펙이 정의하는 예외는 제한적이라, JPA 프로그래밍의 특징에 따라 정의된 일부 예외를 제외하면, 대부분의 DB예외는 벤더가 정의한 예외나 시스템 예외 등으로 포장돼서 전달된다
    - 아래는 추상화된 예외들이다
    
    ![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled%204.png)
    

### 하이버네이트

- 가장 크게 성공한 오픈소스 ORM 프레임워크
- JPA vs 하이버네이트
    - JPA는 자바 어플리케이션에서 관계형 데이터베이스를 사용하는 방식을 정의한 인터페이스 ⇒ **말 그대로 인터페이스!!**
    - JPA는 특정 기능을 하는 라이브러리가 아니다
    자바 어플리케이션에서 관계형 데이터베이스를 어떻게 사용해야 하는지를 정의하는 한 방법일 뿐이다
    - JPA의 핵심이 되는 `EntityManager`는 `javax.persistence.EntityManager` 라는 파일에 `interface`로 정의되어 있다
- Hibernate는 JPA의 구현체
    - 위에서 언급한 `javax.persistence.EntityManager`와 같은 인터페이스를 직접 구현한 라이브러리
        
        ![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled%205.png)
        

### 트랜잭션

- 스프링은 데이터 액세스 기술과 트랜잭션 서비스 사이의 종속성을 제거하고, 스프링이 제공하는 트랜잭션 추상 계층을 이용해서 트랜잭션 기능을 활용하도록 만들어준다
- JPA는 반드시 트랜잭션 안에서 동작하도록 설계되어 있다

### PlatformTransactionManager

- 스프링 트랜잭션 추상화의 핵심 인터페이스
- 모든 스프링의 트랜잭션 기능과 코드는 이 인터페이스를 통해서 로우레벨의 트랜잭션 서비스를 이용할 수 있다
- 스프링에서는 시작과 종료를 트랜잭션 전파 기법을 이용해 자유롭게 조합하고 확장할 수 있다
- 트랜잭션 속성에 따라 새로 시작하거나, 진행중인 트랜잭션에 참여하거나, 진행 중인 트랜잭션을 무시하고 새로운 트랜잭션을 만드는 식으로 상황에 따라 다르게 동작한다
- 종류
    
    ![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled%206.png)
    
    - DataSourceTrasactionManager
        - JDBC 기반 트랜잭션 매니저
        - Connection의 트랜잭션 API를 이용해서 트랜잭션을 관리한다
        - DataSourceTrasactionManager가 사용할 DataSource는 getConnection() 호출될 대마다 매번 새로운 Connection을 돌려줘야 한다
        - 어플리케이션 코드에서 트랜잭션 매니저가 관리하는 Connection을 가져오려면 DataSourceUntils.getConnection()을 사용해야 한다
    - JpaTransactionManager
        - JPA를 이용하는 DAO에는 JpaTransactionManager를 사용한다
        - JTA로 트랜잭션 서비스를 이용하는 경우에는 JpaTransactionManager가 필요 없다
        - JpaTransactionManager는 DataSourceTrasactionManager가 제공하는 DataSource 레벨의 트랜잭션 관리 기능을 동시에 제공한다
        - 따라서 JpaTransactionManager를 사용하면서 동시에 트랜잭션이 적용된 JDBC DAO를 사용할 수도 있다
    - HibernateTrasactionManager
        - JpaTransactionManager와 마찬가지로 DataSource 레벨의 트랜잭션 기능도 동시에 제공한다
    - JtaTransactionManger
        - 하나 이상의 DB 또는 트랜잭션 리소스가 참여하는 글로벌 트랜잭션을 적용하려면 JTA를 이용해야 한다
        - JTA는 여러 개의 트랜잭션 리소스에 대한 작업을 하나의 트랜잭션으로 묶을수도 있고, 여러 대의 서버에 분산되어 진행되는 작업을 트랜잭션으로 연결해주기도 한다

### 경계설정 전략

- 트랜잭션의 시작과 종료가 되는 경계는 보통 서비스 계층 오브젝트의 메소드다
- 트랜잭션 경계를 설정하는 방법은 코드에 의한 프로그램적인 방법과, AOP를 이용한 선언적인 방법으로 구분할 수 있다
    - 코드에 의한 트랜잭션 경계설정
        - 스프링의 트랜잭션 매니저는 모두 PlatformTransactionManager를 구현하고 있다
        - 따라서 이 인터페이스로 현재 등록되어 있는 트랜잭션 매니저 빈을 가져올 수 있다면 트랜잭션 매니저의 종류에 상관 없이 동일한 방식으로 트랜잭션을 제어하는 코드를 만들 수 있다
        - PlatformTransactionManager의 메소드를 직접 사용해도 되지만, try/catch블록을 써야 하는 번거로움이 있다
        - PlatformTransactionManager의 메소드를 직접 사용하는 대신 템플릿/콜백 방식의 TransactionTemplate을 이용하면 편리하다
    - 선언적 트랜잭션 경계설정
        - 선언적 트랜잭션을 이용하면 코드에는 전혀 영향을 주지 않으면서 특정 메소드 실행 전후에 트랜잭션이 시작되고 종료되거나 기존 트랜잭션에 참여하도록 만들 수 있다
        - AOP를 이용해 트랜잭션 기능을 부여 하는 방법
            - aop와 tx 네임스페이스
                - 설정 파일에 명시적으로 포인트컷과 어드바이스를 정의한다
                - 장점 : 코드에 전혀 영향을 주지 않고 일괄적으로 트랜잭션을 적용하거나 변경할 수 있다
                - 단점 : 복잡해보인다
            - @Transactional : 트랜잭션이 적용될 타킷 인터페이스나 클래스, 메소드 등에 @Transacitonal 어노테이션을 부여해서 트랜잭션 대상으로 지정하고 트랜잭션 속성을 제공한다
                - 장점 : aop와 tx 스키마 태그를 사용하는 경우보다 훨씬 세밀한 설정이 가능하다
                - 단점 : 일일이 대상 인터페이스나 클래스, 메소드에 부여하는건 상대적으로 번거로운 작업이 된다

### 트랜잭션 속성

- 트랜잭션 경계는 여섯가지 속성을 갖는다
- readOnly
    - 성능을 최적화하기 위해 사용한다
        - 성능이 왜 최적화?
            - 하이버네이트의 Session Flush Mode가 MANUAL로 설정된다
            - MANUAL로 설정이 되면 강제로 플러시를 호출하지 않는 한 플러시가 일어나지 않아 성능이 향상된다
    - 특정 트랜잭션 작업 안에서 쓰기 작업이 일어나는 것을 의도적으로 방지하기 위해 사용한다
        - 일반적으로는 읽기 전용 트랜잭션이 시작된 이후 INSERT, UPDATE, DELETE같은 쓰기 작업이 진행되면 예외가 발생한다
- timeout
    - 트랜잭션에 제한시간을 지정한다
- rollbackFor / rollbackForClassName
    - 선언적 트랜잭션에서는 런타임 예외가 발생하면 롤백한다
    - 체크 예외지만 롤백 대상으로 삼아야 하는 것이 있다면 rollbackFor를 지정한다
- noRollbackFor / noRollbackForClassName
    - 롤백 대상인 런타임 예외를 트랜잭션 커밋 대상으로 지정한다
- propagation
    - 트랜잭션을 시작하거나 기존 트랜잭션에 참여하는 방법을 결정하는 속성이다
    - 모든 속성이 모든 종류의 트랜잭션 매니저와 데이터 액세스 기술에서 다 지원되지는 않는다
    - 스프링이 지원하는 트랜잭션 전파 속성은 다음 여섯가지다
        - REQUIRED
            - 디폴트 속성
            - 모든 트랜잭션 매니저가 지원한다
            - 미리 시작된 트랜잭션이 있으면 참여하고 없으면 새로 시작한다
        - SUPPORTS
            - 이미 시작된 트래잭션이 있으면 참여하고 그렇지 않으면 트랜잭션 없이 진행한다
        - MANDATORY
            - 이미 시작된 트랜잭션이 있으면 참여하고, 없으면 예외를 발생시킨다
        - REQUIRES_NEW
            - 매번 새로운 트랜잭션을 시작한다
            - 이미 진행중인 트랜잭션이 있으면 트랜잭션을 잠시 보류시킨다
        - NOT_SUPPORTED
            - 트랜잭션을 사용하지 않는다
            - 이미 진행중인 트랜잭션이 있다면 보류시킨다
        - NEVER
            - 트랜잭션을 사용하지 않도록 강제한다
            - 이미 진행중인 트랜잭션도 존재하면 안된다 (예외 발생)
        - NESTED
            - 중첩 트랜잭션
            - 트랜잭션 안에 다시 트랜잭션을 만든다
            - 먼저 시작된 부모 트랜잭션의 커밋과 롤백에는 영향을 받지만, 자신의 커밋과 롤백은 부모 트랜잭션에게 영향을 주지 않는다
- isolation
    - 트랜잭션 격리수준
    - 여러 트랜잭션이 진행될 때에, 트랜잭션의 작업 결과를 여타 트랜잭션에게 어떻게 노출할 것인지를 결정한다
    - READ_UNCOMMITTED (level 0)
        - 가장 낮은 격리수준
        - 하나의 트랜잭션이 커밋되기 전에 그 변화가 다른 트랜잭션에 그대로 노출된다
        - 트랜잭션 A가 특정 데이터를 변경하고 있는 중에(커밋하지 않은 상태) 트랜잭션 B가 read하면 트랜잭션 A가 변경한 데이터를 읽는다
        - 장점 : 성능을 극대화할 수 있다
        - 단점 : 데이터의 일관성이 조금 떨어진다
        - dirty read가 발생한다
        
        ![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled%207.png)
        
    - READ_COMMITTED (level 1)
        - 실제로 가장 많이 사용되는 격리수준
        - 다른 트랜잭션이 커밋하지 않은 정보는 읽을 수 없다 (dirty read 방지)
        대신 하나의 트랜잭션이 읽은 로우를 다른 트랜잭션이 수정할 수 있다
        - Non-Repeatable Read 발생
            - 한 트랜잭션 내에서 같은 쿼리를 두번 수행할 때, 그 사이에 다른 트랜잭션이 값을 수정 또는 삭제함으로써 두 쿼리가 상이하게 나타나는 비 일관성이 발생하는 것
        - 트랜잭션 A가 특정 컬럼 데이터를 변경하고 있는 중에(커밋하지 않은 상태) 트랜잭션 B가 read하면 트랜잭션 A가 변경하기 전 데이터를 읽어온다
        만약 트랜잭션 A가 데이터 변경 후 커밋하게 되면 트랜잭션 B는 변경된 데이터를 읽어온다
        
        ![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled%208.png)
        
    - REPEATABLE_READ (level 2)
        - 항상 일관성 있는 데이터 읽기를 보장한다
        - 하나의 트랜잭션이 읽은 로우를 다른 트랜잭션이 수정할 수 없다
        - (새로운 로우를 추가하는 것은 제한하지 않는다)
        - Phantom Read가 발생한다
            - 한 트랜잭션 안에서 일정범위의 레코드를 두번 이상 읽을 때, 첫 번재 쿼리에서 없던 유령 레코드가 두번째 쿼리에서 나타난다
        
        ![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled%209.png)
        
    - SERIALIZABLE (level 4)
        - 가장 강력한 트랜잭션 격리수준
        - 여러 트랜잭션이 같은 테이블의 정보를 액세스하지 못한다
        - 가장 성능이 떨어진다
        - 극단적으로 안전한 작업이 필요한 경우가 아니라면 자주 사용되지 않는다
    - DEFAULT
        - 사용하는 데이터 액세스 기술 또는 DB 드라이버의 디폴트 설정을 따른다
            - Mysql : REPEATABLE_READ
            - Oracle : READ_COMMITTED
            - h2 : READ_COMMITTED

### JTA를 이용한 글로벌/분산 트랜잭션

- 한개 이상의 DB를 사용할 때, 하나의 트랜잭션 안에서 동작하게 하려면 서버가 제공하는 트랜잭션 매니저를 JTA를 통해 사용해야 한다
- JTA : Java Transaction APIs
- 데이터베이스 여러개를 이용할 경우 트랜잭션을 제어하기 위한 목적으로 사용한다
- JTA는 플랫폼마다 상이한 트랜잭션 매니저들과 어플리케이션들이 상호작용할 수 있는 인터페이스를 정의하고 있다
→ 트랜잭션 처리가 필요한 어플리케이션이 특정 벤더의 트랜잭션 매니저에 의존할 필요가 없음을 의미한다
- JTA의 구현체를 사용할 때에는 주의를 기울여야 한다
‘J2EE 호환됨’이라고 검증을 받은 어플리케이션 서버들도 트랜잭션 관리를 제대로 지원하지 않거나 가상적으로만 지원할 수도 있다

### 스프링 데이터 프로젝트

- 기본 데이터저장소에 대한 특성은 유지하고, 데이터 액세스 방법에 대하여 친숙하고 익숙한 접근방법을 제시하는 목적을 가진 스프링 기반 프로그래밍 모델
- 스프링 데이터를 이용하면 데이터 액세스 기술, 관계/비관계형 데이터베이스, 맵리듀스 프레임워크, 클라우드 기반 데이터 서비스를 쉽게 적용할 수 있다
- 어떤 종류의 데이터저장소라도 하나의 도메인 오브젝트, find 메소드, 정렬/페이징에 대한 CRUD 동작이 필요하다
- 스프링 데이터는 Repository라는 제네릭한 인터페이스를 제공하여 공통된 연산에 implementation을 동적으로 제공한다
- 각 데이터저장소는 스프링데이터의 Repository를 구현하여 자신의 데이터저장소에 맞는 Repository가 제공된다 (예를 들면 JpaRepositroy, MongoRepository)
- 클라이언트는 자신이 사용하려는 Repository를 상속하여 각 저장소에서 정의한 네이밍 컨벤션에 맞게 메소드만 선언하기만 하면 스프링 데이터가 런타임 시, 이름에 맞는 적절한 구현 내용을 제공한다
- 여러 서브 프로젝트들이 있다
    
    ![스크린샷 2021-12-05 오전 9.41.04.png](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2021-12-05_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_9.41.04.png)
    

![Untitled](02%20%E1%84%83%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%90%E1%85%A5%20%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%89%E1%85%A6%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20e326b593d484470698778f5d11e17e4d/Untitled%2010.png)

### 정리

- DAO 패턴을 이용하면 데이터 액세스 계층과 서비스 계층을 깔끔하게 분리할 수 있다
- 데이터 액세스 기술은 자유롭게 변경해서 사용할 수가 있다
- 스프링 JDBC는 JDBC DAO를 템플릿/콜백 방식을 이용해서 편리하게 작성할 수 있게 해준다
- JPA와 하이버네이트를 이용하는 DAO에서는 템플릿/콜백과 자체적인 API를 선택적으로 사용할 수 있다
- 스프링은 하나 이상의 데이터 액세스 기술로 만들어진 DAO를 같은 트랜잭션 안에서 동작하도록 만들어준다

### 참고

[https://d2.naver.com/helloworld/5102792](https://d2.naver.com/helloworld/5102792)

[https://ddd4117.github.io/2021/04/jpa-15장-고급-주제와-성능-최적화/](https://ddd4117.github.io/2021/04/jpa-15%EC%9E%A5-%EA%B3%A0%EA%B8%89-%EC%A3%BC%EC%A0%9C%EC%99%80-%EC%84%B1%EB%8A%A5-%EC%B5%9C%EC%A0%81%ED%99%94/)

[https://suhwan.dev/2019/02/24/jpa-vs-hibernate-vs-spring-data-jpa/](https://suhwan.dev/2019/02/24/jpa-vs-hibernate-vs-spring-data-jpa/)

[https://deveric.tistory.com/86](https://deveric.tistory.com/86)

[https://wannaqueen.gitbook.io/spring5/spring-boot/undefined-1/39.-jta-by-ys](https://wannaqueen.gitbook.io/spring5/spring-boot/undefined-1/39.-jta-by-ys)

[https://emflant.tistory.com/99](https://emflant.tistory.com/99)