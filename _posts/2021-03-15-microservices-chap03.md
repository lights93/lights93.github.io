---
layout: post
title: "공조 마이크로서비스 집합 생성"
date: 2021-03-15
excerpt: "마이크로서비스"
tags: [microservice]
comments: true
---

# 3. 공조 마이크로서비스 집합 생성

## 마이크로서비스 환경 소개

#### 마이크로서비스가 처리하는 정보

**Product 서비스**

제품 정보 관리

- Product ID
- name
- weight

**Review 서비스**

리뷰 정보 관리

- Product ID
- Review ID
- author
- subject
- content

**Recommendation 서비스**

추천 정보 관리

- Product ID
- Recommendation ID
- Author
- Rate
- Content

**Product Composite 서비스**

핵심 서비스에서 수집한 제품 관련 정보 제공

- 제품 정보
- 특정 제품의 리뷰 목록
- 특정 제품의 추천 목록

**인프라 관련 정보**

인프라(ex. 도커, 쿠버네티스)에서 컨테이너로 실행되므로 어떤 컨테이너가 사용자의 요청에 응답하는지 추적해야 한다

#### 임시로 검색 서비스 대체

현 단계에선 서비스 검색 메커니즘이 없으므로 각 마이크로서비스의 포트 번호를 직접 지정

- Product composite 서비스: 7000
- Product 서비스: 7001
- Review 서비스: 7002
- Recommendation 서비스: 7003

## 골격 마이크로서비스 생성

- 빌드 도구로 gradle 사용
- 자바 8과 호환되는 코드 생성
- fat JAR로 프로젝트 패키징
- 액추에이터와 웹플럭스 모듈 의존성 추가

```groovy
buildscript {
	ext {
		springBootVersion = '2.3.2.RELEASE'
	}
	repositories {
		mavenCentral()
	}
	dependencies {
		classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
	}
}

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot' // spring boot 의존성 관리
apply plugin: 'io.spring.dependency-management' // 라이브러리를 추가할 때 스프링 부트에 맞는 라이브러리 버전을 자동적으로 추가해주기 위한 플러그인


group = 'se.magnus.microservices.core.product'
version = '1.0.0-SNAPSHOT'
sourceCompatibility = 1.8

repositories {
	mavenCentral()
}


dependencies {
	implementation project(':api')
	implementation project(':util')
	implementation('org.springframework.boot:spring-boot-starter-actuator')
	implementation('org.springframework.boot:spring-boot-starter-webflux')
	testImplementation('org.springframework.boot:spring-boot-starter-test')
	testImplementation('io.projectreactor:reactor-test')
}

```

#### 그래들에 멀티 프로젝트 빌드 설정

settings.gradle

```groovy
include ':microservices:product-service'
include ':microservices:review-service'
include ':microservices:recommendation-service'
include ':microservices:product-composite-service'

```

## RESTful API 추가

#### api 프로젝트와 util 프로젝트 추가

**api 프로젝트**

라이브러리 프로젝트의 구조는 main 애플리케이션 클래스가 없고 build.gradle 파일이 약간 다르다는 점만 뺴면 애플리케이션 프로젝트와 동일

```groovy
plugins {
	id "io.spring.dependency-management" version "1.0.9.RELEASE"
}


dependencyManagement {
    imports { mavenBom("org.springframework.boot:spring-boot-dependencies:${springBootVersion}") } // 라이브러리를 추가할 때 스프링 부트에 맞는 라이브러리 버전을 자동적으로 추가해주기 위한 것
}
```

**util 프로젝트**

- 예외 클래스
- 유틸리티 클래스(예외 처리, 마이크로서비스 검색)

#### API 구현

1. build.gradle에 의존성 추가

```groovy
dependencies {
	implementation project(':api')
	implementation project(':util')
```

2. api 및 util 프로젝트의 스프링 빈을 감지하도록 기본 애플리케이션 클래스에 @ComponentScan 추가

```java
@SpringBootApplication
@ComponentScan("se.magnus")
public class ProductServiceApplication {
```

3. 인터페이스 구현 생성자 주입

