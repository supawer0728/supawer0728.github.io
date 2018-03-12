---
title: Hazelcast 공유
date: 2018-03-11 00:05:51
tags: [hazlecast, nosql]
categories:
  - nosql
---

# In-Memory Data Grid

## In-Memory Data Grid(IMDG)에 관해서

http://d2.naver.com/helloworld/106824

### 요약

* 분산 저장(scale out)
* 메모리를 사용한다
* 보통 객체를 저장한다(serialize)
* Lock, Transaction, Sharding을 지원한다

<!-- more -->

# Hazelcast

## Hazelcast?

**Distributed Computing, Simplified.**

Map, Queue, Executor Service, IAtomicLong, Lock, Cache, 기타 등등의 기능을 지원한다 https://hazelcast.org/

* java api만 사용하여 의존성이 없다
* P2P 통신
    * 많은 NoSQL과 달리 peer-to-peer 통신을 사용한다.
    * Master/Slave가 아니라 SPoF가 없다.
    * 모든 member가 동일한 양의 데이터를 쥐고 동일한 양의 연산을 수행한다.
    * Hazelcast를 기존 어플리케이션에 내재시키거(embed)나 혹은 Server/Client 모드로 Hazelcast member의 client로써 어플리케이션을 실행시킬 수 있다.
* Scalable
    * member를 간단히 추가할수 있으며, 자동으로 cluster를 찾고 선형으로 메모리 및 처리량을 증가시킨다.
    * 각 member들은 TCP로 서로 연결되어 있으며 모든 통신인 TCP로 이루어진다.
* Fast : 모든 것을 memory에 저장하며, 빠른 읽기/저장이 가능하다
* 중복지원
    * 여러 member에 각 data entry의 백업을 보존한다
    * 한 member에서 실패가 발생시, 백업에서 복구가 되며, 클러스터는 다운되지 않고 동작하게 된다.

## 두가지 사용 방법 - Embedded

