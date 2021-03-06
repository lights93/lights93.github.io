---
title: "토비의 스프링 4장"
date: 2018-09-30
excerpt: "예외"
categories:
  - spring
tags:
  - tobySpring
---

# 4. 예외

## 4.1 사라진 SQLException

* 4.1.1 초난감 예외처리
	* 예외 블랙홀
		* 모든 예외는 적절하게 복구되든지 아니면 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보돼야 한다.
	* **무의미하고 무책임한 throws**

* 4.1.2 예외의 종류와 특징
	* Error
		* 시스템에 뭔가 비정상적인 상황이 발생했을 경우
		* 대응 방법이 없기 때문에 애플리케이션에서는 신경 쓰지 않아도 된다.
	* Exception과 체크 예외(checked exception)
		* 반드시 예외를 처리하는 코드가 필요
	* RuntimeException과 언체크/런타임 예외
		* 개발자의 부주의로 생기는 예외

* 4.1.3 예외처리 방법
	* **예외 복구**
		* 예외상황을 파악하고 문제를 해결해서 정상 상태로 돌려놓는 것
	* 예외처리 회피
		* 예외처리를 자신이 담당하지 않고 자신을 호출한 쪽으로 던져버리는 것
		* 예외를 회피하는 것은 예외를 복구하는 것처럼 의도가 분명해야 한다. 자신을 사용하는 쪽에서 예외를 다루는 게 최선의 방법이라는 확신이 있어야 한다.
	* **예외 전환**
		* 내부에서 발생한 예외를 그대로 던지는 것이 그 예외상황에 대한 적절한 의미를 부여해주지 못하는 경우에, 의미를 분명하게 해줄 수 있는 예외로 바꿔주기 위해 사용
		* 예외를 처리하기 쉽고 단순하게 만들기 위해 포장

* 4.1.4 예외처리 전략
	* 런타임 예외의 보편화
	* add() 메소드의 예외처리
		* **SQLException을 런타임 예외로 전환**해서 던지도록 만든다. 기존의 아이디 중복 때문에 SQLException이 발생한 경우에는 DuplicateUserIdException을 던지게 내벼려 둔다.
	* 애플리케이션 예외
		* 시스템 또는 외부의 예외상황이 원인이 아니라 애플리케이션 자체의 로직에 의해 의도적으로 발생시키고, 반드시 catch해서 무엇인가 조치를 취하도록 요구하는 예외
		* 이러한 예외를 처리할 수 있는 방법 2가지
			1. 정상적인 상황과 에러 상황에서 각 다른 리턴 값을 돌려줌
			2. 정상적인 흐름을 따르는 코드는 그대로 두고, 잔고 부족과 같은 예외상황에서는 비즈니스적인 의미를 띤 예외를 던지도록 만듬(checked exception)

* 4.1.5 SQLException은 어떻게 됐나?
	* 스프링의 JdbcTemplate은 SQLException을 런타임 예외인 DataAccessException으로 포장해서 던져준다.

## 4.2 예외 전환

* 4.2.1 JDBC의 한계
	* DB를 자유롭게 변경해서 사용할 수 있는 유연한 코드를 보장해주지 못하는 한계가 있다.
	* 비표준 SQL
		* 대부분의 DB는 표준을 따르지 않는 비표준 문법과 기능을 제공한다.
	* 호환성 없는 SQLException의 DB 에러정보
		* DB마다 SQL만 다른 것이 아니라 에러의 종류와 원인도 제각각이지만 SQLException 한 가지만 던지도록 설계되어 있다.
* 4.2.2 DB 에러 코드 매핑을 통한 전환
	* DB 종류가 바뀌더라도 DAO를 수정하지 않으려면 위의 2가지 문제를 해결해야 한다.
	* 에러 코드 매핑 파일을 통해 해결
* 4.2.3 DAO 인터페이스와 DataAccessException 계층구조
	* DAO 인터페이스와 구현의 분리
		* 독립적인 인터페이스로 만들려면 메소드 선언에 예외정보가 문제가 될 수 있다.
		* 런타입 예외로 포장해서 던져주면 해결 가능
		* 하지만, 의미 있게 처리할 수 있는 예외가 있다면 어떤 예외인지 알아야 하기 때문에 클라이언트가 DAO의 기술에 의존적이 될 수 밖에 없다.
	* **데이터 액세스 예외 추상화와 DataAccessException 계층구조**
		* DataAccessException은 일부 게술에서만 공통적으로 나타나는 예외를 포함해서 데이터 액세스 기술에서 발생 가능한 대부분의 예외를 계층구조로 분류해놓았다.
		* 낙관적인 락킹(optimistic locking): 같은 정보를 두 명 이상의 사용자가 동시에 조회하고 순차적으로 업데이트를 할 때, 뒤늦게 업데이트한 것이 먼저 업데한 것을 덮어쓰지 않도록 막아주는 데 쓸 수 있는 편리한 기능

* 4.2.4 기술에 독립적인 UserDao 만들기
	* 인터페이스 적용
		* UserDao 인터페이스
		```java
		public interface UserDao {
			void add(User user);
			User get(String id);
			List<User> getAll();
			void deleteAll();
			int getCount();
		}
		```
	* DataAccescException 활용 시 주의사항
		* 하이버네이트나 JPA에서 DuplicateKeyException을 던지지 않고 다른 예외를 던진다. 그래서 사용하기 전에 확인이 필요하다.
		* DAO에서 사용하는 기술의 종류와 상관없이 동일한 예외를 얻고 싶다면 예외를 정의하여 사용하는 것이 좋다.