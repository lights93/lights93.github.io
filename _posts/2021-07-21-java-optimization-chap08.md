---
title: "GC 로깅, 모니터링, 튜닝, 툴"
date: 2021-07-21
excerpt: "자바 최적화"
categories:
  - java
tags:
  - optimization
  - java
---

# 8. GC 로깅, 모니터링, 튜닝, 툴

## 8.1 GC 로깅 개요

GC 로그는 특히, 시스템이 내려간 원인의 단서를 찾는 '콜드 케이스(cold case, 진상이 밝혀지지 않은 범죄사 사고를 가리키는 용어)' 분석을 할 때 매우 유용

다음 두 가시 설정

- GC 로그를 생성
- 애플리케이션 출력과는 별도로 특정 파일에 GC 로그를 보관

GC 로깅은 사실 오버헤드가 거의 없는 것이나 다름없으니 주요 JVM 프로세스는 항상 로깅을 켜놓아야 함

### 8.1.1 GC 로깅 켜기

```
-Xloggc:gc.log -XX:+PrintGCDetails -XX:+PrintTenuringDistribution -XX:+PrintGCTimeStamps -XX:PrintGCDateStamps
```

-Xloggc:gc.log: GC 이벤트에 로깅할 파일을 지정

-XX:+PrintGCDetails: GC 이벤트 세부 정보를 로깅

-XX:+PrintTenuringDistribution: 툴링에 꼭 필요한, 부가적인 GC 이벤트 세부 정보를 추가

-XX:+PrintGCTimeStamps: GC 이벤트 발생 시간을 (VM 시작 이후 경과한 시간을 초 단위로) 출력

-XX:PrintGCDateStamps: GC 이벤트 발생 시간을 (벽시계 시간을 기준으로) 출력

   

주의사항

- 기존 플래그 verbose:gc는 지우고 대신 PrintGCDetails 사용
- PrintTenuringDistribution은 다소 독특한 플래그로, 이 플래그가 제공하는 정보를 사람이 이용하기 어려움. 중요한 메모리압(memory pressure, 메모리 할당 압박) 효과, 조기 승격 등의 이벤트 계산 시 필요한 기초 데이터 제공
- PrintGCDateStamps와 PrintGCTimeStamps는 둘 다 필요함. 전자는 GC 이벤트와 애플리케이션 이벤트를, 후자는 GC와 다른 내부 JVM 이벤트를 각각 연관짓는 용도



로그를 이 정도로 세세히 남겨도 JVM 성능에 이렇다 할 영향 X

필수 플래그 이외에도 로그 순환 관련 플래그

-XX:+UseGCLogFileRotation: 로그 순환 기능을 켠다

-XX:+NumberOfGCLogFiles=<n>:  보관 가능한 최대 로그파일 개수를 설정

-XX:+GCLogFileSize=<size>: 순환 직전 각 파일의 최대 크기를 설정

### 8.1.2 GC 로그 vs JMX

**JMX(Java Management eXtensions)**가 GC에 영향을 주기 때문에 성능 엔지니어는 다음 사항을 숙지해야 함

- GC 로그 데이터는 실제로 가비지 수집 이벤트가 발생해서 쌓이지만, JMX는 데이터를 샘플링하여 얻음
- GC 로그 데이터는 캡처 영향도가 거의 없지만, JMX는 프록시 및 원격 메서드 호출(Remote Method Invocation, RMI) 과정에서도 암묵적인 비용 발생
- GC 로그 데이터에는 자바 메모리 관리에 연관된 성능 데이터가 50가지 이상 있지만, JMX는 10가지도 안 됨

JMX는 성능 데이터 원천으로서 스트리밍된 데이터를 즉시 제공한다는 점에서는 GC로그보다 낫지만, 요즘은 jClarity 센섬같은 툴도 GC 로그 데이터를 스트리밍하는 API를 제공하므로 별반 차이가 없다.

> 기본적인 힙 사용 실태를 파악하는 용도로는 JMX가 제격이지만, 더 깊이 있는 진단을 하려고 하면 금세 부족함을 느끼게 됨

### 8.1.3 JMX의 단점

JMX를 이용해 애플리케이션을 모니터링하는 클라이언트는 대부분 런타임을 샘플링하여 현재 상태를 업데이트 받음

