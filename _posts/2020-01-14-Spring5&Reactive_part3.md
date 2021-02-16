---
layout: post
title: "스트림의 새로운 표준 - 리액티브 스트림"
date: 2019-12-19
excerpt: "스트림의 새로운 표준 - 리액티브 스트림"
tags: [Reactive, Spring5]
comments: true

---

# 3. 스트림의 새로운 표준 - 리액티브 스트림

## 모두를 위한 반응성

### API 불일치 문제

과도하게 많은 선택지로 인해 시스템을 지나치게 복잡하게 만들 수 있음

핵심적인 문제는 라이브러리 공급자가 일관된 API를 만들어낼 수 있는 표준화된 방법이 없다는 사실

### 풀 방식과 푸시 방식

#### 일반적인 풀 방식(요소를 하나씩 요청)

서비스에서 데이터베이스로의 요청에 추가 시간이 소요됨

전체 처리시간 대부분이 유휴 상태

리소스가 사용되지 않더라고 추가 네트워크 작업으로 인해 전체 처리 시간이 2배 또는 3배로 늘어남

데이터베이스가 응답을 서비스에 전달하고 서비스가 응답을 처리한 다음, 데이터의 새 부분을 요청하는 동안 아무 일도 하지 않고 대기하므로 비효율적

#### 풀링과 배치처리 결합

배치처리로 요청하여 퍼포먼스 향상

처리 시간은 여전히 비효율적

데이터를 쿼리하는 동안 클라이언트는 대기상태

#### 데이터 한 번 요청 후 비동기적으로 푸시

처리 흐름 동안 서비스가 첫 번째 응답을 기다리고 있을 때 대기상태 한 번 있음

전체 대기 시간이 짧음

데이터베이스는 필요한 수의 원소를 처리한 이후에도 서비스에서 사용하지 않을 항목을 여전히 생성 가능

### 흐름 제어

푸시 모델을 채택하는 가장 중요한 이유는 요청하는 횟수를 최소화해 전체 처리 시간을 최적화하는 것

푸시 모델만 사용하는 것은 기술적 한계가 존재

프로듀서가 컨슈머의 처리 능력을 무시하면 전반적인 시스템 안정성에 영향을 미칠 수 있음

#### 느린 프로듀서와 빠른 컨슈머

순수 푸시 모델은 실제적 요구를 제공할 수 없기 때문에 동적으로 시스템의 처리량을 증가시키는 것이 불가능함

#### 빠른 프로듀서와 느린 컨슈머

프로듀서가 컨슈머가 처리할 수 있는 것보다 훨씬 많은 데이터를 전송할 수 있으며 이로 인해 부하를 받는 컴포넌트에 치명적인 오류가 생길 수 있음

해결책으로 처리되지 않은 원소를 큐에 수집하는 것

적절한 큐가 필요하다

#### 무제한 큐

메시지 전달을 확실하게 할 수 있음

실제 리소스가 무제한일 수는 없으므로 메시지 전달을 계속 수행하면 프로그램의 복원력이 떨어짐

메모리 한도에 도달하면 전체 시스템 손상

#### 크기가 제한된 드롭 큐

메모리 오버플로를 방지하기 위해 큐가 가득 차면  메세지를 무시하는 형태

메시지의 중요성이 낮을 때 일반적으로 사용

#### 크기가 제한된 블로킹 큐

제한에 도달하면 메시지 유입을 차단

일반적으로 블로킹 큐

하지만 시스템의 비동기를 무효화하여 절대 받아들일 수 없는 시나리오

## 리액티브 스트림의 기본 스펙

리액티브 스트림 스펙에는 Publisher, Subscriber, Subscription, Processor의 네 가지 기본 인터페이스가 정의되어 있음

Publisher (Observable과 유사)는 Publisher와 Subscriber를 연결하기 위한 표준화된 진입점

Subscriber(Obserber와 유사)는 Observer의 메소드와 동일한 세 가지 외에 onSubscribe 제공

onSubscribe는 표준화된 방법으로 Subscriber에게 구독의 성공했음을 알림

**Subscription**은 원소 생성을 제어하기 위해 기본적인 사항 제공

cancel()메소드를 통해 구독 취소 가능

request()를 통해 Publisher와 Subscriber 사이의 상호 작용 확장 가능



Subscriber는 request 메소드를 통해 요청하는 Publisher가 보내줘야 하는 데이터 크기를 알려줄 수 있으며, 이를 통해 Publisher에서 유입되는 원소의 개수가 처리할 수 있는 제한을 초과하지 않을 것임을 확신할 수 있다.

순수 푸시모델과는 달리 스펙에는 **배압**을 적절하게 제어할 수 있는 하이브리드 **푸시-풀** 모델이 포함되어 있음

순수 푸시모델과 풀모델도 지원

### 리액티브 스트림 동작해보기

#### Processor 개념 소개

Publisher와 Subscriber의 혼합 형태

Publisher와 Subscriber 사이에 몇가지 단계를 추가하도록 설계됨

어렵

### 리액티브 스트림 기술 호환성 키트(TCK)

동작을 검증하고 반응 라이브러리를 표준화해 서로 호환하는지 확인하는 공통 도구

TCK는 모든 리액티브 스트림 코드를 방어하고 지정된 규칙에 따라 구현

### JDK9

...

## 참조

1. 실전! 스프링5를 활용한 리액티브 프로그래밍