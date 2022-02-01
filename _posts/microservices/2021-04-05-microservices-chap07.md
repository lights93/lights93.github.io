---
title: "리액티브 마이크로서비스 개발"
date: 2021-04-05
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 7. 리액티브 마이크로서비스 개발

## 논블로킹 동기 API와 이벤트 기반 비동기 서비스의 선택 기준

일반적으로 견고하고 확장성 있는 마이크로서비스를 만들려먼 가능한 한 자율적으로 만드는 것이 중요

런타임 의존성을 최소화해야 함 -> **느슨한 결합(loose coupling)**

따라서 동기 API 방식보다는 이벤트 기반 비동기 메시지 전달 방식을 선호

논블로킹 동기 API를 사용하는 것이 유리한 경우도 있음

- 최종 사용자가 응답을 기다리는 읽기 작업일 때
- 모바일 앱이나 SPA 웹 애플리케이션처럼 동기 API가 알맞은 클라이언트 플랫폼일 때
- 클라이언트가 다른 조직의 서비스에 연결할 때

현재 프로젝트에서의 시스템 환경

- product-composite 마이크로서비스가 공개하는 생성, 읽기, 삭제, 서비스는 동기 API 기반
- 핵심 마이크로서비스가 제공하는 읽기 서비스는 응답을 기다리는 최종 사용자가 있기 때문에 논블로킹 동기 API로 개발
- 핵심 마이크로서비스의 생성 및 삭제 서비스는 이벤트 기반 비동기 서비스로 개발

## 스프링을 사용해 논블로킹 동기 REST API 개발

### 스프링 리액터 소개

스프링 5의 리액티브 지원은 **프로젝트 리액터** 기반

프로젝트 리액터는 리액티브 애플리케이션 구축의 표준인 리액티브 스트림 사양을 기반으로 함

데이터를 스트림으로 처리하는 프로그래밍 모델을 사용하며, 프로젝트 리액터의 Flux와 Mono를 핵심 데이터 유형으로 사용

### 스프링 데이터 MongoDB를 사용한 논블로킹 영속성

```java
public interface ProductRepository extends ReactiveCrudRepository<ProductEntity, String> { //Reactive로 변경
    Mono<ProductEntity> findByProductId(int productId); // Mono로 변경
}

```

#### 테스트 코드 변경

block()을 이용하여 결과를 받을 때까지 기다리거나, StepVerifier를 사용해 검증 가능한 비동기 이벤트 시퀀스 선언

```java
@Test //block() 방식
public void getByProductId() {
  List<RecommendationEntity> entityList = repository.findByProductId(
    savedEntity.getProductId()).collectList().block();

  assertThat(entityList, hasSize(1));
  assertEqualsRecommendation(savedEntity, entityList.get(0));
}

@Test // stepverifier 방식
public void getByProductId() {

  StepVerifier.create(repository.findByProductId(savedEntity.getProductId()))
    .expectNextMatches(foundEntity -> areProductEqual(savedEntity, foundEntity))
    .verifyComplete();
}
```

### 핵심 서비스의 논블로킹 REST API

#### API 변경

```java
Mono<Product> getProduct(@PathVariable int productId); // Mono로 변경
```

#### 서비스 구현 변경

```java
@Override
public Mono<Product> getProduct(int productId) { // Mono 반환

  if (productId < 1)
    throw new InvalidInputException("Invalid productId: " + productId);

  return repository.findByProductId(productId)
    .switchIfEmpty(error(new NotFoundException("No product found for productId: " + productId)))
    .log()
    .map(e -> mapper.entityToApi(e))
    .map(e -> {
      e.setServiceAddress(serviceUtil.getServiceAddress());
      return e;
    });
}
```

#### 테스트 코드 변경

생략

#### 블로킹 코드 처리

RDB에 접근하는 review 서비스는 Scheduler를 사용해 블로킹 코드를 실행

스케줄러는 일정 수의 스레드를 보유한 전용 스레드 플의 스레드에서 블로킹 코드를 실행

스레드 풀을 사용해 블로킹 코드를 처리하면 마이크로서비스에서 사용할 스레드의 고갈을 방지하므로 논블로킹 처리에 영향을 주지 않음

