---
title: "동시성 이슈 해결"
date: 2022-02-01
excerpt: "동시성 이슈"
categories:
- concurrency
tags:
- concurrency
---
## 배경
업무 중 동시성처리하는 이슈가 많아 다양한 방법으로 동시성 처리가 필요했습니다.

```
ex. 한 유저에게만 포인트 지급하는 이벤트에서 한 유저가 여러 기기로 동시 접근하는 상황
포인트 지급은 API호출로 처리하기 때문에 롤백으로 대응할 수 없다.
그래서 동시에 접근이 불가능하도록 설정이 필요하다.
```

## 동시성 이슈 해결 방안
### 1. DB unique key
적립 DB에 먼저 insert를 하고 이후에 적립 API를 호출합니다.

적립 DB는 유저의 고유한 ID + 적립 필수 조건을 key로 설정합니다.

동시에 요청이 왔다면, 약간 늦게 들어온 요청은 unique key로 인해 DB insert시 에러가 발생합니다.

#### 기본적인 FLOW
1. 적립 이벤트 요청
2. 적립 대상인지 확인
3. 적립 DB에 insert
4. 적립 API 호출
5. 결과 응답
#### 동시 요청 FLOW
1. A 유저가 동시에 2개의 요청(1번 요청, 2번 요청)
2. 1번 요청, 2번 요청 모두 적립대상인지 확인 완료
3. 1번 요청에 대해 적립 DB에 insert
4. 2번 요청에 대해 적립 DB insert 시도 -> unique key error -> 실패 응답
5. 1번 요청에서 적립 API 호출
6. 1번 요청에 대한 성공 응답
#### 이슈
- DB에서 unique key로 잡기 어려운 구조에서는 이 방식으로 대응할 수 없었습니다.
- unique key가 아닌 컬럼의 업데이트 대응이 필요한 경우 대응할 수 없었습니다.
### DB exclusive lock(write lock)
#### exclusive lock(베타 락)이란?
베타 Lock은 데이터를 변경하고자 할 때 사용되며, 트랜잭션이 완료될 때까지 유지됩니다.

베타락은 Lock이 해제될 때까지 다른 트랜잭션(읽기 포함)은 해당 리소스에 접근할 수 없습니다.

해당 Lock이 해제되기 전까지는, 다른 공유Lock, 배타적Lock을 설정하는 것이 불가능합니다.

베타 락을 아래와 같이 `FOR UPDATE`를 추가하여 락을 설정합니다.
```sql
SELECT `column`
FROM `table`
FOR UPDATE 
```
#### 기본적인 FLOW
1. 적립 이벤트 요청
2. 트랜잭션 + write lock으로 적립 대상인지 확인
3. 적립 DB에 insert
4. 적립 API 호출
5. 결과 응답
#### 동시 요청 FLOW
1. A 유저가 동시에 2개의 요청(1번 요청, 2번 요청)
2. 1번 요청 적립 대상 확인하면서 write lock 설정
3. 2번 요청 적립 대상인지 확인 시도 -> lock이 걸려 있기 때문에 대기
4. 1번 요청에 대해 적립 DB에 insert
5. 1번 요청에서 적립 API 호출
6. 락 해제 + 1번 요청에 대한 성공 응답
7. 2번 요청 적립 대상인지 확인 -> 1번 요청 정상 종료하여 적립 미대상
8. 2번 요청 종료
#### 이슈
- 위의 예시에서 2번 요청에서는 1번 요청이 진행되는 동안 대기하는 이슈가 있습니다.
- 타임아웃 설정이 필요합니다.
- 정상적인 흐름대로라면 2번 요청은 어차피 적립이 안 되는 상황이기 때문에 대기할 이유가 없습니다.
- 락을 잘못걸게 된다면 데드락 이슈가 발생할 수 있습니다.

### 추가로 id용 테이블 또는 set 사용
redis에 lock을 걸기 위한 set을 만들어 unique key값들을 저장합니다.

동시에 2개의 요청이 들어오는 경우 하나의 요청만 set에 들어가고, 다른 요청은 set에 들어가지 않아 바로 요청을 종료할 수 있습니다.
#### 기본적인 FLOW
1. 적립 이벤트 요청
2. redis sadd로 적립 대상인지 확인
3. 적립 DB에 insert
4. 적립 API 호출
5. 결과 응답
#### 동시 요청 FLOW
1. A 유저가 동시에 2개의 요청(1번 요청, 2번 요청)
2. 1번 요청 redis sadd 요청 -> 1개 추가되었으므로 1 return
3. 2번 요청 redis sadd 요청 -> 0개 추가되었으므로 0 return -> 요청 종료
4. 1번 요청에 대해 적립 DB에 insert
5. 1번 요청에서 적립 API 호출
6. 1번 요청에 대한 성공 응답
#### 이슈
- 동시성을 막기 위해 redis에서 너무 많은 데이터를 가지고 있습니다.(cache가 아닌 key DB로 사용됨)
- 추가적인 redis 세팅이 필요합니다.
- mysql로도 가능하지만 redis에 비해 성능이 떨어집니다.

## 정리
기본적으로 적립 이력은 mysql DB에 저장했기 때문에 특별한 이슈가 없을 때에는 unique key로 동시성을 제어합니다.

unique key로 막을 수 없는 경우에는 write lock 또는 추가적인 id 테이블 또는 id set을 이용하여 동시 접근을 제어합니다.

write lock을 사용하면 추가 테이블 없이 대응 가능하지만, 스핀락 형태이기 때문에 대기하고 있어서 리소스 낭비가 있습니다.

write lock을 잘못 사용하게 된다면 데드락에 빠질 수 있습니다.

추가적인 테이블 또는 set을 이용한다면 추가적인 메모리또는 스택 구성이 필요합니다.

각 상황에 맞는 적절한 처리가 필요해보입니다.

## 이 외 적용하지는 않았지만 사용 가능한 방법
- [레디스를 활용한 분산 락과 안전하고 빠른 락의 구현](https://suhwan.dev/2019/06/09/transaction-isolation-level-and-lock/)
- [Working With the Spring Distributed Lock](https://tanzu.vmware.com/developer/guides/spring-integration-lock/)
## 참고
- [[데이터베이스] Lock에 대해서 알아보자 - 기본편](https://sabarada.tistory.com/121)
- [[DB] Lock이란?](https://chrisjune-13837.medium.com/db-lock-%EB%9D%BD%EC%9D%B4%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80-d908296d0279)
- [MySQL SELECT FOR UPDATE 의 이해](https://jinhokwon.github.io/mysql/mysql-select-for-update/)
- [Lock으로 이해하는 Transaction의 Isolation Level](https://suhwan.dev/2019/06/09/transaction-isolation-level-and-lock/)

---
**잘못된 정보나 다른 좋은 방법이 있으면 댓글로 공유 부탁드립니다!**

감사합니다.