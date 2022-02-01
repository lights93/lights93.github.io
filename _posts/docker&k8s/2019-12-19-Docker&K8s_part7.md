---
title: "쿠버네티스 실전편"
date: 2019-12-19
excerpt: "쿠버네티스 실전편"
categories:
  - Docker
tags:
  - Docker
  - Kubernetes
---

# 7. 쿠버네티스 실전편

## 7.1 쿠버네티스의 그 외 리소스

### 잡

잡은 하나 이상의 파드를 생성해 지정된 수의 파드가 정상 종료될 때까지 이를 관리하는 리소스

정상 종료된 후에도 삭제되지 않고 그대로 남아있기 때문에 작업이 종료된 후에 파드의 로그나 실행 결과 분석 가능

배치 작업 위주의 어플리케이션에 적합

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pingpong
  labels:
    app: pingpong
spec:
  parallelism: 3 # 동시에 실행하는 파드의 수 설정
  template:
    metadata:
      labels:
        app: pingpong
    spec:
      containers:
        - name: pingpong
          image: gihyodocker/alpine:bash
          command: ["/bin/sh"]
          args:
            - "-c"
            - |
              echo [`date`] ping!
              sleep 10
              echo [`date`] pong!
      restartPolicy: Never # 파드 종료 후 재실행 여부/ job에서는 Never 혹은 OnFailure만 가능
```

종료된 파드는 Completed

### 크론잡

크론잡 리소스는 스케줄을 지정행 정기적으로 파드 실행 가능

컨테이너 친화적인 특성을 유지하면서 스케줄에 따른 작업 수행 가능

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: pingpong
spec:
  schedule: "*/1 * * * *" # 크론과 같은 포맷으로 스케줄 정의
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: pingpong
        spec:
          containers:
            - name: pingpong
              image: gihyodocker/alpine:bash
              command: ["/bin/sh"]
              args:
                - "-c"
                - |
                  echo [`date`] ping!
                  sleep 10
                  echo [`date`] pong!
          restartPolicy: OnFailure
```

### 시크릿

쿠버네티스의 시크릿 리소스를 사용하면 기밀 정보 문자열을 BASE64 인코딩으로 만들 수 있다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-secret
type: Opaque
data:
  .htpasswd: bWlubzpoZml2dDhpWURQcHouCg== # base64 인코딩 예시