```java
	@Autowired
	public ReviewServiceApplication(
		@Value("${spring.datasource.maximum-pool-size:10}")
			Integer connectionPoolSize
	) {
		this.connectionPoolSize = connectionPoolSize;
	}

	@Bean
	public Scheduler jdbcScheduler() {
		LOG.info("Creates a jdbcScheduler with connectionPoolSize = " + connectionPoolSize);
		return Schedulers.fromExecutor(Executors.newFixedThreadPool(connectionPoolSize)); // 스레드 풀 구성
	}
```

```java
private final Scheduler scheduler;

@Autowired
public ReviewServiceImpl(Scheduler scheduler) {
  this.scheduler = scheduler;
}
```

```java
@Override
public Flux<Review> getReviews(int productId) {

  if (productId < 1)
    throw new InvalidInputException("Invalid productId: " + productId);

  LOG.info("Will get reviews for product with id={}", productId);

  return asyncFlux(() -> Flux.fromIterable(getByProductId(productId))).log(null, FINE);
}

protected List<Review> getByProductId(int productId) { // 블로킹코드 구현

  List<ReviewEntity> entityList = repository.findByProductId(productId);
  List<Review> list = mapper.entityListToApiList(entityList);
  list.forEach(e -> e.setServiceAddress(serviceUtil.getServiceAddress()));

  LOG.debug("getReviews: response size: {}", list.size());

  return list;
}

// 스레드 풀의 스레드에서 블로킹 코드 실행
private <T> Flux<T> asyncFlux(Supplier<Publisher<T>> publisherSupplier) {
  return Flux.defer(publisherSupplier).subscribeOn(scheduler);
}
```

## 복합 서비스의 논블로킹 REST API

#### API 변경

```java
Mono<ProductAggregate> getCompositeProduct(@PathVariable int productId); // Mono로 변경
```

#### 통합 계층 변경

RestTemplate을 WebClient로 대체

1. 별도 구성 없이 통합 클래스에서 사용할 WebClient 인스턴스를 빌드

   ```java
   public class ProductCompositeIntegration implements ProductService, RecommendationService, ReviewService {
   
       private final WebClient webClient;
   
       @Autowired
       public ProductCompositeIntegration(WebClient.Builder webClient, ...) {
           this.webClient = webClient.build();
       }
   ```

2. webClient 인스턴스를 사용한 논블로킹 방식으로 product 서비스 호출

   ```java
   @Override
   public Mono<Product> getProduct(int productId) {
     String url = productServiceUrl + "/product/" + productId;
     LOG.debug("Will call the getProduct API on URL: {}", url);
   
     return webClient.get().uri(url).retrieve().bodyToMono(Product.class).log().onErrorMap(WebClientResponseException.class, ex -> handleException(ex));
     // onErrorMap은 예외를 자체 예외로 변경
   }
   ```

   ```java
   @Override
   public Flux<Recommendation> getRecommendations(int productId) {
   
     String url = recommendationServiceUrl + "/recommendation?productId=" + productId;
   
     LOG.debug("Will call the getRecommendations API on URL: {}", url);
   
     // api 실패시 다른 정보라도 전달할 수 있도록 empty() 리턴
     return webClient.get().uri(url).retrieve().bodyToFlux(Recommendation.class).log().onErrorResume(error -> empty());
   }
   ```

#### 서비스 구현 변경

zip()을 사용해 세 가지 API를 병렬로 호출

```java
@Override
public Mono<ProductAggregate> getCompositeProduct(int productId) {
  return Mono.zip(
    values -> createProductAggregate((Product) values[0], (List<Recommendation>) values[1], (List<Review>) values[2], serviceUtil.getServiceAddress()), // 3개의 API를 호출한 결괏값은 앞서와 같이 createProductAggregate 헬퍼 메서드로 집계
    integration.getProduct(productId), // 병렬로 호출할 요청의 목록 3가지
    integration.getRecommendations(productId).collectList(),
    integration.getReviews(productId).collectList())
    .doOnError(ex -> LOG.warn("getCompositeProduct failed: {}", ex.toString()))
    .log();
}
```

#### 테스트 코드 변경

