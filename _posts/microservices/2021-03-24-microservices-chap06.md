---
title: "영속성 추가"
date: 2021-03-24
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 6. 영속성 추가

## 진행 방향 확인

**프로토콜 계층(protocol layer)**은 매우 얇으며, 공통 클래스로 구성됨

각 마이크로서비스의 주요 기능은 서비스 계층에 있다

모든 핵심 마이크로서비스에는 자체 데이터베이스와 통신하는 **영속성 계층(Persistence layer)**이 있다.

## 핵심 마이크로서비스에 영속성 계층 추가

스프링 데이터 외에 자바 빈 메핑 도구인 MapStruct 사용

MapStruct를 사용하면 스프링 데이터 엔티티 객체와 API 모델 클래스를 쉽게 상호 변환 가능

#### 의존성 추가

각 핵심 마이크로서비스의 빌드 파일(build.gradle)에 MapStruct의 버전 정보를 담을 변수 선언

```groovy
ext {
    mapstructVersion = "1.3.1.Final"
}
```

MapStruct 의존성 추가

```groovy
implementation("org.mapstruct:mapstruct:${mapstructVersion}")
```

MapStruct가 컴파일 타임에 애노테이션을 처리해 빈 매핑 구현을 생성하려면 annotationProcessor 및 testAnnotationProcessor 의존성 추가 필요

```groovy
annotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
testAnnotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
```

IDE에서 컴파일 타임 생성을 하려면 다음 의존성도 추가

```groovy
compileOnly "org.mapstruct:mapstruct-processor:${mapstructVersion}"
```

product 및 recommendation 마이크로서비스를 위해 스프링 데이터 MongoDB 의존성 추가

```groovy
implementation('org.springframework.boot:spring-boot-starter-data-mongodb')
testImplementation('de.flapdoodle.embed:de.flapdoodle.embed.mongo') // 내장형 MongoDB를 사용해 JUnit 테스트 하기 위해 추가
```

review 마이크로서비스를 위해 의존성 추가

```groovy
implementation('org.springframework.boot:spring-boot-starter-data-jpa')
implementation('mysql:mysql-connector-java') // 런타임 DB
testImplementation('com.h2database:h2') // 테스트용 DB
```

#### 엔티티 클래스를 사용해 데이터 저장

엔티티 클래스 및 대응하는 API 모델 클래스는 포함하고 있는 필드가 비슷함

id 필드와 version 필드는 엔티티 클래스에만 추가



저장된 각 엔티티의 데이터베이스 ID는 id 필드에 보관

보안 관점의 모법 사례를 따라 API로는 id 필드를 공개하지 않음

비즈니스 관점에서 데이터베이스의 일관성을 보장하고자 엔티티를 식별하는 모델 클래스 필드를 해당 엔티티 클래스의 고유 색인으로 지정



version 필드는 낙관적 잠금(optimistic locking)을 구현하고자 사용

즉 스프링 데이터가 동시 업데이트에 의한 겹쳐 쓰기를 확인하고 엔티티를 업데이트하고자 사용

version 필드는 API에 공개 X

```java
@Document(collection="products") // 이 클래스가 MongoDB 엔티티 클래스며, products라는 이름의 MongoDB 컬렉션에 매핑된다는 것을 표시
public class ProductEntity {

    @Id
    private String id;

    @Version
    private Integer version;

    @Indexed(unique = true) // 고유 색인을 가져옴
    private int productId;

    private String name;
    private int weight;
}
```

```java
@Document(collection="recommendations")
@CompoundIndex(name = "prod-rec-id", unique = true, def = "{'productId': 1, 'recommendationId' : 1}") // productId와 recommendationId 필드로 구성된 복합 비즈니스 키를 위한 고유 복합 인덱스 생성
public class RecommendationEntity {

    @Id
    private String id;

    @Version
    private Integer version;

    private int productId;
    private int recommendationId;
    private String author;
    private int rating;
    private String content;
  
  	// 생략
}

```

```java
@Entity // JPA 엔티티 클래스임
@Table(name = "reviews", indexes = { @Index(name = "reviews_unique_idx", unique = true, columnList = "productId,reviewId") }) // SQL 데이터베이스의 products 테이블에 매핑된다는 것을 표시, 고유 복합 인덱스 생성
public class ReviewEntity {

    @Id @GeneratedValue // id값 자동 생성
    private int id;

    @Version
    private int version;

    private int productId;
    private int reviewId;
    private String author;
    private String subject;
    private String content;
}

```

