---
title: "JVM의 코드 실행"
date: 2021-07-26
excerpt: "자바 최적화"
categories:
  - java
tags:
  - optimization
  - java
---

# 9. JVM의 코드 실행

## 9.1 바이트코드 해석

JVM은 다음 세 공간에 주로 데이터를 담아 놓음

- 평가 스택: 메서드별로 하나씩 생성
- 로컬 변수: 결과를 임시 저장
- 객체 힙: 메서드끼리, 스레드끼리 공유됨

### 9.1.1 JVM 바이트코드 개용

JVM에서 각 스택 머신 작업 코드(옵코드)는 1바이트로 나타냄

옵코드는 0부터 255까지 지정 가능하며, 그중에서 약 200개 사용 중

바이트코드 명령어는 스택 상단에 위치한 두 값의 기본형을 구분할 수 있게 표기

> 바이트코드 명령어는 대부분, 한쪽은 각 기본형, 다른 한쪽은 참조형으로 쓸 수 있게 '패밀리(군, 집, 단)' 단위로 구성됨

   

자바는 처음부터 이식성을 염두에 두고 설계된 언어

JVM은 빅 엔디언, 리틀 엔디언 하드웨어 아키텍처 모두 바이트코드 변경없이 실행 가능하도록 명세에 규정됨

load같은 옵코드 군에는 단축형이 있어서 인수를 생략할 수 있고 그만큼 클래스 파일의 인수 바이트 공간을 절약 가능

바이트코드는 개념적으로는 아주 단순하지만, 바이트코드로 나타낼 수 있는 기본 작업보다 훨씬 많은 옵코드가 할당되어 있다.

- 로드/스토어 카테고리: 상수 풀에서 데이터를 로드하거나 스택 상단을 힙에 있는 객체 필드에 저정하는 등의 작업을 함
- 산술 카테고리: 기본형에만 적용되며 순수하게 스택 기반으로 연산을 수행
- 흐름 제어 카테고리: 소스코드의 순회, 분기문
- 메서드 호출 카테고리: 자바 프로그램에서 새 메서드로 제어 권을 넘기는 유일한 장치
- 플랫폼 카테고리: 객체별로 힙 저장 공간을 새로 할당하거나, 고유 락(동기화시 사용하는 모니터)을 다루는 명령어

   

바이트 코드는 구현 복잡도에 따라 대단위(coarse-grained) 바이트코드와 소단위(fine-grained) 바이크코드로 구분

산술 연산은 매우 소단위 작업이고 핫스팟에서 순수 어셈블리어로 구현되는 반면, 대단위 연산은 핫스팟 VM을 다시 호출할 수 밖에 없음   



세이프포인트란 JVM이 어떤 관리 작업을 수행하고 내부 상태를 일관되게 유지하는 데 필요한 지점

일관된 상태를 유지하려면 JVM이 관리 작업 수행 도중 공유 힙이 변경되지 않게 모든 애플리케이션 스레드를 멈추어야 함   



JVM 애플리케이션 스레드 하나하나가 진짜 OS 스레드

인터프리티드 메서드를 실행하는 스레드에 대해 옵코드가 디스패치되는 시점에서 애플리케이션 스레드가 실행하는 것은 유저 코드가 아니라 JVM 인터프리터 코드

따라서 힙 상태 일관성이 보장되고 애플리케이션 스레드를 멈출 수 있음

따라서 '바이트코드 사이사이'가 애플리케이션 스레드를 멈추기에 이상적인 시점이자, 가장 단순한 세이프포인트

JIT 컴파일드 메서드는 기본적으로 JIT 컴파일러가 생성한 기계어 안에 이와 동등한 배리어를 끼워넣음

### 9.1.2 단순 인터프리터

가장 단순한 인터프리터는 switch문이 포함된 while 루프 형태

메서드에서 한번에 한 바이트코드씩 읽어들여 옵코드별로 분기하는 코드

### 9.1.3 핫스팟에 특정한 내용

핫스팟은 단순한 VM 작업을 구현하고 네이티브 플랫폼의 스택 프레임 레이아웃을 최대한 활용하여 성능을 조금이라도 높이기 위해 상당히 많ㅇ느 어셈블리어 코드로 작성돼 있다.

이러한 설계 방식은 다양한 특이 사례를 다루는 데 도움이 됨

## 9.2 AOT와 JIT 컴파일

### 9.2.1 AOT 컴파일

사람이 읽을 수 있는 프로그램 소스 코드를 외부 프로그램(컴파일러)에 넣고 바로 실행 가능한 기계어를 뽑아내는 과정

AOT의 목표는 프로그램을 실행할 플랫폼과 프로세스 아키텍처에 딱 맞은 실행 코드를 얻는 것

