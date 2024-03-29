---
title: "가비지 수집 고급"
date: 2021-07-20
excerpt: "자바 최적화"
categories:
  - java
tags:
  - optimization
  - java
---

# 7. 가비지 수집 고급

## 7.1 트레이드오프와 탈착형 수집기

개발자는 가비지 수집기 선정 시 다음 항목을 충분히 고민해야 함

- 중단 시간(중단 길이 또는 기간)
- 처리율(애플리케이션 런타임 대비 GC 시간 %)
- 중단 빈도(수집기 때문에 애플리케이션이 얼마나 자주 멈추는가?)
- 회수 효율(GC 사이클 당 얼마나 많은 가비지가 수집되는가?)
- 중단 일관성(중단 시간이 고른 편인가?)

## 7.2 동시 GC 이론

범용 가비지 수집기는 중단 결정을 효과적으로 내리는 데 참고할 만한 도메인 지식이 전혀 X

메모리 할당은 불확정성을 유발하는 직접적인 원인으로, 실제로도 많은 자바 응용 시스템에서 들쑥날쑥한 양상을 보임

### 7.2.1 JVM 세이프포인트

JVM은 완전히 선제적(fully prememptive)인 멀티스레드 환경이 아니다

OS는 언제든지 선제 개입(코어에서 스레드를 제거) 가능

한 스레드가 자신에게 할당된 타임 슬라이스를 다 쓰거나, 스스로 wait() 상태로 잠들 때 그렇게 됨

JVM도 애플리케이션 스레드마다 세이프포인트(safepoint)라는 특별한 실행 지점을 둔다.

세이프포인트는 스레드의 내부 자료 구조가 훤히 보이는 지점으로, 여기서 어떤 작업을 하기 위해 스레드는 잠시 중단됨   



풀 STW 가비지 수집기의 경우, 이 수집기가 작동하려면 안정된 객체 그래프가 필요함 -> 전채 애플리케이션 스레드 중단

JVM은 다음 두 가지 규칙에 따라 세이프포인트 처리

- JVM은 강제로 스레드를 세이프포인트 상태로 바꿀 수 없다
- JVM은 스레드가 세이프포인트 상태에서 벗어나지 못하게 할 수 있다

세이프포인트 요청을 받았을 때 그 지점에서 스레드가 제어권을 반납하게 만다는 코드가 VM 인터프리터 구현체 어딘가에 있어야 함

세이프포인트 상태로 바뀌는 몇가지 일반적인 경우

1. JVM이 전역 '세이프포인트 시간' 플래그를 세팅
2. 각 애플리케이션 스레드는 폴링하면서 이 플래그가 세팅됐는지 확인
3. 애플리케이션 스레드는 일단 멈췄다가 다시 깨어날 때가지 대기

**세이프포인트 시간(time to safepoint)** 플래그를 세팅하면 모든 애플리케이션 스레드는 반드시 멈춰야 함



일반 애플리케이션 스레드는 인터프리터에서 바이트코드 2개를 실행할 때마다 폴링

컴파일드 코드에서는, 보통 컴파일드 메서드 밖으로 나가거나 분기가 회귀하는 지점에 JIT컴파일러가 세이프포인트 폴링 코드 삽입   



다음 각 경우에 스레드는 자동으로 세이프포인트 상태가 됨

- 모니터에서 차단
- JNI 코드를 실행

다음 경우에는 스레드가 꼭 세이프포인트 상태가 되는 건 아님

- 바이트코드를 실행하는 동중이다
- OS가 인터럽트를 걸었다

### 7.2.2 삼색 마킹

- GC 루트를 흰색 표시
- 다른 객체는 모두 흰색 표시
- 마킹 스레드가 회색노드로 랜덤하게 이동
- 이동한 노드를 검은색 표시하고 이 노드가 가리키는 모든 흰색 노드를 회색 표시
- 회색 노드가 하나도 남지 않을 때까지 위 과정을 되풀이
- 검은색 객체는 모두 접근 가능한(reachable)한 것이므로 살아남은
- 흰색 노드는 더 이상 접근 불가한 객체이므로 수집 대상

   

동시 수집은 **SATB(snapshop at the beginning)**라는 기법 활용

즉, 수집 사이클을 시작할 때 접근 가능하거나 그 이후에 할당된 객체를 라이브 객체로 간주