#### 스프링 데이터 리포지토리 정의

CrudRepository 클래스는 데이터베이스에 데이터를 생성하거나 데이터베이스에 저장된 데이터를 읽고 업데이트하고 삭제하기 위한 표준 메서드를 제공

PagingAndSortingRepository는 CrudRepository 클래스에 페이징 및 정렬 기능을 추가한 클래스

스프링 데이터는 메서드 서명의 명명 규칙에 따른 쿼리 메서드 정의를 지원

findByProductId(int productId)라면 스프링 데이터는 쿼리를 호출할 때 productId 매개 변수에 지정한 값으로 productId 필드를 설정하고 기본 컬렉션이나 테이블에서 해당 엔티티를 찾아 반환하는 쿼리를 자동으로 작성한다.

```java
public interface ProductRepository extends PagingAndSortingRepository<ProductEntity, String> {
    Optional<ProductEntity> findByProductId(int productId); // 0개 혹은 1개이므로 optional 사용
}
```

```java
public interface RecommendationRepository extends CrudRepository<RecommendationEntity, String> {
    List<RecommendationEntity> findByProductId(int productId);
}
```

```java
public interface ReviewRepository extends CrudRepository<ReviewEntity, Integer> {
    @Transactional(readOnly = true)
    List<ReviewEntity> findByProductId(int productId);
}
```

## 영속성에 중점을 둔 자동 테스트 작성

영속성 테스트는 테스트를 시작할 때 내장형 데이터베이스를 시작하고 테스트가 완료되면 데이터베이스를 중지

하지만 런타임에 필요한 웹 서버 등의 다른 자원이 시작되길 기다리진 않는다.

- `@DataMongoTest`: 테스트를 시작할 때 내장형 MongoDB 데이터베이스를 시작한다
- `@DataJpaTest`: 테스트를 시작할 때 내장형 SQL 데이터베이스를 시작한다.
  - review 마이크로서비스의 필드 파일에 H2 데이터베이스를 테스트 의존성으로 추가 - > H2를 내장형 데이터베이스로 사용
  - 기본적으로 스프링 부트는 다른 테스트에 의한 부작용을 최소화하고자 SQL 데이터베이스에 업데이트한 내용을 롤백하도록 테스트를 구성