![image](http://docs.hazelcast.org/docs/3.9/manual/html-single/images/Embedded.png)

* standalone cache처럼 어플리케이션에 추가하여 사용하는 방식
* 고성능 비동기처리 어플리케이션을 구동할때 유용
* 어플리케이션에 hazelcast의 데이터, 서비스가 모두 올라간다.
* 데이터 접근 지연이 낮아, 처리가 매우 빠르다

## 두가지 사용 방법 - Server/Client

![image](http://docs.hazelcast.org/docs/3.9/manual/html-single/images/ClientServer.png)

* Hazelcast node를 구축하고, application은 client로써 접근
* 성능 예측이 용이
* 개별 확장 가능

## Near Cache

**Global Cache와 Local Cache의 분리**

### put

![image](https://docs.oracle.com/cd/E18686_01/coh.37/e18680/img/near-cache-fetch.jpg)


### read

![image](https://docs.oracle.com/cd/E18686_01/coh.37/e18680/img/near-cache.jpg)

## Near Cache


### 장점

* 네트워크 트래픽 감소
* latency 저하

### 단점

* 메모리 사용량 증가
* 데이터 변경으로 인한 무효화(invalid)가 발생이 잦을 수록 성능 저하 -> 대부분 읽기 작업인 서비스에 적합
* 일관성이 깨질 수 있다

### 특기사항

* Server/Client로 운용시 반드시 Near Cache를 사용할 것
* 원래 키의 TTL과 Near Cache의 TTL 다르므로 주의

## Where To Use?

* cache
* event(message) based programming : publish / subscribe
* Global Scheduler
* Session Store
* Repository
* Streaming Analysis
* Map/Reduce

## 샤딩 지원

* Hazelcast에서 샤딩을 파티셔닝이라고 부른다
* 기본적으로 271개의 파티션이 생성되며, entry key의 hash mod 연산에 의해 나뉘지게 된다.

![image](http://docs.hazelcast.org/docs/3.9/manual/html-single/images/NodePartition.jpg)

## 샤딩 지원 - 노드 추가

![image](http://docs.hazelcast.org/docs/3.9/manual/html-single/images/BackupPartitions.jpg)

* 검은 글씨 : 주 파티션
* 파란 글시 : 레플리카(백업)
* member의 가입, 탈퇴 때마다 파티셔닝이 재연산

## 샤딩 지원

![image](http://docs.hazelcast.org/docs/3.9/manual/html-single/images/4NodeCluster.jpg)

# 분산 자료형

## Map

```
public class DistributedMap {
    public static void main(String[] args) {
        Config config = new Config();
        HazelcastInstance h = Hazelcast.newHazelcastInstance(config);
        ConcurrentMap<String, String> map = h.getMap("my-distributed-map");
        map.put("key", "value");
        map.get("key");
         
        //Concurrent Map methods
        map.putIfAbsent("somekey", "somevalue");
        map.replace("key", "value", "newvalue");
    }
}    

```

이런 느낌의 자료형 : **Map, MultiMap, Queue,** List, Set, Ring Buffer, AtomicLong, AtomicReference, CountdownLatch

## Topic / Reliable Topic(Support Backup)

```
public class Sample implements MessageListener<MyEvent> {

  public static void main( String[] args ) {
    Sample sample = new Sample();
    HazelcastInstance hazelcastInstance = Hazelcast.newHazelcastInstance();
    ITopic topic = hazelcastInstance.getTopic( "default" );
    topic.addMessageListener( sample );
    topic.publish( new MyEvent() );
  }
  
  @Override
  public void onMessage( Message<MyEvent> message ) {
    MyEvent myEvent = message.getMessageObject();
    System.out.println( "Message received = " + myEvent.toString() );
    if ( myEvent.isHeavyweight() ) {
      messageExecutor.execute( new Runnable() {
          public void run() {
            doHeavyweightStuff( myEvent );
          }
      } );
    }
  }
```

## Distributed Computing

```
public static void main( String[] args ) throws Exception {
  HazelcastInstance hazelcastInstance = Hazelcast.newHazelcastInstance();
  IExecutorService executor = hazelcastInstance.getExecutorService( "exec" );
  for ( int k = 1; k <= 1000; k++ ) {
    Thread.sleep( 1000 );
    System.out.println( "Producing echo task: " + k );
    executor.executeOnAllMembers( new EchoTask( String.valueOf( k ) ) );
  }
  System.out.println( "EchoTaskMain finished!" );
}
```

## Transaction

### 자료형

**`TransactionContext`를 통해서, Transaction을 지원하는 Map, Set, List, MultiMap을 사용할 수 있다**

* Queue, Set, List의 경우, 복제를 이용
* Map의 경우, 쓰기 락을 이용

### `ONE_PHASE`와 `TWO_PHASE`

**ONE_PHASE**

* 빠름
* Single Phase로 변경을 적용
* 준비단계가 없기 때문에 충돌 감지 불가능
* 실제 충돌이 발생시 변경사항이 기록되지 않아, 일관성 무너질 수 있음

**TWO_PHASE**

* Prepare Phase(준비 단계)를 실행, 충돌이 발생 시 이 단계에서 실패
* 준비단계가 완료되어야 쓰기 실행
* 다른 멤버로 커밋 로그를 남겨, 실패해도 다른 멤버가 커밋할 수 있음
* 느림

## Transaction

### XA Transaction 지원

**XA Transaction은 글로벌 트랜잭션 내에서, 여러 백엔드 저장소에 접근하기 위한 표준**

* 트랜잭션 매니저가 DB에 어떤 트랜잭션에서 어떠한 일을 수행하는지 알림
* 2-phase commit 수행 방법 정의
* 대기중인 트랜잭션의 복구 방법 정의

## 참고

Hazelcast Architects View : https://hazelcast.com/resources/architects-view-hazelcast/
Java Code Reference Card : https://hazelcast.com/resources/code-reference-card/
Deployment And Operation Guide : https://hazelcast.com/resources/hazelcast-deployment-operations-guide/