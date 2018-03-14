---
title: Spring Cloud를 사용한 Auto Scaling
date: 2018-03-11 02:00:00
tags: [spring-cloud, eureka, spring-actuator, architecture, msa]
categories:
  - [spring,cloud,netflix]
  - [architecture,msa]
---

# Eureka Topology

{% plantuml %}
component [Eureka Server]
component [api-gateway(Ribbon, Eureka Client)]
component [member-api(Eureka Client)]

[api-gateway(Ribbon, Eureka Client)] -left-> [Eureka Server] : 서버 목록 조회
[member-api(Eureka Client)] -up-> [Eureka Server] : 등록
[api-gateway(Ribbon, Eureka Client)] -down-> [member-api(Eureka Client)] : 요청 전송
{% endplantuml %}

* 중앙 레지스트리 컴포넌트에 서비스 식별자, 호스트명, 포트 번호, 동작 상태 등의 메타데이터를 담을 수 있다
* Ribbon과 함께 클라이언트 측의 동적 부하 분산 기능을 담당할 수 있다

**단점**

* 인스턴스 추가를 수동으로 해야한다!!!
* 인스턴스 종료를 수동으로 해야한다!!!

<!-- more -->

# 자동확장(Auto Scale-out)

## 자동확장이란?

* 트래픽 증가시 : 추가 인스턴스를 만듦
* 트래픽 감소시 : 불필요한 여분의 인스턴스를 서비스에서 제외

### 장점

* 고가용성
* 확장성 : 필요한 서비스 그룹만 수평적으로 늘릴 수 있다
* 자원 사용량 최적화

## 확장 모델

* 애플리케이션 자동 확장 : 애플리케이션 바이너리 복제
* 인프라스트럭처 확장 : 전체 가상머신까지 복제

## 확장 모델(어플리케이션 자동 확장)

**A 시나리오**

{% plantuml %}
package "가상머신1" {
  [서비스1]
}

package "가상머신2" {
  [서비스2]
}

package "가상머신3" {
  [서비스3]
}
{% endplantuml %}

{% plantuml %}
package "가상머신4" {
  [서비스1]
}

package "가상머신5" {
  [서비스2 ]
}

package "가상머신6" {
  [서비스2]
}
{% endplantuml %}

* 라이브러리만 바뀌었을 뿐 그 하부 인프라스트럭처는 바뀌지 않는다
* 라이브러리만 교체하고 가상머신은 그대로이므로 인스턴스화가 빠르게 이루어진다
* OS 수준의 튜닝이 필요하다면 효과적이지 않을 수 있다

## 확장 모델(인프라스트럭처 자동 확장)

{% plantuml %}
package "운영 중" {
  package "가상머신2" {
    [서비스2]
  }
  package "가상머신1" {
    [서비스1]
  }
}

package "예비" {
  package "가상머신4" {
    [서비스2 ]
  }
  package "가상머신3" {
    [서비스1 ]
  }
}
{% endplantuml %}

* 대부분의 경우 새로운 가상머신을 그때그때 생성하고 제거
* 예비 인스턴스는 미리 정의된 서비스 인스턴스를 가진 가상머신 이미지로 생성
* `서비스1`에 대한 수요가 생길시 `가상머신3`이 운영 상태로 옮겨짐
* 가상머신 이미지가 본질적으로 무거우므로 새로운 가상머신을 운영 상태로 전환하는 데 시간이 많이 소요될 수 있다
    * 때문에 도커와 같은 가벼운 컨테이너가 선호된다

## 확장 모델(클라우드 자동 확장)

AWS 등의 클라우드 서비스가 제공해주는 기능

## 자동 확장 방식(자원 제약 조건 기반)

**실시간 지표(metric)**

* cpu, 메모리, disc 사용율 뿐 아니라 `Heap memory`와 같이 인스턴스가 스스로 수집할 수 있는 통계에 기반해 확장함

**예제**

* CPU사용량 60%를 초과하면 인스턴스 추가

```uml
[모니터링] .right.> [인스턴스3] : CPU가 60% 이상이면 가동
[모니터링] -down-> [인스턴스1] : CPU 모니터링
[모니터링] -down-> [인스턴스2] : CPU 모니터링
```
* 응답 슬라이딩 : 60초 슬라이딩 윈도우 내에서 특정 트랜잭션의 60% 이상의 응답 시간이 정해진 한계치를 넘으면 인스턴스 추가
* CPU 슬라이딩 : 5분 슬라이딩 윈도우 내에서 CPU 사용량이 70% 넘으면 새 인스턴스를 추가
* 예외 슬라이딩 : 60초 슬라이딩 윈도우 내에서 80% 이상의 트랜잭션이 스레드 풀 부족에 의한 타임아웃이 발생했을 시, 새 인스턴스 추가

