---
title: "분산 추적"
date: 2021-06-19
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 14. 분산 추적

## 스프링 클라우드 슬루스와 집킨을 사용한 분산 추적

전체 워크플로에 대한 추적 정보는 **추적(trace)** 혹은 **추적 트리(trace tree)**라고 부름.

기본 작업 단위라고 할 수 있는 트리의 일부분은 **스팬(span)**   



스프링 클라우드 슬루스는 HTTP를 이용하는 동기 방식이나 RabbitMQ, 카프카 등의 메시지 브로커를 이용하는 비동기 방식으로 집킨에 추적 정보를 전송

## 소스 코드에 분산 추적 추가

1. 스프링 클라우드 슬루스를 활용해 집킨으로 추적 정보를 전송하고자 빌드 파일에 의존성 추가
2. RabbitMQ 및 카프카 의존성 추가
3. RabbitMQ나 카프카로 집킨 서버에 추적 정보를 보내도록 마이크로서비스를 구성
4. 도커 컴포즈 파일에 집킨 서버 추가
5. kafka 스프링 프로필을 스프링 클라우드 프로젝트에 추가

### 빌드 파일에 의존성 추가

```groovy
implementation('org.springframework.cloud:spring-cloud-starter-sleuth')
implementation('org.springframework.cloud:spring-cloud-starter-zipkin')
```

```groovy
implementation('org.springframework.cloud:spring-cloud-starter-stream-rabbit')
implementation('org.springframework.cloud:spring-cloud-starter-stream-kafka')
```

### 스프링 클라우드 슬루스 및 집킨에 대한 구성 추가

```yaml
spring.zipkin.sender.type: rabbit
spring.sleuth.sampler.probability: 1.0 # default는 10%만 보내지만 전체를 보내기 위해 수정
```

kafka 스프링 프로필

```yaml
---
spring.profiles: kafka

management.health.rabbit.enabled: false
spring.cloud.stream.defaultBinder: kafka
spring.zipkin.sender.type: kafka
spring.kafka.bootstrap-servers: kafka:9092
```

### 도커 컴포즈 파일에 집킨 서버 추가

```yaml
zipkin:
  image: openzipkin/zipkin:2.12.9 # 도커 이미지 및 버전
  mem_limit: 512m # 모든 추적 정보를 유지하기 위해 다른 컨테이너에 비해 메모리 소비가 많음
  networks:
    - my-network
  environment:
    - STORAGE_TYPE=mem # 집킨이 모든 추적 정보를 메모리에 유지하게 함
    - RABBIT_ADDRESSES=rabbitmq # rabbitmq 사용하도록 설정
  ports:
    - 9411:9411
  depends_on:
    rabbitmq:
      condition: service_healthy # rabbitMQ가 정상임을 보고할 때까지 집킨 서버 시작 X
```

 카프카용 서버 정의

```yaml
zipkin:
  image: openzipkin/zipkin:2.12.9
  mem_limit: 512m
  networks:
    - my-network
  environment:
    - STORAGE_TYPE=mem
    - KAFKA_BOOTSTRAP_SERVERS=kafka:9092 # 집킨이 카프카를 사용해 추적 정보를 수신하고, kafka 호스트 이름을 사용헤 카프카에 연결
  ports:
    - 9411:9411
  depends_on:
    - kafka
```

## 분산 추적 수행 

생략

## 요약

런타임 컴포넌트 사이의 의존성을 낮추고자 비동기 방식을 사용해 집킨 서버로 추적 정보를 전송하도록 마이크로서비스 환경을 구성



## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)
- https://zipkin.io/