---
layout: post
title: "토비의 스프링 7장"
date: 2018-10-22
excerpt: "스프링 핵심 기술의 응용"
tags: [tobySpring]
comments: true
---

# 7. 스프링 핵심 기술의 응용

## 7.1 SQL과 DAO의 분리

* SQL 변경이 필요한 상황이 발생하면 SQL을 담고 있는 DAO 코드가 수정될 수밖에 없다.
* 7.1.1 XML 설정을 이용한 분리
	* 개별 SQL 프로퍼티 방식
	* SQL 맵 프로퍼티 방식
		* 맵으로 만들어두면 새로운 SQL이 필요할 때 설정에 \<entry>만 추가해주면 되는 모든 SQL을 일일이 프로퍼티로 등록하는 방법에 비해 작업량도 적고 코드도 간단해서 좋다
* 7.1.2 SQL 제공 서비스
	* SQL 서비스 인터페이스
	* 스프링 설정을 사용하는 단순 SQL 서비스
		* 모든 DAO는 SQL을 어디에 저장해두고 가져오는지에 대해서는 전혀 신경 쓰지 않아도 된다.
		* sqlService 빈에는 DAO에는 전혀 영향을 주지 않은 채로 다양한 방법으로 구현된 SqlService 타입 클래스를 적용할 수 있다.

## 7.2 인터페이스의 분리와 자기참조 빈

* 7.2.1 XML 파일 매핑
	* JAXB
		* XML 문서정보를 거의 동일한 구조의 오브젝트로 직접 매핑
		* XML 문서의 구조를 정의한 스키마를 이용해서 매핑할 오브젝트의 클래스까지 자동으로 만들어주는 컴파일러 제공
	* SQL 맵을 위한 스키마 작성과 컴파일
	* 언마샬링
		* 언마샬링: XML 문서를 읽어서 자바의 오브젝트로 변환하는 것
		* 마샬링: 바인딩 오브젝트를 XML 문서로 변환하는 것
* 7.2.2 XML 파일을 애용하는 SQL 서비스
	* SQL 맵 XML 파일
	* XML SQL 서비스
		* 처음 SQL을 읽는 부분이 어디인지 모르니 우선 생성자에서 SQL을 읽어와 내부에 저장
* 7.2.3 빈의 초기화 작업
	* 생성자에서 예외가 밸상할 수도 있는 복잡한 초기화 작업을 다루는 것은 좋지 않음
	* 읽어들일 파일의 위치와 이름이 코드에 고정되어 있다는 점
	* @POstConstruct는 빈 오브젝트의 초기화 메소드를 지정하는 데 사용 
* 7.2.4 변화를 위한 준비: 인터페이스 분리
	* XML 대신 다른 포맷의 파일에서 SQL을 읽어오게 하려면?
	* 책임에 따른 인터페이스 정의
		1. SQL 정보를 외부의 리소스로부터 읽어오는 것
		2. 읽어논 SQL을 보관해두고 있다가 필요할 때 제공해주는 것
	* SqlService의 구현 클래스가 변경 가능한 책임을 가진 SqlReader와 SqlRegistry 두 가지 타입의 오브젝트를 사용하도록 만든다.
	* 정보를 전달하기 위해 일시적으로 Map 타입의 형식을 갖도록 만들어야 한다는 건 불편하다.
	* SqlReader에게 SqlRegistry 전략을 제공해주면서 이를 이용해 SQL 정보를 SqlRegistry에 저장하라고 요청
	```
	sqlReader.readSql(sqlRegistry); // SQL을 지정할 대상인 sqlRegistry 오브젝트를 전달
	```
	* SqlRegistry 인터페이스
	```
	public interface SqlRegistry {
		public interface SqlRegistry {
			void registerSql(String key, String sql); // SQL을 키와 함께 등록한다.
			String findSql(String key) throws SqlNotFoundException; // 키로 SQL 검색
		}
	}
	```
	* SqlReader 인터페이스
	```
	public interface SqlReader {
		void read(SqlRegistry sqlRegistry);
	}
	```
