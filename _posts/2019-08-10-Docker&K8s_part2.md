---
layout: post
title: "도커 컨테이너 배포"
date: 2019-08-10
excerpt: "도커 컨테이버 배포"
tags: [Docker, Kubernetes]
comments: true
---

# 2. 도커 컨테이너 배포

## 2.1 컨테이너로 애플리케이션 실행하기

도커 이미지: 도커 컨테이너를 구성하는 파일 시스템과 실행할 애플리케이션 설정을 하나로 합친 것으로, 컨테이너를 생성하는 템플릿 역할을 한다.

도커 컨테이너: 도커 이미지를 기반으로 생성되며, 파일 시스템과 애플리케이션이 구체화돼 실행되는 상태

컨테이너가 생성될 때 이미지로부터 이를 구체화

컨테이너로 애플리케이션을 실행하려면 컨테이너 형태로 구체화될 템플릿 역할을 하는 이미지를 먼저 만들어야 한다.

### 도커 이미지 만들기

Dockerfile에 전용 도메인 언어로 이미지의 구성을 정의한다.

FROM이나 RUN 같은 키워드를 인스트럭션(명령)이라고 한다.

- FROM 인스트럭션

  도커 이미지의 바탕이 될 베이스 이미지를 저장

  Dockerfile로 이미지를 빌드할 때 먼저 FROM 인스트럭션에 지정된 이미지를 내려받는다.

  FROM에서 받아오는 도커 이미지는 도커 허브라는 레지스트리에 공개된 것

  태그: 각 이미지의 버전을 구별하는 식별자

  각 도커 이미지는 고유의 해시값을 갖는데, 이 해시만으로 필요한 이미지가 무엇인지 특정하기 어렵기 때문에 태그 사용

- RUN 인스트럭션

  도커 이미지를 실행할 때 컨테이너 안에서 실행할 명령을 정의하는 인스트럭션

- COPY 인스트럭션

  도커가 동작 중인 호스트 머신의 파일이나 디렉토리를 도커 컨테이너 안으로 복사하는 인스트럭션

  ADD 인스트럭션이랑 용도가 다르다.

- CMD 인스트럭션

  도커 컨테이너를 실행할 때 컨테이너 안에서 실행할 프로세스 지점

  RUN 인스트럭션은 어미지를 빌드할 때 실행되고, CMD 인스트럭션은 컨테이너를 시작할 때 한 번 실행된다.

  RUN은 애플리케이션 업데이트 및 배치에, CMD는 애플리케이션 자체를 실행하는 명령

  ```bash
  $ go run /echo/main.go
  ```

  이 명령을 CMD 인스트럭션에 기술하면 다음과 같이 명령을 공백으로 나눈 배열로 나타낸다.

  ```dockerfile
  CMD ["go", "run", "/echo/main.go"]
  ```

#### CMD 명령 오버라이드

```bash
$ docker container run $(docker image build -q .) echo test
```

위와 같이 실행하면 CMD에 설정한 명령 대신 echo 실행

### 도커 이미지 빌드하기

docker image build 명령으로 빌드

-t: 이미지명 지정/태그 명 지정(생략 시에는 latest)

-t 옵션을 지정하지 않으면, 빌드는 가능하지만 해시값으로 이미지를 구별해야 하므로 사용하기가 번거롭다.

```bash
$ docker image build -t 이미지명[:태그명] Dockerfile의_경로
```

이미지명의 충돌을 피하기 위해 / 앞에 네임스페이스를 붙일 수 있다(**example**/echo:latest)

예시:  ```docker image build -t example/echo:latest```

빌드를 실행하면 베이스 이미지를 내려받고 RUN이나 COPY 인스트럭션에 지정된 명령이 단계적으로 실행됨

```docker image ls``` 로 빌드가 제대로 되었는 지 확인 가능

#### ENTRYPOINT 인스트럭션으로 명령을 좀 더 세밀하게 실행

