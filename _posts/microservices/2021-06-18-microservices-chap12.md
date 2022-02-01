---
title: "구성 중앙화"
date: 2021-06-18
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 12. 구성 중앙화

## 스프링 클라우드 컨피그 서버 소개

### 구성 저장소의 저장 유형 선택

- 깃 저장소
- 로컬 파일 시스템
- 하시코프 볼트
- JDBC 데이터베이스

### 클라이언트가 먼저 접속할 서버 결정

기본적으로 클라이언트는 구성 서버에 먼저 접속해 구성을 검색하며, 구성에 따라 검색 서버에 자기 자신을 등록

또한 클라이언트는 검색 서버에 먼저 접속해 구성 서버 인스턴스를 찾은 다음 구성 서버에서 구성을 가져올 수도 있는데, 각 접근 방식에는 장단점이 있다.

### 구성 보안

중요한 정보이므로 보안에 신경써야 함

런타임 관점에서는 구성 서버가 에지 서버를 통해 외부로 노출될 이유가 없다

상용 환경에서는 구성 서버에 대한 외부 접근을 차단

#### 구성 정보 보안

마이크로서비스나 구성 서버 API 사용자의 구성 정보 요청은 HTTPS를 사용하는 에지 서버에 의해 도청으로부터 보호됨

구성 서버는 스프링 시큐리티를 사용해 HTTP 기본 인증을 설정하며, SPRING_SECURITY_USER_NAME 및 SPRING_SECURITY_USER_PASSWORD 환경 변수에 자격 증명을 넣음

#### 구성 저장 보안

구성 서버는 구성 정보를 암호화해서 디스크에 저장

구성 서버는 대칭 키와 비대칭 키 모두 지원하는데, 비대칭 키가 더 안전하지만 관리가 어려움

### 구성 서버 API 소개

- /actuator: 모든 마이크로서비스가 노출하는 표준 액추에이터 엔드포인트로 주의를 기울여 사용해야 함
- /encrypt 및 /decrypt: 중요한 정보를 암호화하고 해독하기 위한 엔드포인트
- /{microservice}/{profile}: 지정한 마이크로서비스의 스프링 프로필 구성을 반환

## 구성 서버 설정

1. 스프링 부트 프로젝트 생성

2. build.gradle에 spring-cloud-config-server와 spring-boot-starter-security 읮노성 추가

3. `@EnableConfigServer` 추가

   ```java
   @EnableConfigServer
   @SpringBootApplication
   public class ConfigServerApplication {
   ```

4. application.yml에 구성 서버를 위한 구성 추가

   ```yaml
   server.port: 8888
   
   spring.cloud.config.server.native.searchLocations: file:${PWD}/config-repo
   
   # WARNING: Exposing all management endpoints over http should only be used during development, must be locked down in production!
   management.endpoint.health.show-details: "ALWAYS"
   management.endpoints.web.exposure.include: "*"
   
   logging:
     level:
       root: info
   
   ---
   spring.profiles: docker
   spring.cloud.config.server.native.searchLocations: file:/config-repo
   ```

5. 마이크로서비스 환경 외부에서 구성 서버 API에 접근할 수 있도록 에지 서버에 라우팅 규칙을 추가

6. Dockerfile을 추가하고 3개의 도커 컴포즈 파일에 구성 서버 정의 추가

7. 민감한 구성 매개 변수는 도커 컴포즈 환경 파일인 .env파일로 외부화

8. 공통 빌드 파일인  settings.gradle에 구성 서버 추가

   ```groovy
   include ':spring-cloud:config-server'
   ```

### 에지 서버에 라우팅 규칙 설정

에지 서버로 들어오는 /config로 시작하는 모든 요청은 다음 라우팅 규칙에 의해 구성 서버로 라우팅

```yaml
- id: config-server
  uri: http://${app.config-server}:8888
  predicates:
  - Path=/config/**
  filters:
  - RewritePath=/config/(?<segment>.*), /$\{segment} # /config의 앞 부분을 제거한 후 구성 서버로 보냄
```

### 도커 환경을 위한 구성 서버 설정

```yaml
config-server:
  environment:
    - SPRING_PROFILES_ACTIVE=docker,native # 깃이 아닌 일판 파일 기반위 구성 저장소를 사용하도록 구성 서버 설정 -> native 추가
    - ENCRYPT_KEY=${CONFIG_SERVER_ENCRYPT_KEY} # 대칭키
    - SPRING_SECURITY_USER_NAME=${CONFIG_SERVER_USR} # 자격 증명
    - SPRING_SECURITY_USER_PASSWORD=${CONFIG_SERVER_PWD} # 자격 증명
  volumes:
    - $PWD/config-repo:/config-repo
  build: spring-cloud/config-server
  mem_limit: 350m
```

환경 변수 값은 .env 파일에서 가져옴

```
CONFIG_SERVER_ENCRYPT_KEY=my-very-secure-encrypt-key
CONFIG_SERVER_USR=dev-usr
CONFIG_SERVER_PWD=dev-pwd
```

## 구성 서버의 클라이언트 설정

1. build.gradle에 spring-cloud-starter-config와 spring-retry 의존성 추가

2. application.yml을 구성 저장소로 옮기고, 이름을 spring.application.name 속성에 지정된 클라이언트 이름으로 변경

3. src/main/resources 폴더에 이름이 bootstrap.yml인 파일 추가. 구성 서버에 연결하는 데 필요한 구성 정보가 담겨 있다