## 자동 확장 방식(특정 기간 동안)

**개요**

특정 기간 동안에 집중적으로 발생하는 트래픽을 처리하는 데 사용되는 방식

{% plantuml %}
[모니터링] .right.> [인스턴스3] : 08시에 올린다, 18시에 내린다
[모니터링] -down-> [인스턴스1]
[모니터링] -down-> [인스턴스2]
{% endplantuml %}

## 자동 확장 방식(메시지 큐 길이 기반)

{% plantuml %}
[예매 서비스] -down-> [Message Queue] : 예매 성공시 이벤트 발생!
[메일 발송 서비스1] -up-> [Message Queue] : 구독
[모니터링] -left-> [Message Queue] : 큐 길이 검사 
[모니터링] .down.> [메일 발송 서비스2] : 큐 길이가 100 이상이면 추가
{% endplantuml %}

## 자동 확장 방식(예측에 의한 확장)

* 트래픽이 갑자기 치솟을 때는 전통적인 자동 확장은 도움이 되지 못한다
* 실제 트래픽이 치솟기 전에 예측해서 확장한다
* [Netflix Scryer](https://medium.com/netflix-techblog/scryer-netflixs-predictive-auto-scaling-engine-a3f8fc922270)
* 이력 정보, 현재 트렌드 등, 여러 가지 입력 정보를 바탕으로 발생 가능한 `트래픽 패턴`을 예측 - 빅데이터 분석 등의 솔루션과 연계

# 구현

## 필요한 기능

{% plantuml %}
[MicroService Instance1\n<Eureka Client>] -up-> [서비스 레지스트리\n<Eureka Server>] : 등록
[MicroService Instance2\n<Eureka Client>] -up-> [서비스 레지스트리\n<Eureka Server>] : 등록
[서비스 레지스트리\n<Eureka Server>] <- [라이프 사이클 관리자] : 서버 목록 조회
[라이프 사이클 관리자] --> [MicroService Instance1\n<Eureka Client>] : 성능 측정(/metric)
[라이프 사이클 관리자] --> [MicroService Instance2\n<Eureka Client>] : 성능 측정(/metric)
[라이프 사이클 관리자] -> [Cloud Service(Vendor)] : 필요시 확장(축소) 요청
[Cloud Service(Vendor)] ..> [MicroService Instance3\n<Eureka Client>] : 새 Instance 생성
[MicroService Instance3\n<Eureka Client>] .up.> [서비스 레지스트리\n<Eureka Server>] : 등록
[Cloud Service(Vendor)] ..> [MicroService Instance1\n<Eureka Client>] : 추가 Library 배포하거나
{% endplantuml %}

* 마이크로 서비스 : 지속적으로 상태, 성능 성보를 전달(spring-actuator 사용시: `/mertirc`)
* 서비스 레지스트리 : 모든 서비스와 해당 서비스들의 상태, 메타데이터, 종단점 URI를 지속적으로 추적
* 로드 밸런서 : 사용 가능한 인스턴스 최신 정보를 얻기위해 서비스 레지스트리를 검색
* 라이프 사이클 관리자
    * 측정 지표 수집기 : 모든 서비스 인스턴스로부터 측정 지표 정보를 수집하는 책임, 슬라이딩 윈도우 유지
    * 확장 정책 : CPU 60% 이상 등의 규칙 집합
    * 결정 엔진 : 수집된 측정 지표와 확장 정책으로 확장과 축소를 결정하는 책임
    * 배포 규칙 : 서비스를 위해 최소 4GB의 메모리가 있어야 한다 등의 배포 제약 조건를 사용
    * 배포 엔진 : 레지스트리를 업데이트

> 라이프 사이클 관리자가 직접 인스턴스 생성, 제거를 하더라도 역할 상으로는 문제 없어 보임
> 다만 역할을 분리하여 관리하는 것이, 운영하기가 더 수월할 수도 있겠다는 생각이 듦

## spring-boot를 활용한 사용자 정의 라이프 사이클 관리자 구현

{% plantuml %}
[Eureka 서버]
[라이프 사이클 APP] -up-> [Eureka 서버] : 1. 서비스 리스트 조회
[라이프 사이클 APP] -up-> [Eureka 서버] : 2. 서비스별 서버 리스트 조회
[라이프 사이클 APP] -down-> [스프링 부트 서비스] : 3.각 서비스 상태 체크
[스프링 부트 서비스] -up-> [Eureka 서버] : 등록
[라이프 사이클 APP] .> [새 인스턴스]
{% endplantuml %}

* 스프링 부트 서비스는 마이크로 서비스에 해당
    * spring actuator가 활성화 되어 있고, 라이프 사이클 관리자는 이를 통해 측정 지표를 수집한다
* 라이프 사이클 관리자는 스르핑 애플리케이션이다
    * 백그라운드 잡으로 실행
    * Eureka 서버를 폴링
    * Eureka 서버에서 받아온 서비스 목록의 actuator 종단점을 호출해서 상태와 수치 정보를 가져옴
    * 인스턴스 종료는 각 서비스 인스턴스의 종료 서비스를 호출(`/shutdown 등`)
    * 등록의 경우 SSH로 가상머신에 접속하고 미리 설치된 스크립트를 실행(혹은 스크립트 전달)

## member-api

**spring-actuator의 성능 지표**

`/metrics` 호출시 서비스 상태 반환

참고 : https://cloud.spring.io/spring-cloud-netflix/multi/multi_netflix-metrics.html

```
{
    "mem": 491102,
    "mem.free": 247109,
    "processors": 4,
    "instance.uptime": 15898,
    "uptime": 42016,
    "systemload.average": 3.90234375,
    "heap.committed": 435200,
    "heap.init": 262144,
    "heap.used": 188090,
    "heap": 3728384,
    "nonheap.committed": 56920,
    "nonheap.init": 2496,
    "nonheap.used": 55903,
    "nonheap": 0,
    "threads.peak": 35,
    "threads.daemon": 30,
    ...
    "httpsessions.max": -1,
    "httpsessions.active": 0
}
```

## member-api

분당 transaction 추가

```
@Autowired
private GaugeService gaugeService;

@Bean
public Filter transactionPerMinuteCountFilter() {

    return new OncePerRequestFilter() {
        private SlidingWindowCounter counter = new SlidingWindowCounter(Duration.ofSeconds(10L));

        @Override
        protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
            gaugeService.submit("gauge.request.per.10s", counter.increase());
            filterChain.doFilter(request, response);
        }
    };
}

private class SlidingWindowCounter {
    private Duration duration;
    private LocalDateTime expiry;
    private long count;

    public SlidingWindowCounter(Duration duration) {
        this.duration = duration;
        calculateExpiry();
    }

    private void calculateExpiry() {
        this.expiry = LocalDateTime.now().plus(duration);
    }

    public synchronized long increase() {
        if (expiry.isBefore(LocalDateTime.now())) {
            calculateExpiry();
            count = 1L;
            return count;
        }

        return ++count;
    }
}
```

**`http://localhost:8081/metrics`**

`/members`를 한 번 호출 후

```
{
  ...
  "gauge.request.per.10s()": 1,
  ...
}
```

## LifecycleManagerApplication

### MetricsCollector

`/metrics` 정보 수집
```
while (true) {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    // 서비스 목록 순회
    eurekaClient.getServices().forEach(service -> {
        log.info("service : {}", service);
        Map metrics = restTemplate.getForObject("http://{service}/metrics", Map.class, service);
        decisionEngine.execute(service, metrics);
    });
}
```

## LifecycleManagerApplication

### DecisionEngine

`/metrics` 정보로 배포가 필요한지 판단 후 배포 실행

```
// 서비스 별 정책에 의한 확장 필요여부 판단
if (policies.getPolicy(serviceId).isScaleOutRequired(serviceId, metrics)) {
    return deploymentEngine.scaleOut(deploymentRules.getDeploymentRule(serviceId), serviceId);
}
```

**SamplePolicy**

```
public boolean isScaleOutRequired(String serviceId, Map metrics) {
    // 10초당 100회 이상의 요청이 있는 경우 배포
    if (metrics.containsKey(key)) {
        Long requestsPer10Seconds = (Long) metrics.get(key);
        log.info("{} : {}", key, requestsPer10Seconds);
        return requestsPer10Seconds > 10L;
    }

    return false;
}
```

## LifecycleManagerApplication

### DeplymentEngine

```
public boolean scaleOut(DeploymentRule rule, String serviceId) {

    if (!rule.executable()) {
        return false;
    }

    Thread thread = new Thread(() -> {
        executeDeploy();
    });

    thread.start();
    return true;
}

// TODO: 배포 서비스에 맞춰 배포 실행
private void executeDeploy() {
    log.info("배포 실행");
}
```

# 참고

**서적**

[스프링 마이크로 서비스(라제시 RV, 2017.07.27, acorn+PACKT)](http://www.acornpub.co.kr/book/spring-microservices)