```

생성한 secret을 볼륨으로 마운트한다.

환경변수로 인증정보 파일의 경로 설정

시크릿 리소스는 여러 층위에 걸친 보안 대책 중 하나

#### 인증정보를 환경 변수로 안전하게 관리하기

환경 변수는 쿠버네티스 매니페스트 파일에 포함되기 때문에 평문 그대로 매니페스트 파일에 들어가지 않도록 해야 한다.

1. 인코딩

2. 인코딩된 문자열로 시크릿 리소스 생성

3. 환경변수 설정에서 values 대신 valueFrom.secretKeyRef 값 사용

   ```yaml
   env:
     - name: password
       valueFrom:
         secretKeyRef:
           name: mysql-secret
           key: username
   ```



## 7.2 사용자 관리와 RBAC

쿠버네티스의 사용자 개념

- 일반 사용자: 클러스터 외부에서 쿠버네티스를 조작하는 사용자로, 다양한 방법으로 인증을 거친다.
- 서비스 계정: 쿠버네티스 내부적으로 관리되며 파드가 쿠버네티스 API를 다룰 때 사용하는 사용자

서비스 계정 및 일반 사용자의 권한은 RBAC(role-based access control)라는 매커니즘을 통해 제어됨

RBAC는 롤에 따라 리소스에 대한 권한을 제어하는 기능이자 개념

RBAC를 적절히 사용해 쿠버네티스의 리소스의 보안 확보 가능

### RBAC를 이용한 권한 제어

롤: 어떤 쿠버네티스 API를 사용할 수 있는지가 정의됨

- 롤: 각 쿠버네티스 API의 사용 권항을 정의, 지정된 네임스페이스 안에서만 유효
- 클러스터롤: 각 쿠버네티스 API의 사용 권한을 정의, 클러스터 전체에서 유효

바인딩: 롤을 일반 사용자 및 그룹, 그리고 서비스 계정과 연결해줌

- 롤바인딩: 일반 사용자 및 그룹/서비스 계정과 롤을 연결
- 클러스터롤바인딩: 일반 사용자 및 그룹/서비스 계정과 클러스터롤을 연결

로컬 쿠버네티스 환경에서 수행 불가능 (이하 생략 ...)

## 7.3 헬름

하나 이상의 클러스터를 운영하다 보면 같은 애플리케이션을 여러 클러스터에 배포해야하는 경우 발생

매니페스트 파일을 클러스터 개수만큼 작성하는 환경에 따라 달리 적용하며 배포하기는 쉽지 않은 일이며 실수하기 쉽다.

이런 문제를 해결한 것이 헬름

헬름은 쿠버네티스 차트를 관리하기 위한 도구

헬름인 패키지 관리 도구, 차트가 리소스를 하나로 묶은 패키지

helm: 차트를 관리

차트: 매니페트스 템플릿 패키지/차트를 사용하여 매니페스트 파일 생성

매니페스트 파일에 기초한 쿠버네티스 리소스 관리

### 헬름 설치

### 헬름의 주요 개념

헬름은 클라이언트(cli)와 서버(쿠버네티스 클러스터에 설치되는 틸러)로 구성

클라이언트는 서버를 대상으로 명령을 지시하는 역할

서버는 클라이언트에서 전달받은 명령에 따라 쿠버네티스 클러스터에 패키지 설치, 업데이트, 삭제 등의 작업 수행

쿠버네티스는 매니페스트 파일을 적용하는 방식으로 애플리케이션을 배포

매니페스트 파일을 여러 개 패키징한 것이 차트

차트는 헬름 리포지토리에 tgz파일로 저장, 틸러가 매니페스트를 생성하는 데 사용

#### 리포지토리

- local: 헬름 클라이언트가 설치된 로컬 리포지토리, 로컬에서 생성한 패키지 존재
- stable: 안정 버전에 이른 차트가 존재하는 리포지토리, 일정한 요건을 만족하는 차트만 제공
- Incubator: stable요건을 만족하지 못하는 차트가 제공되는 리포지토리

#### 차트의 구성

```
chart_name / --- templates/ 매니페스트 파일 템플릿 데릭터리
| |- xxx.yaml 각종 쿠버네티스 리소스의 매니페스트 템플릿
| |- _helper.tpl 매니페트스 렌더링에 사용되는 템플릿 헬퍼
| |- NOTE.txt 차트 사용법 등의 참조 문서 템플릿
|
| |- charts/ 이 차트가 의존하는 차트의 디렉터리
|- Chart.yaml 차트 정보가 정의된 파일
|- values.yaml 차트 기본값 value 파일
```

### 차트 설치하기

```bash
helm install [--name 릴리스_네임] 차트_리포지토리/차트명
```

helm install 명령을 실행하면 차트에 포함된 기본값 value파일에 정의된 설정값으로 설치됨

기본값 value를 사용하는 경우는 드물며, 대부분 일부 수정된 커스텀 value 사용

```yaml
redmineUsername: mino
redminePassword: mino
remineLanguage: ko

serviceType: NodePort
```

```bash
helm install -f redmine-test.yaml redmine stable/redmine --version 4.0.0
helm ls # 확인
helm upgrade -f redmine-test.yaml redmine stable/redmine --version 4.0.0 # value 파일 수정 적용
```

### 차트로 설치한 애플리케이션 제거하기

```bash
helm delete redmine # v3에서는 완전 제거인 듯?
helm rollback redmine 2 # 롤백
```

### RBAC를 지원하는 애플리케이션 설치하기

### 사용자 차트 생성하기

생략...

## 7.4 쿠버네티스 배포 전략

컨테이너를 사용한 배포는 각 서버가 도커 이미지를 직접 받아가는 풀 배포이므로 배포 및 스케일 아웃이 쉽다.

쿠버네티스 역시 컨테이너의 장점을 살려 배포 가능

### 롤링 업데이트

디플로이먼트 리소스에서는 파드를 교체하는 전략을 .specs.strategy.type 속성에 정의

```yaml
# 재배포용 
apiVersion: v1
kind: Service
metadata:
  name: echo-version
  labels:
    app: echo-version
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: echo-version

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-version
  labels:
    app: echo-version
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo-version
  template:
    metadata:
      labels:
        app: echo-version
    spec:
      containers:
        - name: echo-version
          image: gihyodocker/echo-version:0.1.0 # 재배포시 버전을 바꾼다.
          ports:
            - containerPort: 8080