```java
@RunWith(SpringRunner.class)
@DataMongoTest
public class PersistenceTests {

	@Autowired
	private ProductRepository repository;

	private ProductEntity savedEntity;

	@Before
	public void setupDb() {
		repository.deleteAll();

		ProductEntity entity = new ProductEntity(1, "n", 1);
		savedEntity = repository.save(entity);

		assertEqualsProduct(entity, savedEntity);
	}

	@Test
	public void create() {

		ProductEntity newEntity = new ProductEntity(2, "n", 2);
		repository.save(newEntity);

		ProductEntity foundEntity = repository.findById(newEntity.getId()).get();
		assertEqualsProduct(newEntity, foundEntity);

		assertEquals(2, repository.count());
	}

	@Test
	public void update() {
		savedEntity.setName("n2");
		repository.save(savedEntity);

		ProductEntity foundEntity = repository.findById(savedEntity.getId()).get();
		assertEquals(1, (long)foundEntity.getVersion());
		assertEquals("n2", foundEntity.getName());
	}

	@Test
	public void delete() {
		repository.delete(savedEntity);
		assertFalse(repository.existsById(savedEntity.getId()));
	}

	@Test
	public void getByProductId() {
		Optional<ProductEntity> entity = repository.findByProductId(savedEntity.getProductId());

		assertTrue(entity.isPresent());
		assertEqualsProduct(savedEntity, entity.get());
	}

	@Test(expected = DuplicateKeyException.class)
	public void duplicateError() {
		ProductEntity entity = new ProductEntity(savedEntity.getProductId(), "n", 1);
		repository.save(entity);
	}

	@Test
	public void optimisticLockError() {

		// Store the saved entity in two separate entity objects
		ProductEntity entity1 = repository.findById(savedEntity.getId()).get();
		ProductEntity entity2 = repository.findById(savedEntity.getId()).get();

		// Update the entity using the first entity object
		entity1.setName("n1");
		repository.save(entity1);

		//  Update the entity using the second entity object.
		// This should fail since the second entity now holds a old version number, i.e. a Optimistic Lock Error
		try {
			entity2.setName("n2");
			repository.save(entity2);

			fail("Expected an OptimisticLockingFailureException");
		} catch (OptimisticLockingFailureException e) {
		}

		// Get the updated entity from the database and verify its new sate
		ProductEntity updatedEntity = repository.findById(savedEntity.getId()).get();
		assertEquals(1, (int)updatedEntity.getVersion());
		assertEquals("n1", updatedEntity.getName());
	}

	@Test
	public void paging() {

		repository.deleteAll();

		List<ProductEntity> newProducts = rangeClosed(1001, 1010)
			.mapToObj(i -> new ProductEntity(i, "name " + i, i))
			.collect(Collectors.toList());
		repository.saveAll(newProducts);

		Pageable nextPage = PageRequest.of(0, 4, ASC, "productId");
		nextPage = testNextPage(nextPage, "[1001, 1002, 1003, 1004]", true);
		nextPage = testNextPage(nextPage, "[1005, 1006, 1007, 1008]", true);
		nextPage = testNextPage(nextPage, "[1009, 1010]", false);
	}

	private Pageable testNextPage(Pageable nextPage, String expectedProductIds, boolean expectsNextPage) {
		Page<ProductEntity> productPage = repository.findAll(nextPage);
		assertEquals(expectedProductIds,
			productPage.getContent().stream().map(p -> p.getProductId()).collect(Collectors.toList()).toString());
		assertEquals(expectsNextPage, productPage.hasNext());
		return productPage.nextPageable();
	}

	private void assertEqualsProduct(ProductEntity expectedEntity, ProductEntity actualEntity) {
		assertEquals(expectedEntity.getId(), actualEntity.getId());
		assertEquals(expectedEntity.getVersion(), actualEntity.getVersion());
		assertEquals(expectedEntity.getProductId(), actualEntity.getProductId());
		assertEquals(expectedEntity.getName(), actualEntity.getName());
		assertEquals(expectedEntity.getWeight(), actualEntity.getWeight());
	}
}
```

## 서비스 계층에서 영속성 계층 사용

1. 데이터베이스 연결 URL 기록

   자체 데이터베이스와 연결된 마이크로서비스를 확장하는 경우 각 마이크로서비스가 실제로 사용하는 데이터베이스가 무엇인지 파악하기 힘든 문제가 있다.

   따라서 마이크로서비스가 시작된 직후에 접속한 데이터베이스의 URL을 기록하는 로그 문을 추가한다.

   ```java
   public class ProductServiceApplication {
   
   	private static final Logger LOG = LoggerFactory.getLogger(ProductServiceApplication.class);
   
   	public static void main(String[] args) {
   
   		ConfigurableApplicationContext ctx = SpringApplication.run(ProductServiceApplication.class, args);
   
   		String mongodDbHost = ctx.getEnvironment().getProperty("spring.data.mongodb.host");
   		String mongodDbPort = ctx.getEnvironment().getProperty("spring.data.mongodb.port");
   		LOG.info("Connected to MongoDb: " + mongodDbHost + ":" + mongodDbPort);
   	}
   ```

2. 새  API 추가

   생성/삭제 API 추가

   ```java
   @PostMapping(
     value    = "/product",
     consumes = "application/json",
     produces = "application/json")
   Product createProduct(@RequestBody Product body);
   ```

