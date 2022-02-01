---
title: "Resilience4j를 사용한 탄력성 개선"
date: 2021-06-19
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 13. Resilience4j를 사용한 탄력성 개선

## Resilience4j의 서킷 브레이커와 재시도 메커니즘 소개

재시도와 서킷 브레이커 메커니즘은 마이크로서비스와 같이 동기 방식으로 연결되는 소프트웨어 컴포넌트에 특히 유용함

### 서킷 브레이커 소개

- 서킷 브레이커는 다량의 오류를 감지하며 서킷을 열어 새 호출을 받지 않는다
- 서킷 브레이커는 서킷이 열려 있을 때 빠른 실패 로직을 수행, 즉 이어지는 호출에서 시간 초과 등으로 말미암은 새로운 오류가 발생하지 않게 하며, **폴백 메서드(fallback 메서드)**로 호출을 리디렉션. 폴백 메서드에 다양한 비즈니스 로직을 적용하면 로컬 캐시의 데이터를 반환하거나 즉각젓인 오류 메시지를 반환하는 등의 최적화된 응답을 생성
- 시간이 지나면 서킷 브레이커는 반열림 상태로 전환돼 새로운 호출을 허용하며, 이를 통해 문제를 일으킨 원인이 사라졌는지 확인. 서킷 브레이커는 새로운 오류를 감지하면 서킷을 다시 열고 빠른 실패 로직을 다시 수행하며, 오류가 사라졌으면 서킷을 닫고 정상 작동 상태로 돌아감

Resilience4j는 런타임에 다양항 방법으로 서킷 브레이커의 정보를 제공

- 서킷 브레이커의 현재 상태는 마이크로서비스 액추에이터의 상태 점검 엔드포인트를 사용해 모니터링 가능
- 서킷 브레이커는 상태 전이 등의 이벤트를 액추에이어 엔드포인텡 계시
- 서킷 브레이커는 스프링 부트의 메트릭 시스템과 통합돼 있으며, 이를 이용해 프로메테우스와 같은 모니터링 도구에 메트릭을 게시 가능

구성 매개 변수

- ringBufferSizeInClosedState: 닫힌 상태에서의 호출 수로, 서킷을 열어야 할지 결정할 때 사용
- failureRateThreshold: 실패한 호출에 대한 임계값으로 이 값을 초과하면 서킷이 열림
- waitInterval: 반열림 상태로 전환하기 전에 서킷을 열린 상태로 유지하는 시간
- ringBufferSizeInHalfOpenState: 반열림 상태에서의 호출 수로, 서킷을 다시 열거나 닫힘 상태로 돌아갈지를 결정할 때 사용
- automaticTransitionFromOpenToHalfOpenEnabled: 대기 시간이 지난 후로 서킷을 반열림 상태로 자동 전환할지, 첫 번째 호출이 들어오길 기다렸다가 반열림 상태로 전환할지를 결정
- ignoreExceptions: 오류로 간주하지 않을 예외를 지정(보통 InvalidInputException이나 NotFoundException은 예외처리)

### 재시도 메커니즘 소개

**재시도(retry)** 메커니증은 일시적인 네트워크 결함과 같이 무작위로 드물게 발생하는 오류에 매우 유용

설정된 대기 시간을 사이에 두고, 실패한 요청을 여러번 다시 시도하는 것

재시도 메커니즘을 사용하기 위한 주요 요건 중 하나는 **멱등성**이 있어야 한다는 점  



Resilience4j는 서킷 브레이커와 같은 방식으로 재시도와 관련된 이벤트 및 메트릭 정보를 공개하지만 상태 정보는 전혀 재공하지 않으며, 재시도 이벤트에 관한 정보는 액추에이터 엔드포인트에서 얻을 수 있다.

- maxRetryAttempts: 첫 번째 호출을 포함한 총 재시도 횟수
- waitDuration: 재시돌르 다시 수행하기 전의 대기 시간
- retryException: 재시도를 트리거하는 예외 목록

## 소스 코드에 서킷 브레이커 및 재시도 메커니즘 추가

- Resilience4j에 대한 스타터 의존성을 빌드 파일에 추가
- 서킷 브레이커 및 재시도 메커니즘을 적용할 소스 코드에 애노테이션을 추가
- 서킷 브레이커 및 재시도 메커니즘의 동작을 제어하는 구성 추가

### 프로그래밍 방식으로 지연 및 무작위 오류 추가

선택적인 쿼리 매개변수 추가

- delay: 일부로 지연시키기 위한 변수
- faultPercentage: 지정한 백분유레 따라 무작위로 예외 발생

#### API 정의 변경

```java
@GetMapping(
    value    = "/product-composite/{productId}",
    produces = "application/json")
Mono<ProductAggregate> getCompositeProduct(
    @PathVariable int productId,
    @RequestParam(value = "delay", required = false, defaultValue = "0") int delay,
    @RequestParam(value = "faultPercent", required = false, defaultValue = "0") int faultPercent
);
```

