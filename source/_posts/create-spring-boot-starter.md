---
title: spring-boot-starter 작성하기
date: 2018-03-15 17:36:38
tags: [spring, spring-boot, starter]
categories:
  - [spring, boot]
  - [spring, practice]
  - [practice]
---

# 개요

`spring-boot`에서 `starter`란 의존성과 설정을 자동화해주는 모듈을 뜻한다.

예를 들어, `spring-boot-starter-jpa`를 의존성 추가했을 때 아래의 일을 해준다.

- `spring-aop`, `spring-jdbc` 등의 의존성을 걸어준다.
- `classpath`를 뒤져서 어떤 Database를 사용하는지 파악하고, 자동으로 entityManager를 구성해준다.
- 해당 모듈들 설정에 필요한 properties 설정을 제공한다(`Configuration Processor`를 사용시 효과 UP)

본 문서는 직접 `spring-boot-starter`를 작성하고 동작하는 방법을 공유한다.
`spring-boot` 버전은 `2.0.0.RELEASE`를 사용한다.

# 구현할 내용

아래 요구 사항을 충족하는 경우 reaquest header에서 `uri`를 logging하는 `spring-boot-starter`를 작성한다.

- `spring-webmvc`를 사용할 것
- `application.yml`에서 `spring.mvc.custom-uri-logging-filter.enabled: true`일 것
- `application.yml`에서 `spring.mvc.custom-uri-logging-filter.level: info` 등으로 지정한 레벨로 찍을 것