* 7.2.5 자기참조 빈으로 시작하기
	* 다중 인터페이스 구현과 간접 참조
		* XmlSqlService 인터페이스를 3개의 인터페이스를 상속 받도록 구조 변경
	* 인터페이스를 이용한 분리
	* 자기참조 빈 설정
* 7.2.6 디폴트 의존관계
	* 확장 가능한 기반 클래스
		* BaseSqlService를 SqlService 빈으로 등록하고 SqlReader와 SqlRegistry를 구현한 클래스 역시 빈으로 등록해서 DI
	* 디폴트 의존관계를 갖는 빈 만들기
		* 디폴트 의존관계: 외부에서 DI받지 않는 경우 기본적으로 자동 적용되는 의존관계
		* JaxbXmlSqlReader를 독립적인 빈으로 설정했을 떄와 달리 디폴트 의존 오브젝트로 직접 넣어줄 때는 프로퍼티를 외부에서 지정할 수 없는 문제 발생
		* 디폴트 파일 이름 추가
		* 설정을 통해 다른 구현 오브젝트를 사용하게 해도 DefaultSqlService는 생서자에서 일단 디폴트 의존 오브젝트를 만든다는 단점이 있다.

## 7.3 서비스 추상화 적용

* 자바에는 JAXB 외에도 다양한 XML과 자바오브젝트를 매핑하는 기술이 있기 때문에 필요에 따라 다른 기술로 손쉽게 바뀌서 사용할 수 있게 해야 한다.
* XML 파일을 좀더 다양한 소스에서 가져올 수 있게 만든다.
* 7.3.1 OXM 서비스 추상화
	* Castol XML, JiBX, XmlBeans, Xstream 존재
	* OXM(Object-XML Mapping): XML과 자바오브젝트를 매핑해서 상호 변환해주는 기술
	* OXM 서비스 인터페이스
	```
	public interface Unmarshaller {
		boolean supports(Class<?> clazz); // 해당 클래스로 어마샬이 가능한지 확인해준다.
		Object unmarshall(Source source) throws IOException, XmlMappingException; // source를 통해 제공받은 XML을 자바오브젝트 트리로 변환해서 그 루트 오브젝트를 돌려준다.
	}
	```
	* JAXB 구현 테스트
	* Castor 구현 테스트
* 7.3.2 OXM 서비스 추상화 적용
	* 멤버 클래스를 참조하는 통합 클래스
		* 하나의 클래스로 만들어 빈의 등록과 설정을 단순해지고 쉽게 사용할 수 있다.
* 7.3.3 리소스 추상화
	* 자바의 클래스패스 안에 존재하는 리소스사 서블릿 컨텍스트의 리소스 또는 임의의 스트림으로 가져올 수 있는 리소르를 지정하는 방법이 없다는 점과 리소스 파일의 존재 여부를 미리 확인할 수 있는 기능이 없다는 단점
	* 리소스
		* 리소스는 빈이 아니라 값으로 취급
	* 리소스 로더
		* 문자열 안에 리소스의 종류와 리소스의 위치를 함께 표현하게 해주는 것
		* 스프링의 애플리	케이션 컨텍스트가 대표적인 예
	* Resource를 이용해 XML 파일 기져오기

## 7.4 인터페이스 상속을 통한 안전한 기능확장
 
* 7.4.1 DI와 기능의 확장
	* DI를 의식하는 설계
		* 확장 고려
	* DI와 인터페이스 프로그래밍
		* 다형성을 얻기 위해
		* 분리 원칙을 통해 클라이언트와 의존 오브젝트 사이의 관계를 명확하게 해줄 수 있기 때문에
		* 인터페이스는 하나의 오브젝트가 여러 개를 구현할 수 있으므로, 하나의 오브젝트를 바라보는 창이 여러가지일 수 있다.
		* 인터페이스 분리 원칙 : 목적과 관심이 각기 다른 클라이언트가 있다면 인터페이스를 통해 이를 적절하게 분리해줄 필요가 있다.
* 7.4.2 인터페이스 상속

## 7.5 DI를 이용해 다양한 구현 방법 적용하기