ENTRYPOINT 인스트럭션을 사용하면 컨테이너의 명령 실행 방식을 조정할 수 있다.

ENTRYPOINT에서 지정한 값이 기본 프로세스를 지정

```bash
$ docker container run example/echo:latest version
```

위의 코드에서 ENTRYPOINT로 ["go"]를 지정하면 "go version"이 컨테이너 안에서 실행됨

ENTRYPOINT는 이미지를 생성하는 사람이 컨테이너의 용도를 어느 정도 제한하려는 경우에도 유용하다.

#### 그 밖의 Dockerfile 인스트럭션

- LABEL: 이미지를 만든 사람의 이름 등을 적을 수 있다.

- ENV: 도커 컨테이너 안에서 사용할 수 있는 환경변수를 지정

- ARG: 이미지를 빌드할 때 정보를 함께 넣기 위해 사용(이미지를 빌드할 때만 사용할 수 있는 일시적인 환경변수)

#### CMD 인스트럭션 작성 방법

- CMD["실행파일", "인자1", "인자2"]: 실행 파일에 인자를 전달(사용 권장)
- CMD 명령 인자1 인자2: 명령과 인자를 지정(쉘에 정의된 변수를 참조 가능)
- CMD ["인자1", "인자2"]: ENTRYPOINT에 지정된 명령에 사용할 인자를 전달

### 도커 컨테이너 실행

예시: ```docer container run example/echo:latest```

컨테이너를 종료하고 싶으면 Ctrl + C(SIGINT 전송)

-d 옵션을 붙여 백그라운드로 컨테이너 실행 가능 ```docker container run -d example/echo:latest```

#### docker 축약 명령

docker run 혹은 docker pull 은 docker container run과 docker image pull을 축약한 것이다.

축약되지 않은 명령을 사용하면 타이핑 수는 조금 늘어나지만 각 명령이 의미하는 바가 명확하다.

그래서 축약되지 않은 명령 사용

#### 포트 포워딩

도커 컨테이너는 가상 환경이지만, 외부에서 봤을 때 독립된 하나의 머신처럼 다룰 수 있다는 특징이 있다.

애플리케이션이 리스닝하고 있는 포트는 컨테이너 밖에서는 바로 사용할 수 없다. -> Connection refused

포트 포워딩: 호스트 머신의 포트를 컨테이너 포트와 연결해 컨테이너 밖에서 온 통신을 컨테이너 포트로 전달

-p 옵션을 붙여 포트 포워딩 가능(-p 호스트-포트:컨테이너-포트)

```bash
$ docker container run -d -p 9000:8080 example/echo:latest
```

호스트 포트를 생략하면 빈 포트가 ephemeral 포트로 자동할당

```bash
$ docker container run -d -p 8080 example/echo:latest
$ docker container ls
```

docker container ls 의 출력 결과의 PORTS 칼럼에서 확인 가능

## 2.2 도커 이미지 다루기

도커 이미지는 도커 컨테이너를 만들기 위한 템플릿

도커 이미지는 우분투같은 운영 체제로 구성된 파일 시스템은 물론, 컨테이너 위에서 실행하기 위한 애플리케이션이나 그 의존 라이브러리, 도구에 어떤 프로세스를 실행할지 등의 실행 환경의 설정 정보까지 포함하는 아카이브

### docker image build - 이미지 빌드

Dockerfile에 기술된 구성을 따라 도커 이미지를 생성하는 명령

-t: 이미지명 지정/태그 명 지정(생략 시에는 latest)

```bash
$ docker image build -t 이미지명[:태그명] Dockerfile의_경로
```

* **-f 옵션**

  기본으로 Dockerfile이라는 이름으로 된 Dockerfile을 찾지만 그 외 파일명으로 된 Dockerfile을 사용하려면 -f 옵션 사용

  ```bash
  $ docker image build -f Dockerfile-test -t example/echo:latest .
  ```