```

```yaml
# 모니터링 용
apiVersion: v1
kind: Pod
metadata:
  name: update-checker
  labels:
    app: update-checker
spec:
  containers:
    - name: kubectl
      image: gihyodocker/fundamental:0.1.0
      command:
        - sh
        - -c
        - |
          while true
          do
            APP_VERSION=`curl -s http://echo-version`
            echo "[`date`] $APP_VERSION "
            sleep 1
          done
```

재배포전 파드 상태: running

재배포 실행 시점의 파드 상태: 기존 파드 running/ 새로운 파드containercreating

파드 교체: 새로운 파드 running/기존 파드 terminating

#### 롤링 업데이트의 동작 제어하기

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-version
  labels:
    app: echo-version
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: echo-version

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-version
  labels:
    app: echo-version
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingupdate:
      maxUnavailable: 3 # 롤링 업데이트 중 동시에 삭제할 수 있는 파드의 최대 개수 (비율로도 지정 가능) (기본값은 25%)
      maxSurge: 4 # 동시에 생성하는 파드의 개수 (기본값은 25%)
  selector:
    matchLabels:
      app: echo-version
  template:
    metadata:
      labels:
        app: echo-version
    spec:
      containers:
        - name: echo-version
          image: gihyodocker/echo-version:0.3.0
          ports:
            - containerPort: 8080

```

maxUnavailable이 크면 동시에 교체되는 파드 수가 늘어나므로 롤링 업데이트에 걸리는 시간이 줄어든다.

하지만, 롤링 업데이트 중 요청이 하나에 몰리기 때문에 트레이드오프 발생

maxSurge 값이 크면 필요판 파드를 빨리 생성하므로 교체 시간이 단축

하지만, 순간적으로 필요한 시스템 자원이 급증하는 부작용

### 실행 중인 컨테이너에 대한 헬스 체크 설정

애플리케이션에 따라 컨테이너가 모두 시작된 후에도 요청을 처리할 수 있는 상태가 될 때까지 시간이 걸리는 경우 존재

쿠버네티스에서 이런 문제를 해결하기 위해 livenessProbe와 readinessProbe 존재

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-version
  labels:
    app: echo-version
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: echo-version

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-version-hc
  labels:
    app: echo-version
spec:
  replicas: 4
  selector:
    matchLabels:
      app: echo-version
  template:
    metadata:
      labels:
        app: echo-version
    spec:
      containers:
        - name: echo-version
          image: gihyodocker/echo-version:0.1.0
          imagePullPolicy: Always
          livenessProbe: # 애플리케이션이 의존하는 컨테이너 안의 파일이 존재하는지를 확인하는 용도
            exec:
              command: # live.txt가 존재하지 않으면 재시작
                - cat
                - /live.txt
            timeoutSeconds: 3 # 헬스체크 요청 타임아웃 시간
            initialDelaySeconds: 15 # 컨테이너를 실행한 후 헬스체크를 실행하는 시간
          ports:
            - containerPort: 8080