3. 영속성 계층 사용

   ```java
   private final ServiceUtil serviceUtil;
   private final ProductRepository repository;
   private final ProductMapper mapper;
   
   @Autowired
   public ProductServiceImpl(ProductRepository repository, ProductMapper mapper, ServiceUtil serviceUtil) {
     this.repository = repository;
     this.mapper = mapper;
     this.serviceUtil = serviceUtil;
   }
   ```

   매퍼 클래스

   ```java
   @Override
   public Product createProduct(Product body) {
     try {
       ProductEntity entity = mapper.apiToEntity(body);
       ProductEntity newEntity = repository.save(entity);
   
       LOG.debug("createProduct: entity created for productId: {}", body.getProductId());
       return mapper.entityToApi(newEntity);
   
     } catch (DuplicateKeyException dke) {
       throw new InvalidInputException("Duplicate key, Product Id: " + body.getProductId());
     }
   }
   
   @Override
   public Product getProduct(int productId) {
   
     if (productId < 1)
       throw new InvalidInputException("Invalid productId: " + productId);
   
     ProductEntity entity = repository.findByProductId(productId)
       .orElseThrow(() -> new NotFoundException("No product found for productId: " + productId));
   
     Product response = mapper.entityToApi(entity);
     response.setServiceAddress(serviceUtil.getServiceAddress());
   
     LOG.debug("getProduct: found productId: {}", response.getProductId());
   
     return response;
   }
   
   @Override
   public void deleteProduct(int productId) {
     LOG.debug("deleteProduct: tries to delete an entity with productId: {}", productId);
     repository.findByProductId(productId).ifPresent(e -> repository.delete(e));
   }
   ```

   삭제 오퍼레이션의 구현은 멱등성(idempotent)이 있어야 한다. 즉 여러번 호출하더라도 같은 결과를 반환해야 함 -> 엔티티가 DB에 존재하지 않더라도 200 return

4. 자바 빈 매퍼 선언

   MapStruct로 매퍼 클래스 선언

   ```java
   @Mapper(componentModel = "spring")
   public interface ProductMapper {
   
       @Mappings({
           @Mapping(target = "serviceAddress", ignore = true) // entity클래스에는 serviceAdress가 필요없으므로 무시
       })
       Product entityToApi(ProductEntity entity);
   
       @Mappings({
           @Mapping(target = "id", ignore = true), // API 모델 클래스에 없는 id와 version 필드를 무시
           @Mapping(target = "version", ignore = true)
       })
       ProductEntity apiToEntity(Product api);
   }
   ```

   ```java
       @Mappings({
           @Mapping(target = "rate", source="entity.rating"), // 이름이 다른 필드의 매핑도 지원
           @Mapping(target = "serviceAddress", ignore = true)
       })
       Recommendation entityToApi(RecommendationEntity entity);
   ```

5. 서비스 테스트 업데이트

   ```java
   @RunWith(SpringRunner.class)
   @SpringBootTest(webEnvironment = RANDOM_PORT, properties = {"spring.data.mongodb.port: 0"})
   public class ProductServiceApplicationTests {
   
   	@Autowired
   	private WebTestClient client;
   
   	@Autowired
   	private ProductRepository repository;
   
   	@Before
   	public void setupDb() { // 테스트 전에 data 초기화
   		repository.deleteAll();
   	}
   
   	@Test
   	public void duplicateError() {
   
   		int productId = 1;
   
   		postAndVerifyProduct(productId, OK);
   
   		assertTrue(repository.findByProductId(productId).isPresent());
   
   		postAndVerifyProduct(productId, UNPROCESSABLE_ENTITY) // 응답이 예상한 오류가 맞는지 확인
   			.jsonPath("$.path").isEqualTo("/product")
   			.jsonPath("$.message").isEqualTo("Duplicate key, Product Id: " + productId);
   	}
   
   	@Test
   	public void deleteProduct() {
   
   		int productId = 1;
   
   		postAndVerifyProduct(productId, OK);
   		assertTrue(repository.findByProductId(productId).isPresent());
   
   		deleteAndVerifyProduct(productId, OK);
   		assertFalse(repository.findByProductId(productId).isPresent());
   
   		deleteAndVerifyProduct(productId, OK); // 없을 때도 OK return 하는지 확인(멱등성)
   	}
   
   	private WebTestClient.BodyContentSpec postAndVerifyProduct(int productId, HttpStatus expectedStatus) {
   		Product product = new Product(productId, "Name " + productId, productId, "SA");
   		return client.post()
   			.uri("/product")
   			.body(just(product), Product.class)
   			.accept(APPLICATION_JSON)
   			.exchange()
   			.expectStatus().isEqualTo(expectedStatus)
   			.expectHeader().contentType(APPLICATION_JSON)
   			.expectBody();
   	}
   
   	private WebTestClient.BodyContentSpec deleteAndVerifyProduct(int productId, HttpStatus expectedStatus) {
   		return client.delete()
   			.uri("/product/" + productId)
   			.accept(APPLICATION_JSON)
   			.exchange()
   			.expectStatus().isEqualTo(expectedStatus)
   			.expectBody();
   	}
   
   }
   ```

## 복합 서비스 API 확장

