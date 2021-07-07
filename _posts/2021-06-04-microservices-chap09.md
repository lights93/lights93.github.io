---
title: "넷플릭스 유레카와 리본을 사용한 서비스 검색"
date: 2021-06-04
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 9. 넷플릭스 유레카와 리본을 사용한 서비스 검색

## 서비스 검색 소개

### DNS 기반 서비스 검색의 문제

라운드 로빈 방식 DNS를 이용해 검색하면 안 되는 이유??

DNS 클라이언트는 보통 리졸브된 IP 주소를 캐시하며, DNS 이름에 대응되는 IP 주소가 여러 개인 경우에는 동작하는 첫 번째 IP 주소를 계속 사용한다.

따라서  DNS 서버와 DNS 프로토콜은 동적으로 변하는 마이크로서비스 인스턴스르 처리하는 데 적합하지 않으며, 실무 관점에서도 DNS에 기반한 서비스 검색이 그리 매력적이지 않다/

### 서비스 검색의 문제

다음 사항을 고려해 마이크로서비스 인스턴스를 추적해야 한다

- 언제든지 새로운 인스턴스가 시작될 수 있다.
- 언제든지 기존 인스턴스가 응답하지 않을 수 있으며, 결국에는 중단될 수 있다.
- 실패한 인스턴스 중 일부는 얼마 후에 정상 상태로 돌아와서 다시 트래픽을 수신하지만 그렇지 못하는 인스턴스도 있다. 실패에서 복구되지 못한 인스턴스는 서비스 레지스트리에서 제거해야 한다.
- 일부 마이크로서비스 인스턴스는 시작하는 데 시간이 걸릴 수 있다.
- 언제든지 의도하지 않는 네트워크 파티셔닝과 그 밖의 네트워크 관련 오류가 발생할 수 있다.

### 넷플릭스 유레카를 이용한 서비스 검색

넷플릭스 유레카는 클라이언트 측 서비스 검색을 구현

즉 클라이언트는 사용 가능한 마이크로서비스 인스턴스의 정보를 얻고자 검색 서비스 넷플릭스 유레카와 통신하는 소프트웨어를 실행

1. 마이크로서비스 인스턴스는 시작할 떄마다 자신을 유레카 서버에 등록
2. 각 마이크로서비스 인스턴스는 자신이 정상이며 요청을 받을 준비가 됐을음 알리고자 정기적으로 유레카 서버에 하트비트 메시지를 보냄
3. 클라이언트는 클라이언트 라이브러리를 사용해, 사용 가능한 서비스의 정보를 정기적으로 유레카 서비스에 요청
4. 클라이언트에서 다른 마이크로서비스로 요청을 보내야 하는 경우에는 검색 서비스에 요청하지 않고도 클라이언트 라이브러리에 보존된 사용 가능한 인스턴스 목록에서 대상을 선택할 수 있으며, 보통은 라운드 로빈 방식으로 인스턴스를 선택

스프링 클라우드는 넷플릭스 유레카와 같은 검색 서비스와 통신하는 방법을 추상화한 DiscoveryClient라는 인터페이스를 제공

DiscoveryClient를 사용하면 검색 서비스와 연계해 사용 가능한 서비스 및 인스턴스에 관한 정보를 얻을 수 있다.

DiscoverClient 인터페이스 구현은 자동으로 스프링 부트 애플리케이션을 검색 서버에 등록하는 기능도 있다.   



스프링 부트는 시작하는 동안 DiscoveryClient 인터페이스 구현을 자동으로 찾음

의존성 추가 필요 -> 넷플릭스 유레카(spring-cloud-starter-netflix-eureka-client)   

 

스프링 클라우드는 로드 밸런서를 통해 검색 서비스에 등록된 인스턴스로 요청을 보내는 방법을 추상화한 LoadBalancerClient 인터페이스를 제공

WebClient.Builder 객체를 반환하는 `@Bean`선언에 `@LoadBalanced` 애노테이션을 추가하면 LoadBalancerClient 구현이 Builder 인스턴스에 ExchangeFilterFunction으로 주입됨

클래스 경로에 spring-cloud-starter-netflix-eureka-client 의존성이 있는 경우 넷플릭스 리본 기반의 로드 밸런서 클라이언트인 RibbonLoadBalancer가 자동으로 주입됨

## 넷플릭스 유레카 서버 설정

1. 스프링 이니셜라이저를 사용해 스프링 부트 프로젝트를 생성

2. spring-cloud-starter-netflix-eureka-client 의존성 추가

3. `@EnableEurekaServer` 애노테이션 추가

4. 포트 8761로 변경

5. 도커 컴포즈 파일에 유레카 서버 추가

   ```yaml
   eureka:
     build: spring-cloud/eureka-server
     mem_limit: 350m
     ports:
       - "8761:8761"
   ```

