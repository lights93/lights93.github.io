---
title: "API 접근 보안"
date: 2021-06-08
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 11. API 접근 보안

## OAuth 2.0 및 OpenID Connect 소개

**인증**은 사용자 이름과 암호 같은, 사용자가 제공한 자격 증명을 확인해 사용자를 식별자를 하는 것

**권한 부여**는 인증된 사용자, 즉 식별된 사용자에게 여러 API에 대한 접근 권한을 부여하는 것   



OAuth2.0은 권한 부여를 위한 공개 표준

OpenID Connection는 클라이언트 애플리케이션이 권한 부여 서버에서 받은 자격 증명을 기반으로 사용자의 신원을 확인할 수 있도록 OAuth2.0에 추가된 기능

### OAuth2.0 소개

OAuth2.0은 권한 부여를 위해 광범위하게 사용되는 공개 표준으로, 사용자에게 권한을 위임받은 서드파티 클라이언트 애플리케이션이 사용자를 대신해 보안 리소스에 접근할 수 있게 함

- **자원 소유자(resource owner)**: 최종 사용자
- **클라이언트(client)**: 최종 사용자의 권한을 위임받아 보안 API를 호출하려는 서드파티 애플리케이션(ex. 웹 앱, 네이티브 모바일 앱)
- **자원 서버(resource server)**: 보호 대상 자원에 대한 API를 제공하는 서버
- **권한부여 서버(authorization server)**: 자원 소유자를 인증하고 자원 소유자의 승인을 받아서 클라이언트에게 토큰 발급. 사용자 정보 관리 및 사용자 인증은 보통 **ID 제공자(Idp, Identity Provider)**에게 위임

   

권한 부여 서버에 등록된 클라이언트는 **클라이언트 ID**와 **클라이언트 시크릿**을 발급받는다

클라이언트는 암호와 마찬가지로 클라이언트 시크릿을 보호해야 함

클라이언트는 **리다이렉트 URI**를 등록해야 하며, 권한 부여 서버는 사용자 인증을 거쳐 발급한 **인증 코드**와 **토큰**을 리다이렉트 URI로 전달한다.   



OAuth 2.0 사양에서는 접근 토큰 발급을 위한 권한 승인 흐름을 네 가지로 정의

- 권한 코드 승인 흐름(authorization code grant flow): 사용자는 웹 브라우저로 권한 부여 서버와 상호 작용해 클라이언트 애플리케이션에게 권한을 위임

  1. 클라이언트 애플리케이션은 웹 브라우저를 통해 사용자를 권한 부여 서버로 보내고 권한 승인 흐름을 시작
  2. 권한 부여 서버는 사용자를 인증하고 사용자의 동의를 요청
  3. 권한 부여 서버는 사용자를 인증 코드와 함께 클라이언트 애플리케이션으로 리다이렉트
  4. 클라이언트 애플리케이션은 인증 코드와 접근 토큰을 교환하고자 서버 측 코드를 사용해 권한 부여 서버를 다시 호출, 클라이언트 애플리케이션은 권한 부여 서버로 인증 코드를 보낼 때 클라이언트 ID와 클라이언트 시크릿도 함께 보내야 함
  5. 권한 부여 서버는 접근 토큰을 발급해 클라이언트 애플리케이션으로 보내며, 선택적으로 재발급 토큰을 발급 및 반환할 수 있다.
  6. 클라이언트는 접근 토큰을 사용해 자원 서버가 공개하는 보안 API에 요청을 보냄
  7. 자원 서버는 접근 토큰을 검사하고 검사가 성공하면 요청을 처리, 접근 토큰이 유효하다면 6단계와 7단계 반복 가능, 접근 토큰이 만료되면 클라이언트는 재발급 토큰을 사용해 접근 토큰을 새로 발급 가능

- 묵시적 승인 흐름: 클라이언트 시크릿을 안전하게 보호할 수 없는 클라이언트 애플리케이션을 대상으로 함

  권한 부여 서버에서 인증 코드 대신 접근 토큰을 받으며, 안전성이 낮음

- 자원 소유자 암호 자격 증명 승인 흐름: 클라이언트 애플리케이션이 웹 브라우저와 상호 작용할 수 없는 경우에 사용

  사용자는 자신의 자격 증명을 클라이언트 애플리케이션과 공유해야 하며, 클라이언트 애플리케이션은 이 자격 증명을 사용해 접근 토큰을 얻음

