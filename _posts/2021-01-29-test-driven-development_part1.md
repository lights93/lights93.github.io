---
title: "TDD 화폐 예제"
date: 2021-01-29
excerpt: "테스트 주도 개발"
categories:
  - tdd
tags:
  - tdd
---

# 1. 화폐 예제

### 다중 통화를 지원하는 Money 객체

앞으로 어떤 일을 해야 하는지 알려주고, 지금 하는 일에 집중할 수 있도록 도와주며, 언제 일이 다 끝나는지 알려줄 수 있게끔 할일 목록을 작성

우선 어떤 테스트가 필요할까? 로 시작



**TDD 리듬**

1. 재빨리 테스트를 하나 추가
2. 모든 테스트를 실행하고 새로 추가한 것이 실패하는지 확인
3. 코드 수정
4. 모든 테스트를 실행하고 전부 성공하는지 확인
5. 리팩토링을 통해 중복 제거



TDD의 핵심은 이런 작은 단계를 밟을 능력을 갖추어야 한다는 것

### 타락한 객체

일반적인 TDD 주기

1. 테스트를 작성
2. 실행 가능하게 만든다
3. 올바르게 만든다

목적은 깔끔한 코드를 얻는 것

'작동하는 깔끔한 코드'를 얻어야 한다는 문제 중에서 '작동하는'에 해당하는 부분을 먼저 해결

그러고 나서 '깔끔한 코드'부분을 해결



최대햔 빨리 초록색을 보기 위해 취할 수 있는 세 전략

- 가짜로 구현하기: 상수를 반환하게 만들고 진짜 코드를 얻을 떄까지 단계적으로 상수를 변수로 바꾸어 간다
- 명백한 구현 사용하기: 실제 구현을 입력한다
- 삼각측량: 일단 skip

### 모두를 위한 평등

값 객체 패턴(value object pattern): 객체를 값처럼 쓸 수 있음

값 객체에 대한 제약사항 중 하나는 객체의 인스턴스 변수가 생성자를 통해서 일단 설정도니 후에는 결코 변하지 않는다는 것

값 객체를 사용하면 별칭 문제를 걱정할 필요가 없다(별칭문제는 다른 곳에서 값을 변했는데, 기존 것에 영향을 주는 경우)

값 객체가 암시하는 것 중 하나는 2장에서와 같이 모든 연산은 새 객체를 반환해야 한다는 것



삼각측량: 만약 라디오 신호를 두 수신국이 감지하고 있을 때, 수신국 사이의 거리가 알려져 있고 각 수신국의 신호의 방향을 알고 있다면, 이 정보들만으로 충분히 신호의 거리와 방위를 알 수 있다.

삼각측량을 이용하려면 예제가 2개 이상 있어야만 코드를 일반화 가능

### 프라이버시

두 테스트가 동시에 실패하면 망한다

테스트와 코드 사이의 결합도는 낮추기 위해, 테스트하는 객체의 새 기능을 사용

### 솔직히 말하자면

테스트 단계

1. 테스트 작성
2. 컴파일되게 하기
3. 실패하는지 확인하기 위해 실행
4. 실행하게 만듦
5. 중복 제거

처음 4단계는 빨리 진행해야 함

주기의 5번째 단계 없이는 앞의 네 단계도 제대로 되지 않는다



코드에서 중복을 제거하기 전까지는 자신의 코드를 파트너를 제외한 어느 누구에게도 보여주려고 하지 않는다

### 중복과 변경

#### 중복 코드 살펴보기

예시로 한 달에 한 번씩 가입자별로 전화 요금을 계산하는 애플리케이션

일반 요금의 요구사항에서 심야 할인 요금 요구사항이 생겨 코드 중복이 생김



코드를 복사하여 구현 시간을 절약한 대가로 지불해야 하는 비용은 예상보다 크다

중복 코드가 존재하기 때문에 언제 터질지 모르는 시한폭탄을 안고 있는 것과 같다

#### 중복 코드 수정하기

세금 추가해야 하는 상황



실수로 한쪽에만 세금 기능을 추가한다면 다른 요금제 가입자에게는 세금이 부과되지 않는 상황 발생 -> 심각한 장애

제대로 수정했어도, 중복 코드를 서로 다르게 수정하기가 쉽다

중복 코드는 새로운 중복 코드를 부른다.

새로운 중복 코드를 추가하는 과정에서 코드의 일관성이 무너질 위험이 항상 도사리고 있다.

더 큰 문제는 중복코드가 늘어날수록 애플리케이션은 변경에 취약해지고 버그가 발생할 가능성이 높아진다는 것