* **--pull 옵션**

  이 옵션을 사용하면 베이스 이미지(FROM)를 강제로 새로 받아온다.

  ```bash
  $ docker image build --pull=true -t example/echo:latest .
  ```

  이미지를 빌드할 때 확실하게 최신 베이스 이미지를 사용하고 싶을 때 옵션 추가

  빌드 속도에서는 불리하기 때문에 실무에서는 latest로 지정하는 것을 피하고 태그로 지정한 베이스 이미지 사용

### docker search - 이미지 검색

도커 허브는 도커 이미지 레지스트리로 사용자나 조직 이름으로 리포지토리를 만들 수 있다.

도커 이미지를 직접 만드는 대신 다른 사람이나 조직에서 만들어 둔 이미지를 사용할 수 있다.

docker search 명령을 통해 도커 허브에 등록된 리포지토리를 검색 가능

```bash
docker search [options] 검색_키워드
$ docker search --limit 5 mysql
```

공식 리포지토리의 네임스페이스는 일률적으로 library다. 공식 리포지토리의 네임스페이스는 생략할 수 있다.

검색 결과는 STARS 순으로 출력

### docker image pull - 이미지 내려받기

```bash
docker image pull [options] 리포지토리명[:태그명]
docker image pull jenkins:latest
```

태그명을 생락하면 기본값으로 지정된 태그가 적용됨(latest)

### docker image ls - 보유한 도커 이미지 목록 보기

현재 호스트 운영 체제에 저장된 도커 이미지의 목록을 보여준다.

```bash
docker iamge ls [options] [리포지토리[:태그]]
```

IMAGE ID 는 CONTAINER ID와 별개

### docker image tag - 이미지에 태그 붙이기

* 도커 이미지의 버전

  IMAGE ID는 도커 이미지의 버전 넘버 역할

  원래 같은 이미지여도 수정하면 다른 IMAGE ID에 할당됨

* 이미지 ID에 태그 부여하기

  docker image tag는 이미지ID에 태그명을 별명으로 붙이는 명령

  도커 이미지의 태그는 어떤 특정 이미지 ID를 갖는 도커 이미지를 쉽게 식별하는 목적

  태그를 지정하지 않고 빌드한 이미지는 기본적으로 latest 태그 부여

  내용을 수정하고 차분 빌드를 적용해 다시 이미지를 빌드하면 해시값이 이전 이미지와 달라지고 새 이미지가 latest 태그 차지

  ```bash
  docker image tag 기반이미지명[:태그] 새이미지명[:태그]
  $ docker image tage example/echo:latest example/echo:0.1.0
  ```

  ### docker image push - 이미지를 외부에 공개하기

  현재 저장된 도커 이미지를 도커 허브 등의 레지스트리에 등록하기 위해 사용

  ```bash
  docker image push [options] 리포지토리명[:태그]
  $ docker image tag example/echo:latest ligths8615/echo:latest
  $ docker push lights8615/echo
  ```

  docker login 명령으로 로그인

  도커 허브는 자신 혹은 소속 기관이 소유한 리포지토리에만 이미지를 등록할 수 있다.

  공개 리포지토리에 등록할 이미지기 때문에 패스워드나 API 키 값 같은 민감한 정보가 포함되지 않도록 주의

  #### 도커 허브

  도커 레지스트리: 많은 수의 이미지를 중앙 집권적으로 관리하기 위한 호스팅 기능

  도커 허브: 도커 사 자체에서 관리하는 도커 레지스트리

## 2.3 도커 컨테이너 다루기

### 도커 컨테이너의 생애주기

도커 컨테이너는 실행 중, 정지, 파기의 3가지 상태를 가짐

각 컨테이너는 같은 이미지로 생성했다고 하더라도 별개의 상태를 가짐