```

readinessProbe는 컨테이너 외부에서 HTTP 요청 같은 트래픽을 발생시켜 이를 처리할 수 있는 상태인지 확인

livenessProbe 또는 readinessProbe Running 상태가 되어도 READY가 0/1로 나오다가 모든 헬스체크 확인 후 1/1로 변경

#### 안전을 위해 애플리케이션이 정지한 후 파드를 삭제

삭제 중인 파드의 컨테이너가 해당 시점에 사용자로부터 받은 요청을 처리 중이라면 사용자는 애플리케이션의 응답을 받지 못한다.

이런 일을 방자히려면 애플리케이션이 확실히 안전하게 종료된 다음에 컨테이너를 삭제해야 한다.

미리 규정된 안전한 절차를 거쳐 애플리케이션을 셧다운하는 것을 Graceful Shutdown(안전 종료)라고 한다.



파드에 종료 명령이 전달되면 파드에 속하는 컨테이너 프로세스에 SIGTERM 시그널 전달.

SIGTERM 시그널을 받은 컨테이너는 terminationGracePeriodSeconds(기본값 30초)에 설정된 시간 안에 정상적으로 애플리케이션이 종료되면 SIGKILL 시그널을 보내 컨테이너를 강제종료

종료처리가 오래 걸리는 컨테이너는 terminationGracePeriodSeconds값을 늘려 설정 필요



Nginx는 SIGTERM시그널을 받으면 즉시 종료되므로 다른 방법이 필요

lifecycle.preStop 속성에서 컨테이너 종료 시작 시점의 훅을 정의 가능

### 블루-그린 배포

새 버전 파드와 구버전 파드가 불가피하게 같이 존재하는 시간이 발생

그래서 의도치 않은 부작용 발생 가능

#### 블루-그린 배포란?

기존 서버군과 새로운 버전이 배포된 서버군을 구성하고, 로드 밸런서 혹은 서비스 디스커버리 수준에서 참조 대상을 교체하는 방식으로 이뤄지는 배포

일시적으로나마 애플리케이션을 배포할 서버군을 2계통으로 유지해야 하기 때문에 롤링업데이트보다 필요 리소스의 양이 늘어난다.

장점으로는 신버전과 구버전이 혼재하는 시간 없이 순간적인 교체가 가능하며, 한쪽 서버군을 릴리스 전 스탠바이 상태로 사용 가능하다.

#### 2계통의 디플로이먼트 준비하기

배포 시에 기존 디플로이먼트를 업데이하는 것이 아니라 새로운 디플로이먼트 리소스를 준비한 다음 기존 디플로이먼트와 교체한 후 기존 것을 폐기하는 과정

```yaml
# echo-version-blue.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-version-blue
  labels:
    app: echo-version
    color: blue
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo-version
      color: blue
  template:
    metadata:
      labels:
        app: echo-version
        color: blue
    spec:
      containers:
        - name: echo-version
          image: gihyodocker/echo-version:0.1.0
          ports:
            - containerPort: 8080
```

```yaml
# echo-version-green.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-version-green
  labels:
    app: echo-version
    color: green
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo-version
      color: green
  template:
    metadata:
      labels:
        app: echo-version
        color: green
    spec:
      containers:
        - name: echo-version
          image: gihyodocker/echo-version:0.2.0
          ports:
            - containerPort: 8080
```

#### 셀렉터 레이블을 변경해 디플로이먼트 교체

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-version
  labels:
    app: echo-version
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: echo-version
    color: blue # 먼저 블루 버전으로 배포
```

그린 버전으로 변경

```bash
kubectl patch service echo-version -p '{"spec": {"selector": {"color": "green"}}}'
```

구버전과 신버전이 함께 존재하는 문제가 없으며 교체에 걸리는 시간도 제로에 가까움

#### 서비스 메시를 구현하는 Linkerd와 Istio

마이크로 서비스의 장점

- 각 서비스를 가장 적합한 언어나 프레임워크로 구현 가능
- 서비스 단위의 배포 가능
- 장애가 발생할 때 장애 범위 최소화

서비스 간 라우팅 관리나 통신 대상 서비스가 오류를 일으킬 경우에 대한 적절한 대처 등 운영 상 고려해야 할 점이 늘어나는 단점도 존재



이런 문제를 해결하기 위한 것이 서비스 메시

서비스 메시는 서비스와 네트워크 사이에 위치하는 것으로, 개발자가 애플리케이션 코드를 작성하지 않고도 고기능 라우팅 제어를 할 수 있다. 또 일부 서비스가 장애를 일으켜도 트래픽이 과도하게 몰리지 않도록 하는 서킷 브레이커 등 장애에 대한 내구성을 높이는 기능도 갖추고 있음

서비스 메시를 구축하기 위해 대표적으로 사용하는 오픈소스는 Linkerd와 Istio

Linkerd와 Istio 모두 블루-그린 배포나 카나리아 릴리스 적용 가능하며, 지표 수집 기능도 제공

## 참조

1. 도커/쿠버네티스를 활용한 컨테이너 개발 실전 입문