1. 복합 서비스 API에 새 오퍼레이션 추가

   기존과 유사하여 생략

2. 통합 계층에 메서드 추가

   ```java
   @Override
   public Product createProduct(Product body) {
   
     try {
       String url = productServiceUrl;
       LOG.debug("Will post a new product to URL: {}", url);
   
       Product product = restTemplate.postForObject(url, body, Product.class);
       LOG.debug("Created a product with id: {}", product.getProductId());
   
       return product;
   
     } catch (HttpClientErrorException ex) {
       throw handleHttpClientException(ex);
     }
   }
   ```

3. 새 복합 API 오퍼레이션 구현

   ```java
   @Override
   public void createCompositeProduct(ProductAggregate body) {
   
     try {
   
       LOG.debug("createCompositeProduct: creates a new composite entity for productId: {}", body.getProductId());
   
       Product product = new Product(body.getProductId(), body.getName(), body.getWeight(), null);
       integration.createProduct(product);
   
       if (body.getRecommendations() != null) {
         body.getRecommendations().forEach(r -> {
           Recommendation recommendation = new Recommendation(body.getProductId(), r.getRecommendationId(),
                                                              r.getAuthor(), r.getRate(), r.getContent(), null);
           integration.createRecommendation(recommendation);
         });
       }
   
       if (body.getReviews() != null) {
         body.getReviews().forEach(r -> {
           Review review = new Review(body.getProductId(), r.getReviewId(), r.getAuthor(), r.getSubject(),
                                      r.getContent(), null);
           integration.createReview(review);
         });
       }
   
       LOG.debug("createCompositeProduct: composite entites created for productId: {}", body.getProductId());
   
     } catch (RuntimeException re) {
       LOG.warn("createCompositeProduct failed", re);
       throw re;
     }
   }
   ```

   각  create 메서드 호출

   이외 유사하여 생략

   오류 상황에서 깨지기 쉬움(현재 상황)

4. 복합 서비스 테스트 업데이트

   ```java
   	@Test
   	public void createCompositeProduct1() {
   
   		ProductAggregate compositeProduct = new ProductAggregate(1, "name", 1, null, null, null);
   
   		postAndVerifyProduct(compositeProduct, OK);
   	}
   
   	@Test
   	public void createCompositeProduct2() {
   		ProductAggregate compositeProduct = new ProductAggregate(1, "name", 1,
   				singletonList(new RecommendationSummary(1, "a", 1, "c")),
   				singletonList(new ReviewSummary(1, "a", "s", "c")), null);
   
   		postAndVerifyProduct(compositeProduct, OK);
   	}
   
   	@Test
   	public void deleteCompositeProduct() {
   		ProductAggregate compositeProduct = new ProductAggregate(1, "name", 1,
   				singletonList(new RecommendationSummary(1, "a", 1, "c")),
   				singletonList(new ReviewSummary(1, "a", "s", "c")), null);
   
   		postAndVerifyProduct(compositeProduct, OK);
   
   		deleteAndVerifyProduct(compositeProduct.getProductId(), OK);
   		deleteAndVerifyProduct(compositeProduct.getProductId(), OK);
   	}
   ```

## 도커 컴포즈 환경에 데이터베이스 추가

도커 컴포즈가 제어하는 환경에  MongoDB와 MySQL 추가

도커 컨테이너로 실행될 때도 데이터베이스를 찾을 수 있도록 마이크로서비스 구성에 추가

#### 도커 컴포즈 구성

```yaml
  # $ mongo
  mongodb:
    image: mongo:3.6.9 # mongo 공식이미지 사용
    mem_limit: 350m
    ports:
      - "27017:27017" # 로컬 호스트를 바탕으로 접근할 수 있도록 기본 포트를 도커 호스트에 전달
    command: mongod --smallfiles

  # $ mysql -uroot -h127.0.0.1 -p
  mysql:
    image: mysql:5.7 # mysql 공식이미지 사용
    mem_limit: 350m
    ports:
      - "3306:3306" # 로컬 호스트를 바탕으로 접근할 수 있도록 기본 포트를 도커 호스트에 전달
    environment: # 환경변수 선언
      - MYSQL_ROOT_PASSWORD=rootpwd
      - MYSQL_DATABASE=review-db
      - MYSQL_USER=user
      - MYSQL_PASSWORD=pwd
    healthcheck: # mysql 데이터베이스의 상태를 확인하기 위한 헬스체크 방법 선언
      test: ["CMD", "mysqladmin" ,"ping", "-uuser", "-ppwd", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 10
      
  product:
    depends_on: # 몽고DB 서비스에 의존하도록 설정 mongoDB 컨테이너 시작 전에 product가 시작하지 않는다.
      - mongodb
      
  review:
    depends_on: # mysql 컨테이너가 실행 중이고 헬스체크 결과가 정상일 때만 실행, 초기화 과정을 진행하는 동안 review 서비스의 시작을 보류시킴
      mysql:
        condition: service_healthy
      
# 생락...
```