문제는 가비지 수집

각 수집 사이클 전후의 메모리 상태 역시 알 수가 없으므로 GC 데이터를 깊이 있게, 정확하게 분석할 수 없다

JMX는 장기적 추이를 파악하는 정도로 쓸 수 밖에 없다

특히, 각 수집 전후의 힙 상태 정보가 대단히 중요

또 메모리압(할당률)을 분석하는 활동이 매우 중요한데, JMX는 데이터를 수집하는 방식 때문에 이마저도 불가능

RMI 기반 통신 채널의 고질적인 문제점에도 취약함

접속 해제시 가비지 수집기를 돌려 객체를 회수해야 함

RMI를 사용하는 애플리케이션은 기본 1시간에 한번씩 풀 GC 발생

### 8.1.4 GC 로그 데이터의 장점

전체 구성 컴포넌트가 서로 맞물려 작동하면서 최종적 동작, 성능이 귀결되는 소프트웨어를 **발현적(emergent)**이라고 함

> GC 로그는 핫스팟 JVM 내부에서 논블로킹(non-blocking) 쓰기 메커니즘 사용 -> 성능에 미치는 영향은 거의 0

GC 로그에 쌓인 기초 데이터는 특정 GC 이벤트와 연관 지을 수 있어서 모든 의미 있는 분석 작업을 수행 가능

## 8.2 로그 파싱 툴

### 8.2.1 센섬

센섬은 최고의 GC 로그 파싱, 정보 추출, 자동 분석 기능을 제공하는 것이 목표

### 8.2.2 GCViewer

GC 로그 파싱 및 그래프 출력 등 기본 기능을 갖춘 데스크톱 툴

오픈소스라서 무료

### 8.2.3 같은 데이터를 여러 가지 형태로 시각화하기

## 8.3 GC 기본 튜닝

"GC는 언제 튜닝해야 할까?"

1. GC가 성능 문제를 일으키는 근원이라고 확인하거나 그렇지 않다고 배제하는 행위는 저렴
2. UAT에서 GC 플래그를 켜는 것도 저렴한 행위
3. 메모리 프로파일러, 실행 프로파일러를 설정하는 작업은 결코 저렴하지 않음

엔지이너는 튜닝을 수행하면서 다음 네 가지 주요 인자를 면밀히 관찰/측정해야 함

- 할당
- 중단 민감도
- 처리율 추이
- 객체 수명

   

힙 크기 조정 플래그

-Xms<size>: 힙 메모리의 최소 크기를 설정

-Xmx<size>: 힙 메모리의 최대 크기를 설정

-XX:MaxPermSize=<size>: 펌젠 메모리의 최대 크기를 설정 (자바 7 이전)

-XX:MaxMetaspaceSize=<size>: 메타스페이스 메모리의 최대 크기를 설정 (자바 8 이후)



튜닝 시 GC 플래그는 다음과 같이 추가

- 한번에 한 플래그씩 추가
- 각 플래그가 무슨 작용을 하는지 숙지해야 함
- 부수 효과를 일으키는 플래그 조합도 있음을 명심



성능 문제를 일으키는 원인이 GC인지 아닌지 판단하기

- CPU 사용률이 100%에 가까운가?
- 대부분의 사긴이 유저 공간에서 소비되는가?
- GC 로그가 쌓이고 있다면 현재 GC가 실행 중이라는 증거

세 가지 조건이 다 맞는다면 GC가 성능 이슈를 일으키고 있을 가능성이 크고 철저한 조사와 튜닝이 필요함

### 8.3.1 할당이란?

할당률 분석은 튜닝 방법뿐만 아니라, 실제로 가비지 수집기를 튜니앟면 성능이 개선될지 여부를 판단하는 데 꼭 필요한 과정



초기 할당 전략은 다음 네 가지 단순 영역에 집중

- 굳이 없어도 그만인, 사소한 객체 할당(예. 로그 디버깅 메시지)
- 박싱 비용
- 도메인 객체
- 엄청나게 많은 논JDK 프레임워크 객체



드물지만 도메인 객체가 메모리를 많이 차지하는 일도 있다.

