---
title: "스프링 부트 소개"
date: 2021-03-11
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 2. 스프링 부트 소개

## 스프링 부트

#### 설정보다 관례와 팻 JAR 파일

스프링 부트는 스프링 프레임워크와 서드파티 제품으로 구성된 핵심 모듈의 설정 방식을 개선해 상용 스프링 애플리케이션을 빠르게 개발하기 위한 프레임워크

설정보다 관례

팻 JAR은 아파치 톰캣과 같은 JAVA EE 웹서버를 별도로 설치하지 않아도 시작할 수 있도록 함 -> 도커에서 실행하기에 매우 적합

#### 스프링 부트 애플리케이션 설정에 대한 코드 예제

- `@SpringBootApplication` 애노테이션
  - 컴포넌트 검색을 활성화해 애플리케이션 클래스와 패키지와 모든 하위 패키지에서 스프링 컴포넌트와 구성 클래스를 검색
  - 애플리케이션 클래스 자체를 구성 클래스로 만든다
  - 자동 설정을 활성화해 스프링 부트가 설정 가능한 JAR파일을 클래스패스에서 자동으로 찾게함(ex. 톰캣)
- 컴포넌트 검색(`@ComponentScan`)
- 자바기반 구성(`@Configuration`)

## 스프링 웹플럭스

스프링 웹플럭스는 두 가지 프로그래밍 모델을 지원

- 애노테이션 기반 명령형 방식: 기존 웹 프레임워크인 스프링 웹 MVC와 유사하지만 리액티브 서비스를 지원
- 함수 지향(function-oriented) 모델 기반의 라우터 및 핸들러 방식

#### REST 서비스 설정에 대한 코드 예제

**스타터 의존성**

**속성 파일**

## 스프링 폭스

스웨거를 사용해 RESTful 서비스 문서를 공개하는 기능을 다수의 주요 API 게이트웨이가 내장

스프링 폭스는 스프링 프레임워크와는 별개의 오픈 소스 프로젝트로, 런타임에 스웨거 기반의 API 문서를 생성

## 스프링 데이터

스프링 데이터는 다양한 유형의 데이터베이스 엔진에 데이터를 저장히기 위한 공통 프로그래밍 모델을 제공

스프링 데이터 프로그래밍 모델의 핵심 개념은 entity와 repository

#### 엔티티

스프링 데이터가 저장하는 데이터

보통 엔티티 클래스에는 일반 스프링 데이터 애노테이션과 데이터베이스 기술에 따른 애노테이션을 혼합해 사용

#### 리포지토리

리포지토리는 여러 유형의 데이터베이스에 데이터를 저장하고 접근하고자 사용

스프링 데이터는 리포지토리를 간단히 정의할 수 있도록 CrudRepository 등의 몇 가지 기본 자바 인터페이스를 제공

리액티브 기반 인터페이스인 ReactiveCrudRepository도 제공

## 스프링 클라우드 스트림

스프링 클라우드 스트림은 게시-구독 통합 패턴을 기반으로 하는 메시징 방식의 스트리밍 추상화를 제공

스프링 클라우드 스트림은 현재 아파치 카프카와 RabbitMQ를 기본 지원

스프링 클라우드 스트림의 핵심 개념

- 메시지(Message): 메시징 시스템과 주고받는 데이터를 설명하는 데이터 구조
- 게시자(Publisher): 메시징 시스템에 메시지를 보낸다
- 구독자(Subscriber): 메시징 시스템에서 메시지를 받는다
- 채널(Channel): 메시징 시스템과 통신하는 데 사용, 게시자는 출력 채널을 사용하고 구독자는 입력 채널 사용
- 바인더(Binder): 특정 메시징 시스템과의 통합 기능을 제공

소비자 그룹(consumer group), 파티셔닝(partitioning), 영속성(persistence), 내구성(durability) 등의 메시징 기능과 오류 처리를 위한 재시도, 데드 레터 대기열 등을 재정의 가능

#### 스프링 클라우드 스트림을 사용한 메시지 송수신 예제

스프링 클라우드 스트림은 기본 입력 채널(Sink)와 출력 채널(Source)을 제공하므로 직접 만들 필요 X

아래 처럼 메시지 게시

```java
@EnableBinding(Source.class)
public class MyPublisher {
  @Autowired privet Source mysource;
  
  public string processMessage(MyMessage message) {
    mysource.output().send(MessageBuilder.withPayload(message).build());
  }
}
```

아래처럼 메시지 받은

```java
@EnableBinding(Sink.class)
public class MySubscriber {
  @StreamLister(target = Sink.INPUT)
  public void receive(MyMessage message) {
    LOG.info("Received {}", message);
  }
}
```

yaml 파일을 통헤 게시자 구성

```yaml
spring.cloud.stream:
	default.contentType: application/json
	bindings.output.destination: mydestination
```

yaml 파일을 통해 구독자 구성

```yaml
spring.cloud.stream:
	default.contentType: application/json
	bindings.input.destination: mydestination
```

## 도커

컨테이너는 리눅스 호스트에서 실행되며, 리눅스 네임스페이스를 이용해 사용자, 프로세스, 파일 시스템, 네트워킹 등의 전역 시스템 리소스를 컨테이너에 분배

또한 리눅스 제어 그룹을 사용해 컨테이너가 사용할 수 있는 CPU와 메모리를 제한

컨테이너는 개발과 테스트에 매우 유용



단일 커멘드로 여러 컨테이너를 시작 및 중지하고 싶다면 도커 컴포즈를 사용하는 게 좋다

```yaml
product:
	build: microservices/product-service
	
recommendation:
	build: microservices/recommendation-service
	
review:
	build: microservices/review-service
	
composite:
	build: microservices/composite-service
	ports:
		- "8080:8080"
```

- build 옵션은 각 마이크로 서비스에 사용할 Dockerfile을 지정. 도커 컴포즈는 Docerfile을 이용해 빌드한 후 이 이미지를 사용해 도커 컨테이너 시작
- ports 옵션은 도커를 실행하는 서버의 8080포트와 컨테이너의 8080포트를 매핑

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)

  