```java
@Before
public void setUp() {
// Mono.just() 와 Flux.fromIterable()을 이용하여 return하도록 변경
  when(compositeIntegration.getProduct(PRODUCT_ID_OK)).
    thenReturn(Mono.just(new Product(PRODUCT_ID_OK, "name", 1, "mock-address")));

  when(compositeIntegration.getRecommendations(PRODUCT_ID_OK)).
    thenReturn(Flux.fromIterable(singletonList(new Recommendation(PRODUCT_ID_OK, 1, "author", 1, "content", "mock address"))));

  when(compositeIntegration.getReviews(PRODUCT_ID_OK)).
    thenReturn(Flux.fromIterable(singletonList(new Review(PRODUCT_ID_OK, 1, "author", "subject", "content", "mock address"))));

  when(compositeIntegration.getProduct(PRODUCT_ID_NOT_FOUND)).thenThrow(new NotFoundException("NOT FOUND: " + PRODUCT_ID_NOT_FOUND));

  when(compositeIntegration.getProduct(PRODUCT_ID_INVALID)).thenThrow(new InvalidInputException("INVALID: " + PRODUCT_ID_INVALID));
}
```

## 이벤트 기반 비동기 서비스 개발

복합 서비스는 생성 및 삭제 이벤트를 각 핵신 서비스의 토픽에 게시한 후 핵심 서비스의 처리를 기다리지 않고 호출자에게 OK응답 반환

### 메시징 관련 문제를 처리하도록 스프링 클라우드 스트림 구성

#### 소비자 그룹

소비자 그룹을 만들어 소비자 유형별로 하나의 인스턴스가 메시지를 처리하게 해야 함

```yaml
spring.cloud.stream:
  bindings.input:
    destination: products
    group: productsGroup # group 필드를 사용해 소비자 그룹으로 묶음
```

#### 재시도 및 데드 레터 대기열

소비자가 메시지 처리에 실패하면 메시지는 실패한 소비자가 성공적으로 처리할 때까지 대기열로 다시 보내지거나 사라짐

내용이 잘못된 메시지인 경우엔 수동으로 메시지를 제거할 떄까지 다른 메시지를 처리하지 못하도록 소비자를 차단

네트워크 오류로 데이터베이스에 연결할 수 없는 경우와 같이 일시적인 문제로 실패한 경우에는 여러 번의 재시도로 처리가 성공할 수 있음



결함 분석 및 수정을 위해 메시지를 다른 저장소로 이동하기 전에 수행할 재시도 횟수를 지정할 수 있어야 함

```yaml
spring.cloud.stream.bindings.input.consumer:
  maxAttempts: 3 # 재시도 횟수
  backOffInitialInterval: 500 # 첫 번째 재 시도는 500ms 후에 실행
  backOffMaxInterval: 1000 # 다른 두 번의 재시도는 1000ms후에 실행
  backOffMultiplier: 2.0

spring.cloud.stream.rabbit.bindings.input.consumer:
  autoBindDlq: true # dlq에 autobind하는지..
  republishToDlq: true # true로 설정하면 failed message가 dlq로 이동

spring.cloud.stream.kafka.bindings.input.consumer:
  enableDlq: true # dlq는 dead letter queue 의 줄임말인듯
```

#### 순서 보장 및 파티션

파티션을 사용하면 성능과 확장성을 잃지 않으면서도 전송됐을 때의 순서 그대로 메시지를 전달 가능

비즈니스 로직에서 메시지가 전송된 순서대로 메시지를 소비하고 처리해야 하는 경우엔 여러 개의 소비자 인스턴스를 사용해 처리 성능을 높일 수 없다

대부분 엄격하게 순서를 지켜서 메시지를 처리해야 하는 경우는 같은 비즈니스 엔티티에만 영향을 줄 때이다.

이런 문제를 해결하려면 하위 토픽이라고도 알려진 **파티션**을 도입해 메시징 시스템이 같은 키를 가진 메시지 사이의 순서를 보장하고자 사용할 키를 각 메시지에 지정

같은 키를 가진 메시지는 언제나 같은 파티션에 배치되며, 하나의 동일 파티션에 속한 메시지만 전달 순서가 보장됨

메시지 순서 보장을 위해 소비자 그룹 안의 각 파티션마다 하나의 소비자 인스턴스가 배정됨   



