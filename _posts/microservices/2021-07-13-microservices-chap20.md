---
title: "마이크로서비스 모니터링"
date: 2021-07-13
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 20. 마이크로서비스 모니터링

## 프로메테우스와 그라파나를 사용한 성능 모니터링

프로메테우스가 쿠버네티스 포드에 적용한 애노테이션을 사용해 마이크로서비스의 메트릭을 수집함

## 애플리케이션 메트릭 수집을 위한 소스 코드 변경

스프링 부트2는 마이크로미터(Micoremeter) 라이브러리를 사용해 프로메테우스 형식에 맞춘 성능 메트릭을 생성

```groovy
implementation('io.micrometer:micrometer-registry-prometheus')
```

프로메테우스가 엔드포인트를 인지할 수 있도록 적용

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "4004"
  prometheus.io/scheme: http
  prometheus.io/path: "/actuator/prometheus"
```

메트릭을 어디에서 수집했는지 프로메테에수그 쉽게 식별할 수 있도록 메트릭을 생성한 마이크로서비스의 이름을 태그로 지정

```yaml
management.metrics.tags.application: ${spring.application.name}
```

## 마이크로서비스 빌드 및 배포

생략

## 그라파나 대시보드를 사용한 마이크로서비스 모니터링

일반적으로 대시보드는 초당 요청 수와 응답 시간, 요청 처리 주엥 발생한 오류 비율과 같은 애플리케이션 수준의 성능 메트릭에 중점을 둠

생략...

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)

  
