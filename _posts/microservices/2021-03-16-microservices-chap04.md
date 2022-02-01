---
title: "도커를 사용한 마이크로서비스 배포"
date: 2021-03-16
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 4. 도커를 사용한 마이크로서비스 배포

## 도커 소개

도커는 가상머신을 대체하는 경량 컨테이너 개념을 일반화

컨테이너는 리눅스 호스트에서 실행되며, 리눅스 네임스페이스를 이용해서 사용자, 프로세스, 파일 시스템, 네트워킹 등의 전역 시스템 리소스를 컨테이너에 분배

또한 **리눅스 제어 그룹**을 사용해 컨테이너가 사용할 수 있는 CPU와 메모리를 제한

시작 시간이 훨씬 단축되며, CPU와 메모리 사용량 측면의 오버헤드도 크게 줄어듬

## 도커에서 자바를 실행할 때의 문제

자바가 리눅스 cgroup으로 지정한 자원 할당량을 무시했기 때문에 도커와는 잘 맞지 않았다.

자바는 컨테이너에 허용된 메모리를 JVM에 할당하는 게 아니라 도커 호스트의 전체 메모리를 할당했다

또한 컨테이너에 허용된 CPU 코어만 JVM에 할당하는 게 아니라 도커 호스트의 스레드 풀 등의 CPU 관련 전체 자원을 컨테이너의 JVM에 할당했다.

SE9 버전부터 조금씩 도커 지원 시작

#### 도커에서 자바 커맨드 실행

**CPU**

자바 SE12는 컨테이너의 제약 조건 설정을 준수하므로 스레드 풀과 같은 CPU 관련 자원을 제대로 구성할 수 있다.

**메모리**

문제 X

#### 자바 SE 9 도커 컨테이너의 문제

CPU 제한 준수 X

메모리 제약 조건 준수 X



도커와 자바로 심각한 작업을 수행하는 경우엔 자바 SE 10 이상을 사용해야 한다

## 도커로 단일 마이크로서비스 실행

다른 마이크로서비스와 다른 호스트에서 실행되므로 포트 충돌 X

localhost로 통신 불가능

마이크로서비스를 컨테이너에서 실행하더라도 소스 코드를 변경할 필요는 없으며, 마이크로서비스의 구성만 변경하면 됨



마이크로서비스를 로컬에서 실행할 때와 도커 컨테이너로 실행할 때의 구성이 다르므로 이를 처리하고자 스프링 프로필 사용

#### 소스코드 변경

application.yml 끝에 추가

```yaml
---
spring.profiles: docker

server.port: 8080
```

Dockerfile 생성

```dockerfile
# OpenJDK의 공식 도커 이미지 기반
FROM openjdk:12.0.2
# 다른 도커 컨테이너에 8080포트 공개
EXPOSE 8080
# 그래들 빌드 폴더에 있는 팻JAR 파일을 도커 이미지에 추가
ADD ./build/libs/*.jar app.jar
# 도커 이미지로 컨테이너를 시작할 때 도커에서 사용할 커맨드를 지정
ENTRYPOINT ["java","-jar","/app.jar"]

```

#### 도커 이미지 빌드

팻 JAR 빌드

```bas
./graldew :microservices:product-service:build
```

#### 서비스 시작

```bash
docker run --rm -p8080:8080 -e "SPRING_PROFILES_ACTIVE=docker" product-service
```

1. docker run: 컨테이너를 시작하고 터미널에 로그 출력
2. --rm: ctrl+c를 입력해 컨테이너를 중지하면 도커가 컨테이너를 제거함
3. -p8080:8080: 컨테이너의 8080포트를 도커 호스트의 8080 포트에 매핑해 외부에서 호출할 수 있게 함. 도커 호스트의 특정 포트에는 컨테이너를 하나만 매핑 가능
4. -e: 컨테이너의 환경 변수 지정 가능
5. product-service: 도커가 컨테이너를 시작할 때 사용할 도커 이미지의 이름

#### 컨테이너를 분리 모드로 실행