그래서 삼색 표시 알고리즘은 사소하지만 몇 가지 단점 존재

변경자 스레드(mutator thread, 힙에 위치한 객체를 변경하는 프로그램)가 수집을 하는 도중에는 검은색 상태, 수집을 안 하는 동안에는 흰색 상태로 새 객체 생성   



삼색 마킹 알고리즘에서 실행 중인 애플리케이션 스레드가 변경한 것 떄문에 라이브 객체가 수집되는 현상을 방지하려면 몇 가지 로직이 더 추가돼야 함

동시 수집기에서는 마킹 스레드가 삼색 알고리즘을 실행하는 도중에도 변경자 스레드가 계속 객체 그래프를 변경하기 때문



객체 색깔을 검은색 -> 회색으로 바꾸고 변경자 스레드가 업데이트하며 처리할 노드 세트에 도로 추가하여 해결 가능

삼색 불변의 원칙을 위배할지 모를 모든 변경 사항을 큐 형태로 넣어두고, 주 단계가 끝난 다음 부차적인 조정 단계에서 바로잡는 방법도 있음

### 7.3 CMS

중단 시간을 아주 짧게 하려고 설계된, 테뉴어드(올드) 공간 전용 수집기

CMS는 가비지 수집의 두 번째 원칙(아직 살아 있는 객체를 수집하면 안 된다)를 위반하지 않도록 반드시 레코드를 바로잡아야 함

1. 초가 마킹(Initial Mark) (STW)
2. 동시 마킹(Concurrent Mark)
3. 동시 사전 정리(Concurrent Preclean)
4. 재마킹(Remark) (STW)
5. 동시 스위프(Concurrent Sweep)
6. 동시 리셋(Concureent Reset)

전체적으로 한 차례 긴 STW 중단을 일반적으로 매우 짧은 두 차례 STW 중단으로 대체     



초기 마킹 단계의 목적은, 해당 영역 내부에 위치한 확실한 GC 출발점(**내부 포인터**라고 하며 수집 사이클 목적상 GC 루트와 동일함)을 얻는 것

동시 마킹 단계에서는 삼색 마킹 알고리즘을 힙에 적용하면서 나중에 조정해야 할지 모를 변경 사항을 추적

동시 사전 정리 단계의 목표는 재마킹 단계에서 가능한 STW 시간을 줄이는 것

재마킹 단계는 카드 테이블을 이용해 변경자 스레드가 동시 마킹 단계 도중 영향을 끼친 마킹을 조정



CMS를 적용했을 때 효과

1. 애플리케이션 스레드가 오랫동안 멈추지 않는다
2. 단일 풀 GC 사이클 시간이 더 길다
3. CMS GC 사이클이 실행되는 동안, 애플리케이션 처리율은 감소
4. GC가 객체를 추적해야 하므로 메모리를 더 많이 쓴다
5. GC 수행에 훨씬 더 많은 CPU 시간이 필요
6. CMS는 힙을 압착하지 않으므로 테뉴어드 영역은 단편화 가능

### 7.3.1 CMS 작동 원리

CMS는 대부분 애플리케이션 스레드와 동시에 작동

기본적으로 가용 스레드 절반을 동원해 GC 동시 단계를 수행하고, 나머지 절반은 애플리케이션 스레드가 자바 코드를 실행하는 떼 씀



만약 CMS 실행 도중 에덴 공간이 꽉 차버리면??

애플리케이션 스레드가 더 이상 진행할 수 없으니 당연히 실행이 중단되고 CMS 도중 영 GC가 일어남

그런데 이 영 GC는 코어 절반만 사용하므로 병렬 수집기의 영 GC보다 오래 걸림

영 수집이 끝나고 일부 객체는 테뉴어드로 승격됨 -> 조정이 필요 -> CMS는 조금 다른 영 수집기 사용



할당률 급증 -> 조기 승격 -> 테뉴어드 공간조차 부족한 사태 : **동시 모드 실패(concurrent mode failure, CMF)**

CMF자 자주 일어나지 않게 하려면 테뉴어드 꽉 차기 전에 CMS가 수집 사이클을 개시해야 함. 디폴트는 75%   



힙 단편화는 CMF를 유발하는 또 다른 원인