1. 실행 중 상태

   docker container run 명령의 인자로 지정된 도커 이미지를 기반으로 컨테이너가 생성되면 정의된 애플리케이션이 실행됨

   애플리케이션이 실행 중인 상태가 컨테이너의 실행 중 상태

   실행이 끝나면 정지 상태

2. 정지 상태

   실행 중 상태에 있는 컨테이너를 사용자가 명시적으로 정지하거나 컨테이너에서 실행된 애플리케이션이 정상/오류 여부를 막론하고 종료된 경우 자동으로 정지 상태

   컨테이너를 정지시키면 디스크에 컨테이너가 종료되던 시점의 상태가 저장돼 남는다.

3. 파기 상태

   정지 상태의 컨테이너는 명시적으로 파기하지 않는 이상 디스크에 그대로 남아 있다.(호스트 운영 체제를 종료해도 남아 있다.)

   컨테이너를 자주 생성하고 정지해야 하는 상황에서는 디스크를 차지하는 용량이 점점 늘어나므로 불필요한 컨테이너를 완전히 삭제하는 것이 바람직하다.

   한 번 파기한 컨테이너는 다시 실행할 수 없다.

### docker container run - 컨테이너 생성 및 실행

도커 이미지로부터 컨테이너를 생성하고 실행하는 명령

```bash
docker container run [options] 이미지명[:태그] [명령] [명령인자 ...]
docker container run [options] 이미지ID [명령] [명령인자 ...]
$ docker container run -d -p 9000:8000 example/echo:latest
```

- docker container run 명령의 인자

  CMD 인스트럭션 오버라이드

  ```bash
  $ docker container run -it alpine:3.7 uname -a
  ```

- 컨테이너에 이름 붙이기

  ```--name``` 옵션을 사용하여 컨테이너에 원하는 이름을 붙일 수 있다.

  ```bash
  docker container run --name [컨테이너명] [이미지명][:태그]
  $ docker container run -t -d --name mino-echo example/echo:latest
  ```

  이름 붙인 컨테이너는 개발용으로 비교적 자주 사용되지만, 같은 이름의 컨테이너를 새로 실행하려면 같은 이름을 갖는 기존의 컨테이너를 먼저 삭제해야 하기 때문에 운영 환경에서는 거의 사용되지 않는다.

#### 도커 명령에서 자주 사용되는 옵션

1. -i : 컨테이너를 실행할 때 컨테이너 쪽 표준 입력과의 연결을 그대로 유지(컨테이너 셸에서 명령실행 가능)
2. -t : 유사 터미널 기능 활성화(-it 로 -i옵션과 주로 묶어서 사용)
3. —rm: 컨테이너를 종료할 때 컨테이너를 파기하도록 하는 옵션
4. -v: 호스트와 컨테이너 간에 디렉터리나 파일 공유 옵션

### docker container ls - 도커 컨테이너 목록 보기

```bash
docker container ls [options]
```

- 컨테이너 ID만 추출하기

- ```bash
  $ docker container ls -q
  ```

- 컨테이너 목록 필터링하기

  ```bash
  docker container ls --filter "필터명=값"
  $ docker container ls --filter "name=echo1"
  $ docker container ls --filter "ancestor=example/echo"
  ```

  name은 이름과 컨테이너명이 일치

  ancestor는 컨테이너를 생성한 이미지 기준

- 종료된 컨테이너 목록 보기

  ```bash
  $ docker container ls -a
  ```

### docker container stop - 컨테이너 정지하기

```bash
docker container stop 컨테이너ID_또는_컨테이너명
$ docker container stop cd78dc51873b
$ docker container stop echo1
```

### docker container restart - 컨테이너 재시작하기

```bash
docker container restart 컨테이너ID_또는_컨테이너명
$ docker container restart echo1
```

### docker container rm - 컨테이너 파기하기