```java
@GetMapping(
    value    = "/product/{productId}",
    produces = "application/json")
Mono<Product> getProduct(
     @PathVariable int productId,
     @RequestParam(value = "delay", required = false, defaultValue = "0") int delay,
     @RequestParam(value = "faultPercent", required = false, defaultValue = "0") int faultPercent
);
```

#### Product Composite 마이크로서비스의 코드 변경

```java
@Override
public Mono<ProductAggregate> getCompositeProduct(int productId, int delay, int faultPercent) {

    return Mono.zip(
      	...
      	integration.getProduct(productId, delay, faultPercent)
      	...
}
```

```java
public Mono<Product> getProduct(int productId, int delay, int faultPercent) {

    URI url = UriComponentsBuilder.fromUriString(productServiceUrl + "/product/{productId}?delay={delay}&faultPercent={faultPercent}").build(productId, delay, faultPercent);
    LOG.debug("Will call the getProduct API on URL: {}", url);

    return getWebClient().get().uri(url)
        ...
```

### Product 마이크로서비스의 소스 코드 변경

```java
public Mono<Product> getProduct(int productId, int delay, int faultPercent) {

    if (productId < 1) throw new InvalidInputException("Invalid productId: " + productId);

    if (delay > 0) simulateDelay(delay);

    if (faultPercent > 0) throwErrorIfBadLuck(faultPercent);

    return repository.findByProductId(productId)
        .switchIfEmpty(error(new NotFoundException("No product found for productId: " + productId)))
        .log()
        .map(e -> mapper.entityToApi(e))
        .map(e -> {e.setServiceAddress(serviceUtil.getServiceAddress()); return e;});
}

private void simulateDelay(int delay) {
  LOG.debug("Sleeping for {} seconds...", delay);
  try {Thread.sleep(delay * 1000);} catch (InterruptedException e) {}
  LOG.debug("Moving on...");
}

private void throwErrorIfBadLuck(int faultPercent) {
  int randomThreshold = getRandomNumber(1, 100);
  if (faultPercent < randomThreshold) {
    LOG.debug("We got lucky, no error occurred, {} < {}", faultPercent, randomThreshold);
  } else {
    LOG.debug("Bad luck, an error occurred, {} >= {}", faultPercent, randomThreshold);
    throw new RuntimeException("Something went wrong...");
  }
}

private final Random randomNumberGenerator = new Random();
private int getRandomNumber(int min, int max) {

  if (max < min) {
    throw new RuntimeException("Max must be greater than min");
  }

  return randomNumberGenerator.nextInt((max - min) + 1) + min;
}
```

### 서킷 브레이커 추가

#### 빌드 파일에 의존성 추가

```groovy
ext {
   resilience4jVersion = "1.3.1"
}

dependencies {
   ...
   implementation("io.github.resilience4j:resilience4j-spring-boot2:${resilience4jVersion}")
   implementation("io.github.resilience4j:resilience4j-reactor:${resilience4jVersion}")
}
```

### 서킷 브레이커 및 시간 초과 로직 추가

```java
@CircuitBreaker(name = "product") // 서킷 브레이커 적용
public Mono<Product> getProduct(int productId, int delay, int faultPercent) {

    URI url = UriComponentsBuilder.fromUriString(productServiceUrl + "/product/{productId}?delay={delay}&faultPercent={faultPercent}").build(productId, delay, faultPercent);
    LOG.debug("Will call the getProduct API on URL: {}", url);

    return getWebClient().get().uri(url)
        .retrieve().bodyToMono(Product.class).log()
        .onErrorMap(WebClientResponseException.class, ex -> handleException(ex))
        .timeout(Duration.ofSeconds(productServiceTimeoutSec)); // 시간 초과시 예외 발생
}
```

#### 폴백 로직 추가

```java
public Mono<ProductAggregate> getCompositeProduct(int productId, int delay, int faultPercent) {

    return Mono.zip(
      			...
            integration.getProduct(productId, delay, faultPercent)
                .onErrorReturn(CallNotPermittedException.class, getProductFallbackValue(productId)), // 서킷이 열려있을 때 발생하는 예외를 잡아서 예외 처리
      			...
}

private Product getProductFallbackValue(int productId) {

  LOG.warn("Creating a fallback product for productId = {}", productId);

  if (productId == 13) {
    String errMsg = "Product Id: " + productId + " not found in fallback cache!";
    LOG.warn(errMsg);
    throw new NotFoundException(errMsg);
  }

  return new Product(productId, "Fallback product" + productId, productId, serviceUtil.getServiceAddress());
}
```

#### 구성 추가

```yaml
app.product-service.timeoutSec: 2 # 시간초과

resilience4j.circuitbreaker:
  backends:
    product:
      registerHealthIndicator: true # 서킷 브레이커 정보를 상태 점거(health) 엔드포인트에 추가 여부
      ringBufferSizeInClosedState: 5
      failureRateThreshold: 50
      waitDurationInOpenState: 10000
      ringBufferSizeInHalfOpenState: 3
      automaticTransitionFromOpenToHalfOpenEnabled: true
      ignoreExceptions:
        - se.magnus.util.exceptions.InvalidInputException
        - se.magnus.util.exceptions.NotFoundException
```

