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
- Global Cache에서 장애가 발생한 경우, Hystrix를 고려하자

<!-- more -->

# 서비스 입장에서 본 구조

MVC의 Controller에서 로직을 실행하기 위해 Service를 호출할 때 캐시를 사용하려 한다.

{% plantuml %}
[service.get()] --> [Cache] : cache를 먼저 호출
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

> 지금까지의 소스는 https://github.com/supawer0728/simple-cache/tree/simple-cache 에서 받을 수 있다.

# 2차 캐시 구성

캐시에 많은 부하가 몰릴 수 있는 시스템의 경우 캐시를 수직적으로 나누어 1차, 2차 캐시를 사용하는 것이 좋을 때가 있다. 서비스 입장에서 구성을 나타내면 아래와 같다.

{% plantuml %}
[service.get()] --> [Local Cache(JVM)] : heap을 사용하는 local cache 호출
[Local Cache(JVM)] --> [Global Cache(Remote)] : local cache에 값이 없을 때 remote cache 호출
[Global Cache(Remote)] --> [Repository] : remote cache에 값이 없을 때 repository 호출
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

`CacheManager`에서 실제 구현을 담고 있는 `Cache`를 저장한다. 우리가 구현할 내용은 순서를 지정해주고 순서에 따라 값을 넣는 것이다.

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

# 이후 10초간 local-cache만 호출
03:24:29  INFO LoggingCache     : Local-Cache.put(Object, Object) called
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