```bash
docker container rm 컨테이너ID_또는_컨테이너
$ docker container rm 6d451a8ed47f
$ docker container rm echo2
$ docker cotainer rm -f 2833385cc36b
```

현재 실행중인 컨테이너는 일반적인 rm 명령으로 삭제 불가능 -> -f 옵션 사용

* docker container run --rm을 사용해 컨테이너를 정지할 때 함께 삭제하기

  —rm을 붙여 생성한 컨테이너는 실행이 끝나면 자동으로 파기

  이름이 붙은 컨테이너를 자주 생성하고 정지해야 할 때 주로 사용(같은 이름의 컨테이너가 2개 생성이 불가능하기 때문에)

### docker container logs - 표준 출력 연결하기

현재 실행 중인 특정 도커 컨테이너의 표준 출력 내용을 확인 가능

표준 출력으로 출력된 내용만 확인할 수 있으므로 파일 등에 출력된 로그는 볼 수 없다.

```bash
docker container logs [options] 컨테이너ID_또는_컨테이너명
$ docker container logs -f $(docker contain ls --filter "ancestors=jenkins" -q)
```

실제 운영 단계에서는 실행 중인 컨테이너의 로그를 수집해 웹 브라우저나 명령행 도구로 열람하게 해주는 기능을 사용하기 때문에 docker container logs 명령을 실제로 사용하는 경우는 그다지 많지 않다.

### docker container exec - 실행 중인 컨테이너에서 명령 실행하기

```bash
docker container exec [options] 컨테이너ID_또는_컨테이너명 컨테이너에서_실행할_명령
$ docker container exec echo pwd
$ docker container exec -it echo sh
```

마치 ssh로 로그인한 것처럼 컨테이너 내부를 조작 가능

표준 입력 연결을 유지하는 -i옵션과 유사 터미널을 할당하는 -t 옵션을 조합하면 컨테이너를 셸을 통해 다룰 수 있다.

컨테이너 안에 든 파일을 수정하는 것은 애플리케이션에 의도하지 않은 부작용을 초래할 수 있으므로 운영 환경에서는 절대 해서는 안 된다.

### docker container cp - 파일 복사하기

컨테이너끼리 혹은 컨테이너와 호스트 간에 파일을 복사하기 위한 명령

실행 중인 컨테이너와 파일을 주고받기 위한 명령

```bash
docker container cp [options] 컨테이너ID_또는_컨테이너명:원본파일 대상파일
docker container cp [options] 호스트_원본파일 컨테이너ID_또는_컨테이너명:대상파일
$ docker container cp echo:/echo/main.go .
$ docker container dummy.txt echo:/tmp
```

아직 파기되지 않은 정지 상태의 컨테이너에 대해서도 실행할 수 있다.

## 2.4 운영과 관리를 위한 명령

#### prune - 컨테이너 및 이미지 파기

* docker container prune

  필요 없는 이미지나 컨테이너를 일괄 삭제 가능

  실행 중이 아닌 모든 컨테이너를 삭제

  ```bash
  docker container prune [options]
  $ docker container prune
  ```

* docker image prune

  태그가 붙지 않은 모든 이미지를 삭제

  ```bash
  docker image prune [options]
  $ docker image prune
  ```

  남은 이미지는 실행 중인 컨테이너의 이미지 등 이유가 있어 남겨 놓은 것

* docker system prune

  사용하지 않는 도커 이미지 및 컨테이너, 볼륨, 네트워크 등 모든 도커 리소스를 일괄적으로 삭제

### docker container stats - 사용 현황 확인하기

시스템 리소스 사용 현황을 컨테이너 단위로 확인(top과 유사)

```bash
docker container stats [options] [대상_컨테이너ID ...]
$ docker container stats
```

## 2.5 도커 컴포즈로 여러 컨테이너 실행하기

도커 컨테이너 = 단일 애플리케이션이라고 봐도 무방하다.

