---
title: Spring Cache 장애 대응 방안
date: 2018-04-18 01:20:53
tags: [spring, cache, fault-tolerance]
category:
  - spring
  - practice
---

# 서론

Spring은 [캐시 추상화](https://docs.spring.io/spring/docs/5.0.5.RELEASE/spring-framework-reference/integration.html#cache)를 통해서 쉽게 캐시를 사용할 수 있게 해준다. `CacheManager`를 잘 구현한다면 `@Cacheable`, `@CachePut`, `@CacheEvict` 등을 통해서 얼마든지 커스터마이징된 캐시를 사용할 수 있다. 하지만 의외로 장애에 대응하는 방안에 대해서는 Reference 상에서도 구체적인 설명을 해주지 않고 있다. 덕분에 다시 글 쓸 거리가 생긴 것 같다.

사용하는 캐시 시스템에 문제가 생겼을 때 어떻게 대처해야하는지는 어떤 정보를 캐시하느냐에 따라 다르다. 경우의 수 자체가 너무 많다. 또한 캐시 자체가 여러 용도로 쓰일 수 있다. HTTP 응답을 캐시하든지, Repository의 결과를 캐시하든지 혹은 Service의 결과를 캐시할 수도 있다. 본문에서는 아래 두 경우로 간단하게 나누고 이에 대한 해결법을 궁리해보고자 한다.

- 부하가 많이 걸린다면? 2차 캐시 구성을 고려하자.(Local Cache, Global Cache 구성)
- Global Cache에서 장애가 발생할 경우를 대비, Hystrix를 고려하자

<!-- more -->

# 서비스 입장에서 본 구조

MVC의 Controller에서 로직을 실행하기 위해 Service를 호출할 때 캐시를 사용하려 한다.

{% plantuml %}
[service.get()] --> [Cache] : cache를 먼저 호출
[Cache] -right-> [RedisServer]
[RedisServer] -left-> [Cache]
[Cache] --> [Repository] : cache에 정보가 없으면 호출
{% endplantuml %}

# 기본 예제

우선 Spring에서 캐시를 사용하는 기본 예제를 간단히 작성하자. 여기서는 Redis를 캐시로 사용한다.

## 의존성 설정

`Spring Boot`는 `2.0.1.RELEASE`를 사용했다.

2.0.x부터 redis를 사용시 `lettuce` 라이브러리가 기본으로 설정된다. 하지만 예전부터 Spring을 사용하는 경우라면 jedis에 익숙한 개발자가 많을 것으로 예상된다. 어차피 본문에서는 라이브러리에 따라 크게 달라지는 설정이 없으므로 익숙한 jedis를 사용하려고 한다. 

```gradle
dependencies {
    compile('org.springframework.boot:spring-boot-starter-cache')
    compile('org.springframework.boot:spring-boot-starter-data-redis') {
        exclude group: 'io.lettuce', module: 'lettuce-core'
    }
    compile('redis.clients:jedis')
    compile('org.springframework.boot:spring-boot-starter-web')
    compileOnly('org.projectlombok:lombok')
    testCompile('org.springframework.boot:spring-boot-starter-test')
}
```

## 도메인

**Person**

```java
@Data
public class Person {
    private Long id;
    private String name;
    private int age;

    // 테스트 정보를 넣기 위한 생성자
    public Person(@NonNull String name, int age) {
        this.name = name;
        this.age = age;
    }

    // 캐시에 저장된 json string을 인스턴스로 만들기 위한 생성자
    @JsonCreator
    public Person(@JsonProperty("id") Long id,
                  @JsonProperty("name") String name,
                  @JsonProperty("age") int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }
}
```

**PersonRepository**

본문에서는 Repository의 내용이 중요하진 않으므로 Method Signature만 적고 넘어가려고 한다.

```java
@Repository
public class PersonRepository {

    public Person save(Person person) { ... }
    public Person findOne(Long id) { ... }
}
```

## 응용 계층

**PersonService**
```java
@Service
public class PersonService {

    private final PersonRepository personRepository;

    public PersonService(@NonNull PersonRepository personRepository) {
        this.personRepository = personRepository;
    }

    // 메서드 항상 실행, 결과값에 id가 null이 아닌 경우 캐싱
    @CachePut(cacheNames = "person", key = "#person.id", unless = "#result.id != null")
    public Person save(Person person) {
        log.info("save() called");
        return personRepository.save(person);
    }

    // 캐시에 값이 없는 경우에는 로그를 남기고 repository 실행
    @Cacheable(cacheNames = "person")
    public Person get(@NonNull Long id) {
        log.info("get(Long) called");
        return personRepository.findOne(id);
    }
}
```

## Controller

```java
@RestController
@RequestMapping("/people")
public class PersonController {

    private final PersonService personService;

    public PersonController(@NonNull PersonService personService) {
        this.personService = personService;
    }

    @GetMapping("/{id}")
    public Person get(@PathVariable Long id) {
        return personService.get(id);
    }

    @PostMapping
    public Person save(@RequestBody Person person) {
        return personService.save(person);
    }
}
```

## 캐시 설정

```java
@EnableCaching(proxyTargetClass = true)
@Configuration
public class CacheConfig extends CachingConfigurerSupport {

    @Value("${redis.host:localhost}")
    private String redisHost;
    @Value("${redis.port:6379}")
    private Integer redisPort;

    @Bean
    public JedisConnectionFactory jedisConnectionFactory() {
        // 현 버전에서 factory.setHostName() 등의 api는 Deprecated되었다.
        return new JedisConnectionFactory(new RedisStandaloneConfiguration(redisHost, redisPort));
    }

    @Bean
    @Override
    public CacheManager cacheManager() {
        RedisCacheManager.RedisCacheManagerBuilder builder = RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(jedisConnectionFactory());

        // 값은 json 문자열로 넣는다. @class 필드로 클래스 정보가 들어간다.
        RedisCacheConfiguration defaultConfig =
                RedisCacheConfiguration.defaultCacheConfig()
                                       .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
                                       .entryTtl(Duration.ofSeconds(20L));

        builder.cacheDefaults(defaultConfig);

        return builder.build();
    }
}
```

## 기본 데이터 인입

```java
@SpringBootApplication
public class SimpleCacheApplication implements CommandLineRunner {

    @Autowired
    private PersonRepository personRepository;

    @Override
    public void run(String... args) throws Exception {
        personRepository.save(new Person("John", 12));
        personRepository.save(new Person("Michel", 16));
        personRepository.save(new Person("Chris", 52));
        personRepository.save(new Person("Michael", 25));
        personRepository.save(new Person("Susan", 34));
        personRepository.save(new Person("Kim", 44));
    }
}
```

## 테스트

**/people/1**를 호출하면 `get(Long) called`가 로그에 한 번 남고, 이후에는 20초 동안 남지 않는다. 그리고 `redis-cli`로 키를 직접 조회해보면 아래와 같이 정보가 잘 인입되었음을 확인할 수 있다.

```
127.0.0.1:6379> get person::1
"{\"@class\":\"com.parfait.study.simplecache.person.domain.Person\",\"id\":1,\"name\":\"John\",\"age\":12}"
```

> 소스코드 : https://github.com/supawer0728/simple-cache/tree/simple-cache

# 2차 캐시 구성

캐시에 많은 부하가 몰릴 수 있는 시스템의 경우 캐시를 수직적으로 나누어 1차, 2차 캐시를 사용하는 것이 좋을 때가 있다. 서비스 입장에서 구성을 나타내면 아래와 같다.

{% plantuml %}
[service.get()] --> [LocalCache] : heap을 사용하는 local cache 호출
[LocalCache] -> [ehCache(Heap)]
[ehCache(Heap)] -> [LocalCache]
[LocalCache] --> [GlobalCache] : local cache에 값이 없을 때 remote cache 호출
[GlobalCache] -right-> [RedisServer]
[RedisServer] -left-> [GlobalCache]
[GlobalCache] --> [Repository] : remote cache에 값이 없을 때 repository 호출
{% endplantuml %}

이러한 구성을 할 때 주의할 점이 있다. local cache와 global cache의 정보가 맞지 않는 기간이 발생할 수 있다는 것이다. 당연한 이야기지만 local cache의 만료 시간이 길면 길수록 그 기간은 길어진다. 때문에 일관된 데이터가 필요한 만큼 local cache의 만료 시간을 짧게 가져가야한다.

아쉽게도 이러한 n차 캐시 구성은 Spring에서 지원해주지 않는다. `CompositeCacheManager`라는 클래스가 있는데, 이 클래스는 등록된 여러 CacheManager 중에서 캐시명으로 Cache를 조회해서 하나만 내어준다.

**CompositeCacheManager**

```java
@Override
@Nullable
public Cache getCache(String name) {
  for (CacheManager cacheManager : this.cacheManagers) {
    Cache cache = cacheManager.getCache(name);
    if (cache != null) {
      return cache;
    }
  }
  return null;
}
```

우리가 필요한 것은 CacheManager의 목록을 연결(Chained) 구성하는 것이다. 다행히 핵심 로직은 이미 라이브러리들이 구현해두었다. 우리는 CacheManager와 Cache만 연결 구현하면 된다. 

## 예제

### ChainedCacheManager

```java
public class ChainedCacheManager implements CacheManager {
    private final List<CacheManager> cacheManagers;
    private final Map<String, Cache> cacheMap = new ConcurrentHashMap<>();

    // 반드시 하나 이상의 CacheManager를 등록
    public ChainedCacheManager(@NonNull CacheManager... cacheManagers) {
        if (cacheManagers.length < 1) {
            throw new IllegalArgumentException();
        }
        this.cacheManagers = Collections.unmodifiableList(Arrays.asList(cacheManagers));
    }

    // 특정 캐시명으로 조회시 map에 없으면 ChainedCache를 생성
    @Override
    public Cache getCache(String name) {
        return cacheMap.computeIfAbsent(name, key -> new ChainedCache(getCaches(key)));
    }

    private List<Cache> getCaches(String name) {
        return cacheManagers.stream().map(manager -> manager.getCache(name)).collect(Collectors.toList());
    }

    @Override
    public Collection<String> getCacheNames() {
        return cacheManagers.stream()
                            .flatMap(manager -> manager.getCacheNames().stream())
                            .collect(Collectors.toSet());
    }
}
```

실제 로직을 담고 있는 `cacheManagers`를 두고 위임자 패턴으로 구현하였다. `getCache(String)` 호출이 있으면, `cacheManagers`의 순서로 해당 `CacheManager`의 `Cache`를 불러와 `ChainedCache`를 새로 생성한다.

### ChainedCache

```java
public class ChainedCache implements Cache {

    // 실제 로직을 위임할 캐시들(local, global)
    private final List<Cache> caches;
    private final int first = 0;
    private final int size;

    public ChainedCache(List<Cache> caches) {
        this.caches = Collections.unmodifiableList(caches);
        this.size = caches.size();
    }
    
    @Override
    public ValueWrapper get(Object key) {

        // 순서대로 캐시에서 값을 불러온다
        for (int i = first; i < size; i++) {
            ValueWrapper valueWrapper = caches.get(i).get(key);
            if (valueWrapper != null && valueWrapper.get() != null) {
                // 첫번째 캐시에 값이 있으면 그대로 반환
                if (i == first) {
                    return valueWrapper;
                }
                
                // 첫번째 이후의 캐시에 값이 있으면 이전 index의 캐시에 각각 저장
                putIntoPreviousIndexedCaches(i, key, valueWrapper.get());
                return valueWrapper;
            }
        }

        return null;
    }
    
    private void putIntoPreviousIndexedCaches(int index, Object key, Object value) {
        for (int i = index - 1; i >= first; i--) {
            singleCachePut(caches.get(i), key, value);
        }
    }
    
    //...
}
```

`CacheManager`에서 실제 구현을 담고 있는 `Cache`를 저장한다. 우리가 구현할 내용은 순서를 지정해주고 순서에 따라 값을 넣는 것이다. 실제 동작은 `caches`에 위임하자.

### CacheConfig

```java
@EnableCaching(proxyTargetClass = true)
@Configuration
public class CacheConfig extends CachingConfigurerSupport {

    @Value("${redis.host:localhost}")
    private String redisHost;
    @Value("${redis.port:6379}")
    private Integer redisPort;
    @Value("${spring.cache.jcache.provider}")
    private String jCacheProvider;
    @Value("${spring.cache.jcache.config}")
    private Resource jCacheConfig;

    @Bean
    public JedisConnectionFactory jedisConnectionFactory() {
        return new JedisConnectionFactory(new RedisStandaloneConfiguration(redisHost, redisPort));
    }

    @Bean
    public CacheManager redisCacheManager() {
        RedisCacheManager.RedisCacheManagerBuilder builder = RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(jedisConnectionFactory());

        RedisCacheConfiguration defaultConfig =
                RedisCacheConfiguration.defaultCacheConfig()
                                       .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
                                       .entryTtl(Duration.ofSeconds(20L));

        builder.cacheDefaults(defaultConfig);

        return new LoggingCacheManager(builder.build(), "Global-Cache");
    }

    @Bean
    public CacheManager jCacheCacheManager() {
        CachingProvider provider = Caching.getCachingProvider(jCacheProvider);

        try {
            return new LoggingCacheManager(new JCacheCacheManager(provider.getCacheManager(jCacheConfig.getURI(), provider.getDefaultClassLoader())), "Local-Cache");
        } catch (IOException e) {
            throw new IllegalStateException("can't create URI with spring.cache.jcache.config");
        }
    }

    @Bean
    @Primary
    @Override
    public CacheManager cacheManager() {

        return new ChainedCacheManager(jCacheCacheManager(), redisCacheManager());
    }
}
```

`RedisCacheManager`와 `JCacheCacheManager(ehCache)`를 각각 원격, 로컬 캐시로 잡았다. `LoggingCacheManager`는 필자가 로그를 남기기 위해 작성한 것이다.

## 테스트

```
# 최초 호출시 service의 get(Long)까지 부른다
03:24:17  INFO LoggingCache     : Local-Cache.getName() called
03:24:17  INFO LoggingCache     : Local-Cache.get(Object) called
03:24:17  INFO LoggingCache     : Global-Cache.get(Object) called
03:24:17  INFO PersonService    : get(Long) called
03:24:17  INFO LoggingCache     : Local-Cache.put(Object, Object) called
03:24:17  INFO LoggingCache     : Global-Cache.put(Object, Object) called

# 이후 10초 동안 local-cache의 값 호출
03:24:19  INFO LoggingCache     : Local-Cache.getName() called
03:24:19  INFO LoggingCache     : Local-Cache.get(Object) called
03:24:23  INFO LoggingCache     : Local-Cache.getName() called
03:24:23  INFO LoggingCache     : Local-Cache.get(Object) called
03:24:26  INFO LoggingCache     : Local-Cache.getName() called
03:24:26  INFO LoggingCache     : Local-Cache.get(Object) called

# 10초가 지나면 local-cache가 만료됨. global-cache까지 호출
03:24:29  INFO LoggingCache     : Local-Cache.getName() called
03:24:29  INFO LoggingCache     : Local-Cache.get(Object) called
03:24:29  INFO LoggingCache     : Global-Cache.get(Object) called
03:24:29  INFO LoggingCache     : Local-Cache.put(Object, Object) called

# 이후 10초간 local-cache만 호출
03:24:34  INFO LoggingCache     : Local-Cache.getName() called
03:24:34  INFO LoggingCache     : Local-Cache.get(Object) called
03:24:38  INFO LoggingCache     : Local-Cache.getName() called
03:24:38  INFO LoggingCache     : Local-Cache.get(Object) called

# 10초 뒤 모든 캐시 만료되고 다시 service 호출
03:24:43  INFO LoggingCache     : Local-Cache.getName() called
03:24:43  INFO LoggingCache     : Local-Cache.get(Object) called
03:24:43  INFO LoggingCache     : Global-Cache.get(Object) called
03:24:43  INFO PersonService    : get(Long) called
03:24:43  INFO LoggingCache     : Local-Cache.put(Object, Object) called
03:24:43  INFO LoggingCache     : Global-Cache.put(Object, Object) called
```

원하는 대로 정상적으로 동작하고 있다.

> 소스코드 : https://github.com/supawer0728/simple-cache/tree/double-cache

# Hystrix 구성

전역 캐시가 부하를 받아 Timeout이 발생하거나 특정 이유로 접근이 되지 않는 등의 장애가 발생한 경우, 일전에 공유했던 Hystrix를 고려해볼 수 있다. **전역 캐시에 장애를 감지한 시점부터는 요청을 보내면 안된다.** 물론 이 방법 또한 만병 통치약인 것은 아니다. 아래의 경우를 생각해보자.

- 단순 네트워크 단절 : 다행이다. 1차 캐시와 repository로 2차 캐시를 복구할 때까지 운용할 수 있다.
- 트래픽 부하로 인한 장애 : 전역 캐시에 장애를 일으켰던 트래픽을 고스란히 1차 캐시와 repository로 견뎌내어야 한다. 2차 캐시를 복구하기 위한 잠깐의 시간 벌기는 가능할 것이다.

위와 같은 한계가 있음을 숙지하고, 이제 Hystirx를 어떻게 구성할 수 있을지 예제로 작성하려한다.

{% plantuml %}
[service.get()] --> [LocalCache] : heap을 사용하는 local cache 호출
[LocalCache] -> [ehCache(Heap)]
[ehCache(Heap)] -> [LocalCache]
[LocalCache] --> [GlobalCache] : local cache에 값이 없을 때 remote cache 호출
[GlobalCache] -down-> [GlobalCache] : RedisServer 장애로 인한 circuit open!
[GlobalCache] --> [Repository] : remote cache에 값이 없을 때 repository 호출
{% endplantuml %}

## HystirxCacheManager

사실상 Spring에서 캐시 추상화(`CacheManager`, `Cache`)를 제공하니 우리는 앞에서 한 작업의 반복을 할 뿐이다. 실제 구현체를 `delegate`로 잡아두고 위임자 패턴을 사용하여 구현하자.

```java
public class HystrixCacheManager implements CacheManager {

    private final CacheManager delegate;
    private final Map<String, Cache> cacheMap = new ConcurrentHashMap<>();

    public HystrixCacheManager(@NonNull CacheManager delegate) {
        this.delegate = delegate;
    }

    @Override
    public Cache getCache(String name) {
        return cacheMap.computeIfAbsent(name, key -> new HystrixCache(delegate.getCache(key)));
    }

    @Override
    public Collection<String> getCacheNames() {
        return delegate.getCacheNames();
    }
}
```

## HystrixCache

```java
@Slf4j
public class HystrixCache implements Cache {

    private final Cache delegate;

    public HystrixCache(Cache delegate) {
        this.delegate = delegate;
    }

    @Override
    public ValueWrapper get(Object key) {
        return new HystrixCacheGetCommand(delegate, key).execute();
    }

    @Override
    public void put(Object key, Object value) {
        new HystrixCachePutCommand(delegate, key, value).execute();
    }

    @Override
    public void evict(Object key) {
        new HystrixCacheEvictCommand(delegate, key).execute();
    }
    
    // ...
}
```

아쉽게도 `spring-netflix-starter-hystrix`에서 지원해주는 애노테이션들을 이용한 설정은 어렵다. 애노테이션을 사용해서 Hystirx설정을 하기 위해서는 설정할 인스턴스가 `ApplicationContext`에 Bean으로 등록되어야 한다. 때문에 여기서는 Spring의 도움 없이 직접 Hystrix API를 사용하였다. `execute()`는 HystrixCommand를 동기로 실행한다.

## HystrixCacheGetCommand

앞에서 get, push, evict에 대해서 모두 `HystirxCacheXXXCommand`로 작성하였으나 본문에서는 `HystrixCacheGetCommand`만 살펴보려고 한다. 나머지든 대동소이하다.

```java
@Slf4j
public class HystrixCacheGetCommand extends HystrixCommand<ValueWrapper> {

    private final Cache delegate;
    private final Object key;

    public HystrixCacheGetCommand(Cache delegate, Object key) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("testGroupKey"))
                    .andCommandKey(HystrixCommandKey.Factory.asKey("cache-get"))
                    .andCommandPropertiesDefaults(
                            HystrixCommandProperties.defaultSetter()
                                                    .withExecutionTimeoutInMilliseconds(500)
                                                    .withCircuitBreakerErrorThresholdPercentage(50)
                                                    .withCircuitBreakerRequestVolumeThreshold(5)
                                                    .withMetricsRollingStatisticalWindowInMilliseconds(20000)));
        this.delegate = delegate;
        this.key = key;
    }

    @Override
    protected ValueWrapper run() {
        return delegate.get(key);
    }

    @Override
    protected ValueWrapper getFallback() {
        log.warn("get fallback called, circuit is {}", super.circuitBreaker.isOpen() ? "opened" : "closed");
        return null;
    }
}
```

20초간 API의 성공/실패 여부를 측정하며, 5번 이상 실행되고 50% 이상 실패했을 경우 회로가 열린다. Timeout은 500ms로 설정했다. `execute()`를 실행하면 `run()`이 실행되며, 실패한 경우 `getFallback()`이 실행된다.

## CacheConfig

```java
@EnableCaching(proxyTargetClass = true)
@Configuration
public class CacheConfig extends CachingConfigurerSupport {

    // ...

    @Bean
    public CacheManager redisCacheManager() {
        RedisCacheManager.RedisCacheManagerBuilder builder = RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(jedisConnectionFactory());

        RedisCacheConfiguration defaultConfig =
                RedisCacheConfiguration.defaultCacheConfig()
                                       .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
                                       .entryTtl(Duration.ofSeconds(20L));

        builder.cacheDefaults(defaultConfig);

        return new HystrixCacheManager(new LoggingCacheManager(builder.build(), "Global-Cache"));
    }
    // ...
}
```

앞서 만들었던 부분을 그대로 생성자로 주입하자.

## 테스트

서버를 실행 후 Redis 서버를 Down 시킨 후 `/people/1`을 호출해보았다.

```
2018-04-18 22:55:09.653  INFO LoggingCache           : Local-Cache.get(Object) called
2018-04-18 22:55:09.653  INFO LoggingCache           : Global-Cache.get(Object) called
2018-04-18 22:55:09.661  WARN HystrixCacheGetCommand : get fallback called, circuit is closed
2018-04-18 22:55:09.662  INFO PersonService          : get(Long) called
2018-04-18 22:55:09.662  INFO LoggingCache           : Local-Cache.put(Object, Object) called
2018-04-18 22:55:09.663  INFO LoggingCache           : Global-Cache.put(Object, Object) called
2018-04-18 22:55:10.164  WARN HystrixCachePutCommand : put fallback called, circuit is closed
2018-04-18 22:55:14.085  INFO LoggingCache           : Local-Cache.get(Object) called
2018-04-18 22:55:14.086  INFO LoggingCache           : Global-Cache.get(Object) called
2018-04-18 22:55:14.588  WARN HystrixCacheGetCommand : get fallback called, circuit is closed
2018-04-18 22:55:14.588  INFO PersonService          : get(Long) called
2018-04-18 22:55:14.588  INFO LoggingCache           : Local-Cache.put(Object, Object) called
2018-04-18 22:55:14.588  INFO LoggingCache           : Global-Cache.put(Object, Object) called
2018-04-18 22:55:15.090  WARN HystrixCachePutCommand : put fallback called, circuit is closed
2018-04-18 22:55:16.104  INFO LoggingCache           : Local-Cache.get(Object) called
2018-04-18 22:55:16.105  INFO LoggingCache           : Global-Cache.get(Object) called
2018-04-18 22:55:16.606  WARN HystrixCacheGetCommand : get fallback called, circuit is closed
2018-04-18 22:55:16.606  INFO PersonService          : get(Long) called
2018-04-18 22:55:16.606  INFO LoggingCache           : Local-Cache.put(Object, Object) called
2018-04-18 22:55:16.607  INFO LoggingCache           : Global-Cache.put(Object, Object) called
2018-04-18 22:55:17.108  WARN HystrixCachePutCommand : put fallback called, circuit is closed
2018-04-18 22:55:17.973  INFO LoggingCache           : Local-Cache.get(Object) called
2018-04-18 22:55:17.974  INFO LoggingCache           : Global-Cache.get(Object) called
2018-04-18 22:55:18.474  WARN HystrixCacheGetCommand : get fallback called, circuit is closed
2018-04-18 22:55:18.474  INFO PersonService          : get(Long) called
2018-04-18 22:55:18.474  INFO LoggingCache           : Local-Cache.put(Object, Object) called
2018-04-18 22:55:18.475  INFO LoggingCache           : Global-Cache.put(Object, Object) called
2018-04-18 22:55:18.977  WARN HystrixCachePutCommand : put fallback called, circuit is closed
2018-04-18 22:55:19.112  INFO LoggingCache           : Local-Cache.get(Object) called
2018-04-18 22:55:19.112  INFO LoggingCache           : Global-Cache.get(Object) called
2018-04-18 22:55:19.613  WARN HystrixCacheGetCommand : get fallback called, circuit is closed
2018-04-18 22:55:19.613  INFO PersonService          : get(Long) called
2018-04-18 22:55:19.613  INFO LoggingCache           : Local-Cache.put(Object, Object) called
2018-04-18 22:55:19.614  INFO LoggingCache           : Global-Cache.put(Object, Object) called
2018-04-18 22:55:20.114  WARN HystrixCachePutCommand : put fallback called, circuit is closed
2018-04-18 22:55:20.241  INFO LoggingCache           : Local-Cache.get(Object) called
2018-04-18 22:55:20.241  INFO LoggingCache           : Global-Cache.get(Object) called
2018-04-18 22:55:20.742  WARN HystrixCacheGetCommand : get fallback called, circuit is closed
2018-04-18 22:55:20.742  INFO PersonService          : get(Long) called
2018-04-18 22:55:20.743  INFO LoggingCache           : Local-Cache.put(Object, Object) called

# 일정 횟수 fallback이 호출된 후 회로 열림!
2018-04-18 22:55:20.743  WARN HystrixCachePutCommand : put fallback called, circuit is opened
2018-04-18 22:55:21.312  INFO LoggingCache           : Local-Cache.get(Object) called
2018-04-18 22:55:21.312  WARN HystrixCacheGetCommand : get fallback called, circuit is opened
2018-04-18 22:55:21.312  INFO PersonService          : get(Long) called
2018-04-18 22:55:21.312  INFO LoggingCache           : Local-Cache.put(Object, Object) called
2018-04-18 22:55:21.312  WARN HystrixCachePutCommand : put fallback called, circuit is opened
2018-04-18 22:55:22.250  INFO LoggingCache           : Local-Cache.get(Object) called
2018-04-18 22:55:22.250  WARN HystrixCacheGetCommand : get fallback called, circuit is opened
```

> 소스코드 : https://github.com/supawer0728/simple-cache/tree/hystrix-cache

# 마무리

Spring의 캐시 추상화를 사용하며, 부하 분산을 통해 장애에 대응할 수 있는 방안에 대해 다뤄보았다.

우선 n차 캐시 구성을 통해서 Heap을 사용하여 외부 시스템의 호출을 줄여서 전역 리소스(전역 캐시, repository)의 부하를 줄였다. n차 캐시 구성을 사용하는 경우, 리소스 간의 일관성이 무너질 수 있다. 일관성이 중요할수록 로컬 캐시의 수명을 짧게해서 사용해야 한다. 

원격 캐시의 경우 파티션(장애)이 발생할 수 있다. 이에 대응하기 위해 Hystrix를 활용할 수 있다. 회로를 열어 원격에 요청을 보내지 않고, 빠른 실패처리를 할 수 있다. 하지만 여기서도 유의해야할 점이 있는데. 트래픽이 몰리는 상황에서 부하를 견디지 못해 장애가 발생한 경우, 이를 1차 캐시와 repository가 받아내게 된다. 본문에서는 fallback에서 null을 던져 repository를 실행시켰다. 원격 캐시에서 장애가 발생했을 때, 사용자 정의 Exception을 던져 캐시 오류인 경우의 응답을 따로 내려줘서 repository를 지켜내는 것도 방법이 될 수 있을 것 같다.