컨테이너를 실행하면서 터미널을 잠그지 않으려면 분리(detached) 모드로 컨테이너를 시작

```bash
docker run -d -p8080:8080 -e "SPRING_PROFILES_ACTIVE=docker" --name my-prd-srv product-service
```

-d 옵션을 사용해 분리모드로 컨테이너 시작

--name 옵션으로 컨테이너의 이름을 지정

로그 확인

```bash
docker logs my-prd-srv -f
```

-f 옵션은 터미널에 로그가 출력되는 동안 커맨드를 종료하지 않고 계속해서 로그를 출력하게 함

컨테이너 중지 및 제거

```bash
docker rm -f my-prd-srv
```

#### 도커 컴포즈를 사용한 마이크로서비스 환경 관리

도커 컴포즈를 사용하려면 도커 컴포즈가 관리할 마이크로서비스를 설명하는 구성 파일을 만들어야 한다

product-compositie-service는 핵심 서비스를 찾을 수 있어야 하기 때문에 복잡

product-compositie-service의 docker 프로필

```yaml
---
spring.profiles: docker

server.port: 8080

app:
  product-service:
    host: product
    port: 8080
  recommendation-service:
    host: recommendation
    port: 8080
  review-service:
    host: review
    port: 8080
```

docker 프로필에서 사용한 호스트 이름은 docker-compose.yml에 지정되어 있음

```yaml
version: '2.1'

services:
  product:
    build: microservices/product-service
    mem_limit: 350m
    environment:
      - SPRING_PROFILES_ACTIVE=docker

  recommendation:
    build: microservices/recommendation-service
    mem_limit: 350m
    environment:
      - SPRING_PROFILES_ACTIVE=docker

  review:
    build: microservices/review-service
    mem_limit: 350m
    environment:
      - SPRING_PROFILES_ACTIVE=docker

  product-composite:
    build: microservices/product-composite-service
    mem_limit: 350m
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker

```

각 마이크로서비스에 대해 다음과 같은 사항을 지정

- 마이크로서비스의 이름. 도커 내부 네트워크에서 사용하는 컨테이너의 호스트 이름이기도 함
- 도커 이미지 빌드에 사용할 Dockerfile의 위치를 지정하는 빌드 지시문
- 메모리 제한 -> 전체 메모리 크기에 맞춰야 함
- 컨테이너에서 사용할 환경 변수(스프링 프로필)

포트를 매핑한 product-composite 서비스만 도커 외부에서 접근 가능

#### 마이크로서비스 환경 시작

```bash
docker-compose up -d
docker-compose logs -f
docker-compose down
```

## 도커 컴포즈를 사용한 마이크로서비스 환경 테스트

테스트 스크립트에 도커 컴포즈를 통합