중복 코드의 양이 많아질수록 버그의 수는 증가하며 그에 비례해 코드를 변경하는 속도는 점점 더 느려진다.

#### 타입 코드 사용하기

두 클래수 사이의 중복 코드를 제거하는 한 가지 방법은 클래스를 하나로 합치는 것

구분하는 타입코드를 추가하고 타입 코드의 값에 따라 로직을 분리시킬 수 있음 -> 낮은 응집도와 높은 결합도라는 문제 발생

### 돌아온 '모두를 위한 평등'

적절한 테스트를 갖지 못한 코드에서 TDD를 해야 하는 경우가 좋좋 있을 것이다

충분한 테스트가 없다면 지원 테스트가 갖춰지지 않은 리팩토링을 만나게 될 수밖에 없다.

있으면 좋을 것 같은 테스트를 작성하라

그렇게 하지 않으면 결국에는 리팩토링하다가 뭔가 깨트릴 것이다.

### 사과와 오렌지

### 객체 만들기

하위 클래스의 존재를 테스트에서 분리함으로써 어떤 모들 코드에도 영향을 주지 않고 상속 구조를 마음대로 변경할 수 있도록 함

### 우리가 사는 시간

TDD를 하는 동안 계속 일종의 조율이 필요함

### 흥미로운 시간

### 모든 악의 근원

### 드디어, 더하기

가지고 있는 객체가 원하는 방식으로 동작하지 않을 경우엔 그 객체와 외부 프로토콜이 같으면서 내부 구현은 다른 새로운 객체(imposter)를 만들 수 있다.

### 진짜로 만들기

모든 중복이 제거되기 전까지는 테스트를 통과한 것으로 치지 않았다

### 바꾸기

......



### Money 회고

#### 다음에 할일은 무엇인가

만약 시스템이 크다면, 늘 건드리는 부분들을 절대적으로 견고해야 한다.

그래야 나날이 수정할 때 안심할 수 있기 때문

어떤 테스트 들이 추가로 더 필요할까?? 생각

할일 목록이 빌 때가 그때까지 설계한 것을 검토하기에 적절한 시기

#### 메타포

메타포가 막강함

단지 이름이 아니다!

#### JUnit 사용도

보통 일 분에 한 번 정도 테스트

#### 코드 메트릭스

- 코드와 테스트 사이에 대략 비슷한 양의 함수와 줄이 있다
- 테스트 코드의 줄 수는 공통된 테스트 픽스처를 뽑아내는 작업을 통해 줄일 수 있다. 하지만 모델 코드와 테스트 코드 사이의 대략적인 줄 수의 비율은 비슷하게 유지될 것이다.
- 회기성 복잡도(cyclomatic complexity)는 기존의 흐름 복잡도와 같다. 명시적인 흐름 제어 대신 다형성을 주로 사용했기 때문에 실제 코드의 복잡도 역시 낮다

#### 프로세스

- 작은 테스트를 추가
- 모든 테스트를 실행하고, 실패하는 것을 확인한다
- 코드에 변화를 줌
- 모든 테스트를 실행하고, 성공하는 것을 확인
- 중복을 제거하기 위해 리팩토링

#### 테스트의 질

TDD의 부산물로 자연히 생기는 테스트들은 시스템의 수명이 다할 때까지 함께 유지돼야 할 만큼 확실히 유용하다

하지만 이 테스트들이 다음과 같은 다른 종류의 테스트들을 대체할 수 있을 거라고 예상해서는 안 된다

- 성능 테스트
- 스트레스 테스트
- 사용성 테스트



지표

- 명령문 커버리지(statement coverage)는 테스트의 질에 대한 충분한 평가 기준이 될 수 없음이 확실하지만, 테스트의 시작점

- 결합 삽입(defect insertion)은 테스트의 질을 평가하는 또 다른 방법

  코드의 의미를 바꾼 후에 테스트가 실패하는지 보는 것

#### 최종 검토

TDD를 가르칠 떄 사람들이 자주 놀라느 세가지

- 테스트를 확실히 돌아가게 만드는 세 가지 접근법: 가짜로 구현하기, 삼각측량법, 명백하게 구현하기
- 설계를 주도하기 위한 방법으로 테스트 코드와 실제 코드 사이의 중복을 제거하기
- 길이 미끄러우면 속도를 줄이고 상황이 좋으면 속도를 높이는 식으로 테스트 사이의 간격을 조절할 수 있는 능력


## 참고

- 테스트 주도 개발(http://www.yes24.com/Product/Goods/12246033)