#### 데이터베이스 연결 구성

product 서비스

```yaml
spring.data.mongodb: # 도커 없이 실행시
  host: localhost
  port: 27017
  database: product-db

logging:
  level:
    org.springframework.data.mongodb.core.MongoTemplate: DEBUG # 어떤 몽고DB에 연결되었는지 확인하기 위한 레벨 설정
---
spring.profiles: docker

spring.data.mongodb.host: mongodb # 도커 컨테이너 연결
```

review 서비스

```yaml
spring.jpa.hibernate.ddl-auto: update # 시작하는 동안 기존 SQL 테이블을 새로 만들거나 업데이트하도록 스프링 데이터 JPA에 지시, 실서비스에서는 주로 사용 X

spring.datasource: # 도커 없이 실행시
  url: jdbc:mysql://localhost/review-db
  username: user
  password: pwd

spring.datasource.hikari.initializationFailTimeout: 60000 # 스프링 부트 애플리케이션이 시작될 때, 최대 60초 동안 데이터베이스 연결을 기다림

logging:
  level: # 하이버네이트가 사용하는 SQL문과 실제 사용값을 출력하기 위함
    org.hibernate.SQL: DEBUG 
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE

---
spring.profiles: docker

spring.datasource: # 도커 컨테이너 연결
  url: jdbc:mysql://mysql/review-db
```

#### MongoDB 및 MySQL CLI 도구

```bash
docker-compose exec mongodb mongo --quiet
```

```bash
docker-compose exec mysql mysql --uuser -p review-db
```

## 새  API 및 영속성 계층의 수동 테스트

생략

## 마이크로서비스 환경의 자동 테스트 업데이트

```bash
function recreateComposite() { # 지우고 생성
    local productId=$1
    local composite=$2

    assertCurl 200 "curl -X DELETE http://$HOST:$PORT/product-composite/${productId} -s"
    curl -X POST http://$HOST:$PORT/product-composite -H "Content-Type: application/json" --data "$composite"
}

function setupTestdata() { # 테스트 데이터 세팅

    body=\
'{"productId":1,"name":"product 1","weight":1, "recommendations":[
        {"recommendationId":1,"author":"author 1","rate":1,"content":"content 1"},
        {"recommendationId":2,"author":"author 2","rate":2,"content":"content 2"},
        {"recommendationId":3,"author":"author 3","rate":3,"content":"content 3"}
    ], "reviews":[
        {"reviewId":1,"author":"author 1","subject":"subject 1","content":"content 1"},
        {"reviewId":2,"author":"author 2","subject":"subject 2","content":"content 2"},
        {"reviewId":3,"author":"author 3","subject":"subject 3","content":"content 3"}
    ]}'
    recreateComposite 1 "$body"

    body=\
'{"productId":113,"name":"product 113","weight":113, "reviews":[
    {"reviewId":1,"author":"author 1","subject":"subject 1","content":"content 1"},
    {"reviewId":2,"author":"author 2","subject":"subject 2","content":"content 2"},
    {"reviewId":3,"author":"author 3","subject":"subject 3","content":"content 3"}
]}'
    recreateComposite 113 "$body"

    body=\
'{"productId":213,"name":"product 213","weight":213, "recommendations":[
    {"recommendationId":1,"author":"author 1","rate":1,"content":"content 1"},
    {"recommendationId":2,"author":"author 2","rate":2,"content":"content 2"},
    {"recommendationId":3,"author":"author 3","rate":3,"content":"content 3"}
]}'
    recreateComposite 213 "$body"

}
```



## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)
- 낙관적 잠금(https://reiphiel.tistory.com/entry/understanding-jpa-lock)
- MapStruct(https://meetup.toast.com/posts/213)