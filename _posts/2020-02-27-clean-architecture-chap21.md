---
title: "소리나는 아키텍처"
date: 2020-02-27
excerpt: "클린 아키텍처"
categories:
  - Architecture
tags:
  - Architecture
---

# 21. 소리나는 아키텍처

## 아키텍처의 테마

소프트웨어 아티텍처는 시스템의 유스케이스를 지원하는 구조

아키텍처를 프레임워크 중심으로 만들어버리면 유스케이스가 중심이 되는 아키텍처는 절대 나올 수 없다.

## 아키텍처의 목적

좋은 아키텍처는 유스케이스를 그 중심에 두기 때문에, 지원하는 구조를 아무런 문제없이 기술할 수 있다.

좋은 소프트웨어 아키텍처는 프레임워크, 데이터베이스, 웹 서버, 그리고 여타 개발 환경 문제나 도구에 대해서는 결정을 미룰 수 있도록 만든다.

좋은 아키텍처는 유스케이스에 중점을 두며, 지엽적인 관심사에 대한 결합은 분리시킨다.

## 하지만 웹은?

웹은 아키텍처일까?? 당연히 아니다!

## 프레임워크는 도구일 뿐, 삶의 방식은 아니다.

프레임워크가 아키텍처의 중심을 차지하는 일을 막을 수 있는 전략을 개발하라.

## 테스트하기 어려운 아키텍처

프레임워크를 전혀 준비하지 않더라도, 필요한 유스케이스 전부에 대해 단위 테스트를 할 수 있어야 한다.

## 결론

아키텍처는 시스템을 이야기해야 하며, 시스템에 적용한 프레임워크에 대해 이야기해서는 안 된다.

## 참조

1. 클린 아키텍처(Clean Architecture)

