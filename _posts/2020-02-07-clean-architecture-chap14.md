---
layout: post
title: "컴포넌트 결합"
date: 2020-02-07
excerpt: "클린 아키텍처"
tags: [Architecture]
comments: true
---

# 14. 컴포넌트 결합

## ADP: 의존성 비순환 원칙(Acyclic Dependencies Principle)

**컴포넌트 의존성 그래프에 순환이 있어서는 안 된다.**

많은 개발자가 동인한 소스 파일을 수정하는 환경에서 문제 발생

해결책: 주 단위 빌드, 순환 의존성 제거

#### 주 단위 빌드(Weekly Build)

5일 중 4일 동안 각자 개발/1일 동안 머지

프로젝트가 커질수록 통합이 늦어짐

빌드 주기가 계속 늦어지게 됨

#### 순환 의존성 제거하기



## 참조

1. 클린 아키텍처(Clean Architecture)

