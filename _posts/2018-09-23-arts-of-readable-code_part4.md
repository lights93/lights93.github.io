---
title: "읽기 좋은 코드가 좋은 코드다 part4"
date: 2018-09-23
excerpt: "읽기 좋은 코드가 좋은 코드다 part4"
categories:
  - CodeReadability
tags:
  - artsOfReadableCode
---

# 선택된 주제들

## 14. 테스트와 가독성

* #### 각 테스트의 최상위 수준은 최대한 간결해야 한다. 이상적으로는 각 테스트의 입출력이 한 줄의 코드로 설명될 수 있어야 한다.

* #### 테스트에 실패하면, 버그를 추적해서 수정하는 데 도움이 될 만한 에러 메시지를 출력해야 한다.

* #### 코드의 구석구석을 철저하게 실행하는 가장 간단한 입력을 사용하라.

* #### 무엇이 테스트되는지 분명하게 드러나도록 테스트 함수에 충분한 설명이 포함된 이름을 부여하라.
	* TEST1() 대신, TEST_함수이름_상황 과 같은 이름을 사용하라.

## 15. 분/시간 카운터를 설계하고 구현하기