덩치 큰 배열만 곧바로 테뉴어드에 할당될 가능성이 큼

핫스팟은 TLAB 및 큰 객체의 조기 승격에 관한 튜닝 플래그 제공

```
-XX:PretenureSizeThreshold=<n>
-XX:MinTLABSize=<n>
```

각 스위치가 어떤 영향을 미치는지 제대로 벤치마킹도 안 하고 확실한 근거 없이 막연히 사용하면 안 됨

   

할댱률은 테뉴어드로 승격되는 객체 수에 영향을 끼침

단명 자바 객체의 수명이 불변이라고 가정하면 할당률이 높을수록 영 GC 발생 주기는 짧아짐

너무 자주 수집이 일어나면 단명 객체는 장례를 치를 시간도 없이 테뉴어드로 잘못 승격될 가능성이 큼

즉, 할당이 폭주하면 조기 승격 문제 발생

조기 승격 문제에는 다음 스위치가 요긴하게 쓰임

```
-XX:MaxTenuringThreshold=<n>
```

테뉴어드 영역으로 승격되기 전까지 객체가 통과해야 할 가비지 수집 횟수를 설정

이 값을 바꿀 때는 다음 두 가지 상충되는 관심사를 잘 따져봐야 함

- 한계치가 높을수록 진짜 장수한 객체를 더 많이 복사
- 한계치가 너무 낮으면 단명 객체가 승격되어 테뉴어드에 메모리압을 가중시킴

### 8.3.2 중단 시간이란?

중단 시간 튜닝 시 유용한 휴리스틱

1. \> 1초: 1초 이상 걸려도 괜찮다: 대부분 Parallel
2. 1초 ~ 100밀리초: 대부분 Parallel/G1
3. < 100밀리초: 대부분 CMS

### 8.3.3 수집기 스레드와 GC 루트

GC 루트 탐색 시간은 다음과 같은 요인의 영향을 받음

- 애플리케이션 스레드 개수
- 코드 캐시에 쌓인 컴파일드 코드량
- 힙 크기



ex. 엄청나게 큰 Object[] -> 단일 스레드에서 탐색 -> 오래 걸림 -> 전체 마킹 시간 결정

## 8.4 Parallel GC 튜닝

이 수집기의 목표와 트레이드 오프

- 풀 STW
- GC 처리율이 높고 계산 비용이 싸다
- 부분 수집이 일어날 가능성은 없다
- 중단 시간은 힙 크기에 비례하여 늘어난다

이와 같은 특성들이 별문제가 안 되는 애플리케이션에서는 Parallel GC가 효과적

예전 GC 힙 크기 조정 플래그

-XX:NewRatio=<n>: 영 세대/전체 힙 비율

-XX:SurvivorRatio=<n>: 서바이버 공간/영 세대 비율

-XX:NewSize=<n>: 최소 영 세대 크기

-XX:MaxNewSize=<n>: 최대 영 세대 크기

-XX:MinHeapFreeRatio=<n>: 팽창을 막기 위한 GC 이후 최소 힙 여유 공간 비율

-XX:MaxHeapFreeRatio=<n>: 수축을 막기 위한 GC 이후 최대 힙 여유 공간 비율

   

대부분의 최신 애플리케이션은 사람보다 프로그램이 크기를 알아서 잘 결정하기 때문에 이렇게 명시적으로 크기를 설정하는 일은 삼가는 게 좋음

### 8.5 CMS 튜닝

CMS는 튜닝이 까다롭기로 소문난 수집기

CMS처럼 중단 시간이 짧은 수집기는 정말로 STW 중단 시간을 단축시켜야 하는 유스케이스에 한해 어쩔 수 없을 때만 사용해야 함

CMS 플래그를 바꿀 때 안티패턴의 늪에 빠질 우려가 있음

   

CMS 수집이 일어나면 기본적으로 코어 절반은 GC에 할당되므로 애플리케이션 처리율은 반토막 남

CMS 수집이 끝나자마자 곧바로 새 CMS 수집이 시작되는 **백투백** 수집 현상은 동시 수집기가 얼마 못 가 고장날 거라는 신호

백투백 현상이 일어나면 사실상 전체 애플리케이션 실행 처리율은 50%나 떨어짐   