도커 컨테이너로 시스템을 구축하면 하나 이상의 컨테이너가 서로 통신하며, 그 사이에 의존관계가 생긴다.

컨테이너의 동작을 제어하기 위한 설정 파일이나 환경 변수를 어떻게 전달할지, 컨테이너 간의 의존관계를 고려할 때 포트 포워딩을 어떻게 설정해야 하는지 등의 요소를 적절히 관리해야 한다.

### docker-compose 명령으로 컨테이너 실행하기

Compose는 yaml 포맷으로 기슬된 설정 파일로, 여러 컨테이너의 실행을 한 번에 관리할 수 있게 해준다.

```yaml
version: "3" # 내용을 해석하는 데 필요한 문법 버전
services:
    echo: # 컨테이너 이름
        image: example/echo:latest # 실행할 이미지
        ports: # 포트포워딩 설정
            - 9000:8080
```

```bash
$ docker-compose up # docker-compose.yml로 지정한 컨테이너 실행
$ docker-compose down # docker-compose.yml에 지정된 모든 컨테이너 정지 혹은 삭제 
```

컴포즈를 사용하여 이미 존재하는 도커 이미지뿐만 아니라 docker-compose up 명령을 실행하면서 이미지를 함께 빌드해 새로 생성한 이미지를 실행할 수도 있다.

```yaml
version: "3" # 내용을 해석하는 데 필요한 문법 버전
services:
    echo: # 컨테이너 이름
        build: . # 해당 경로에 있는 Dockerfile에서 이미지 빌드
        ports: # 포트포워딩 설정
            - 9000:8080
```

```bash
$ docker compose up -d --build
```

이미 컴포즈가 이미지를 빌드한 적이 있다면 빌드를 생략하고 컨테이너가 실행된다.

—build 옵선을 사용하면 도커 이미지를 강제로 다시 빌드하게 할 수 있다.

## 2.6 컴포즈로 여러 컨테이너 실행하기

docker-compose.yaml을 작성하면 기존 docker 명령을 사용해 컨테이너를 실행할 때 매번 부여하던 옵션을 설정 파일로 관리할 수 있다.

### 젠킨스 컨테이너 실행하기

```yaml
version: "3"
services:
  master:
    container_name: master
    image: jenkinsci/jenkins:2.142-slim
    ports:
      - 8888:8080
    volumes:
      - ./jenkins_home:/var/jenkins_home
```

volumes는 호스트와 컨테이너 사이에 파일을 복사하는 것이 아니라 파일을 공유하는 매커니즘

호스트 쪽 현재 작업 디렉토리 바로 애래에 jenkins_home 디렉터리를 젠킨스 컨테이너의 /var/jenkins_home에 마운트

/var/jenkins_home에 저장되기 때문에 컨테이너를 종료했다가 재시작해도 초기 설정 유지

### 마스터 젠킨스 용 SSH 키 생성

관리 기능이나 작업 실행 지시 등은 마스터 인스턴스가 맡고, 작업을 실제로 진행하는 것은 슬레이브 인스턴스가 담당하는 구조

### 슬레이브 젠킨스 컨테이너 생성

```yaml
version: "3" # 내용을 해석하는 데 필요한 문법 버전
services:
  master:
    container_name: master
    image: jenkinsci/jenkins:2.142-slim
    ports:
      - 8888:8080
    volumes:
      - ./jenkins_home:/var/jenkins_home
    links:
      - slave01

  slave01:
    container_name: slave01
    image: jenkinsci/ssh-slave
    environment:
      - JENKINS_SLAVE_SSH_PUBKEY=ssh-rsa asjfiowe~~~~~#!@$!@
```

### 더 나은 컨테이너 개발

이상적인 도커 구성 관리는 애플리케이션을 바로 이용할 수 있는 수준까지 수작업 없이 배포하는 것이 목표

## 참조

1. 도커/쿠버네티스를 활용한 컨테이너 개발 실전 입문