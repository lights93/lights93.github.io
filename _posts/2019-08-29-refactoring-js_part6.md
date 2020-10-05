---
layout: post
title: "간단한 구조 리팩토링"
date: 2019-08-29
excerpt: "간단한 구조 리팩토링"
tags: [JavaScript, RefactroingJS]
comments: true

---

# 6. 간단한 구조 리팩토링

## 6.1 코드

## 6.2 확신 전략

## 6.3 이름 바꾸기

의미가 없는 것들의 이름을 변경

좋지 않은 이름을 찾을 때 고려사항

- 철자가 틀린 단어
- 짧은 이름(약어 및 한 문자 이름)
- 비논리적이고 일반적인 이름
- 변수 이름에 있는 숫자
- 중복된 이름
- 생성자를 위한 대문자가 아닌 것들
- 카멜 케이스가 아닌 것들

## 6.4 불필요한 코드

- 죽은 코드
- 추측 코드 및 주석
- 공백
- 아무것도 하지 않는 코드(ex. if(!!booleans))
- 디버깅/로깅

## 6.5 변수

- 매직 넘버

- 긴 코드 줄(변수)

- 인라인 함수 호출

- 변수 도입(소개): 같은 것을 반복하면 변수로 대체

- 변수 호이스팅(함수 호이스팅도 포함)

  #### 함수 호이스팅

  ```javascript
  function myCoolFunction(){}; // 전체 함수가 맨 위로 호이스팅 됨
  var myCoolFunction = function(){} // 함수 이름만 호이스팅 되어서 함수가 실행이 안 됨 (할당 X)
  var add = (x, y) => x + y; // 위와 동일
  ```

## 6.6 문자열

- 문자열 연결, 매직, 템플릿
- 문자열을 처리하는 기본 정규식
- 긴 코드 줄: 문자열

## 참고자료

리팩토링 자바스크립트(Refactoring JavaScript)

https://github.com/gilbutITbook/006963/tree/master/ch05