### 재시도 메커니즘 추가

#### 재시도 애노테이션 추가

```java
@Retry(name = "product") // 재시도 메커니즘 적용
@CircuitBreaker(name = "product")
@Override
public Mono<Product> getProduct(int productId, int delay, int faultPercent) { ...
```

#### 재시도 예외 처리

```java
@Override
public Mono<ProductAggregate> getCompositeProduct(int productId, int delay, int faultPercent) {

    return Mono.zip(
            values -> createProductAggregate((SecurityContext) values[0], (Product) values[1], (List<Recommendation>) values[2], (List<Review>) values[3], serviceUtil.getServiceAddress()),
            ReactiveSecurityContextHolder.getContext().defaultIfEmpty(nullSC),
            integration.getProduct(productId, delay, faultPercent)
                .onErrorReturn(CallNotPermittedException.class, getProductFallbackValue(productId)),
            integration.getRecommendations(productId).collectList(),
            integration.getReviews(productId).collectList())
        .doOnError(ex -> LOG.warn("getCompositeProduct failed: {}", ex.toString()))
        .log();
}
```

#### 구성 추가

```yaml
resilience4j.retry:
  backends:
    product:
      maxRetryAttempts: 3
      waitDuration: 1000
      retryExceptions:
      - org.springframework.web.reactive.function.client.WebClientResponseException$InternalServerError
```

### 자동 테스트 추가

```bash
function testCircuitBreaker() {

    echo "Start Circuit Breaker tests!"

    EXEC="docker run --rm -it --network=my-network alpine"

    # 서킷 브레이커가 닫혀있는지 확인
    assertEqual "CLOSED" "$($EXEC wget product-composite:8080/actuator/health -qO - | jq -r .components.circuitBreakers.details.product.details.state)"

    # 3초짜리 delay를 보내 실패하게 하여 서킷 브레이커 오픈 상태로 변경
    for ((n=0; n<3; n++))
    do
        assertCurl 500 "curl -k https://$HOST:$PORT/product-composite/$PROD_ID_REVS_RECS?delay=3 $AUTH -s"
        message=$(echo $RESPONSE | jq -r .message)
        assertEqual "Did not observe any item or terminal signal within 2000ms" "${message:0:57}"
    done

    # fallback 값이 오는지 확인
    assertCurl 200 "curl -k https://$HOST:$PORT/product-composite/$PROD_ID_REVS_RECS?delay=3 $AUTH -s"
    assertEqual "Fallback product2" "$(echo "$RESPONSE" | jq -r .name)"

    # fallback 값이 오는지 확인


    assertCurl 200 "curl -k https://$HOST:$PORT/product-composite/$PROD_ID_REVS_RECS $AUTH -s"
    assertEqual "Fallback product2" "$(echo "$RESPONSE" | jq -r .name)"

    # ID가 13인 제품 조회시 404 확인
    assertCurl 404 "curl -k https://$HOST:$PORT/product-composite/$PROD_ID_NOT_FOUND $AUTH -s"
    assertEqual "Product Id: $PROD_ID_NOT_FOUND not found in fallback cache!" "$(echo $RESPONSE | jq -r .message)"

    # 10초 후 반열림 상태로 바뀌므로 10초 대기
    echo "Will sleep for 10 sec waiting for the CB to go Half Open..."
    sleep 10

    # 반 열림 상태로 바뀌었는지 확인
    assertEqual "HALF_OPEN" "$($EXEC wget product-composite:8080/actuator/health -qO - | jq -r .components.circuitBreakers.details.product.details.state)"

    # 정상 요청 3번 보냄
    for ((n=0; n<3; n++))
    do
        assertCurl 200 "curl -k https://$HOST:$PORT/product-composite/$PROD_ID_REVS_RECS $AUTH -s"
        assertEqual "product name C" "$(echo "$RESPONSE" | jq -r .name)"
    done

    # 그대로 닫힌 상태인제 확인
    assertEqual "CLOSED" "$($EXEC wget product-composite:8080/actuator/health -qO - | jq -r .components.circuitBreakers.details.product.details.state)"

    # 상태 전이가 제대로 됐는지 확인
    assertEqual "CLOSED_TO_OPEN"      "$($EXEC wget product-composite:8080/actuator/circuitbreakerevents/product/STATE_TRANSITION -qO - | jq -r .circuitBreakerEvents[-3].stateTransition)"
    assertEqual "OPEN_TO_HALF_OPEN"   "$($EXEC wget product-composite:8080/actuator/circuitbreakerevents/product/STATE_TRANSITION -qO - | jq -r .circuitBreakerEvents[-2].stateTransition)"
    assertEqual "HALF_OPEN_TO_CLOSED" "$($EXEC wget product-composite:8080/actuator/circuitbreakerevents/product/STATE_TRANSITION -qO - | jq -r .circuitBreakerEvents[-1].stateTransition)"
}
```

## 서킷 브레이커 및 재시도 메커니즘 테스트

생략

## 요약

서킷 브레이커는 서킷이 열려 있을 때 빠른 실패, 폴백 메서드를 작동시킴

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)