6. 유레카 서버 및 마이크로서비스에 대한 구성을 설정

## 넷플릭스 유레카 서버에 마이크로서비스 연결

마이크로서비스 인스턴스 등록

1. 빌드 파일에 spring-cloud-starter-netflix-eureka-client 의존성 추가

   ```groovy
   implementation('org.springframework.cloud:spring-cloud-starter-netflix-eureka-client')
   ```

2. 단일 마이크로서비스 테스트 시에는 유레카 구성일 필요 없어서 비활성화 필요

   ```java
   @SpringBootTest(webEnvironment=RANDOM_PORT, properties = {"eureka.client.enabled=false"})
   ```

3. 구성 설정

spring.application.name 속성을 사용해 각 마이크로서비스에 가상 호스트 이름, 즉 유레카 서비스가 각 마이크로서비스를 식별하는 데 사용하는 이름을 부여



마이크로서비스 인스턴스를 찾을 수 있도록 다음 단계 수행

1. webclient.builder에 `@LoadBalanced` 애노테이션 추가

   ```java
   	@Bean
   	@LoadBalanced
   	public WebClient.Builder loadBalancedWebClientBuilder() {
   		final WebClient.Builder builder = WebClient.builder();
   		return builder;
   	}
   ```

2. 통합 클래스에서 생성자가 실행될 떄까지는 수행되지 않음, 늦은 초기화 방식으로 webClient 생성이 필요함

   ```java
   private WebClient getWebClient() {
     if (webClient == null) {
       webClient = webClientBuilder.build();
     }
     return webClient;
   }
   ```

3. getter 메서드를 이용하여 접근 필요

   ```java
   @Override
   public Mono<Product> getProduct(int productId) {
       String url = productServiceUrl + "/product/" + productId;
       LOG.debug("Will call the getProduct API on URL: {}", url);
   
       return getWebClient().get().uri(url).retrieve().bodyToMono(Product.class).log().onErrorMap(WebClientResponseException.class, ex -> handleException(ex));
   }
   ```

4. 하드 코딩해서 구성했던 마이크로서비스의 목록 제거 후 대체 선언

   ```java
   private final String productServiceUrl = "http://product"; // spring.application.name 임
   private final String recommendationServiceUrl = "http://recommendation";
   private final String reviewServiceUrl = "http://review";
   ```

## 개발 프로세스에서 사용할 구성 설정

넷플릭스 유레카를 사용하는 경우에는 기본 구성 값 때문에 시작 시간이 길어질 수 있음

개발 과정에선 이런 대기 시간을 최소화하는 구성을 사용하는 것이 유용하며, 상용화 과정에선 기본값 바탕의 구성을 사용

### 유레카 구성 매개 변수

- eureka.server: 유레카 서버 관련 매개 변수
- eureka.client: 유레카 서버와 통신하는 클라이언트를 위한 것
- eukera.instance: 유레카 서버에 자신을 등록하려는 마이크로서비스 인스턴스를 위한 것

### 유레카 서버 구성

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  # from: https://github.com/spring-cloud-samples/eureka/blob/master/src/main/resources/application.yml
  server: # 유레카 서버의 시작시간을 최소화하기 위하여 아래 2개 추가
    waitTimeInMsWhenSyncEmpty: 0
    response-cache-update-interval-ms: 5000
```

### 유레카 서버에 연결할 클라이언트 구성

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/ # 유레카 서버를 찾기 위함, 이외 변수들은 시간 최소화용
    initialInstanceInfoReplicationIntervalSeconds: 5
    registryFetchIntervalSeconds: 5
  instance:
    leaseRenewalIntervalInSeconds: 5
    leaseExpirationDurationInSeconds: 5
    
# 다른 마이크로서비스를 조회하는 서비스에만 필요함 (시간 줄이기 위한 설정)
ribbon.ServerListRefreshInterval: 5000
ribbon.NFLoadBalancerPingInterval: 5
---
spring.profiles: docker

eureka.client.serviceUrl.defaultZone: http://eureka:8761/eureka/
```

## 검색 서비스 사용

### 유레카 서버의 장애 상황 테스트

#### 유레카 서버 중지

서버가 없어도 클라이언트가 기존 인스턴스 호출 가능

#### review 인스턴스 중지

여전히 2개의 인스턴스가 실행 중인 줄 안다

#### product 인스턴스 추가

새로운 인스턴스가 나타난 줄 모름

#### 유레카 서버 다시 시작

업데이트 됨

## 요약

넷플릭스 유레카는 런타임 특성으로 내결함성, 견고성, 탄력성을 가지는 매우 뛰어난 서비스 검색 서

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)