게시자 측과 소비자 측 양쪽에서 스프링 클라우드 스트림을 구성하고, 게시자 측에서 키 및 파티션 수를 지정해야 함

```yaml
spring.cloud.stream.bindings.output:
	destination: products
	producer:
		partition-key-expression: payload.key # 페이로드의 key 필드에서 키를 가져옴
		partition-count: 2 # 사용하는 파티션 개수
```

```yaml
spring.cloud.stream.bindings.input:
	destination: products
	group: productsGroup
	consumer:
		partitioned: true
		instance-index: 0 # 0번 파티션의 메시지만 소비함
```

### 토픽 및 이벤트 정의

메시징 시스템은 보통 헤더와 본문으로 구성된 메시지를 다루는데, 어떤 상황이 발생했다는 것을 설명하는 메시지가 바로 이벤트다.

이벤트 메시지 본문에는 이벤트 유형, 이벤트 데이터, 타임스탬프(이벤트 발생 시간)가 들어 있다.

```java
public class Event<K, T> { 

    public enum Type {CREATE, DELETE}

    private Event.Type eventType; // 이벤트 유형(예: 생성, 삭제)
    private K key; // 데이터 식별을 위한 키
    private T data; // 실제 이벤트 데이터
    private LocalDateTime eventCreatedAt; // 이벤트 발생 시간

    public Event() {
        this.eventType = null;
        this.key = null;
        this.data = null;
        this.eventCreatedAt = null;
    }

    public Event(Type eventType, K key, T data) {
        this.eventType = eventType;
        this.key = key;
        this.data = data;
        this.eventCreatedAt = now();
    }

    public Type getEventType() {
        return eventType;
    }

    public K getKey() {
        return key;
    }

    public T getData() {
        return data;
    }

    public LocalDateTime getEventCreatedAt() {
        return eventCreatedAt;
    }
}
```

#### 그래들 빌드 파일 변경

```groovy
dependencies { // 스프링 클라우드 스트림 관련 의존성 추가
  implementation('org.springframework.cloud:spring-cloud-starter-stream-rabbit')
	implementation('org.springframework.cloud:spring-cloud-starter-stream-kafka')
	testImplementation('org.springframework.cloud:spring-cloud-stream-test-support')
}

ext { // 스프링 클라우드 버젼 지정
	springCloudVersion = "Hoxton.SR6"
}

dependencyManagement { // 스프링 클라우드 의존성 설정
	imports {
		mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
	}
}
```

#### 복합 서비스에서 이벤트 게시

1. 메시지 소스를 선언하고 통합 계층에서 이벤트 게시

   1. ProductCompositeIntegration 클래스의  MessageSources 인터페이스에 메시지 채널을 선언하고, 해당 인스턴스를 생성자로 주입

      ```java
      @EnableBinding(ProductCompositeIntegration.MessageSources.class)
      @Component
      public class ProductCompositeIntegration implements ProductService, RecommendationService, ReviewService {
          private MessageSources messageSources;
      
          public interface MessageSources {
      
              String OUTPUT_PRODUCTS = "output-products";
              String OUTPUT_RECOMMENDATIONS = "output-recommendations";
              String OUTPUT_REVIEWS = "output-reviews";
      
              @Output(OUTPUT_PRODUCTS)
              MessageChannel outputProducts();
      
              @Output(OUTPUT_RECOMMENDATIONS)
              MessageChannel outputRecommendations();
      
              @Output(OUTPUT_REVIEWS)
              MessageChannel outputReviews();
          }
      
          @Autowired
          public ProductCompositeIntegration(MessageSources messageSources) {
              this.messageSources = messageSources;
          }
      ```

   2. MessageBuilder 클래스를 사용해 이벤트가 담긴 메시지를 작성

      ```java
      @Override
      public void deleteProduct(int productId) {
        messageSources.outputProducts().send(MessageBuilder.withPayload(new Event(DELETE, productId, null)).build());
      }
      ```