- 클라이언트 자격 증명 승인 흐름: 클라이언트 애플리케이션이 특정 사용자와 관련 없는 API를 호출해야 하는 경우

  클라이언트 ID와 클라이언트 시크릿을 사용해 접근 토큰을 얻음

### OpenID Connect 소개

OIDC는 클라이언트 애플리케이션이 사용자의 신원을 확인하게 하려고 OAuth2.0에 추가된 기능

OIDC를 사용하면 승인 흐름이 완료된 후 클라이언트 애플리케이션이 권한 부여 서버에서 받아오는 토큰인 ID 토큰이 추가됨



ID 토큰은 JWT로 인코딩되며, 사용자 ID, 이메일 주소와 같은 다수의 클레임을 포함

ID 토큰은 JSON 웹 서명으로 디지털 서명됨



클라이언트 애플리케이션은 권한 부여 서버의 공개 키로 디지털 서명을 검사함으로써 ID 토큰의 정보를 신뢰 가능



ID 토큰과 같은 방식으로 접근 토큰을 인코딩하고 서명할 수도 있지만, 사양에 따른 필수사항은 아님

OIDC는 **디스커버리 엔드포인트**를 정의하고 있는데 이는 주요 엔드포인트에 대한 정보를 제공하기 위한 표준

## 시스템 환경 보안

- HTTPS를 사용해 공개된 API에 대한 외부 요청과 응답을 암호화하고 도청을 방지
- OAuth2.0 및 OpenID Connect를 사용해 API에 접근하는 사용자 및 클라이언트 애플리케이션에 대한 인증 및 권한 부여를 수행
- HTTP 기본 인증을 사용해 검색 서비스에 대한 접근을 보호

1. 외부 통신엔 HTTPS를 사용하고 시스템 환경 안에선 일반 텍스트를 사용
2. 외부에선 에지 서버를 거쳐야 로컬 OAuth 2.0 권한 부여 서버에 접근할 수 있다.
3. 에지 서버와 product-composite 마이크로서비스는 서명된 JWT 토큰으로 접근 토큰을 검사
4. 에지 서버와 product-compositbe 마이크로서비스는 jwk-set 엔드포인트에서 가져온 권한 부여 서버의 공개 키로 JWT 기반 접근 토큰의 서명을 검사

## 시스템 환경에 권한 부여 서버 추가

스프링 시큐리티 OAuth에서 쓸 만한 권한 부여 서버 제공



이 권한 부여 서버는 JWT로 인코딩된 접근 토큰을 사용하며, OpenID Connect Discovery 표준의 일부인 **JSON 웹 키 집합(JWKS, JSON Web Key Set)** 엔드포인트 제공

이 키 집합에는 권한 부여 서버가 발급한 JWT 토큰을 자원 서버에서 검사할 때 사용하는 공개 키가 포함돼 있음



샘플 프로젝트 변경 사항

- 다른 마이크로서비스와 같은 방식으로 유케라 클라이언트를 추가
- 상태 점검 엔드포인트에 접근할 수 있도록 스프링 부트 액추에이터 추가
- 권한 부여 서버를 도커 컨테이너로 실행할 수 있도록 도커파일을 추가
- 그래들 빌드 파일 설정
- 승인 유형 추가, 스코프 이름 변경
- 권한 부여 서버에 등록된 사용자 정보 변경

권한 부여 서버를 시스템 환경에 통합하고자 적용한 변경 사항

- 공틍 빌드 파일(settings.gradle)에 권한 부여 서버 추가
- 3개의 도커 컴포즈 파일에 권한 부여 서버 추가
- 에지 서버에 health에 추가, /oauth/로 시작하는 URI 경로 추가

## HTTPS를 사용한 외부 통신 보호

- **인증서 생성**: 개발 목적의 자체 서명 인증서 생성
- **에지 서버 구성**: 인증서를 사용해 HTTPS 기반 외부 트래픽만 허용하도록 에지 서버를 구성

인증서와 HTTPS를 사용하도록 에지 서버를 구성하고자 게이트웨이 프로젝트의 application.yml 파일 수정

```yaml
server.port: 8443 # HTTPS와 통신하는 것을 나타내고자 8443으로 변경

server.ssl:
  key-store-type: PKCS12
  key-store: classpath:keystore/edge.p12 # 인증서 경로 명시
  key-store-password: password # 인증서 암호
  key-alias: localhost
```

클래스패스를 사용해 인증서를 제공하는 것은 개발 환경에서만 사용해야 함

