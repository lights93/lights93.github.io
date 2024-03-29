---
title: "JVM 이야기"
date: 2021-03-02
excerpt: "자바 최적화"
categories:
  - java
tags:
  - optimization
  - java
---

# 2. JVM 이야기

## 2.1 인터프리팅과 클래스로딩

JVM은 스택 기반의 해석 머신

레지스터는 없지만 일부 결과를 실행 스택에 보관하며, 이 스택의 맨 위에 쌓인 값을 가져와 계산

JVM 인터프리터의 기본 로직은, 평가 스택을 이용해 중간값들을 담아두고 가장 마지막에 실행된 명령어와 독립적으로 프로그램을 구성하는 opcode를 하나씩 순서대로 처라히는 'while 루프 안의 switch문'

자바 프로세스가 새로 초기화되면 사슬처럼 줄지어 연결된 클래스로더가 차례차례 작동



자바는 프로그램 실행 중 처음 보는 새클래스를 **디펜던시**(dependency)에 로드

클래스를 찾지 못한 클래스로더는 기본적으로 자신의 부모 클래스로더에게 대신 룩업을 넘김

계속 찾지 못하면 ClassNotFoundException 발생

## 2.2 바이트코드 실행

자바 컴파일러 javac를 이용해 컴파일

javac가 하는 일은 자바 소스 코드를 바이트코드로 가득 찬 .class 파일로 바꾸는 것

바이트 코드는 특정 컴퓨터 아키텍처에 특정하지 않은, **중간 표현형**(Intermediate Representation, IR)



모든 클래스 파일은 매직 넘버로 시작

그 다음은 클래스 파일 포맷 버전

버전이 맞지 않으면 런타임에 UnsupportedClassVersionError 예외 발생



상수 풀에는 코드 곳곳에 등장하는 상숫값(클래스명, 인터페이스명, 필드명 등)

JVM은 코드를 실행할 때 런타임에 배치된 메모리 대신, 상수 풀 테이블을 찾아보고 필요한 값을 참조

## 2.3 핫스팟 입문

제로-오버헤드 원칙

- 사용하지 않는 것에는 대가를 치르지 않습니다
- 여러분이 사용하는 코드보다 더 나은 코드를 건네줄 수는 없습니다.

컴퓨터와 OS가 실제로 어떻게 작동해야 하는지 언어 유저(개발자)가 아주 세세한 저수준까지 일러주어야 함 -> 학습 부담

개발자는 결코 자동화 시스템보다 더 나은 코드를 작성할 수 없다는 전제가 깔려 있음 -> 대부분 어셈블리어 언어로 코딩 안하기 때문에 자동화가 더 유리함



자바는 이러한 제로-오버헤드 추상화 철학을 한번도 동조한 적 X

오히려 핫스팟은 프로그램의 런타임 동작을 분석하고 성능에 가장 유리한 방향으로 영리한 최적화를 적용하는 가상 머신

### 2.3.1 JIT 컴파일이란?

프로그램이 성능을 최대로 내려면 네이티브 기능을 활용해 CPU에서 직접 프로그램을 실행시켜야 함

이를 위해 핫스팟은 프로그램 단위를 인터프리티드 바이트코드에서 네이티브 코드로 컴파일 -> **JIT(just-in-time) 컴파일**



핫스팟은 인터프리티드 모드로 실행하는 동안 애플리케이션을 모니터링하면서 가장 자주 실행되는 코드 파트를 발견해 JIT 컴파일을 수행

특정 메서드가 어느 한계치를 넘어가면 프로파일러가 특정 코드 섹션을 컴파일/최적화함



자바처럼 프로필 기반 최적화를 응용하는 환경에서는 대부분의 AOT 플랫폼에서 불가능한 방식으로 런타임 정보를 활용할 여지가 있으므로 동적 인라이닝 또는 가상 호출등으로 성능을 개선할 수 있다.

## 2.4 JVM 메모리 관리

수십 년간 프로그래밍 역사를 거치면서 메모리 관리 용어나 패턴조차 제대로 모르는 개발자가 태반임



자바는 **가비지 수집**(garbage collection)이라는 프로세스를 이용해 힙 메모리를 자동 관리하는 방식으로 해결

가비지 수집이란 한 마디로, JVM이 더 많은 메모리를 할당해야 할 때 불필요한 메모리를 회수하거나 재사용하는 불확정적 프로세스



일단 GC가 실행되면 그동안 다른 애플리케이션은 모두 중단되고 하던 일은 멈춰야 함

## 2.5 스레딩과 자바 메모리 모델(JMM)

1990년대 후반부터 자바의 멀티스레드 방식은 다음 세 가지 기본 설계 원칙에 기반

- 자바 프로세스의 모든 스레드는 가비지가 수집되는 하나의 공용 힙을 가진다
- 한 스레드가 생성한 객체는 그 객체를 참조하는 다른 스레드가 액세스할 수 있다
- 기본적으로 객체는 변경 가능하다. 즉, 객체 필드에 할당된 값은 프로그래머가 Final 키워드로 불변 표시하지 않는 한 바뀔 수 있다

JMM은 서로 다른 실행 스레드가 객체 안에 변경되는 값을 어떻게 바라보는지를 기술한 공식 메모리 모델

### 2.6 JVM 구현체 종류

- OpenJDK

  자바 기준 구현체를 제공하는 특별한 오픈 소스 프로젝트(GPL)

- 오라클 자바(Oracle)

  OpenJDK가 기반이지만 오라클 상용 라이선스로 재라이선스를 받음

- 줄루(Zulu)

- 아이스티(IcedTea)

- 징(Zing)
- J9

- 애비안(Avian)

- 안드로이드(Android)

#### 2.6.1 JVM 라이선스

오라클 자바(자바 9 이후) 라이선스 체계는 좀 복잡함

오라클 JDK와 OpenJDK는 라이선스 외에는 아무런 차이가 없다

오라클 라이선스에는 개발자가 조심해야 할 몇 가지 조항이 포함되어 있음

- 회사 밖으로 오라클 바이너리를 재배포하는 행위는 허용되지 않음
- 사전 동의 없이 오라클 바이너리를 함부로 패치하면 안 됨

## 2.7 JVM 모니터링과 툴링

JVM은 실행 중인 애플리케이션을 instrumentation(오류 진단이나 성능 개선을 위해 애플리케이션에 특정한 코드를 끼워 넣는 것), 모니터링, 관측하는 다양한 기술을 제공

- JMX(Java Management Extensions)
- Java agent
- JVMTI(JVM Tool Interface)
- SA(Serviceabillity Agent)

**JMX**는 JVM과 그 위에서 동작하는 애플리케이션을 제어하고 모니터링하는 강력한 범용 툴

메서드를 호출하고 매개변수를 바꿀 수 있음



**자바 에이전트**는 메서드 바이트코드 조작

### 2.7.1 VisualVM

JVM 어태치 매커니즘을 이용해 실행 프로세스를 실시간 모니터링

## 참조

1. Optimizing Java(자바 최적화)