CMS는 테뉴어드를 압착하지 않음

테뉴어드의 빈 공간은 단일 연속 블록이 아니기 때문에 CMS가 작업을 마친 후 승격된 객체를 기존에 들어찬 객체 사이사이로 밀어 넣어야 함   



유일한 해결책은 (압착 수집기인) ParallelOld GC로 풀 수집해서 객체를 승격시킬 만한 연속 공간을 충분히 확보하는 방법 뿐



이렇게 영 수집이 CMS보다 더 빠르거나 힙 단편화가 발생하면 풀 STW ParallelOld GC 수집으로 회귀할 수 밖에 없음

CMF를 방지하기 위해 CMS를 적용한 저지연 애플리케이션에서는 사실상 튜닝 자체가 주요 이슈   



CMS는 내부적으로 프리 리스트를 이용해 사용 가능한 빈 공간을 관리

동시 스위프 단계에서 스위퍼 스레드가 여유 공간을 더 큰 덩어리로 만들고 단편화로 인해 CMF가 발생하지 않도록 연속된 빈 블록들을 하나로 뭉침

하지만 스위퍼는 변경자와 동시에 작동하므로 스레드가 서로 적절히 동기화되지 않는 한 새로 할당된 블록이 잘못 스위프될 가능성이 있다

이런 일이 없게끔 스위퍼 스레드는 작업 도중 프리 리스트를 잠금

### 7.3.2 CMS 기본 JVM 플래그

CMS 수집기는 다음 플래그로 작동

```
-XX:+UseConcmarkSweepGC
```

최신 핫스팟 버전에서 이 플래그를 쓰면 ParNew GC도 함께 작동함

## 7.4 G1

- CMS보다 훨씬 튜닝하기 쉬움
- 조기 승격에 덜 취약
- 대용량 힙에서 확장성이 우수
- 풀 STW 수집을 없앨 수 있다

### 7.4.1 G1 힙 레이아웃 및 영역

G1 힙은 영역으로 구성됨

영역은 디폴트 크기가 1메가바이트인 공간

영역을 이용하면 세대를 불연속적으로 배치할 수 있고, 수집기가 매번 실행될 때마다 전체 가비지를 수집할 필요가 없다

> 물론 전체 G1 힙은 메모리상에서 연속돼 있다.

   

G1 알고리즘에서는 1,2,4 ... 64 메가바이트 크기의 영역을 사용 가능

영역 크기 = 힙 크기 / 2048

영역 개수 = 힙 크기 / 영역 크기

### 7.4.2 G1 알고리즘 설계

- 동시 마킹 단계를 이용
- 방출 수집기
- 통계적으로 압착

> 영역을 절반 이상을 점유한 객체는 거대 객체로 간주하여 거대 영역이라는 별도 공관에 곧바로 할당됨. 거대 영역은 테뉴어드 세대에 속한, 연속된 빈 공간

G1에서도 에덴, 서바이버 영역으로 이루어진 영 세대 개념은 같지만, 세대를 구성하는 영역이 연속되어 있지 않다는 차이점이 있다

영 세대의 크기는 전체 중단 시간 목표에 따라 조정됨.  



G1 수집기에서 **기억 세트(remembered set, RSet)**라는 장치로 영역을 추적함

RSet은 영역별로 하나씩, 외부에서 힙 영역 내부를 참조하는 레퍼런스를 관리하기 위한 장치

G1은 영역 내부를 바라보는 레퍼런스를 찾으려고 전체 힙을 다 뒤질 필요 없이 RSet만 꺼내 보면 됨

RSet, 카드 테이블은 모두 **부유 가비지(floating garbage)**라는 GC 문제를 해결하는 데 유용함

부유 가비지는 현재 수집 세트 외부에서 죽은 객체가 참조하는 바람에 이미 죽었어야 할 객체가 계속 살아 있는 현상

### 7.4.3 G1 단계

1. 초기 마킹(STW)
2. 동시 루트 탐색
3. 동시 마킹
4. 재마킹(STW)
5. 정리(STW)

동시 루트 탐색은 초기 마킹 단계의 서바이버 영역에서 올드 세대를 가리키는 레퍼런스를 찾는 동시 단계로, 반드시 다음 영 GC 탐색을 시작하기 전에 끝내야 함