하지만 대부분의 실행 코드는 자신이 어떤 플랫폼에서 실행될지 모르는 상태에서 생성되므로 AOT 컴파일은 자신이 사용 가능한 프로세서 기능에 대하 가장 보수적인 선택을 해야 함

결국, AOT 컴파일한 바이너리는 CPU 기능을 최대한 활용하지 못하는 경우가 다반사

### 9.2.2 JIT 컴파일

런타임에 프로그램을 고도로 최적화한 기계어로 변혼하는 기법

프로그램의 런타임 실행 정보를 수집해서 어느 부분이 자주 쓰이고, 어느 부분을 최적화해야 가장 효과가 좋은지 프로파일을 만들어 결정을 내리는 것

JIT 서브시스템은 실행 프로그램과 VM 리소스를 공유하므로 프로파일링 및 최적화 비용 및 성능 향상 기대치 사이의 균형을 맞추어야 함   



바이트코드를 네이티브 코드로 컴파일하는 비용은 런타임에 지불됨

이 과정에서 프로그램 실행헤만 온전히 동원됐을 일부 리소스가 소비되므로 JIT컴파일은 산발적으로 수행됨

또 VM은 최적화하면 가장 좋은 지점을 파악하기 위해 각 종 프로그램 관련 지표를 수집

프로파일링 서브시스템은 현재 어느 메서드가 실행 중인지 항시 추적

컴파일하기 적정한 한계치를 넘어선 메서드가 발견되면 에미터 서브시스템이 컴파일 스레드를 가동해 바이트코드를 기계어로 변환

애플리케이션을 실행할 때마다 성능이 심한 편차를 보이는 현상이 흔함(미리 컴파일하는 것이 성능이 안 좋은 경우가 많음)

그래서 핫스팟은 프로파일링 정보를 보관하지 않고 VM이 꺼지면 일체 폐기

### 9.2.3 AOT 컴파일 vs JIT 컴파일

AOT 컴파일을 상대적으로 이해하기 쉬움

AOT는 최적화 결정을 내리는 데 유용한 런타임 정보를 포기하는 만큼 장점이 상쇄됨

AOT 컴파일 도중 프로세서의 특정 기능을 타깃으로 정하면 해당 프로세서에만 사용 가능함 -> 확장성이 떨어짐   



핫스팟은 새로 릴리즈를 할 때마다 새로은 프로세서 기능에 관한 최적화 코드를 추가할 수 있고, 기존 클래스 및 JAR 파일을 다시 컴파일하지 않아도 신기능 활용 가능   



자바 프로그램도 AOT 컴파일 가능

## 9.3 핫스팟 JIT 기초

핫스팟의 기본 컴파일 단위는 전체 메서드

한 메서드에 해당하는 바이크코드는 한꺼번에 네이티브 코드로 컴파일됨

핫스팟은 핫 루프를 **온-스택 치환(on-stack-replacement, OSR)**이라는 기법을 이용해 컴파일하는 기능도 지원

OSR은 어떤 메서드가 컴파일할 만큼 자주 호출되지는 않지만, 컴파일하기 적합한 루프가 포함돼 있고 루프 바디 자체가 메서드인 경우 사용

### 9.3.1 klass 워드, vtable, 포인터 스위즐링

한마디로 핫스팟은, 멀티스레드 C++ 애플리케이션

그래서 실행 중인 모든 자바 프로그램은 OS 관점에서는 실제로 한 멀티스레드 애플리케이션의 일부

JIT 컴파일 서비시스템을 구성하는 스레드는 핫스팟 내부에서 가장 중요한 스레드들

컴파일 대상 메서드를 찾아내는 프로파일링 스레드와 실제 기계어를 생성하는 컴파일러 스레드도 다 여기에 포함됨

컴파일 대상으로 낙점된 메서드는 컴파일러 스레드에 올려놓고 백그라운드에서 컴파일함



최적화된 기계어가 생성되면 해당 klass와 vtable은 새로 컴파일된 코드를 가리키도록 수정됨

> vtable 포인터를 업데이트 하는 작업을 **포인터 스위즐링(pointer swizzling)**

### 9.3.2 JIT 컴파일 로깅

```
-XX:+PrintCompilation
```

이 스위치를 켜면 컴파일 이벤트 로그가 표준 출력 스트림에 생성되므로 성능 엔지니어는 이 로그를 보고 어떤 메서드가 컴파일되고 있는지 파악 가능

PrintComplilation 출력 결과 형식은 비교적 단순함

메서드가 컴파일 된 시간(VM 시작 이후 ms)이 제일 먼저 나오고,

그 다음에 이번 차례에 컴파일된 메서드의 순번이 표시됨

그 밖의 필드

- n: 네이티브 메서드
- s: 동기화 메서드
- !: 예외 핸들러를 지닌 메서드
- %: OSR을 통해 컴파일된 메서드

