---
title: (Spring Boot) 다중 DataSource 사용하기 with Sharding
date: 2018-05-06 23:34:38
tags: [spring, spring-boot, sharding]
categories:
  - [spring, spring-boot]
  - [spring, practice]
  - [practice]
---

# 서론

Spring Boot를 사용해서 Database Sharding을 처리할 수 있을까?
요즘 NoSQL에서는 Sharding과 관련해서 많은 편의를 제공한다. 알아서 Sharding을 제공해주고, 클러스터에 노드가 추가되면 Shard key를 기반으로 자동으로 새로운 노드로 값들을 분배해주거나, 노드를 제거하면 대상 노드에 있던 값들을 Shard Key에 맞추어 남은 노드로 분배해준다. 자세히 알아보지는 않았지만 RDBMS 측의 각종 벤더들도 이러한 아키텍처를 구현기위한 방법을 제공할 것이라 생각된다.(예를 들어 MySQL의 Fabric - 현재 MySQL Utilities에 통합)

이번 포스트에서는 RDBMS Sharding을 각 벤더에 의존해서 구현하지 않고, Spring Boot로 편리하게 설정을 추상화하여 사용할 방법에 대해 공유하려 한다. 앞서 Sharding에 대해서 거창하게 이야기를 했지만, 본문에서 다루고자 하는 영역은 아래의 두 가지 기능이다.

- 이름 기반으로 다중 DataSource 정보를 등록 : AbstractRoutingDataSource
- 특정 값을 기반으로 대상 DataSource를 사용 : Spring AOP

<!-- more -->

# 이름 기반으로 다중 DataSource 정보 등록하기

## AbstractRoutingDataSource

우선 `spring-jdbc` 모듈에 있는 [AbstractRoutingDataSource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/lookup/AbstractRoutingDataSource.html)에 대해서 먼저 소개를 해야할 것 같다. 여러 DataSource를 등록하고, 특정 상황에 맞게 원하는 DataSource를 사용할 수 있도록 추상화한 클래스다. 이 클래스의 `public void setTargetDataSources(Map<Object, Object> targetDataSource)`를 호출하여 `String:DataSource`을 `키:값`으로하는 `Map`을 저장할 수 있다. 또한 `Object determineCurrentLooupKey()`를 오버라이드해서 상황에 맞게 Key를 반환하도록 구현할 수 있다.