4. 도커 컴포즈 파일에 구성 서버 접근에 필요한 자격 증명 추가

   ```yaml
   environment:
     - SPRING_PROFILES_ACTIVE=docker
     - CONFIG_SERVER_USR=${CONFIG_SERVER_USR}
     - CONFIG_SERVER_PWD=${CONFIG_SERVER_PWD}
   ```

5. 스프링 부트 기반의 자동 테스트를 실행할 때는 구성 서버를 사용하지 않도록 비활성화

   ```java
   @DataMongoTest(properties = {"spring.cloud.config.enabled=false"})
   
   @DataJpaTest(properties = {"spring.cloud.config.enabled=false"})
   
   @SpringBootTest(webEnvironment=RANDOM_PORT, properties = {"eureka.client.enabled=false","spring.cloud.config.enabled=false"})
   
   ```

### 연결 정보 설정

bootstrap.yml

```yaml
app.config-server: localhost

spring:
  application.name: product-composite
  cloud.config:
    failFast: true
    retry:
      initialInterval: 3000
      multiplier: 1.3
      maxInterval: 10000
      maxAttempts: 20
    uri: http://${CONFIG_SERVER_USR}:${CONFIG_SERVER_PWD}@${app.config-server}:8888

---
spring.profiles: docker

app.config-server: config-server
```

1. 클라이언트가 도커 외부에서 실행될 때는 http://localhost:8888 URL로 구성 서버에 접속하고, 도커 컨테이너에서 실행될 때는 http://config-server:8888 URL로 구성 서버에 접속
2. CONFIG_SERVER_USR 및 CONFIG_SERVER_PWD 속성 값을 이용해 HTTP 기본 인증 수행
3. 필요한 경우 시작하는 동안 최대 20번까지 구성 서버에 재접속을 시도
4. 접속에 실패하면 클라이언트는 3초 동안 기다린 후 재접속 시도
5. 최대 재접속 대기 시간은 10초
6. 20번까지 재접속을 시도해고 구성 서버에 접속하지 못하면 클라이언트는 시작에 실패

### 파티셔닝 구성을 도커 컴포즈 파일에서 구성 저장소로 이동

파티션을 추러하기 위한 추가 구성도 중앙 구성 저장소로 옮겨야 함

재사용성을 높이고자 구성을 여러 개의 스프링 프로필로 나누고, 구성 저장소에 있는 해당 구성 파일로 이동

메시지 소비자인 서비스의 구성 파일에 추가

```yaml
---
spring.profiles: streaming_partitioned # 메시지 브로커에서 파티션을 사용하기 위한 속성이 포함됨
spring.cloud.stream.bindings.input.consumer:
  partitioned: true
  instanceCount: 2

---
spring.profiles: streaming_instance_0 # 첫 번째 파티션에서 메시지를 소비하기 위한 속성이 포함됨
spring.cloud.stream.bindings.input.consumer.instanceIndex: 0

---
spring.profiles: streaming_instance_1 # 두 번째 파티션에서 메시지를 소비하기 위한 속성이 포함됨
spring.cloud.stream.bindings.input.consumer.instanceIndex: 1

---
spring.profiles: kafka # 카프카를 메시징 브로커로 사용하기 위한 속성이 포함됨

management.health.rabbit.enabled: false
spring.cloud.stream.defaultBinder: kafka
```

메시지 프로듀서인 서비스의 구성 파일에 다음 구성 추가

```yaml
---
spring.profiles: streaming_partitioned

spring.cloud.stream.bindings.output-products.producer:
  partition-key-expression: payload.key
  partition-count: 2

spring.cloud.stream.bindings.output-recommendations.producer:
  partition-key-expression: payload.key
  partition-count: 2

spring.cloud.stream.bindings.output-reviews.producer:
  partition-key-expression: payload.key
  partition-count: 2

---
spring.profiles: kafka

management.health.rabbit.enabled: false
spring.cloud.stream.defaultBinder: kafka
```



카프카 product 토픽의 첫 번째 파티션에서 메시지를 소비하는 product 소비자를 위한 구성

```yaml
product:
  environment:
    - SPRING_PROFILES_ACTIVE=docker,streaming_partitioned,streaming_instance_0,kafka
    - CONFIG_SERVER_USR=${CONFIG_SERVER_USR}
    - CONFIG_SERVER_PWD=${CONFIG_SERVER_PWD}
```

## 구성 저장소 구조화



## 스프링 클라우드 컨피그 서버 사용

### 빌드 및 자동화 테스트 실행

생략

### 구성 서버 API로 구성 조회

- 응답에는 여러 개의 프로퍼티 소스와 그 속성들이 포함돼 있으며, 각 프로퍼티 소스는 API로 요청한 스프링 프로필과 속성 파일에서 온 것
- 프로퍼티 소스는 우선순위에 따라 반환

### 민감한 정보의 암호화 및 해독

encrypt와 decrypt 엔드포인트 사용

민감한 정보 암호는 다음과 같이 저장

```yaml
password: '{cipher}17fcf0ae5b8c5cf87de6875b699be4a1746dd493a99d926c7a26a68c422117ef'
```

## 요약

구성 서버는  HTTP 기본 인증올 사용해 구성 정보를 보호하므로 자격 증명을 제공하는 사용자만  API 사용 가능

구성 서버 API는 도청 방지를 위해  HTTPS를 사용하는 에지 서버를 통해 외부로 노출

민감한 정보는 암호화하여 저장

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)