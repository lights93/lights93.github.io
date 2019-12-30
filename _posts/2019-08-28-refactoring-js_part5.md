---
layout: post
title: "기본적인 리팩토링 목표"
date: 2019-08-28
excerpt: "기본적인 리팩토링 목표"
tags: [JavaScript, RefactroingJS]
comments: true
---

# 5. 기본적인 리팩토링 목표

프레임워크가 자바스크립트의 자율성을 완전히 제한해서는 안 됨

기본 기능의 구성 요소 6가지

- 대규모(bulk) (코드 경로와 여러 코드 줄)
- 입력
- 출력 (값을 반환)
- 부가 작용
- this: 암시적 입력
- 비공개

## 5.1 대규모 함수

대규모(bulk)라는 용어는 함수 몸체를 표현할 때 사용 -> 복잡성, 코드 줄

대규모 코드의 문제는 코드를 이해하기 힘들게 만들고 테스트하기 어려움(반대되는 의견도 있음)

## 5.2 입력

- 명시적

  ```javascript
  function add(a, b) {
  	return a + b;
  }
  ```

  위의 함수에서 a와 b는 명시적 입력

- 암시적

  ``` javascript
  var calculator = {
    add: function(a, b) {
      return a + b;
    },
    addTwo: function(a) {
      return this.add(a, this.two);
    }
    two: 2
  };
  ```

  this는 암묵적 매개 변수

  명시적 입력(a)와 암시적 입력(two)

- 비지역적

  ```javascript
  var name = "Max";
  var punctuation = "!";
  function sayHi() {
    return "Hi " + name + punctuation;
  }
  ```

  name과 punctuation 2개의 비지역적 변수

  비지역적 변수가 재정의되어도 함수는 여전히 어디서나 호출될 수 있고, 함수가 이 변수들의 재정의에 통제권이 없다.

  그래서 문제가 생길 여지가 높음

가능한 한 많은 함수가 명시적 입력(함수형 프로그래밍 스타일) -> 암시적 입력(객체지향 스타일) -> 비지역적 입력을 따르는 것을 권장

비지역적 상태를 경계하는 가장 쉬운 방법은 코드를 모듈, 함수, 클래스 형태로 만드는 것

함수를 호출할 때 원하는 모든 유형의 매개변수 전달 가능

형식 매개변수의 수가 호출에 전달된 숫자와 일치할 필요가 없음

```javascript
function add(a, b) {
	return a + b;
}
```

위의 코드에서 add(2, 3, 4)를 호출해도 5를 반환하고, add(2)를 호출하면 NaN을 반환

객체 및 함수도 매개변수로 전달 가능

```javascript
function doesSomething(options) {
  if(options.a) {
    var a = options.a
  }
  ...
}
```

params나 options 처럼 일반적인 이름을 사용하겨 객체 내부에서 필요한 값을 숨긴다면, 함수 본문에서  해당 값의 사용 방법 및 해당 객체의 실마리에서 지침을 제공해야 함(특정이름으로 만드는 것이 좋음)

권장 사항

- 전반적으로 입력 값이 작을수록 대량의 코드를 제어하고 테스트하는 것이 쉬움
- 비지역적 입력이 적을수록 코드를 이해하고 테스트하는 것이 더 쉬움
- 모든 함수에는 암시적 입력 수단으로 this가 있지만, this는 많은 경우 사용되지 않거나 심지어 정의되지 않을 수 있음
- 명시적 입력을 this나 비지역적 입력보다 신뢰할 수 있음

## 5.3 출력

함수에서 반환된 값

#### void 함수

​	return 키워드를 사용하여 명시적으로 반환하지 않는다면 undefined 반환

​	부가 작용을 하는 코드에서도 명시적으로 undefined/null을 반환하거나 실제 값을 반환하는 것이 좋다.

출력은 가능하면 항상 일관되고 단순한 값을 반환하고 값이 null이거나 정의되지 않은 값을 피하는 것을 권장

```javascript
function moreThanThree(number) {
  if (number > 3) {
    return true;
  } else {
    return "No. number" + number;
  }
}
```

이 함수는 bool 또는 문자열을 반환하여 좋지 않다. 간단한 출력 유형을 사용하는 것이 좋다.

## 5.4 부가 작용

부가 작용은 DOM과 데이터베이스를 업데이트하기도 하고 console.log를 발생시키기도 함

부가 작용이 있는 기능은 테스트하기 어렵고, 디자인을 복잡하게 만들기 때문에, 부가 작용의 범위를 제한해야 함

## 5.5 암시적 입력

this는 최상위 범위

### 5.5.1 Strict 모드에서 this

​	Strict 모드에서는 모든 함수에 this가 없음

​	일반적으로 최상위 범위 안에서 하나 또는 소수 변수 범위가 지정되길 원함

새로운 실행 환경을 만드는 방법

1. 리터럴 객체

   ```javascript
   var anObject = {
     number: 5,
     getNumber: function(){ return this.number }
   }
   
   console.log(anObject.getNumber());
   ```

2. Object.create()

  ```javascript
   var anObject =
       Object.create(null, {"number": {value: 5},
       "getNumber": {value: function(){return this.number}}});
   console.log(anObject.getNumber());
   ```

3. 클래스

  ```javascript
  class AnObject{
   constructor(){
     this.number = 5;
     this.getNumber = function(){return this.number}
   }
  }
  anObject = new AnObject;
  console.log(anObject.getNumber());
  ```

this는 call, apply, bind 함수를 사용해서 변경 가능