2. 이벤트 게시를 위한 구성 추가

   1. 기본 메시징 시스템이  RabbitMQ고, 기본 콘텐츠 유형이 JSON임을 선언

      ```yaml
      spring.cloud.stream:
        defaultBinder: rabbit
        default.contentType: application/json
      ```

   2. 출력 채널과 토픽 이름을 바인드(bind)

      ```yaml
      bindings:
      	output-products:
        	destination: products
        output-recommendations:
        	destination: recommendations
        output-reviews:
        	destination: reviews
      ```

   3. 카프카 및 RabbitMQ에 대한 연결 정보 선언

      ```yaml
      spring.cloud.stream.kafka.binder:
        brokers: 127.0.0.1
        defaultBrokerPort: 9092
      
      spring.rabbitmq:
        host: 127.0.0.1
        port: 5672
        username: guest
        password: guest
        
      ---
      spring.profiles: docker
      
      spring.rabbitmq.host: rabbitmq
      
      spring.cloud.stream.kafka.binder.brokers: kafka
      ```

3. 테스트 코드 변경

   1. 각 토픽으로 전송되는 메시지 검사용 대기열을 MessagingTests 테스트 클래스에 설정

      ```java
      	@Autowired
      	private MessageCollector collector;
      
      	BlockingQueue<Message<?>> queueProducts = null;
      	BlockingQueue<Message<?>> queueRecommendations = null;
      	BlockingQueue<Message<?>> queueReviews = null;
      
      	@Before
      	public void setUp() {
      		queueProducts = getQueue(channels.outputProducts());
      		queueRecommendations = getQueue(channels.outputRecommendations());
      		queueReviews = getQueue(channels.outputReviews());
      	}
      
      	private BlockingQueue<Message<?>> getQueue(MessageChannel messageChannel) {
      		return collector.forChannel(messageChannel);
      	}
      
      }
      
      ```

   2. 테스트 코드에서 제품 생성 대기열의 내용 확인

      ```java
      @Test
      public void createCompositeProduct1() {
      
        ProductAggregate composite = new ProductAggregate(1, "name", 1, null, null, null);
        postAndVerifyProduct(composite, OK);
      
        // Assert one expected new product events queued up
        assertEquals(1, queueProducts.size());
      
        Event<Integer, Product> expectedEvent = new Event(CREATE, composite.getProductId(), new Product(composite.getProductId(), composite.getName(), composite.getWeight(), null));
        assertThat(queueProducts, is(receivesPayloadThat(sameEventExceptCreatedAt(expectedEvent))));
      
        // Assert none recommendations and review events
        assertEquals(0, queueRecommendations.size());
        assertEquals(0, queueReviews.size());
      }
      
      private void postAndVerifyProduct(ProductAggregate compositeProduct, HttpStatus expectedStatus) {
        client.post()
          .uri("/product-composite")
          .body(just(compositeProduct), ProductAggregate.class)
          .exchange()
          .expectStatus().isEqualTo(expectedStatus);
      }
      ```

#### 핵심 서비스에서 소비

1. 메시지 프로세서 선언

   각 메시지 프로세서는 하나의 토픽만 수신하므로  Sink 인터페이스로 토픽을 바인딩

   ```java
   @EnableBinding(Sink.class) // Sink 인터페이스를 사용하기 위한 애노테이션 설정
   public class MessageProcessor {
   ```

   ```java
   @StreamListener(target = Sink.INPUT) // 수신 채널 지정
   public void process(Event<Integer, Product> event) {
   ```

   ```java
   switch (event.getEventType()) {
   
   case CREATE:
       Product product = event.getData();
       LOG.info("Create product with ID: {}", product.getProductId());
       productService.createProduct(product);
       break;
   
   case DELETE:
       int productId = event.getKey();
       LOG.info("Delete recommendations with ProductID: {}", productId);
       productService.deleteProduct(productId);
       break;
   
   default:
       String errorMessage = "Incorrect event type: " + event.getEventType() + ", expected a CREATE or DELETE event";
       LOG.warn(errorMessage);
       throw new EventProcessingException(errorMessage);
   }
   ```