예시로 MyBatis의 [ReplicationRoutingDataSource](https://github.com/hatunet/spring-data-mybatis/blob/master/src/main/java/org/springframework/data/mybatis/replication/datasource/ReplicationRoutingDataSource.java)를 살펴보자.

아래 소스는 `ReplicationRoutingDataSource`의 내용을 간략화한 것이다.

```java
public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {
  // 1
  public ReplicationRoutingDataSource(DataSource master, List<DataSource> slaves) {
    // ...
    super.setDefaultTargetDataSource(master);
    Map<Object, Object> targetDataSources =
      IntStream.range(0, slaves.size)
               .boxed()
               .collect(toMap(n -> "slave_" + n, n -> slaves.get(n));
    targetDataSources.put("master", master);
    super.setTargetDataSources(targetDataSources);
  }

  // 2
  @Override
  protected Object determineCurrentLookupKey() {
    boolean transactionActive = TransactionSynchronizationManager.isActualTransactionActive();
    if (transactionActive) {
      boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
      if (readOnly) {
        return "slave_" + dataSourceSelectPolicy.select(slaves);
      }
    }
    return "master";
  }
}
```

1: 생성자를 통해서 `master`로 사용할 `DataSource`와 `slave`로 사용할 `DataSource` 목록을 받고 있다.
2: 현재 트랜잭션이 활성화되어 있고, 읽기 전용이면 `slave` 중에 하나를 선택해서 반환하도록 되어 있다.

이를 응용해서 이름 기반으로 동작하는 `AbstractRoutingDataSource`를 구현할 수 있다.

## application.yml을 이용해서 구현해보기

아래와 같은 구조로 RDBMS가 구성되어 있다고 가정해보자.

{% plantuml %}
database master(localhost:3306) {
  [core]
  [shard1]
  [shard2]
}

database slave(localhost:3307) {
  [core(slave)]
  [shard1(slave)]
  [shard2(slave)]
}
{% endplantuml %}

이러한 구조를 아래와 같은 property를 등록하여 설정되도록 하려고 한다.

```yaml
named-routing-data-source:
  enabled: true
  slave-suffix: -slave
  default-data-source: default
  data-sources:
    - name: core
      hikari:
        username: root
        password: root
        jdbc-url: jdbc:mysql://localhost:3306/core?useSSL=false
    - slave-of: core # slave의 경우에는 `slave-of`로 등록한다
      hikari:
        username: root
        password: root
        jdbc-url: jdbc:mysql://localhost:3307/core?useSSL=false
    - name: shard1
      hikari:
        username: root
        password: root
        jdbc-url: jdbc:mysql://localhost:3306/shard1?useSSL=false
    - slave-of: shard1
      hikari:
        username: root
        password: root
        jdbc-url: jdbc:mysql://localhost:3307/shard1?useSSL=false 
    - - name: shard2
      hikari:
        username: root
        password: root
        jdbc-url: jdbc:mysql://localhost:3306/shard2?useSSL=false
    - slave-of: shard2
      hikari:
        username: root
        password: root
        jdbc-url: jdbc:mysql://localhost:3307/shard2?useSSL=false 
```

위 properties는 java로 아래와 같이 나타낼 수 있다.

```java
// 전역 설정
@Data
@ConfigurationProperties(prefix = "named-routing-data-source")
public class NamedRoutingDataSourceGlobalProperties {

  private Boolean enabled;
  private String slaveSuffix = "-slave";
  private String defaultDataSource = "default";
  private List<NamedRoutingDataSourceTargetProperties> dataSources;
}

// DataSource 항목 설정
@Data
public class NamedRoutingDataSourceTargetProperties {

  private String name;
  private String slaveOf;
  private HikariConfig hikari;

  public boolean isSlave() {
    return !StringUtils.isEmpty(slaveOf);
  }
  
  public void check() {
    boolean hasName = !StringUtils.isEmpty(name);
    if (hasName == isSlave) {
      throw new IllegalStateException("name 혹은 slaveOf 둘 중 하나만 있어야 합니다");
    }
  }
}
```

- `NamedRoutingDataSourceTargetProperties`에서 `name`이나 `slaveOf` 둘 중 하나만 값이 있어야 한다.
  - `slaveOf`가 `a`인 경우 `name`은 `"a" + slaveSuffix`가 된다.

## NamedRoutingDataSource와 NamedDataSource

`NamedRoutingDataSource`는 `AbstractRoutingDataSource`를 구현하였다.  `NamedDataSource`는 `NamedRoutingDataSource`에서 들고 있을 대상이다. 먼저 `NamedDataSource`부터 살펴보자.

```java
@Value
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class NamedDataSource implements DataSource {

  private final String name;
  private final String slaveOf;
  @Delegate(types = DataSource.class)
  private final DataSource delegate;

  public static NamedDataSource slaveOf(String slaveOf, String slaveSuffix, DataSource delegate) {
    return new NamedDataSource(slaveOf + slaveSuffix, slaveOf, delegate);
  }

  public static NamedDataSource asMaster(String name, DataSource delegate) {
    return new NamedDataSource(name, null, delegate);
  }

  public boolean isSlave() {
    return !StringUtils.isEmpty(slaveOf);
  }
}
```

`NamedDataSource`는 slave(`slaveOf()`)로, 혹은 master(`asMaster()`)로 생성된다. 그리고 `DataSource`를 구현하고 있는데, 구현에 필요한 메서드는 모두 `delegate`에 위임하였다.

다음은 이 `NamedDataSource`를 가지고 routing 처리를 할 `NamedRoutingDataSource`를 살펴보자.

```java
public class NamedRoutingDataSource extends AbstractRoutingDataSource {

  private final String defaultDataSource;
  // 이름을 key로 하여 실제 대상을 메모리로 들고 있는다
  private final Map<String, NamedDataSource> dataSourceMap;
  // master key에 대한 slave key를 들고 있는다.
  // 여러 slave를 등록할 수 있도록 구현하고 싶다면 Map<String, List<String>>으로 구현할 수 있다.
  private final Map<String, String> masterSlaveMap;

  public NamedRoutingDataSource(String defaultDataSource, List<NamedDataSource> dataSources) {

    this.defaultDataSource = defaultDataSource;
    this.dataSourceMap = createDataSourceMap(dataSources);
    this.masterSlaveMap = dataSourceMap.values()
                                       .stream()
                                       .filter(NamedDataSource::isSlave)
                                       .collect(toMap(NamedDataSource::getSlaveOf, NamedDataSource::getName));
   
    // AbstractRoutingDataSource에 등록
    super.setTargetDataSources(dataSourceMap.entrySet().stream().collect(toMap(Entry::getKey, Entry::getValue)));
    super.setDefaultTargetDataSource(dataSourceMap.get(defaultDataSource));
  }

  private Map<String, NamedDataSource> createDataSourceMap(List<NamedDataSource> dataSources) {
    return dataSources.stream().collect(toMap(NamedDataSource::getName, Function.identity()));
  }

  // routing 로직
  @Override
  protected Object determineCurrentLookupKey() {
    String dataSourceName = NamedRoutingDataSourceManager.getCurrentDataSourceName();

    if (dataSourceName == null) {
      dataSourceName = defaultDataSource;
    }

    boolean transactionActive = TransactionSynchronizationManager.isActualTransactionActive();
    if (transactionActive) {
      boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
      if (readOnly) {
        dataSourceName = masterSlaveMap.getOrDefault(dataSourceName, dataSourceName);
      }
    }

    log.debug("dataSourceName: {}", dataSourceName);
    return dataSourceName;
  }

  public List<NamedDataSource> masters() {
    return dataSourceMap.values().stream().filter(dataSource -> !dataSource.isSlave()).collect(Collectors.toList());
  }
}
```

`NamedRoutingDataSourceManager`에서 현재 사용할 DataSource의 이름을 가지고 올 수 있다(`getDataSourceName()`). `NamedRoutingDataSourceManager`의 구현은 아래와 같다.

```java
public class NamedRoutingDataSourceManager {

  private static final ThreadLocal<String> currentDataSourceName = new NamedThreadLocal<>("name of DataSource selected in this thread");

  private NamedRoutingDataSourceManager() {
    throw new UnsupportedOperationException("this class can't be instance");
  }

  public static String getCurrentDataSourceName() {
    return currentDataSourceName.get();
  }

  public static void setCurrentDataSourceName(String name) {
    currentDataSourceName.set(name);
  }

  public static void removeCurrentDataSourceName() {
    currentDataSourceName.remove();
  }
}
```

`ThreadLocal`로 현재 사용할 DataSource의 이름을 설정할 수 있다. 어디선가는 이 이름을 설정해줘야 `Sharding`이 제대로 동작할 것이다. 만약 이름이 없다면 `NamedRoutingDataSource`는 항상 `defaultDataSource`만 반환할 것이다.

## Spring AOP를 활용해서 DataSource 이름 설정하기

긴 설명보다는 아래 소스를 보고 어떻게 동작할 지 설명하는게 빠를 것 같다.

```java
@Repository
public class BoardRepository {
  
  // 1
  @SetDataSourceName(getterType = HashModularDataSourceNameGetter.class)
  public List<Board> findByWriterId(String id) {
    // ...
  }
  
  // 2
  @SetDataSourceName(spel = "#this[0].id", getterType = HashModularDataSourceNameGetter.class)
  public List<Board> findByWriter(Writer writer) {
    // ...
  }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface SetDataSourceName {

  String spel() default "#this[0]";
  Class<? extends DataSourceNameGetter> getterType();
}

public interface DataSourceNameGetter {
  String getDataSourceName(Object object);
}

@Component
public class HashModularDataSourceNameGetter implements DataSourceNameGetter {
  
  private final int modValue;
  
  public HashModularDataSourceNameGetter(NamedRoutingDataSource dataSource) {
    this.modValue = dataSource.masters().size();
  }

  @Override
  public String getDataSourceName(Object object) {
    return "shard" + (object.hashCode % modValue + 1);
  }
}
```

1: `findByWriterId`의 `id`값의 `hashCode()` 결과에 modular 연산을 수행하여 대상 DataSource의 이름을 설정할 수 있다.
2: [SpEL](https://docs.spring.io/spring/docs/5.0.5.RELEASE/spring-framework-reference/core.html#expressions)를 적용하여 메서드의 특정 argument를 대상으로 DataSource의 이름을 설정할 수 있다.

이를 가능하게 하는 `SetDataSourceNameAspect` 클래스는 다음과 같이 정의할 수 있다.

```java
@RequiredArgsConstructor
@Aspect
public class SetDataSourceNameAspect {

  private final ExpressionParser parser = new SpelExpressionParser();
  private final BeanFactory factory;

  @Pointcut("@annotation(setDataSourceName)")
  public void hasAnnotation(SetDataSourceName setDataSourceName) {
  }

  @Around("hasAnnotation(setDataSourceName)")
  public Object around(ProceedingJoinPoint joinPoint, SetDataSourceName setDataSourceName) throws Throwable {

    Object getterKey = parser.parseExpression(setDataSourceName.spel()).getValue(joinPoint.getArgs());
    // Spring Context에 등록되어 있는 Bean 중에서 Type을 기반으로 검색
    DataSourceNameGetter getter = factory.getBean(setDataSourceName.getterType());
    String dataSourceName = getter.getDataSourceName(getterKey);
    // NamedRoutingDataSourceManager에 사용할 DataSource의 이름 등록
    NamedRoutingDataSourceManager.setCurrentDataSourceName(dataSourceName);

    try {
      return joinPoint.proceed();
    } finally {
      // ThreadLocal을 사용하므로 반드시 `remove()`를 실행할 것
      NamedRoutingDataSourceManager.removeCurrentDataSourceName();
    }
  }
}
```

# 결론

여러 DataSource를 이름 기반으로 등록하고, Spring AOP를 통해서 사용할 DataSource의 이름을 정하도록 소스를 작성해보았다. `AbstractRoutingDataSource`와 Spring AOP의 원리를 이용해 간단하게 추상화해보고 구현까지 해보았다. 해당 소스에서는 복잡성을 증가시키지 않기 위해 `slave`는 하나라고 단정하고 진행하였는데, 필요하다면 `slave` 용도의 DataSource를 따로 `SlaveNamedRoutingDataSource`같은 클래스로 묶어서 단순한 round-robin 정도라면 간단히 load balancing까지 쉽게 할 수 있을 것이다.
한 가지 애매한 것이 DataSource의 이름은 무슨 기준으로 정하느냐 하는 것이다. String으로 되어 있어, 등록하는 부분(`NamedDataSource`)과 실제로 값을 가져오는 부분(`DataSourceNameGetter`)에서 문자열을 잘못 입력하여 런타임 오류가 발생하는 가능성도 있는데, 이 부분에 있어서 좋은 해결책이 딱히 떠오르지는 않았다. 만약 강한 제약을 넣고 싶다면 `NamedDataSource`가 아니라 `EnumedDataSource`를 사용하는 것도 하나의 옵션으로 고려할 수 있지 않을까 생각된다.

위에서 소개한 내용 중에 `application.yml`에 설정된 property를 기반으로 `NamedRoutingDataSource`를 생성하는 소스는 빠져있다. property와 java 소스가 대칭을 이루기에 굳이 구현 소스를 넣지는 않았다. 상세한 구현이 궁금하다면 github 소스를 참고하자.

Github 소스 : https://github.com/supawer0728/parfait-spring-boot-starter-sharding