마킹 작업은 재마킹 단계에서 완료됨

레퍼런스를 처리하고 SATB 방식으로 정리하는 작업도 재마킹 단계에서 함

정리 단계는 어카운팅(accounting) 및 RSet 씻기 태스크를 수행하며 대부분 STW를 일으킴

어카운팅은 이제 완전히 자유의 몸이 되어 재사용 준비를 마친 영역을 식별하는 작업

### 7.4.4 G1 기본 JVM 플래그

자바 8 이전까지 다음 스위치로 G1 작동

```
+XX:UseG1GC
```

G1의 주목표는 중단 시간 단축

그래서 가비지 수집이 일어날 때마다 애플리케이션의 최대 중단 시간을 개발자가 지정 가능

하지만, 목표치에 불과하며 실제로 애플리케이션이 이 기준에 맞추리란 보장 X

디폴트 중단 시간 목표를 200밀리초로 설정하는 스위치

```
-XX:MaxGCPauseMillis=200
```

디폴트 영역 크기 변경하는 방법

```
-XX:G1HeapRegionSize=<n>
```

## 7.5 셰난도아

레드햇 진영에서 OpenJDK 프로젝트 일환으로 제작한 자체 수집기

셰난도아 역시 G1처럼 주목표는 중단 시간 단축

1. 초기 마킹(STW)
2. 동시 마킹
3. 최종 마킹(STW)
4. 동시 압착

   

셰난도아의 가장 두드러진 특징은 **브룩스 포인터(Brooks pointer)**

객체당 메모리 워드를 하나 더써서 이전 가비지 수집 단계에서 객체가 재배치됐는지 여부를 표시하고 새 버전 객체 콘텐츠의 위치를 가리킴

재배치되지 않은 객체의 브룩스 포인터는 그냥 메모리 다음 워드를 가리킴

> 브룩스 포인터는 하드웨어 수준에서 지원되는 CAS(compare-and-swap) 기능에 의존하여 포워딩 주소를 아토믹하게 수정

   

동시 마킹 단계에서는 힙을 죽 훑어 살아 있는 객체를 모두 마킹

포워딩 포인터가 있는 oop를 가리키는 객체 레퍼런스가 있으면 새 oop 위치를 직접 참조하도록 레퍼런스를 수정

   

최종 마킹 단계에서는 STW하고 루트 세르를 재탐색한 후, 방출한 사본을 가리키도록 루트를 복사하고 수정

### 7.5.1 동시 압착

1. 객체를 TLAB로 복사
2. CAS로 브룩스 포인터가 추측성 사본을 가리키도록 수정
3. 이 작업이 성공하면 압착 스레드가 승리한 것으로, 이후 이 버전의 객체는 모두 브룩스 포인터를 경유해서 액세스하게 됨
4. 이 작업이 실패하면 압착 스레드가 실패한 것으로, 추측성 사본을 원상복ㄷ구하고 승리한 스레드가 남긴 브룩스 포인터를 따라감

### 셰난도아 얻기

레드햇 페도라 같은 리눅스 배포판에 아이스티 바이너리 일부로 실려 있음

다음 스위치로 셰난도아 작동 가능

```
-XX:+UseShenandoahGC
```

## 7.6 C4(아줄 징)

생략

## 7.7 밸런스드(IBM J9)

생략

## 7.8 레거시 핫스팟 수집기

### 7.8.1 Serial 및 Serial Old

CPU 한 코어만 사용해 GC 수행

풀 STW

최신 멀티코어 시스템에서 절대 사용 X

### 7.8.2 증분 CMS(iCMS)

자바9부터 사라짐

### 7.8.3 디프리케이트되어 사라진 GC 조합

생략

### 7.8.4 엡실론

엡실론 수집기는 레거시 수집기는 아니지만, 어느 운영계 환경에서건 절대 사용 금물인 수집기

엡실론은 테스트 전용으로 설계된, 아무 일도 안 하는 시험 수집기

가비지 수집 활동을 일체 하지 않음   



다음과 같은 작업에 유용

- 테스트 및 마이크로벤치마크 수행
- 회귀 테스트
- 할당률이 낮거나 0인 자바 애플리케이션 또는 라이브러리 코드의 테스트

## 참조

1. Optimizing Java(자바 최적화)