자세한 정보를 보기 위해서는 아래 스위치도 필요함

```
-XX:+LogCompliation
-XX:+UnlockDiagnosticVMOptions
```

VM이 바이트코드를 네이티브 코드로 어떻게 최적화했는지, 큐잉은 어떻게 처리했는지 관련 정보를 XML 태그 형태로 담은 로그파일로 출력

### 9.3.3 핫스팟 내부의 컴파일러

핫스팟 JVM에는 C1,C2라는 두 JIT 컴파일러가 있다.

C1,C2 컴파일러 모두 핵심 측정값, 즉 메서드 호출 횟수에 따라 컴파일이 트리거링됨   



컴파일 프로세스는 가장 먼저 메서드의 내부 표현형을 생성한 다음, 인터프리티드 단계에서 수집한 프로파일링 정보를 바탕으로 최적화 로직을 적용

C1은 C2보다 컴파일 시간이 더 짧고 단순하게 설계된 까닭에 C2처럼 풀 최적화는 안 됨

변수를 일체 재할당하지 않는 코드로 변환하는 **단일 정적 할당(single static assignment, SSA)**는 두 컴파일러에서 모두 사용하는 공통 기법

### 9.3.4 핫스팟의 단계별 컴파일(tiered compilation)

인터프리터 모드로 실행되다가 단순한 C1 컴파일 형식으로 바뀌고, 다시 이를 C2가 보다 고급 최적화를 수행하는 방식으로 단계를 바꿈

VM 내부에는 5개의 실행 레벨이 존재

- 레벨 0: 인터프리터
- 레벨 1: C1 - 풀 최적화(프로파일링 없음)
- 레벨 2: C1 - 호출 카운터(invocation counter) + 백엣지 카운터(backedge counter)
- 레벨 3: C1 - 풀 프로파일링
- 레벨 4: C2

이 모든 레벨을 다 거치는 것은 아니고, 컴파일 방식마다 경로가 다름

## 9.4 코드 캐시

JIT 컴파일드 코드는 **코드 캐시**라는 메모리 영역에 저장됨

이곳에는 인터프리터 부속 등 VM 자체 네이티브 코드가 함께 들어가 있음



VM 시작 시 코드 캐시는 설정된 값으로 최대 크기가 고정되므로 확장이 불가능

코드 캐시가 꽉 차면 그때부터 더 이상 JIT 컴파일은 안 되며, 컴파일되지 않은 코드는 인터프리테어만 실행됨

결국 최대로 낼 수 있는 성능에 한참 못 미치는 상태로 작동   



코드 캐시는 미할당 영역과 프리 블록 연결 리스트를 담은 힙으로 구현됨

네이티브 코드가 제거될 때마다 해당 블록이 프리 리스트에 추가됨

블록 재활용은 **스워퍼(sweeper, 청소기)**라는 프로세스가 담당   



네이티브 메서드가 새로 저장되면 컴파일드 코드를 담기에 크기가 충분한 블록을 프리 리스트에서 찾아봄

만약 그런 블록이 없으면 여유 공간이 충분한 코드 캐시 사정에 따라 미할당 공간에서 새 블록을 생성   



네이티브 코드가 코드 캐시에서 제거

- (추측성 최적화를 적용한 결과 틀린 것으로 판명되어) 역최적화(deoptimization)될 때
- 다른 컴파일 버전으로 교체됐을 때(단계별 컴파일)
- 메서드를 지닌 클래스가 언로딩될 때

   

코드 캐시의 최대 크기는 다음 VM 스위치로 조정

```
-XX:ReservedCodeCacheSize=<n>
```

### 9.4.1 단편화

압착을 안 하면 코드 캐시는 단편화되고 컴파일은 중단됨

## 9.5 간단한 JIT 튜닝법

JIT 튜닝의 대원칙: '컴파일을 원하는 메서드에게 아낌없이 리소스를 베풀라'

이런 목표를 달성하기 위한 점검 사항

1. PrintCompilation 스위치를 켜도 애플리케이션을 실행
2. 어느 메서드가 컴파일됐는지 기록된 로그 수집
3. ReservedCodeCacheSize를 통해 코드 캐시를 늘림
4. 애플리케이션 재실행
5. 확장된 캐시에서 컴파일드 메서드 살펴봄

   

JIT 컴파일에 내재된 불확정성을 고려해야 함

- 캐시 크기를 늘리면 컴파일드 메서드 규모가 유의미한 방향으로 커지는가?
- 주요 트랜잭션 경로상에 위치한 주요 메서드가 모두 컴파일되고 있는가?

코드 캐시 공간이 모자라는 일이 없게 함으로써 JIT 컴파일이 절대 끊기지 않도록 보장하는 전략

## 참조

1. Optimizing Java(자바 최적화)

