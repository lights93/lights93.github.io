---
title: "로또-TDD 리뷰"
date: 2022-01-30
excerpt: "우아한테크캠프PRO"
categories:
- woowahan-techcamp-pro 
tags:
- nextstep
- woowahan-techcamp-pro
- jpa
---
## 부가 설명
테스트하기 쉬운 부분과 어려운 부분을 분리해야 테스트코드를 작성하기 좋습니다.   
분리한 후 도메인 객체에 대해서만 먼저 테스트코드를 작성합니다. (로직은 서비스레이어가 아닌 도메인에 개발해야 합니다.)  
분리된 것으로 생성한 것(ex. random, shuffle, 날짜)은 파라미터로 전달받습니다.   
TDD 리팩토링을 위한 과도기 단계에서는 복붙과 같은 임시 단계를 사용하는 것도 괜찮습니다.
## 1단계
https://github.com/next-step/jwp-qna/pull/224
### 리뷰사항
1.`@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)`를 사용하면 테스트 DB가 아닌 실제 DB 사용(`@ActiveProfiles`을 설정하여 테스트용 DB를 바라보게 설정 가능)
2. `@DisplayName` 을 활용하여 설명을 추가
## 2단계
https://github.com/next-step/jwp-qna/pull/331
### 리뷰사항
1. 요구사항에 맞게 `@Lob` 사용
2. 항상 같이 조회하는 경우가 아니라면 지연로딩 속성 추가(`@OneToMan`y는 LAZY가 디폴트이지만 `@ManyToOne`는 아니기 때문입니다.)
3. `toString()`로 객체를 출력할때 양방향 연관관계일 경우 순환참조가 일어날 수 있기 때문에 주의 필요
4. `@Where`를 활용하면 삭제되지 않은 것만 가지고 올 수 있다 (`@Where(clause = "deleted = false")`)
5. Entity 클래스의 `equals()` 는 id 값으로 비교하는게 좋을 수 있다. 
   1. 1차 캐시를 초기화한 후에 다시 데이터베이스에서 동일한 엔티티를 읽어오는 경우 새로운 객체가 생성 되어도 두 객체를 equals() 했을 때 같아야 같은 row의 데이터임을 확인 가능
   2. 하지만 일반적인 equals()와 다르게 id 값으로만 비교를 하고 아직 영속화되지 않은 Entity에 대해서도 고민 필요
## 3단계
https://github.com/next-step/jwp-qna/pull/356
### 리뷰사항
1. 정적 팩토리 메소드를 만들어보면 인자수도 줄면서 코드의 가독성 높일 수 있다.
   1. 분리했을때 이점은 객체 내에서 스스로 상태와 관련된 검증이나 로직을 처리할 수 있다
2. 순환참조를 예방하는 다양한 방법이 있지만 실무에서는 주로 DTO를 넘기도록 사용
3. 리턴이 없는 메소드는 테스트 하기 힘들뿐더러 지금같은 경우 예외발생(일종의 리턴)에 대한 테스트를 하는 것이 더 좋다
   1. `assertDoesNotThrow(() -> answer.validateOwner(UserTest.JAVAJIGI));`
4. 관련있는 엔티티끼리 패키지를 묶음
## 정리
1. jpa에 대해 모르는 것이 많아 어려웠다.
2. 리뷰를 상세하게 해주셔서 많은 것을 알 수 있었다.


## 참고

- [JPA @where 어노테이션](https://cheese10yun.github.io/jpa-where/)
- [요청과 응답으로 엔티티(Entity) 대신 DTO를 사용하자](https://tecoble.techcourse.co.kr/post/2020-08-31-dto-vs-entity/)
- [JPA 양방향 Entity 무한재귀 문제해결](https://thxwelchs.github.io/JPA%20%EC%96%91%EB%B0%A9%ED%96%A5%20Entity%20%EB%AC%B4%ED%95%9C%20%EC%9E%AC%EA%B7%80%20%EB%AC%B8%EC%A0%9C%20%ED%95%B4%EA%B2%B0/)
- [JPA cascade 종류](https://data-make.tistory.com/668)
- [JPA CascadeType.REMOVE vs orphanRemoval = true](https://tecoble.techcourse.co.kr/post/2021-08-15-jpa-cascadetype-remove-vs-orphanremoval-true/)
- [JPA 임베디드 타입](https://velog.io/@conatuseus/JPA-%EC%9E%84%EB%B2%A0%EB%94%94%EB%93%9C-%ED%83%80%EC%9E%85embedded-type-8ak3ygq8wo)
- [JPA @Embedded 사용시 주의사항](https://jojoldu.tistory.com/559)
- [우테코 JPA 참고](https://tecoble.techcourse.co.kr/tags/jpa/)
- [정적 팩토리 메서드는 왜 사용할까?](https://tecoble.techcourse.co.kr/post/2020-05-26-static-factory-method/)
- [아이템1: 생성자 대신 정적 팩토리 메소드를 고려하라](https://devlog-wjdrbs96.tistory.com/256)