```javascript
var anObject = {
  number: 5
}
var anotherObject = {
  getNumber: function(){ return this.number }
}
console.log(anotherObject.getNumber.call(anObject));
console.log(anotherObject.getNumber.apply(anObject));
var callForTheFirstObject = anotherObject.getNumber.bind(anObject);
console.log(callForTheFirstObject());
```

함수는 anotherObject에 있지만 bind, call, apply를 사용하여 this를 anOjbect에 명시적으로 할당

```javascript
var anObject = {
  number: 5
}
var anotherObject = {
  getIt: function(){ return this.number },
  setIt: function(value){ this.number = value; return this; }
}
console.log(anotherObject.setIt.call(anObject, 3));
```

setIt 코드는 this를 반환

this를 반환하지 않는다면 어떤 일이 일어났는지 명확히 확인 불가능

## 5.6 비공개

일부는 비공개 함수가 세부 구현이라고 느끼기 때문에 테스트를 요구하지 않음

익명 함수: 만들어졌다가 사라짐 -> 즉시 호출된 함수식(IIFE, Immediately Invoked Function Expressions)

노출식 모듈 패턴(revealing module pattern)

```javascript
var diary = (function(){
  var key = 12345;
  var secrets = 'rosebud';

  function privateUnlock(keyAttempt){
    if(key===keyAttempt){
      console.log('unlocked');
      diary.open = true; //this로 접근하면 호출한 privateTryLock에 접근
    }else{
      console.log('no');
    }
  };

  function privateTryLock(keyAttempt){
    privateUnlock(keyAttempt);
  };

  function privateRead(){
    if(this.open){
      console.log(secrets);
    }else{
      console.log('no');
    }
  };

  return {
    open: false,
    read: privateRead,
    tryLock: privateTryLock
  }

})();

// 다음과 함께 실행
diary.tryLock(12345);
diary.read();
```

diary가 최상위 범위에서 선언되면 이전에 선언된 함수에 대한 모든 것을 알 수 있다.

diary를 호출하는 것이 이상하여 this를 that으로 명시적 전달

```javascript
var diary = (function(){
  var key = 12345;
  var secrets = 'programming is just syntactic sugar for labor';

  function privateUnlock(keyAttempt, that){
    if(key===keyAttempt){
      console.log('unlocked');
      that.open = true;
    }else{
      console.log('no');
    }
  };

  function privateTryLock(keyAttempt){
    privateUnlock(keyAttempt, this);
  };

  function privateRead(){
    if(this.open){
      console.log(secrets);
    }else{
      console.log('no');
    }
  }

  return {
    open: false,
    read: privateRead,
    tryLock: privateTryLock
  }

})();
```

privateUnlock 함수에 대한 2개의 명시적 입력과 1개의 비지역적 입력이 있다는 것이 유일한 차이점

함수를 수정할 때 call, apply, bind 사용

```javascript
var diary = (function(){
  var key = 12345;
  var secrets = 'sitting for 8 hrs/day straight considered harmful';

  function privateUnlock(keyAttempt){
    if(key===keyAttempt){
      console.log('unlocked');
      this.open = true;
    }else{
      console.log('no');
    }
  };

  function privateTryLock(keyAttempt){
    privateUnlock.call(this, keyAttempt);
  };

  function privateRead(){
    if(this.open){
      console.log(secrets);
    }else{
      console.log('no');
    }
  }

  return {
    open: false,
    read: privateRead,
    tryLock: privateTryLock
  };

})();
```

비지역적 매개변수 -> 암시적 매개변수로 변경

클래스로 작성

```javascript
class Diary {
  constructor(){
    this.open = false;
    this._key = 12345;
    this._secrets = 'the average human lives around 1000 months';
  };

  _unlock(keyAttempt){
    if(this._key===keyAttempt){
      console.log('unlocked');
      this.open = true;
    }else{
      console.log('no')
    }
  };
  tryLock(keyAttempt){
    this._unlock(keyAttempt);
  };

  read(){
    if(this.open){
      console.log(this._secrets);
    }else{
      console.log('no');
    }
  }
}
d = new Diary();
d.tryLock(12345);
d.read();
```

private 변수와 _unlock 함수가 노출됨

밑줄 표시는 비공개 표시

모듈화를 해도 진짜 비공개가 아님

### 5.6.1 자바스크립트에도 비공개가 있을까?

자바스크립트에 비공개는 없음

실제적으로 자바스크립트에는 비공개 방식을 취하는 두 가지 선택 가능한 규칙

1. 비공개를 포기하고, 특성을 다른 'this'에 붙이며 익명 함수를 감싸는 방식
2. 'this'를 내보내는 모듈의 전역 'this'에 연결하는 방법

- 클래스는 생성자 함수를 위한 간결한 표현이 아닌 고유한 구문을 만드는 더 많은 기능을 제공
- 함수형 프로그래밍은 객체 지향의 미래

## 5.7 마무리

- 대규모 코드 수준을 낮게 유지
- 총 입력 횟수를 적게 유지
- 비지역적 입력에서 명시적 입력을 선호
- 비지역적 입력으로 객체 이름을 하드 코딩하는 대신 this를 명시적으로 전달할지 아니면 함수 호출에 바인딩할지 선택
- 부가 작용에 대한 실제적이고 의미 있는 반환값을 선호
- 부가 작용은 최소화하거나 존재하지 않게 할 것
- 비지역적 입력과 전역 변수를 줄이려면 클래스(또는 적어도 객체) 일부로 함수와 다른 변수에 가능한 한 잘 정의된 this를 저장
- 비공개 코드를 위한 작성 방식이 접근 영역에 영향을 미칠 수 있음

## 참고자료

리팩토링 자바스크립트(Refactoring JavaScript)

[링크](https://github.com/gilbutITbook/006963/tree/master/ch05).