* 7.5.1 ConcurrentHashMap을 이용한 수정 가능 SQL 레지스트리
	* HashMap은 멀티스레드 환경에서 동시에 수정을 시도하거나 수정과 동시에 요청하는 경우 예상하지 못한 결과가 발생할 수 있다.
	* 그래서 동기화된 해시 데이터 조작에 최적화도도록 만들어진 ConcurrentHashMap을 사용
	* 수정 가능 SQL 레지스트리 테스트
	* 수정 가능 SQL 레지스트리 구현
* 7.5.2 내장형 데이터베이스를 이용한 SQL 레지스트리 만들기
	* ConcurrentHashMap이 멀티스레드 환경에서 최소한의 동시성을 보장해주고 성능도 그리 나쁜 편은 아니지만, 저장되는 데이터의 ㅑㅇㅇ이 많아지고 작은 조회와 변경이 일어나는 환경이라면 한계가 있다.
	* 내장형 DB: 애플리케이션에 내장돼서 애플리케이션과 함께 시작되고 종료되는 DB
		* 데이터가 메모리에 저장되기 떄문에 IO로 인해 발생하는 부하가 적어서 성능이 뛰어나다.
		* Map과 같은 컬렉션이나 오브젝트를 이용해 메모리에 데이터를 저장해두는 방법에 비해 매우 효과적이고 안정적인 방법으로 등록, 수정, 검색이 가능
	* 스프링의 내장형 DB 지원 기능
		* Derby, HSQL, H2 등이 있다
		* 스프링은 내장현 DB를 초기화하는 작업을 지원하는 편리한 내장형 DB빌더 제공
	* 내장형 DB 빌더 학습 테스트
		* SQL 준비 및 등록
		```
			new EmbeddedDatabaseBuilder() // 빌더 오브젝트 생성
				.setType(내장형DB종류) // EmbeddedDatabaseType의 HSQL, DERBY, H2 중에 하나 선택
				.addScript(초기화에 사용할 DB 스크립트의 리소스) // 테이블 생성과 데이터 초기화를 위해 사용할 SQL 문장을 담은 SQL 스크립트의 위치 저장
				...
				.build() // 주어진 조건에 맞는 내장형 DB를 준비하고 초기화 스크립트를 모두 실행한 뒤에 이에 접근할 수 있는 EmbeddedDatabase를 돌려준다.
		```
	* 내장형 DB를 이용한 SqlRegistry 만들기
		* EmbeddedDatabaseBuilder는 초기화 코드가 필요하다
		* 따라서 EmbeddedDatabaseBuilder는를 활용해서 EmbeddedDatabase 타입의 오브젝트를 생성해주는 팩토리 빈 필요
	* UpdatableSqlRegistry 테스트 코드의 재사용
	* XML 설정을 통한 내장형 DB의 생성과 적용
* 7.5.3 트랜잭션 적용
	* 여러 개의 SQL을 변경하는 작업을 진해하는 중에 존재하지 않는 키가 발견되면??
	* 트랜잭셩 적용 필요
	* 다중 SQL 수정에 대한 트랜잭션 테스트
	* 코드를 이용한 트랜잭션 적용

## 7.6 스프링 3.1의 DI

* 자바 언어의 변화와 스프링
	* 애노테이션의 메타정보 활용
		* 애노테이션은 오변에 따라 컴파일된 클래스에 존재하거나 애플리케이션이 동작할 떄 메모리에 로딩되기도 하지만 자바 코드가 실행되는 데 직접 참여하지 못한다
		* 애노테이션은 애플리케이션을 핵심 로직을 담은 자바 코드와 이를 지원하는 IoC 방식의 프레임워크, 그리고 프레임워크가 참조하는 메타정보라는 세 가지로 구성하는 방식에 잘 어울린다.
		* 애노테이션 하나를 자바 코드에 넣는 것만으로도, 애노테이션을 참고하는 코드에서는 다양한 부가 정보를 얻어낼 수 있다는 장점이 있다.
		* 그리고 XML에 비해 짧다는 장점이 있다.
		* 변경할 때마다 새로 컴파일해줘야 하는 단점이 있다.
	* 정책과 관례를 이용한 프로그래밍
		* 스프링은 점차 애노테이션으로 메타정보를 작성하고, 미리 정해진 정책과 관례를 활용해서 간결한 코드에 많은 내용을 담을 수 있는 방식을 적극 도입하고 있다.