#### 런타임에 자체 서명 인증서 교체

테스트나 상용 환경과 같은 런타임 환경에서는 공인된 인증기관(CAT에서 서명한 인증서를 사용해야 한다.

## 검색 서비스 접근 보안

### 유레카 서버 변경

1. build.gradle에 스프링 시쿠리티 의존성 추가

   ```groovy
   implementation 'org.springframework.boot:spring-boot-starter-security'
   ```

2. 보안 구성 추가

   - 사용자 정의

     ```java
     @Override
     public void configure(AuthenticationManagerBuilder auth) throws Exception {
         auth.inMemoryAuthentication()
         .passwordEncoder(NoOpPasswordEncoder.getInstance())
         .withUser(username).password(password)
         .authorities("USER");
     }
     ```

   - 구성 파일에서 가져온 사용자 이름과 암호를 생성자에 삽입

     ```java
     @Autowired
     public SecurityConfig(
         @Value("${app.eureka-username}") String username,
         @Value("${app.eureka-password}") String password
     ) {
         this.username = username;
         this.password = password;
     }
     ```

   - HTTP 기본 인증으로 모든 API 및 웹 페이지를 보호하고자 다음과 같이 정의

     ```java
     @Override
     protected void configure(HttpSecurity http) throws Exception {
         http
             // Disable CRCF to allow services to register themselves with Eureka
             .csrf()
                 .disable()
             .authorizeRequests()
               .anyRequest().authenticated()
               .and()
               .httpBasic();
     }
     ```

3. 사용자 자격 증명은 구성 파일(application.yml)에서 설정

   ```yaml
   app:
     eureka-username: u
     eureka-password: p
   ```

4. 테스트 클래스는 구성 파일에 있는 자격 증명을 사용해 유레카 서버의 API를 테스트

   ```java
   @Value("${app.eureka-username}")
   private String username;
   
   @Value("${app.eureka-password}")
   private String password;
   
   @Autowired
   public void setTestRestTemplate(TestRestTemplate testRestTemplate) {
      this.testRestTemplate = testRestTemplate.withBasicAuth(username, password);
   }
   ```

### 유레카 클라이언트 변경

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: "http://${app.eureka-username}:${app.eureka-password}@${app.eureka-server}:8761/eureka/"
```

### 보안 유레카 서버 테스트

생략

## OAuth 2.0과 OpenID Connect를 사용한 API 접근 인증 및 권한 부여

### 에지 서버와 product-composite 서비스 변경

- OAuth 2.0 자원 서버를 만들고자 build.gradle에 스프링 시큐리티 5.1 의존성 추가

  ```groovy
  implementation('org.springframework.boot:spring-boot-starter-security')
  implementation('org.springframework.security:spring-security-oauth2-resource-server')
  implementation('org.springframework.security:spring-security-oauth2-jose')
  ```

- 보안 구성 추가

  ```java
  @EnableWebFluxSecurity
  public class SecurityConfig {
  
     @Bean
      SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        http
           .authorizeExchange()
              .pathMatchers("/actuator/**").permitAll() // 보호 대상이 아닌 URL에 대한 접근 허용
              .pathMatchers(POST, "/product-composite/**").hasAuthority("SCOPE_product:write")
              .pathMatchers(DELETE, "/product-composite/**").hasAuthority("SCOPE_product:write")
              .pathMatchers(GET, "/product-composite/**").hasAuthority("SCOPE_product:read")
              .anyExchange().authenticated() // 다른 모든 URL에 대한 접근은 인증이 필요함
              .and()
           .oauth2ResourceServer()
              .jwt(); // jwt로 인코딩된 OAuth 2.0 접근 토큰을 기반으로 이증 및 권한 부여를 수행
        return http.build();
     }
  }
  ```

- 권한 부여 서버의 jwk-set 엔드포인트를 구성 파일에 추가

  ```yaml
  spring.security.oauth2.resourceserver.jwt.jwk-set-uri: http://${app.auth-server}:9999/.well-known/jwks.json
  ```

### product-composite 서비스 변경

- 접근 토큰에 있는 OAuth 2.0 스코프를 바탕으로 접근을 허용하도록 보안 구성

  ```java
  .pathMatchers(POST, "/product-composite/**").hasAuthority("SCOPE_product:write") // SCOPE_를 붙이는 것은 관례
  .pathMatchers(DELETE, "/product-composite/**").hasAuthority("SCOPE_product:write")
  .pathMatchers(GET, "/product-composite/**").hasAuthority("SCOPE_product:read")
  ```

- API를 호출할 때마다 관련된 JWT 접근 토큰을 기록하고자 메서드 추가

  ```java
  private void logAuthorizationInfo(SecurityContext sc) {
      if (sc != null && sc.getAuthentication() != null && sc.getAuthentication() instanceof JwtAuthenticationToken) {
          Jwt jwtToken = ((JwtAuthenticationToken)sc.getAuthentication()).getToken();
          logAuthorizationInfo(jwtToken);
      } else {
          LOG.warn("No JWT based Authentication supplied, running tests are we?");
      }
  }
  
  private void logAuthorizationInfo(Jwt jwt) {
    if (jwt == null) {
      LOG.warn("No JWT supplied, running tests are we?");
    } else {
      if (LOG.isDebugEnabled()) {
        URL issuer = jwt.getIssuer();
        List<String> audience = jwt.getAudience();
        Object subject = jwt.getClaims().get("sub");
        Object scopes = jwt.getClaims().get("scope");
        Object expires = jwt.getClaims().get("exp");
  
        LOG.debug("Authorization info: Subject: {}, scopes: {}, expires {}: issuer: {}, audience: {}", subject, scopes, expires, issuer, audience);
      }
    }
  }
  ```

- 스프링 기반 통합 테스트를 실행할 때는 OAuth를 비활성화해야 함

  - 테스트 클래스가 모든 자원제 접근할 수 있도록 허용하고자 테스트용 보안 구성 클래스인 TestSecurityConfig를 추가

    ```java
    @TestConfiguration
    public class TestSecurityConfig {
    
        @Bean
        public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
            http.csrf().disable().authorizeExchange().anyExchange().permitAll();
            return http.build();
        }
    }
    ```

  - 기존 보안 구성을 재정의하고자 모든 스프링 통합 테스트 클래스 변경

    ```java
    @SpringBootTest(
       webEnvironment=RANDOM_PORT,
       classes = {ProductCompositeServiceApplication.class, TestSecurityConfig.class },
       properties = {"spring.main.allow-bean-definition-overriding=true","eureka.client.enabled=false"})
    ```

### 테스트 스크립트 변경

상태 점검 API 이외의 모든 API는 접근 토큰이 있어야 호출 가능

접근 토큰을 확보해 생성 및 삭제 API를 호출하고자 writer 클라이언트 자격으로 다음 커맨드 실행

```bash
ACCESS_TOKEN=$(curl -k https://writer:secret@$HOST:$PORT/oauth/token -d grant_type=password -d username=magnus -d password=password -s | jq .access_token -r)
```

스코프 기반 권한 부여가 작동하는지 확인하고자 테스트 스크립트에 두 가지 테스트 추가

- 첫 번째 테스트는 접근 토큰 없이 API를 호출하며, API는 401 unauthorized HTTP 상태를 반환
- 두 번째 테스트는 reader 클라이언트를 사용해 업데이트 API를 호출하며, read스코프만 있느 접근 토큰 사용. API 403 unauthorized 반환

```bash
# Verify that a request without access token fails on 401, Unauthorized
assertCurl 401 "curl -k https://$HOST:$PORT/product-composite/$PROD_ID_REVS_RECS -s"

# Verify that the reader - client with only read scope can call the read API but not delete API.
READER_ACCESS_TOKEN=$(curl -k https://reader:secret@$HOST:$PORT/oauth/token -d grant_type=password -d username=magnus -d password=password -s | jq .access_token -r)
READER_AUTH="-H \"Authorization: Bearer $READER_ACCESS_TOKEN\""

assertCurl 200 "curl -k https://$HOST:$PORT/product-composite/$PROD_ID_REVS_RECS $READER_AUTH -s"
assertCurl 403 "curl -k https://$HOST:$PORT/product-composite/$PROD_ID_REVS_RECS $READER_AUTH -X DELETE -s"
```

## 로컬 권한 부여 서버를 사용한 테스트

1. 소스 코드를 빌드하고 테스트 스크립트를 실행해 문제가 없는지 확인
2. OAuth 2.0 승인 흐름을 사용해 접근 토큰을 얻는 방법을 배움
3. 접근 토큰을 사용해 API를 호출

### 자동 테스트 빌드 및 실행

생략

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)
- 유레카 관련 기본 설정(https://cloud.spring.io/spring-cloud-static/Greenwich.RELEASE/single/spring-cloud.html)