```java
@RestController
public class ProductServiceImpl implements ProductService {

	private static final Logger LOG = LoggerFactory.getLogger(ProductServiceImpl.class);

	private final ServiceUtil serviceUtil;

	@Autowired
	public ProductServiceImpl(ServiceUtil serviceUtil) {
		this.serviceUtil = serviceUtil;
	}

	@Override
	public Product getProduct(int productId) {
		LOG.debug("/product return the found product for productId={}", productId);

		if (productId < 1)
			throw new InvalidInputException("Invalid productId: " + productId);

		if (productId == 13)
			throw new NotFoundException("No product found for productId: " + productId);

		return new Product(productId, "name-" + productId, 123, serviceUtil.getServiceAddress());
	}
}
```

4. 포트번호 및 로깅 설정

```yaml
server.port: 7001
server.error.include-message: always

logging:
  level:
    root: INFO
    se.magnus: DEBUG
```

## 복합 마이크로서비스 추가

핵심 서비스로의 발신 요청을 처리하는 통합 컴포넌트와 복합 서비스 자체 구현의 두 부분으로 나뉨

책임을 이렇게 나누는 것은 단위 테스트와 통합 테스트를 간편하게 자동화하고 통합 컴포넌트를 모의 객체(mock)로 대체해 서비스 구현을 개별적으로 테스트하기 위함

## 예외 처리 추가

## API 수동 테스트

curl로 테스트

## 자동화된 마이크로서비스 테스트

구현을 마무리하려면 테스트 자동화 필요

통합 테스트에서 내장협 웹 서버로 API를 시작한 후 테스트 클라이언트를 사용해 HTTP 요청을 보내고 결과를 검증

```java
package se.magnus.microservices.composite.product;

import static java.util.Collections.*;
import static org.mockito.Mockito.*;
import static org.springframework.boot.test.context.SpringBootTest.WebEnvironment.*;
import static org.springframework.http.HttpStatus.*;
import static org.springframework.http.MediaType.*;

import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.reactive.server.WebTestClient;

import se.magnus.api.core.product.Product;
import se.magnus.api.core.recommendation.Recommendation;
import se.magnus.api.core.review.Review;
import se.magnus.microservices.composite.product.services.ProductCompositeIntegration;
import se.magnus.util.exceptions.InvalidInputException;
import se.magnus.util.exceptions.NotFoundException;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = RANDOM_PORT)
public class ProductCompositeServiceApplicationTests {

	private static final int PRODUCT_ID_OK = 1;
	private static final int PRODUCT_ID_NOT_FOUND = 2;
	private static final int PRODUCT_ID_INVALID = 3;

	@Autowired
	private WebTestClient client;

	@MockBean
	private ProductCompositeIntegration compositeIntegration;

	@Before
	public void setUp() {

		when(compositeIntegration.getProduct(PRODUCT_ID_OK)).
			thenReturn(new Product(PRODUCT_ID_OK, "name", 1, "mock-address"));
		when(compositeIntegration.getRecommendations(PRODUCT_ID_OK)).
			thenReturn(singletonList(new Recommendation(PRODUCT_ID_OK, 1, "author", 1, "content", "mock address")));
		when(compositeIntegration.getReviews(PRODUCT_ID_OK)).
			thenReturn(singletonList(new Review(PRODUCT_ID_OK, 1, "author", "subject", "content", "mock address")));

		when(compositeIntegration.getProduct(PRODUCT_ID_NOT_FOUND)).thenThrow(
			new NotFoundException("NOT FOUND: " + PRODUCT_ID_NOT_FOUND));

		when(compositeIntegration.getProduct(PRODUCT_ID_INVALID)).thenThrow(
			new InvalidInputException("INVALID: " + PRODUCT_ID_INVALID));
	}

	@Test
	public void contextLoads() {
	}

	@Test
	public void getProductById() {

		client.get()
			.uri("/product-composite/" + PRODUCT_ID_OK)
			.accept(APPLICATION_JSON)
			.exchange()
			.expectStatus().isOk()
			.expectHeader().contentType(APPLICATION_JSON)
			.expectBody()
			.jsonPath("$.productId").isEqualTo(PRODUCT_ID_OK)
			.jsonPath("$.recommendations.length()").isEqualTo(1)
			.jsonPath("$.reviews.length()").isEqualTo(1);
	}

	@Test
	public void getProductNotFound() {

		client.get()
			.uri("/product-composite/" + PRODUCT_ID_NOT_FOUND)
			.accept(APPLICATION_JSON)
			.exchange()
			.expectStatus().isNotFound()
			.expectHeader().contentType(APPLICATION_JSON)
			.expectBody()
			.jsonPath("$.path").isEqualTo("/product-composite/" + PRODUCT_ID_NOT_FOUND)
			.jsonPath("$.message").isEqualTo("NOT FOUND: " + PRODUCT_ID_NOT_FOUND);
	}

	@Test
	public void getProductInvalidInput() {

		client.get()
			.uri("/product-composite/" + PRODUCT_ID_INVALID)
			.accept(APPLICATION_JSON)
			.exchange()
			.expectStatus().isEqualTo(UNPROCESSABLE_ENTITY)
			.expectHeader().contentType(APPLICATION_JSON)
			.expectBody()
			.jsonPath("$.path").isEqualTo("/product-composite/" + PRODUCT_ID_INVALID)
			.jsonPath("$.message").isEqualTo("INVALID: " + PRODUCT_ID_INVALID);
	}
}

```

