---
title: "로또-TDD 리뷰"
date: 2022-01-11
excerpt: "우아한테크캠프PRO"
categories:
- woowahan-techcamp-pro 
tags:
- nextstep
- woowahan-techcamp-pro
- tdd
---
## 부가 설명
테스트하기 쉬운 부분과 어려운 부분을 분리해야 테스트코드를 작성하기 좋습니다.   
분리한 후 도메인 객체에 대해서만 먼저 테스트코드를 작성합니다. (로직은 서비스레이어가 아닌 도메인에 개발해야 합니다.)  
분리된 것으로 생성한 것(ex. random, shuffle, 날짜)은 파라미터로 전달받습니다.   
TDD 리팩토링을 위한 과도기 단계에서는 복붙과 같은 임시 단계를 사용하는 것도 괜찮습니다.
## 1단계
PASS

## 2단계
### 리뷰사항
- 매직넘버 반영

## 3단계
https://github.com/next-step/java-lotto-pro/pull/167
### 리뷰사항
1. 프린트하는 클래스에 Printable 인터페이스를 도입
  -> 도메인은 값만 제대로 뷰 객체에게 전달하면 되기 때문에 인터페이스 불필요함

2. getter를 최대한 사용하지 않는 방법
3. 변수명에 불필요한 이름 추가 X(ex. `String moneyText`)
4. 역할에 맞는 새로운 도메인 객체 추가
5. parseInt 메서드는 뷰에서 처리
### 정리
도메인과 뷰를 분리하는 것이 어려웠습니다.  
그 이유는 도메인 내부를 getter로 노출하고 싶지 않아 다른 방법으로 하기 위해 계속 고민했기 때문이었습니다.  
도메인 내부에 있는 것을 노출시키고 싶지 않아 내부를 직접 출력하는 printable 인터페이스를 도입했는데,  
해당 인터페이스에 의존하여 오히려 더 안 좋은 상황이 발생했습니다.  
그래서 내부를 getter로 일부 노출하는 대신 프린트를 해주는 printable 인터페이스를 제거하여 printable 인터페이스에 대한 의존을 제거했습니다.   
그리고, 뷰에서 출력되는 것에 대한 처리는 모두 뷰단으로 옮겼습니다.

## 4단계
https://github.com/next-step/java-lotto-pro/pull/299
### 리뷰사항
1. Review 클래스 내부에서 관련 로직 처리할 수 있도록 추가
2. 입출력은 view로 이동
3. 스태틱 팩토리 메서드에서 기본 생성자는 private으로 막는다.
### 정리
1. 팩토리 메서드는 자주 사용하지 않아 기본 생성자를 private으로 막는 것에 익숙하지가 않았습니다. 이번 리뷰 이후에는 계속 private 생성자를 추가했습니다.
2. spring에서는 입출력을 알아서 처리해주다보니 입출력 처리에 익숙하지 않아, view단 관련 리뷰사항이 지속되었습니다.

## 5단계
https://github.com/next-step/java-lotto-pro/pull/303
### 리뷰사항
- 클래스의 모호한 네이밍으로 인하여 잘못된 역할로 오인함
- 파싱하는 역할을 뷰로 옮기도록 수정 -> 파싱하는 역할의 parser를 추가했습니다.
- 객체에 캐싱 적용
### 정리
1. 도메인 개발과 view가 익숙하지 않아 역할이 모호하여 잘못 처리된 부분이 많았습니다.
2. Integer 같은 클래스에서 숫자에 캐싱을 적용하는 것에 대해 알 수 있었고, 적용도 해보았습니다.


## 참고

- [우아한 태크캠프 PRO 3기](https://edu.nextstep.camp/s/Reggx5FJ)
- [spring jackson 라이브러리 이해하기](https://mommoo.tistory.com/83)
- [private constructor in java](https://www.upgrad.com/blog/private-constructor-in-java/)
- [JAVA INTEGER 캐시](https://jwdeveloper.tistory.com/140)