CMS 수집 중 GC에 할당된 코어 수를 줄이는 방법도 있다

그만큼 수집 수행 CPU 시간이 줄어들고 부하 급증 시 애플리케이션의 회복력이 떨어지는 위험을 감수해야 함

```
-XX:ConGCThreads=<n>
```

   

CMS에서 STW는 두 단계에서 발생

- 초기 마킹: GC 루트가 직접 가리키는 내부 노드를 마킹
- 재마킹: 카드 테이블을 이용해 조정 작업이 필요한 객체를 식별

모든 애플리케이션 스레드는 CMS가 한번 일어날 때마다 반드시 2회 멈추는데, 세이프포인트에 예미한 저지연 애플리케이션에서는 중요한 영향을 미칠 수 있다.

```
-XX:CMSInitiatingOccupancyFraction=<n>
-XX:UseCMSInitiatingOccupancyOnly
```

CMSInitiatingOccupancyFraction(CMS 초기 점유율)는 CMS가 언제 수집을 시작할지 설정하는 플래그

CMS가 실행되면 영 수집을 통해 올드 영역으로 승격되는 객체들을 수용햘 여유 공간이 필요함

여유 공간 역시 JVM 자체 수집한 통계치에 따라 그 크기가 조정되지만, 첫 번째 CMS를 가동시킬 추정치를 미리 정해놓음

기본적으로 75%

UseCMSInitiatingOccupancyOnly를 함께 설정하면 초기 점유 공간을 동적 크기 조정하는 기능이 꺼짐   



할당률이 심하게 튀는 CMS 애플리케이션이라면 여유 공간을 늘리고(매개변수 값 줄임) 능동적 크기 조정 기능을 끄는 전략을 구사

### 8.5.1 단편화로 인한 CMF

```
-XX:PrintFLSStatistics=1
```

위의 JVM 스위치를 추가하면 GC 로그에 몇몇 추가 정보가 표시됨

평균 블록 크기와 최대 청크 크기를 보니 메모리 청크의 크기 분포를 대략 짐작 가능

덩치 큰 라이브 객체를 테뉴어드로 옮기려고 하는데 그만한 크기의 청크가 바닥난 경우 GC 승격이 약하되여 결국 CMF 발생

로그를 파싱하거나 센섬 툴을 써서 CMF에 근접했다는 사실을 자동 감지 가능

## 8.6 G1 튜닝

엔드 유저가 최대 힙 크기와 최대 GC 중단 시간을 간단히 설정하면 나머지는 수집기가 알아서 처리하게 하는 것이 G1 튜닝의 목표

튜닝이 필요한 경우 다음 스위치를 반드시 지정

```
-XX:+UnlockExperimentalVMOptions
```

G1 튜닝에서 가장 큰 문제는, 이 수집기가 처음 등장한 이후로 내부적으로 상당히 많이 변화를 겪었다는 사실

   

G1 수집기는 할당률에 뒤쳐지지 않는 한 계속 조금씩 압착하므로 CMF가 일어날 가능성은 전혀 없다

어떤 애플리케이션에서 할당률이 계속 높은 상태로 대부분 단명 객체가 생성되고 있다면 다음 튜닝을 고려해봄 직함

- 영 세대를 크게 설정
- 테뉴어드 한계치를 최대 15 정도로 늘려 잡음
- 애플리케이션에서 수용 가능한 최장 중단 시간 목표를 정함

이와 같이 에덴 및 서바이버 영역을 구성하면 순수 단명 객체가 승격될 가능성이 현저히 줄어듬

올드 세대압도 낮아지고 올드 영역을 정리할 일도 줄어듬

## 8.7 jHiccup

JVM이 연속적으로 실행되지 목한 지점을 보여주는 계측 도구

히컵을 일으키는 가장 흔한 원인은 GC STW 중단이지만, OS나 플랫폼 관련 문제 때문에 발생하기도 함

jHiccup은 GC 튜닝에도 좋지만 초저지연 작업을 할 때 유용

   

```bash
jHiccupt -p <프로세스 ID>
```

실행 중인 애플리케이션에 jHiccup을 주입하는 명령

## 참조

1. Optimizing Java(자바 최적화)