2. 서비스 구현 변경

   MongoDB를 위한 논블로킹 리액티브 영속성 계층을 사용하도록 product 및  recommendation 서비스의 생성 및 삭제 메서드 새로 작성

   ```java
   public class ProductServiceImpl implements ProductService {
   
   	@Override
   	public Product createProduct(Product body) {
   
   		if (body.getProductId() < 1)
   			throw new InvalidInputException("Invalid productId: " + body.getProductId());
   
   		ProductEntity entity = mapper.apiToEntity(body);
   		Mono<Product> newEntity = repository.save(entity)
   			.log()
   			.onErrorMap(
   				DuplicateKeyException.class,
   				ex -> new InvalidInputException("Duplicate key, Product Id: " + body.getProductId()))
   			.map(e -> mapper.entityToApi(e));
   
   		return newEntity.block();
       // 메시지 프로세서는 블로킹 프로그래밍 모델을 기반으로 하므로 영속성 계층에서 받은 Mono 객체의 block() 메서드를 호출하지 않으면 메시징 시스템이 서비스 구현에서 발생한 오류를 처리하지 못하므로 이벤트가 대기열로 다시 들어가지 못하고 데드 레터 대기열로 이동하게됨
   	}
   }
   ```

3. 이벤트 소비를 위한 구성 추가

   게시자를 위해 했던 것과 유사

   생략

4. 테스트 코드 변경

   ```java
   @Autowired
   private Sink channels;
   
   private AbstractMessageChannel input = null;
   
   @Before
   public void setupDb() {
     input = (AbstractMessageChannel) channels.input();
     repository.deleteAll().block();
   }
   
   private void sendCreateProductEvent(int productId) {
     Product product = new Product(productId, "Name " + productId, productId, "SA");
     Event<Integer, Product> event = new Event(CREATE, productId, product);
     input.send(new GenericMessage<>(event));
   }
   
   private void sendDeleteProductEvent(int productId) {
     Event<Integer, Product> event = new Event(DELETE, productId, null);
     input.send(new GenericMessage<>(event));
   }
   ```

## 리액티브 마이크로서비스 환경의 수동 테스트

세 가지 구성을 별도의 도커 컴포즈 파일로 준비

- 파티션 없이  RabbitMQ 사용
- 토픽당 2개의 파티션으로 RabbitMQ 사용
- 토픽당 2개의 파티션으로 카프카 사용

### 이벤트 저장

각 토픽에 게시된 이벤트를 확인할 수 있도록 각 토픽에 게시된 이벤트를 별도의 소비자 그룹인  auditGroup에 저장하도록 구성

```yaml
spring.cloud.stream:
  bindings:
    output-products:
      destination: products
      producer:
        required-groups: auditGroup
```

### 상태 점검  API 추가

마이크로서비스가 요청 및 메시지를 처리할 준비가 됐는지 쉽게 확인하고자 모든 마이크로서비스에 상태 점검 API를 추가

상태 점검 API는 스프링 부트 액추에이터 모듈에서 제공하는 상태 점검 엔드포인트를 기반으로 작동

액추에이터 기반의 health 엔드포인트는 마이크로서비스와 마이크로서비스가 의존하는 모든 의존성이 정상인 경우에  UP으로 응답, HTTP 상태 코드 200 반환

정상이 아닌 경우  DOWN으로 응답하며,  HTTP 상태 코드 500 반환

   

health 엔드포인트를 확장하면 스프링 부트가 다루지 않는 부분의 상태도 점검 가능

```java
public Mono<Health> getProductHealth() {
  return getHealth(productServiceUrl);
}

public Mono<Health> getRecommendationHealth() {
  return getHealth(recommendationServiceUrl);
}

public Mono<Health> getReviewHealth() {
  return getHealth(reviewServiceUrl);
}

private Mono<Health> getHealth(String url) {
  url += "/actuator/health";
  LOG.debug("Will call the Health API on URL: {}", url);
  return webClient.get().uri(url).retrieve().bodyToMono(String.class)
    .map(s -> new Health.Builder().up().build())
    .onErrorResume(ex -> Mono.just(new Health.Builder().down(ex).build()))
    .log();
}
```

```java
@Autowired
ProductCompositeIntegration integration;

@Bean
ReactiveHealthContributor coreServices() {

  Map<String, ReactiveHealthContributor> map = new LinkedHashMap<>();
  map.put("product", (ReactiveHealthIndicator)() -> integration.getProductHealth());
  map.put("recommendation", (ReactiveHealthIndicator)() -> integration.getRecommendationHealth());
  map.put("review", (ReactiveHealthIndicator)() -> integration.getReviewHealth());
  return CompositeReactiveHealthContributor.fromMap(map);
}
```

