---
title: (Spring)다중 DataSource 처리
date: 2018-03-22 13:54:08
tags: [spring]
categories:
  - spring
---

# 서론

Spring Application을 만들면서 여러 `DataSource`와 `transaction`이 존재하고 하나의 transaction 내에 commit과 rollback이 잘 동작하도록 하려면 어떻게 설정해야 할까? 실제로 구현을 해본 적은 없지만 세 가지 방법이 머릿속에 떠올랐다.

- `@Transactional`의 propagation을 이용
- `spring-data-commons`의 `ChainedTransactionManager` 이용
- `JtaTransactionManager` 이용

이 방법들이 실제로 써먹을 수 있을지 확인해보려고 한다.

<!-- more -->

# 구현 1 - @Transactional의 propagation 이용

## Transactional.propagation에 대한 간단한 설명

Spring의 `@Transactional`의 `propagation` 속성으로 다음과 같은 설정을 할 수 있다. 자세한 설명을 해둔 [블로그(Rednics Blog)](http://springsource.tistory.com/136)가 있어 링크를 남긴다. [spring reference](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/transaction.html#tx-propagation)도 참조하자

- REQUIRED : 기본 설정, 진행중인 transaction이 있으면 참여, 없으면 새로 생성
- SUPPORTS : 진행중인 transaction이 있으면 참여, 없으면 transaction 없이 실행
- MANDATORY : 진행중인 transaction이 있으면 참여, 없으면 예외.
- REQUIRES_NEW : 새로운 transaction 시작. 진행중인 transaction은 보류.
- NOT_SUPPORTED : 진행중인 transaction이 있으면 보류, transaction 없이 실행.
- NEVER : transaction없이 실행. 진행중인 transaction이 있으면 예외.
- NESTED : 중첩 transaction 실행. 자식 tx은 부모 tx에게 영향을 주지 않지만, 부모는 자식에게 영향을 줌.

## 요구 사항

{% plantuml %}
[testController] --> [<memberTx>\nlogicService] : method 실행
[<memberTx>\nlogicService] --> [<memberTx>\nmemberService] : 1. insertMember
[<memberTx>\nlogicService] --> [<boardTx>\nboardService] : 2. insertBoard(예외발생!)
{% endplantuml %}
하고자 하는 일을 다이어그램으로 나타내면 위와 같다.
주로 사용하는 DataSource와 transaction이 존재하고, 거기에 부가 transaction이 참여하는 모양새다. 2번에서 예외가 발생했을 때, 2번도 rollback이 되고 1번도 같이 rollback이 되었으면 좋겠다. 

memberTx와 boardTx가 `REQUIRED`, `REQUIRES_NEW`, `NESTED`의 propagation을 가질 수 있을 때, 총 9가지 경우의 수가 나온다. 어떤 조합에서 `commit`과 `rollback`이 어떻게 실행될지 직접 구현해보겠다.

다음 단락부터 이를 구현한 예제가 나올텐데, 좀 지루한 내용이라 결과만 알고 싶다면 바로 결론으로 가자.

## 의존성

본문에서는 `spring-boot`, `mysql(docker 사용)`, `mybatis`를 사용한다.
`spring-boot`의 버전은 `2.0.0.RELEASE`다.

```gradle
dependencies {
    compile('org.springframework.boot:spring-boot-starter-web')
    compile('org.mybatis.spring.boot:mybatis-spring-boot-starter:1.3.2')
    compileOnly('org.projectlombok:lombok')
    runtime('mysql:mysql-connector-java')
    testCompile('org.springframework.boot:spring-boot-starter-test')
}
```

## SpringApplication 설정

### application.yml

```yml
spring:
  datasource:
    hikari1:
      username: multitxtest
      password: multitxtest
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:11306/multi_tx_test
    hikari2:
      url:
      username: multitxtest
      password: multitxtest
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://127.0.0.1:11307/multi_tx_test
```

### 두 개의 SqlSession 생성

#### Board용 SqlSession

```java
// board
@Configuration
@EnableConfigurationProperties(Hikari2Properties.class)
public class BoardSqlSessionConfig {

    @Bean
    public DataSource boardDataSource(Hikari2Properties properties) {
        return DataSourceCreator.createHikariDataSource(properties);
    }

    @Bean
    public PlatformTransactionManager boardTxManager(DataSource boardDataSource) {
        return new DataSourceTransactionManager(boardDataSource);
    }

    @Bean
    public SqlSessionFactory boardSqlSessionFactory(DataSource boardDataSource) throws Exception {
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(boardDataSource);
        return factoryBean.getObject();
    }

    @Bean(destroyMethod = "clearCache")
    public SqlSession boardSqlSession(SqlSessionFactory boardSqlSessionFactory) {
        return new SqlSessionTemplate(boardSqlSessionFactory);
    }

    @Bean
    public MapperFactoryBean<BoardMapper> boardMapper(SqlSessionFactory boardSqlSessionFactory) {

        MapperFactoryBean<BoardMapper> factoryBean = new MapperFactoryBean<>(BoardMapper.class);
        factoryBean.setSqlSessionFactory(boardSqlSessionFactory);
        return factoryBean;
    }
}
```

#### Member용 SqlSession

```java
// member
@Configuration
@EnableConfigurationProperties(Hikari1Properties.class)
public class MemberSqlSessionConfig {

    @Bean
    public DataSource memberDataSource(Hikari1Properties properties) { ... }

    @Bean
    @Primary
    public PlatformTransactionManager memberTxManager(DataSource memberDataSource) { ... }

    @Bean
    public SqlSessionFactory memberSqlSessionFactory(DataSource memberDataSource) throws Exception { ... }

    @Bean(destroyMethod = "clearCache")
    public SqlSession memberSqlSession(SqlSessionFactory memberSqlSessionFactory) { ... }

    @Bean
    public MapperFactoryBean<MemberMapper> memberMapper(SqlSessionFactory memberSqlSessionFactory) { ... }
}
```

#### DataSourceCreator

```java
@UtilityClass
public class DataSourceCreator {

    public DataSource createHikariDataSource(DataSourceProperties properties) {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(properties.getUrl());
        dataSource.setUsername(properties.getUsername());
        dataSource.setPassword(properties.getPassword());
        dataSource.setDriverClassName(properties.getDriverClassName());

        return dataSource;
    }
}
```

## DAO

### BoardMapper

```java
public interface BoardMapper {

    @Insert("INSERT INTO board(title, content) VALUES(#{title}, #{content})")
    @SelectKey(statement = "SELECT LAST_INSERT_ID()", keyColumn = "id", keyProperty = "id", before = false, resultType = Long.class)
    int insert(Board board);

    @Delete("TRUNCATE TABLE board")
    void truncate();
}
```

### MemberMapper

```java
public interface MemberMapper {

    @Insert("INSERT INTO member(name, age) VALUES(#{name}, #{age})")
    @SelectKey(statement = "SELECT LAST_INSERT_ID()", keyColumn = "id", keyProperty = "id", before = false, resultType = Long.class)
    int insert(Member member);

    @Delete("TRUNCATE TABLE member")
    void truncate();
}
```

## Service

### MemberService

```java
@Service
@Transactional(transactionManager = "memberTxManager")
public class MemberService {

    @Autowired
    private MemberMapper memberMapper;

    public void saveWithRequired() {
        memberMapper.insert(Member.createForTest(1));
    }

    @Transactional(transactionManager = "memberTxManager", propagation = Propagation.REQUIRES_NEW)
    public void saveWithRequiresNew() {
        memberMapper.insert(Member.createForTest(1));
    }

    @Transactional(transactionManager = "memberTxManager", propagation = Propagation.NESTED)
    public void saveWithNested() {
        memberMapper.insert(Member.createForTest(1));
    }
}
```

### BoardService

```java
@Service
@Transactional("boardTxManager")
public class BoardService {
    @Autowired
    private BoardMapper boardMapper;

    public int saveWithRequired() {
        boardMapper.insert(Board.createForTest(1));
        throw new IllegalStateException("this method throw exception");
    }

    @Transactional(transactionManager = "boardTxManager", propagation = Propagation.REQUIRES_NEW)
    public int saveWithRequiresNew() {
        boardMapper.insert(Board.createForTest(1));
        throw new IllegalStateException("this method throw exception");
    }

    @Transactional(transactionManager = "boardTxManager", propagation = Propagation.NESTED)
    public int saveWithNested() {
        boardMapper.insert(Board.createForTest(1));
        throw new IllegalStateException("this method throw exception");
    }
}
```

### LogicService

```java
@Service
@Transactional
public class LogicService {

    @Autowired
    private final MemberService memberService;
    @Autowired
    private final BoardService boardService;

    public void required_required() {
        memberService.saveWithRequired();
        boardService.saveWithRequired();
    }

    public void required_requiresNew() {
        memberService.saveWithRequired();
        boardService.saveWithRequiresNew();
    }

    public void required_nested() {
        memberService.saveWithRequired();
        boardService.saveWithNested();
    }

    public void requiresNew_required() {
        memberService.saveWithRequiresNew();
        boardService.saveWithRequired();
    }

    public void requiresNew_requiresNew() {
        memberService.saveWithRequiresNew();
        boardService.saveWithRequiresNew();
    }

    public void requiresNew_nested() {
        memberService.saveWithRequiresNew();
        boardService.saveWithNested();
    }

    public void nested_required() {
        memberService.saveWithNested();
        boardService.saveWithRequired();
    }

    public void nested_requiresNew() {
        memberService.saveWithNested();
        boardService.saveWithRequiresNew();
    }

    public void nested_nested() {
        memberService.saveWithNested();
        boardService.saveWithNested();
    }
}
```

## 결과

### LogicService에 @Transactional이 없는 경우

| member propagation | board propagation | member insert | board insert |
| - | - | - | - |
| required | required | commit | rollback |
| required | requires-new | commit | rollback |
| required | nested | commit | rollback |
| requires-new | required | commit | rollback |
| requires-new | requires-new | commit | rollback |
| requires-new | nested | commit | rollback |
| nested | required | commit | rollback |
| nested | requires-new | commit | rollback |
| nested | nested | commit | rollback |

#### 설명

logicalService에서는 transaction이 시작되지 않는다. 따라서 memberService와 boardService는 각각 별개의 transaction에서 독립적으로 실행된다. 예외가 발생한 boardService의 내용만 rollback된다.

### LogicService에 @Transactional이 있는 경우(memberTxManager)

| member propagation | board propagation | member insert | board insert |
| - | - | - | - |
| required | required | rollback | rollback |
| required | requires-new | rollback | rollback |
| required | nested | rollback | rollback |
| requires-new | required | commit | rollback |
| requires-new | requires-new | commit | rollback |
| requires-new | nested | commit | rollback |
| nested | required | rollback | rollback |
| nested | requires-new | rollback | rollback |
| nested | nested | rollback | rollback |

#### 설명

- `memberService`가 `REQUIRED`인 경우에는, `logicService`의 transaction의 영향을 받아서 rollback된다. 
- `REQUIRES_NEW`인 경우에는 새로운 transaction을 생성하기 때문에 자신의 method execution이 종료됨과 함께 commit을 해버린다.
- `NESTED`의 경우, 부모 transaction의 영향을 받기 때문에 memberService의 내용은 rollback된다.

**주의**

여기까지 테스트를 해보면, 정상적으로 동작하는 것처럼 보인다. 속지말자.

### LogicService에 @Transactional이 있는 경우(boardTxManager)

| member propagation | board propagation | member insert | board insert |
| - | - | - | - |
| required | required | commit | rollback |
| required | requires-new | commit | rollback |
| required | nested | commit | rollback |
| requires-new | required | commit | rollback |
| requires-new | requires-new | commit | rollback |
| requires-new | nested | commit | rollback |
| nested | required | commit | rollback |
| nested | requires-new | commit | rollback |
| nested | nested | commit | rollback |

#### 설명

확인하고 싶었던 부분 1이다. 서로 다른 transactionManager에서 관리하는 transaction context에 참여할 수 있을까? 결과를 확인해보니 불가능했다.

### 전체 @Transactional에서 boardTxManager를 쓰는 경우

| member propagation | board propagation | member insert | board insert |
| - | - | - | - |
| required | required | commit | rollback |
| required | requires-new | commit | rollback |
| required | nested | commit | rollback |
| requires-new | required | commit | rollback |
| requires-new | requires-new | commit | rollback |
| requires-new | nested | commit | rollback |
| nested | required | commit | rollback |
| nested | requires-new | commit | rollback |
| nested | nested | commit | rollback |

#### 설명

확인하고 싶었던 부분 2다. `boardTxManager`가 `memberDataSource`에 대한 작업을 rollback할 수 있을까? 불가능하다.

## 결론

별다른 장치 없이 `DataSource` 두 개, `TransactionManager` 두 개를 사용하면 100% 위험하다. 원하는 동작을 장담할 수 없다. 이 방법은 얼른 벗어나야 한다. 이러한 설정을 사용하면서 어디에서 버그가 나고 있는지 찾지 않길 바란다.

# 구현 2 - ChainedTransactionManager 사용

구현 2에서는 `spring-data-commons`에서 제공해주는 `ChainedTransactionManager`를 사용하려고 한다. javadoc을 읽어보면 다음과 같이 설명하고 있다.

{% blockquote ChainedTransactionManager https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html javadoc%}
PlatformTransactionManager 구현체로 transaction 생성, commit, rollback을 위임자 패턴으로 구성한다. 이 구현체를 사용하는 것의 전제되는 가정은, transaction의 rollback을 야기하는 오류는 대부분 transaction이 완료되기 전, 혹은 가장 안쪽의 PlatformTransactionManager에서 발생한다는 것이다.

이 구현체의 인스턴스는 지정된 순서대로 transaction을 시작하고, 역순으로 commit/rollback한다. 즉, transaction을 중단시킬 가능성이 가장 큰 PlatformTransactionManager가 마지막에 설정되어야 한다. commit 중에 예외를 던지는 PlatformTransactionManager는 자동으로 다른 transaction manager의 rollback을 일으킨다.
{% endblockquote %}
<br>
javadoc의 설명대로라면 원하는 구현을 할 수 있다. 하지만 한가지 유의할 점이 있다. 소스를 까보면 `for`문을 돌면서 `.getTransaction()`을 호출하는 데, 이는 성능 문제를 야기할 수 있다. 알 사람은 다 아는 `LazyConnectionDataSourceProxy`를 사용할 차례다. spring은 기본적으로 `transaction`을 미리 가져오는데, 저 `DataSource`의 구현체는 필요한 때에 `transaction`을 가져오도록 connection 호출 시점을 뒤로 미룰 수 있다. 따로 `LazyConnectionDataSourceProxy`를 상세히 설명하기 보다는, 이 구현체를 알게된 [블로그](http://kwon37xi.egloos.com/m/5364167)를 소개하겠다. 

## 의존성 추가

```gradle
dependencies {
    compile('org.springframework.data:spring-data-commons')
}
```

## Spring Application 설정

### DataSouceCreator

`LazyConnectionDataSourceProxy`를 반환하도록 변경한다.

```java
@UtilityClass
public class DataSourceCreator {

    public DataSource createHikariDataSource(DataSourceProperties properties) {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(properties.getUrl());
        dataSource.setUsername(properties.getUsername());
        dataSource.setPassword(properties.getPassword());
        dataSource.setDriverClassName(properties.getDriverClassName());

        return new LazyConnectionDataSourceProxy(dataSource);
    }
}
```

### ChainedTxConfig 

`ChainedTransactionManager`를 primary로 등록하자.

```java
@Configuration
public class ChainedTxConfig {

    @Bean
    @Primary
    public PlatformTransactionManager transactionManager(PlatformTransactionManager memberTxManager, PlatformTransactionManager boardTxManager) {
        return new ChainedTransactionManager(memberTxManager, boardTxManager);
    }
}
```

## LogicService, MemberService, BoardService 수정

`ChainedTransactionManager` bean을 primary로 등록했으니, `@Transactional`에서 지정했던 transactionManager 설정을 지우자.

```java
@Service
@Transactional
public class LogicService { ... }

@Service
@Transactional
public class MemberService { ... }

@Service
@Transactional
public class BoardService { ... }
```

## 결과

| member propagation | board propagation | member insert | board insert |
| - | - | - | - |
| required | required | rollback | rollback |
| required | requires-new | rollback | rollback |
| required | nested | rollback | rollback |
| requires-new | required | commit | rollback |
| requires-new | requires-new | commit | rollback |
| requires-new | nested | commit | rollback |
| nested | required | rollback | rollback |
| nested | requires-new | rollback | rollback |
| nested | nested | rollback | rollback |
| not-supported | required | commit | rollback |
| not-supported | requires-new | commit | rollback |
| not-supported | nested | commit | rollback |
| not-supported | not-supported | commit | commit |

#### 설명

일부러 `NOT_SUPPORTED`를 추가해봤다. 부모의 영향을 받는 부분(`REQUIRED`, `NESTED`)는 모두 rollback이 동작하고, 나머지는 모두 commit되었다. `ChainedTransactionManager`가 아주 잘 동작하고 있다.

# 구현 3 - JtaTransactionManager

JTA(Java Transaction Api)는 자바 표준으로써, 분산 transaction을 가능하게 해준다. 매우 간단하게 설명하자면, JTA를 지원하는 자원을 가리키는 XA Resource 인터페이스의 구현체들을 등록하면, 해당 구현체들에 대해서 전역 transaction을 지원해준다. 때문에 어떤 자원이든 transaction을 지원할 수 있도록 정의한 셈인데, DataSource, JMS 외에는 쓰고 있는 곳이 없는 것 같다.
Java EE Application Server에서는 전역 tranction을 지원하기 위해서, JTA를 사용하기도 한다. Spring에서는 JNDI에서 Java EE Container가 사용 중인 DataSource를 가져와, JtaTransactionManager에 설정할 수도 있다.

JTA에 대해서 설명하는 것은 여기까지 하고, 직접 사용하면서 여러 DataSource를 대상으로 하는 transaction이 잘 동작하는지 확인해보자.

## 의존성 추가

```gradle
dependencies {
    compile('org.springframework.boot:spring-boot-starter-jta-atomikos')
}
```

## Spring Application 설정

### application.yml

```yml
spring:
  jta:
    enabled: true
    atomikos:
      datasource:
        member:
          unique-resource-name: memberDataSource
          xa-data-source-class-name: com.mysql.jdbc.jdbc2.optional.MysqlXADataSource
          xa-properties:
            user: root
            password: multitxtest
            url: jdbc:mysql://127.0.0.1:11306/multi_tx_test?useSSL=false
        board:
          unique-resource-name: boardDataSource
          xa-data-source-class-name: com.mysql.jdbc.jdbc2.optional.MysqlXADataSource
          xa-properties:
            user: root
            password: multitxtest
            url: jdbc:mysql://127.0.0.1:11307/multi_tx_test?useSSL=false
```

### BoardSqlSessionConfig

```java
@Configuration
@EnableConfigurationProperties
public class BoardSqlSessionConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.jta.atomikos.datasource.board")
    public DataSource boardDataSource() {
        return new AtomikosDataSourceBean();
    }
    // ...
}
```

### MemberSqlSessionConfig

```java
@Configuration
@EnableConfigurationProperties
public class MemberSqlSessionConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.jta.atomikos.datasource.member")
    public DataSource memberDataSource() {
        return new AtomikosDataSourceBean();
    }
    // ...
}
```

## 결과

| member propagation | board propagation | member insert | board insert |
| - | - | - | - |
| required | required | rollback | rollback |
| required | requires-new | rollback | rollback |
| required | nested | rollback | rollback |
| requires-new | required | commit | rollback |
| requires-new | requires-new | commit | rollback |
| requires-new | nested | commit | rollback |
| nested | required | rollback | rollback |
| nested | requires-new | rollback | rollback |
| nested | nested | rollback | rollback |
| not-supported | required | commit | rollback |
| not-supported | requires-new | commit | rollback |
| not-supported | nested | commit | rollback |
| not-supported | not-supported | commit | commit |

### 설명

잘 동작한다. 부모의 영향을 받는 전파 수준과 받지 않는 전파 수준에서 기대하는 동작이 잘 수행되고 있다.

# 마무리

여러 `DataSource`가 존재하는 상황에서 transaction 설정을 어떻게 가져가야 하는지에 대해 3가지 방법을 사용해서 살펴보았다. `ChainedTransactionManager`나 `JtaTransactionManager`를 사용해서, 전역 transaction을 지원하도록 설정하지 않는 한, 개발자가 원하는 동작을 하지 않음을 알 수 있었다.
`ChainedTransactionManager`는 기본 개념이 쉽다. 내부 동작이 어떻게 돌고 있는지 간단하게 파악된다. 별다른 학습 비용 없이 쉽게 사용할 수 있다. 하지만 `JtaTransactionManager`의 경우에는 그에 비해 공부할 내용들이 꽤 있다. 결국은 둘 다 `PlatformTransactionManager`를 구현하고 있다. 개발자가 spring을 사용하는 한, 둘 다 유연하게 적용할 수 있다.
명심하자, DataSource를 두 개 이상 사용할 때에는 반드시 `ChainedTransactionManager`나 `JtaTransactionManager`로 전역 transaction 설정을 해야한다.

소스 코드 : https://github.com/supawer0728/simple-multi-tx