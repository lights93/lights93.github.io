---
title: "OpenAPI/스웨거를 사용한 API 문서화"
date: 2021-03-23
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 5. OpenAPI/스웨거를 사용한 API 문서화

## 스프링 폭스 소개

스프릭 폭스를 사용하면 API를 구현하느 소스 코드와 연동해 API를 문서화 가능

자바 코드와 API문서의 수명 주기가 다르면 시간이 지나면서 쉽게 어긋나기 때문에 중요함

RESTful API의 문서화 관점에서 보면 API를 구현하는 자바 클래스가 아닌, API를 설명하는 자바 인터페이스에 API 문서를 추가하는 게 낫다

스프링 폭스는 개발자가 설정한 구성 정보나 스프링 웹플럭스, 스웨거 애노테이션에서 얻은 구성 정보를 바탕으로 API 문서 생성

## 소스 코드 변경

다음 두 묘둘의 소스 코드를 변경해야 함

- product-compositve-service: 스프링 폭스 구성을 설정하고 API에 대한 일반 정보를 기술
- api: RESTful 서비스를 설명하는 자바 인터페이스에 스웨거 애노테이션 추가

#### 그래들 빌드 파일에 의존성 추가

- springfox-swaager2: 스웨거2 기반의 문서를 생성
- springox-spring-webflux: 스프링 웹플럭스 기반의 RESTful 오퍼레이션을 지원
- springflox-swagger-ui: 마이크로서비스에 스웨거 뷰어를 내장

product-composite-service 모듈의 그래들 빌드 파일에 의존성 추가

```groovy
	implementation('io.springfox:springfox-boot-starter:3.0.0')
```

api 프로젝트에는 하나만 추가

```groovy
	implementation('io.springfox:springfox-swagger2:3.0.0')
```

#### ProductCompositeServiceApplication에 구성과 API 정보 추가

스프링 폭스가 스프링 웹플럭스로 구현한 RESTful 서비스의 스웨거 V2 문서를 생성할 수 있도록 @EnableSwagger2WebFlux 애노테이션을 추가

스프링 폭스의 Docket 빈을 반환하는 스프링 빈을 정의.

Docket 빈은 스프링 폭스 구성에 사용됨

```java
// 구성에 들어가는 API 정보 초기화
@Value("${api.common.version}")
String apiVersion;
@Value("${api.common.title}")
String apiTitle;
@Value("${api.common.description}")
String apiDescription;
@Value("${api.common.termsOfServiceUrl}")
String apiTermsOfServiceUrl;
@Value("${api.common.license}")
String apiLicense;
@Value("${api.common.licenseUrl}")
String apiLicenseUrl;
@Value("${api.common.contact.name}")
String apiContactName;
@Value("${api.common.contact.url}")
String apiContactUrl;
@Value("${api.common.contact.email}")
String apiContactEmail;

@Bean
public Docket apiDocumentation() {

  return new Docket(SWAGGER_2) // 스웨거 V2 문서를 생성하고자 Docket 빈 초기화
    .select()
    .apis(basePackage("se.magnus.microservices.composite.product")) // 스프링 폭스가 API 정보를 찾을 위치 지정
    .paths(PathSelectors.any()) // 스프링 폭스가 API 정보를 찾을 위치 지정
    .build()
    .globalResponseMessage(GET, emptyList()) // 기본 HTTP 응답 코드를 추가하지 않게 함
    .apiInfo(new ApiInfo(
      apiTitle,
      apiDescription,
      apiVersion,
      apiTermsOfServiceUrl,
      new Contact(apiContactName, apiContactUrl, apiContactEmail),
      apiLicense,
      apiLicenseUrl,
      emptyList()
    ));
}
```

#### ProductCompositeService에 API 정보 추가

각 Restful 오퍼레이션의 해당 자바 메서드에 @ApiResponse 애노테이션 및 @ApiOperation 애노테이션을 추가해 오퍼레이션과 오류 응답을 설명



스프링 폭스는 @GetMapping 스프링 애노테이션을 검사해 오퍼레이션의 입력 매개 변수와 응답 유형을 파악

```java
@Api(description = "REST API for composite product information.")
public interface ProductCompositeService {

	/**
	 * Sample usage: curl $HOST:$PORT/product-composite/1
	 *
	 * @param productId
	 * @return the composite product info, if found, else null
	 */
	@ApiOperation(
		value = "${api.product-composite.get-composite-product.description}",
		notes = "${api.product-composite.get-composite-product.notes}")
	@ApiResponses(value = {
		@ApiResponse(code = 400, message = "Bad Request, invalid format of the request. See response message for more information."),
		@ApiResponse(code = 404, message = "Not found, the specified id does not exist."),
		@ApiResponse(code = 422, message = "Unprocessable entity, input parameters caused the processing to fails. See response message for more information.")
	}) // 여기서는 속성 자리 표시자를 사용할 수 없어 직접 넣음
	@GetMapping(
		value = "/product-composite/{productId}",
		produces = "application/json")
	ProductAggregate getProduct(@PathVariable int productId);
}
```

#### 속성 파일에 API 설명 추가

```yaml
api:

  common:
    version: 1.0.0
    title: Sample API
    description: Description of the API...
    termsOfServiceUrl: MINE TERMS OF SERVICE URL
    license: License
    licenseUrl: MY LICENSE URL
    
    
  product-composite:

    get-composite-product:
      description: Returns a composite view of the specified product id
```

## 마이크로서비스 환경 구축 및 시작

```bash
./gradlew build && docker-compose build && docker-compose up -d
```

## 스웨거 문서 사용법

생략

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)
- swagger 마이그레이션(https://stackoverflow.com/questions/62773219/suddenly-springfox-swagger-3-0-is-not-working-with-spring-webflux)
- globalResponses deprecated 관련(https://github.com/springfox/springfox/issues/3343)
- API description deprecated 관련(https://stackoverflow.com/questions/38074936/api-annotations-description-is-deprecated)