## 반자동화된 마이크로서비스 환경 테스트

자동으로 모든 마이크로서비스를 테스트하고 그 결과를 확인할 수 있어야 함

이런 이유료 curl로 RESTful API를 호출하고 jq를 이용해 상태 코드외 JSON 응답을 확인하는 bash 스크립트 생성

```bash
#!/usr/bin/env bash
#
# Sample usage:
#
#   HOST=localhost PORT=7000 ./test-em-all.bash
#
: ${HOST=localhost}
: ${PORT=7000}

function assertCurl() {

  local expectedHttpCode=$1
  local curlCmd="$2 -w \"%{http_code}\""
  local result=$(eval $curlCmd)
  local httpCode="${result:(-3)}"
  RESPONSE='' && (( ${#result} > 3 )) && RESPONSE="${result%???}"

  if [ "$httpCode" = "$expectedHttpCode" ]
  then
    if [ "$httpCode" = "200" ]
    then
      echo "Test OK (HTTP Code: $httpCode)"
    else
      echo "Test OK (HTTP Code: $httpCode, $RESPONSE)"
    fi
  else
      echo  "Test FAILED, EXPECTED HTTP Code: $expectedHttpCode, GOT: $httpCode, WILL ABORT!"
      echo  "- Failing command: $curlCmd"
      echo  "- Response Body: $RESPONSE"
      exit 1
  fi
}

function assertEqual() {

  local expected=$1
  local actual=$2

  if [ "$actual" = "$expected" ]
  then
    echo "Test OK (actual value: $actual)"
  else
    echo "Test FAILED, EXPECTED VALUE: $expected, ACTUAL VALUE: $actual, WILL ABORT"
    exit 1
  fi
}
set -e

echo "HOST=${HOST}"
echo "PORT=${PORT}"


# Verify that a normal request works, expect three recommendations and three reviews
assertCurl 200 "curl http://$HOST:$PORT/product-composite/1 -s"
assertEqual 1 $(echo $RESPONSE | jq .productId)
assertEqual 3 $(echo $RESPONSE | jq ".recommendations | length")
assertEqual 3 $(echo $RESPONSE | jq ".reviews | length")

# Verify that a 404 (Not Found) error is returned for a non existing productId (13)
assertCurl 404 "curl http://$HOST:$PORT/product-composite/13 -s"

# Verify that no recommendations are returned for productId 113
assertCurl 200 "curl http://$HOST:$PORT/product-composite/113 -s"
assertEqual 113 $(echo $RESPONSE | jq .productId)
assertEqual 0 $(echo $RESPONSE | jq ".recommendations | length")
assertEqual 3 $(echo $RESPONSE | jq ".reviews | length")

# Verify that no reviews are returned for productId 213
assertCurl 200 "curl http://$HOST:$PORT/product-composite/213 -s"
assertEqual 213 $(echo $RESPONSE | jq .productId)
assertEqual 3 $(echo $RESPONSE | jq ".recommendations | length")
assertEqual 0 $(echo $RESPONSE | jq ".reviews | length")

# Verify that a 422 (Unprocessable Entity) error is returned for a productId that is out of range (-1)
assertCurl 422 "curl http://$HOST:$PORT/product-composite/-1 -s"
assertEqual "\"Invalid productId: -1\"" "$(echo $RESPONSE | jq .message)"

# Verify that a 400 (Bad Request) error error is returned for a productId that is not a number, i.e. invalid format
assertCurl 400 "curl http://$HOST:$PORT/product-composite/invalidProductId -s"
assertEqual "\"Type mismatch.\"" "$(echo $RESPONSE | jq .message)"

```

모두 테스트

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)

- https://dingue.tistory.com/17
- https://jahyun-dev.github.io/posts/gradle-1/

