---
title: "스프링 클라우드 게이트웨이를 에지 서버로 사용"
date: 2021-06-08
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 10. 스프링 클라우드 게이트웨이를 에지 서버로 사용

## 시스템 환경에 에지 서버 추가

외부 클라이언트는 모든 요청을 에지 서버로 보냄

서비스를 에지 서버 뒤로 숨기려면 도커 컴포즈 파일에 있는 두 서비스의 포트 선언을 제거해야 함

## 스프링 클라우드 게이트웨이 설정

1. 스프링 이니셜라이저로 스프링 부트 프로젝트 생성

2. spring-cloud-start-gateway 의존성 추가

3. spring-cloud-starter-netflix-eureka-client 의존성 추가

4. settings.gradle에 에지 서버 추가

   ```groovy
   include ':spring-cloud:gateway'
   ```

5. 마이크로서비스와 같은 내용의 Dockerfile을 추가

6. 도커 컴포즈 파일에 에지 서버 추가

   ```yaml
   gateway:
   	environment:
   		- SPRING_PROFILES_ACTIVE=docker
   	build: spring-cloud/gateway
   	mem_limit: 350m
   	ports:
   		- "8080:8080"
   ```

7. 라우팅 규칙을 위한 구성을 추가
8. 복합 상태 점검을 에저 서버로 옮김

### 복합 상태 점검 추가

에지서버에 추가

1. 상태 표시기(health indicator) 선언

   ```java
   @Bean
   ReactiveHealthContributor healthcheckMicroservices() {
   
     Map<String, ReactiveHealthContributor> map = new LinkedHashMap<>();
     map.put("product", (ReactiveHealthIndicator)() -> getHealth("http://product"));
     map.put("recommendation", (ReactiveHealthIndicator)() -> getHealth("http://recommendation"));
     map.put("review", (ReactiveHealthIndicator)() -> getHealth("http://review"));
     map.put("product-composite", (ReactiveHealthIndicator)() -> getHealth("http://product-composite"));
     return CompositeReactiveHealthContributor.fromMap(map);
   }
   
   private Mono<Health> getHealth(String url) {
      url += "/actuator/health";
      LOG.debug("Will call the Health API on URL: {}", url);
      return getWebClient().get().uri(url).retrieve().bodyToMono(String.class)
         .map(s -> new Health.Builder().up().build())
         .onErrorResume(ex -> Mono.just(new Health.Builder().down(ex).build()))
         .log();
   }
   ```

   

2. 상태 표시기 구현에 사용할 WebClient.builder 빈을 선언

   ```java
   @Bean
   @LoadBalanced
   public WebClient.Builder loadBalancedWebClientBuilder() {
      final WebClient.Builder builder = WebClient.builder();
      return builder;
   }
   ```

### 스프링 클라우드 게이트웨이 구성

가장 중요한 점은 라우팅 규칙 설정

1. 넷플릭스 유레카를 사용해 트래픽을 보낼 마이크로서비스를 찾으므로 유레카 클라이언트를 구성

2. 개발 환경을 위한 스프링부트 액추에이터 구성

   ```yaml
   management.endpoint.health.show-details: "ALWAYS"
   management.endpoints.web.exposure.include: "*"
   ```

3. 로그 레벨 구성

   ```yaml
   logging:
     level:
       root: INFO
       org.springframework.cloud.gateway.route.RouteDefinitionRouteLocator: INFO
       org.springframework.cloud.gateway: TRACE
   ```

#### 라우팅 규칙

라우팅 경로 정의 규칙

1. 조건자(predicate): 수신되는 HTTP 요청 정보를 바탕으로 경로를 선택
2. 필터(filter): 요청이나 응답을 수정
3. 대상 URI(destination URI): 요청을 보낼 대상
4. ID: 라우트 경로 이름

#### product-composite API로 요청 라우팅

```yaml
spring.cloud.gateway.routes:

- id: product-composite # 경로 이름 설정
  uri: lb://product-composite # 검색 서비스인 넷플릭스 유레카를 통해 product-composite이라는 서비스로 요청이 라우팅됨 lb://는 스프링 클라우드 게이트웨이가 클라이언트 측 로드 밸런서를 사용해 검섹 서비스에 대상을 찾도록 지시
  predicates:
  - Path=/product-composite/** # 라우팅 규칙이 처리할 요청을 지정 **은 0개 이상의 문자
```

#### 유레카 서버의 API와 웹 페이지로 요청 라우팅

유레카의 API와 웹 페이지를 명확하게 분리하고자 라우트 경로 설정

- 에지 서버로 전송된 경로가 /eureka/api로 시작하는 요청은 유레카 API에 대한 호출로 처리
- 에지 서버로 전송된 경로가 /eureka/web으로 시작하는 요청은 유레카 웹 페이지에 대한 호출로 처리

```yaml
- id: eureka-api
  uri: http://${app.eureka-server}:8761 # yaml 파일에 정의된 속성을 받음
  predicates:
  - Path=/eureka/api/{segment} # {segment}는 0개 이상의 문자와 일치
  filters:
  - SetPath=/eureka/{segment} # {segment} 부분 대체
```

```yaml
- id: eureka-web-start
  uri: http://${app.eureka-server}:8761
  predicates:
  - Path=/eureka/web
  filters:
  - SetPath=/

- id: eureka-web-other
  uri: http://${app.eureka-server}:8761
  predicates:
  - Path=/eureka/**
```

#### 조건자와 필터를 이용한 요청 라우팅

```yaml
- id: host_route_200
  uri: http://httpstat.us
  predicates:
  - Host=i.feel.lucky:8080
  - Path=/headerrouting/**
  filters:
  - SetPath=/200

- id: host_route_418
  uri: http://httpstat.us
  predicates:
  - Host=im.a.teapot:8080
  - Path=/headerrouting/**
  filters:
  - SetPath=/418

- id: host_route_501
  uri: http://httpstat.us
  predicates:
  - Path=/headerrouting/**
  filters:
  - SetPath=/501
```

## 에지 서버 테스트

생략

## 요약

조건자, 필터, 대상 URI를 바탕으로 유연한 라우팅 규칙 정의 가능.

검색 서비스를 사용해 대상 마이크로서비스 인스턴스를 조회하도록 스프링 클라우드 게이트웨이 구성 가능



## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)