```bash
#!/usr/bin/env bash
#
# ./gradelw clean build
# docker-compose build
# docker-compose up -d
#
# Sample usage:
#
#   HOST=localhost PORT=7000 ./test-em-all.bash
#
: ${HOST=localhost}
: ${PORT=8080}

function assertCurl() {

  local expectedHttpCode=$1
  local curlCmd="$2 -w \"%{http_code}\""
  local result=$(eval $curlCmd)
  local httpCode="${result:(-3)}"
  RESPONSE='' && (( ${#result} > 3 )) && RESPONSE="${result%???}"

  if [ "$httpCode" = "$expectedHttpCode" ]
  then
    if [ "$httpCode" = "200" ]
    then
      echo "Test OK (HTTP Code: $httpCode)"
    else
      echo "Test OK (HTTP Code: $httpCode, $RESPONSE)"
    fi
  else
      echo  "Test FAILED, EXPECTED HTTP Code: $expectedHttpCode, GOT: $httpCode, WILL ABORT!"
      echo  "- Failing command: $curlCmd"
      echo  "- Response Body: $RESPONSE"
      exit 1
  fi
}

function assertEqual() {

  local expected=$1
  local actual=$2

  if [ "$actual" = "$expected" ]
  then
    echo "Test OK (actual value: $actual)"
  else
    echo "Test FAILED, EXPECTED VALUE: $expected, ACTUAL VALUE: $actual, WILL ABORT"
    exit 1
  fi
}

function testUrl() {
    url=$@
    if curl $url -ks -f -o /dev/null
    then
          echo "Ok"
          return 0
    else
          echo -n "not yet"
          return 1
    fi;
}

function waitForService() {
    url=$@
    echo -n "Wait for: $url... "
    n=0
    until testUrl $url
    do
        n=$((n + 1))
        if [[ $n == 100 ]]
        then
            echo " Give up"
            exit 1
        else
            sleep 6
            echo -n ", retry #$n "
        fi
    done
}


set -e

echo "Start:" `date`

echo "HOST=${HOST}"
echo "PORT=${PORT}"

if [[ $@ == *"start"* ]]
then
    echo "Restarting the test environment..."
    echo "$ docker-compose down"
    docker-compose down
    echo "$ docker-compose up -d"
    docker-compose up -d
fi

waitForService http://$HOST:$PORT/product-composite/1

# Verify that a normal request works, expect three recommendations and three reviews
assertCurl 200 "curl http://$HOST:$PORT/product-composite/1 -s"
assertEqual 1 $(echo $RESPONSE | jq .productId)
assertEqual 3 $(echo $RESPONSE | jq ".recommendations | length")
assertEqual 3 $(echo $RESPONSE | jq ".reviews | length")

# Verify that a 404 (Not Found) error is returned for a non existing productId (13)
assertCurl 404 "curl http://$HOST:$PORT/product-composite/13 -s"

# Verify that no recommendations are returned for productId 113
assertCurl 200 "curl http://$HOST:$PORT/product-composite/113 -s"
assertEqual 113 $(echo $RESPONSE | jq .productId)
assertEqual 0 $(echo $RESPONSE | jq ".recommendations | length")
assertEqual 3 $(echo $RESPONSE | jq ".reviews | length")

# Verify that no reviews are returned for productId 213
assertCurl 200 "curl http://$HOST:$PORT/product-composite/213 -s"
assertEqual 213 $(echo $RESPONSE | jq .productId)
assertEqual 3 $(echo $RESPONSE | jq ".recommendations | length")
assertEqual 0 $(echo $RESPONSE | jq ".reviews | length")

# Verify that a 422 (Unprocessable Entity) error is returned for a productId that is out of range (-1)
assertCurl 422 "curl http://$HOST:$PORT/product-composite/-1 -s"
assertEqual "\"Invalid productId: -1\"" "$(echo $RESPONSE | jq .message)"

# Verify that a 400 (Bad Request) error error is returned for a productId that is not a number, i.e. invalid format
assertCurl 400 "curl http://$HOST:$PORT/product-composite/invalidProductId -s"
assertEqual "\"Type mismatch.\"" "$(echo $RESPONSE | jq .message)"

if [[ $@ == *"stop"* ]]
then
    echo "We are done, stopping the test environment..."
    echo "$ docker-compose down"
    docker-compose down
fi

echo "End:" `date`
```

```bash
./test-em-all.bash start stop
```

#### 테스트 실행 문제 해결

1. 실행 중인 마이크로서비스의 상태 확인

   ```bash
   docker-compose ps
   ```

2. 모든 마이크로서비스가 동작하는지 확인

3. UP 상태가 아닌 마이크로서비스가 있다면 로그 확인

   ```bash
   docker-compose logs product
   ```

4. 만약 로그에 디스크 용량 부족과 관련된 오류가 있다면 복구

   ```bash
   docker system prune -f --volumes
   ```

5. 필요하다면 docker-compose up -d --scale 커맨드로 비정상인 마이크로서비스 다시 시작

   ```bash
   docker-compose up -d --scale product=0
   docker-compose up -d --scale product=1
   ```

6. 충돌 등의 이유로 마이크로서비스가 사라졌으면 시작

   ```bash
   docker-compose up -d --scale product=1
   ```

7. 모든 마이크로서비스가 정상 동작하게 되면 테스트 스크립트를 start 없이 다시 실행

   ```bash
   ./test-em-all.bash
   ```

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)