* 7.6.1 자바 코드를 이용한 빈 설정
	* 먼저 XML 제거
	* 테스트 컨텍스트의 변경
		* DI 정보로 사용될 자바 클래스(TestApplicationContext) 생성
		* 다 옮기기 힘드니 우선 @ImportResource를 이용하여 XML 설정정보를 가져와서 사용
	* \<context:annotaion-config/> 제거
		* @Configuration이 붙은 설정 클래스를 사용하는 컨테이너가 사용되면 더 이상 \<context:annotaion-config/>
	* \<bean/>의 전환
		```
		@Bean
		public DataSource dataSource() {
		SimpleDataSource dataSource = new SimpleDataSource();
		dataSource.setDriverClass(Driver.class);
		dataSource.setUrl("");
		dataSource.setUserame("spring");
		dataSource.setPassword("book");
	}
		```
	* 전용 태그 전환
* 7.6.2 빈 스캐닝과 자동와이어링
	* @Autowired를 이용한 자동와이어링
		* @Autowired는 자동와이어링 기법을 이용해서 조건에 맞는 빈을 찾아 자동으로 수정자 메소드나 필드에 넣어준다.
		* @Autowired는 일단 타입을 기준으로 적용할 빈을 찾아보고, 같은 타입의 빈이 두 개 이상 발견되면 이름을 기준으로 다시 최종 후보를 찾는다.
	* @Component를 이용한 자동 빈 등록
		* @Component는 클래스에 부여됨
		* @ComponentScan: 특정 패키지 아래서만 찾도록 기준이 되는 패키지를 지정
		* 메타 애노테이션에도 사용 가능
		* 메타 애노테이션: 애노테이션의 정의에 부여된 애노테이션
* 7.6.3 컨텍스트 분리와 @Import
	* 성격이 다른 DI 정보 분리(test, 실제)
	* 테스트용 컨텍스트 분리
	* @Import
		* sql 관련 빈 분리
* 7.6.4 프로파일
	* 테스트환경과 운영환경에서 각기 다른 빈 정의가 필요한 경우가 있다.
	* @Profile과 @ActiveProfile
	```
	@Profile("test") // 프로파일 등록
	```
	```
	@ActiveProfiles("test") // test 프포파일 설정
	```
	* 컨테이너의 빈 등록 정보 확인
	* 중첩 클래스를 이용한 프로파일 적용
* 7.6.5 프로퍼티 소스
	* @PropertySource
	```
	database.properties 파일
	db.driverClass=com.mysql.jdbc.Driver
	db.url=jdbc:mysql://~~~~~~~~~
	db.username=spring
	db.password=book
	```
		* 프로퍼티 소스: 컨테이너가 프로퍼티 값을 가져오는 대상
		* @PropertySource로 등록한 리소스로부터 가져오는 프로퍼티 값은 컨테이너가 관리하는 Environment 타입의 환경 오브젝트에 저장
	* PropertySourcesPlaceholderConfigurer
		* @Value 애노테이션을 이용하여 Enviroment 오브젝트를 사용하지 않고 받을 수 있다.
		* 치환자 이용 ex. @Value("${db.url}") String url;
		* 치환자를 이용해 프로퍼티 값을 주입하려면 PropertySourcesPlaceholderConfigurer 빈을 선언해야 한다.
		* @Value를 이용하면 dirverClass처럼 문자열을 그대로 사용하지 않고 타입 변환이 필요한 프로퍼티를 스프링이 알아서 처리해준다는 장점이 있다.
* 7.6.6 빈 설정의 재사용과 @Enable
	* 빈 설정자
		* SQL 서비스를 사용하는 각 애플리케이션은 SQL 매핑파일의 위치를 직접 지정할 수 있어야 하는데 지금은 예제 코드의 UserDao 위치로 고정되어 있다.
		* 템플릿/콜백 적용
	@ Enable* 애노테이션
		* @Import를 @Enable로 시작하는 애노테이션으로 대체
		* 의미가 잘 드러나고 깔끔한 장점
		* 엘리먼트를 넣어서 옵션 지정 가능