```yaml
management.endpoint.health.show-details: "ALWAYS" # UP, DOWN 뿐만 아니라 의존성에 대한 상태까지 확인
management.endpoints.web.exposure.include: "*" # 모든 엔드포인트 공개
```

### 파티션 없이 RabbitMQ 사용

```yaml
  product-composite:
    build: microservices/product-composite-service
    mem_limit: 350m
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
    depends_on: # rabbitMQ 서비스가 정상 동작할 떄 까지 기다림
      rabbitmq: 
        condition: service_healthy

	rabbitmq:
    image: rabbitmq:3.7.8-management
    mem_limit: 350m
    ports:
      - 5672:5672
      - 15672:15672
    healthcheck:
      test: ["CMD", "rabbitmqctl", "status"]
      interval: 10s
      timeout: 5s
      retries: 10
```

### 토픽당 2개의 파티션으로  RabbitMQ 사용

```yaml
  product-p1:
    build: microservices/product-service
    mem_limit: 350m
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_CLOUD_STREAM_BINDINGS_INPUT_CONSUMER_PARTITIONED=true
      - SPRING_CLOUD_STREAM_BINDINGS_INPUT_CONSUMER_INSTANCECOUNT=2 # 2개의 제품 인스턴스 할당
      - SPRING_CLOUD_STREAM_BINDINGS_INPUT_CONSUMER_INSTANCEINDEX=1
    depends_on:
      mongodb:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

```

### 토픽당 2개의 파티션으로 카프카 사용

```yaml
kafka:
  image: wurstmeister/kafka:2.12-2.1.0
  mem_limit: 350m
  ports:
    - "9092:9092"
  environment:
    - KAFKA_ADVERTISED_HOST_NAME=kafka
    - KAFKA_ADVERTISED_PORT=9092
    - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
  depends_on:
    - zookeeper

zookeeper:
  image: wurstmeister/zookeeper:3.4.6
  mem_limit: 350m
  ports:
    - "2181:2181"
  environment:
    - KAFKA_ADVERTISED_HOST_NAME=zookeeper
```

## 리액티브 마이크로서비스 환경의 자동 테스트

```bash
waitForService curl http://$HOST:$PORT/actuator/health # 환경 동작 여부 확인
```

```bash
function testCompositeCreated() {

    # Expect that the Product Composite for productId $PROD_ID_REVS_RECS has been created with three recommendations and three reviews
    if ! assertCurl 200 "curl http://$HOST:$PORT/product-composite/$PROD_ID_REVS_RECS -s"
    then
        echo -n "FAIL"
        return 1
    fi

    set +e
    assertEqual "$PROD_ID_REVS_RECS" $(echo $RESPONSE | jq .productId)
    if [ "$?" -eq "1" ] ; then return 1; fi

    assertEqual 3 $(echo $RESPONSE | jq ".recommendations | length")
    if [ "$?" -eq "1" ] ; then return 1; fi

    assertEqual 3 $(echo $RESPONSE | jq ".reviews | length")
    if [ "$?" -eq "1" ] ; then return 1; fi

    set -e
}

function waitForMessageProcessing() { # 비동기 생성 서비스가 전체 테스트 데이터를 생성할 때까지 대기
    echo "Wait for messages to be processed... "

    # Give background processing some time to complete...
    sleep 1

    n=0
    until testCompositeCreated
    do
        n=$((n + 1))
        if [[ $n == 40 ]]
        then
            echo " Give up"
            exit 1
        else
            sleep 6
            echo -n ", retry #$n "
        fi
    done
    echo "All messages are now processed!"
}
```

```bash
./test-em-all.bash start stop # 테스트 실행
```

## 요약

스프링 웹플럭스, 스프링 웹클라이언트, 스프링 데이터는 스프링 리액터를 사용해 리액티브 및 논블로킹 기능을 제공

스프링 데이터 스트림과 메시징 시스템을 이용해 코드 변경 없이 이벤트 기반 비동기 서비스 개발

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)
- https://docs.spring.io/spring-cloud-stream/docs/current/reference/html/