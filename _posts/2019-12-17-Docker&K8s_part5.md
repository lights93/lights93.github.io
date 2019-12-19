---
layout: post
title: "쿠버네티스 입문"
date: 2019-12-17
excerpt: "쿠버네티스 입문"
tags: [Docker, Kubernetes]
comments: true

---

# 5. 쿠버네티스 입문

## 5.1 쿠버네티스란 무엇인가?

쿠버네티스: 컨테이너 운영을 자동화하기 위한 컨테이너 오케스트레이션 도구

많은 수의 컨테이너를 협조적으로 연동시키기 위한 통합 시스템

컨테이너를 다루기 위한 API 및 명령행 도구 제공

도커 호스트 관리/서버 리소스의 여유를 고려한 컨테이너 배치/스케일링/로드 밸런싱/헬스체크

### 쿠버네티스의 역할

높은 수준의 관리 기능을 제공하는 도구



## 5.3 쿠버네티스 주요 개념

- 노드(node): 컨테이너가 배치되는 서버
- 네임스페이스(namespace): 쿠버네티스 클러스터 안의 가상 클러스터
- 파드(pod): 컨테이너의 집합 중 가장 작은 단위, 컨테이너의 실행 방법 정의
- 레플리카세트(replicaset): 같은 스펙을 같은 파드를 여러 개 생성하고 관리하는 역할
- 디플로이먼트(deployment): 레플리카세트의 리비전 관리
- 서비스(service): 파드의 집합에 접근하기 위한 경로 정의 
- 인그레스(ingress): 서비스를 쿠버네티트 클러스터 외부로 노출시킴
- 컨피그맵(configmap): 설정 정보를 정의하고 파드에 전달
- 퍼시스턴트볼륨(persistentvolume): 파드가 사용할 스토리지의 크기 및 종류 정의
- 퍼시스턴트볼륨클레임(persistentvolumeclaim): 퍼시스턴트 볼륨을 동적으로 확보
- 스토리지클래스(storageclass): 퍼시스턴트 볼륨이 확보하는 스토리지의 종류 정의
- 스테이트풀세트(statefulset): 같은 스펙으로 모두 동일한 파드를 여러 개 생성하고 관리
- 잡(job): 상주 실행을 목적으로 하지 않는 파드를 여러 개 생성하고 정상적인 종료를 보장
- 크론잡(cronjob): 크론 문법으로 스케줄링 되는 잡
- 시크릿(secret): 인증 정보 같은 기밀 데이터 정의
- 롤(role): 네임스페이스 안에서 조작 가능한 쿠버네티스 리소스와 규칙 정의
- 롤바인딩(rolebinding): 쿠버네티스 리소스 사용자와 롤을 연결
- 클러스터롤(clusterrole): 클러스터 전체적으로 조작 가능한 쿠버네티스 리소스의 규칙 정의
- 클러스터롤바인딩(clusterrolebinding): 쿠버네티스 리소스 사용자와 클러스터롤을 연결
- 서비스 계정: 파드가 쿠버네티스 리소스를 조작할 때 사용하는 계정

## 5.4 쿠버네티스 클러스터와 노드

쿠버네티스 클러스터는 여러 리소스를 관리하기 위한 집합체

쿠버네티스 리소스 중 가장 큰 개념은 노드

노드는 컨테이너가 배치되는 대상

쿠버네티스 클러스터는 마스터와 노드의 그룹으로 구성

쿠버네티스는 노드의 리소스 사용 현황 및 배치 전략을 근거로 컨테이너를 적절히 배치

클러스터에 배치된 노드의 수,노드의 사양 등에 따라 배치할 수 있는 컨테이너수가 결정됨

```
kubectl get nodes
```

#### 마스터를 구성하는 관리 컨포넌트

- kube-apiserver: 쿠버네티스 API를 노출하는 컴포넌트 kubectl로부터 리소스를 조작하라는 지시를 받음
- etcd: 고가용성을 갖춘 분산 키-값 스토어. 쿠버네티스 클러스터의 백킹 스토어로 쓰임
- kube-scheduler: 노드를 모니터링하고 컨테이너를 배치할 적절한 노드를 선택
- kube-controller-manager: 리소스를 제어하는 컨트롤러 실행

## 5.5 네임스페이스

네임스페이스: 클러스터 안의 가상 클러스터

```bash
kubectl get namespace
```

네임스페이스마다 권한 설정 가능

## 5.6 파드

파드: 컨테이너가 모인 집합체의 단위

결합이 강한 컨테이너를 파드로 묶어 일괄 배포

파드는 노드에 배치

한 파드 안의 컨테이너는 모두 같은 노드에 배치

### 파드 생성 및 배포하기

파드 생성은 kubectl만 사용해도 가능하지만, 버전 관리 관점에서 yaml로 정의하는 것이 좋다.

매니페스트 파일: 쿠버네티스의 여러 가지 리소스를 정의하는 파일

```yaml
apiVersion: v1
kind: Pod # 리소스의 유형을 지정, 값에 따라 아래의 스펙 변경
metadata: # 리소스에 부여되는 메타데이터
  name: simple-echo
spec: # 리소스를 정의하기 위한 속성 (파드의 경우 containers 아래에 정의)
  containers:
    - name: nginx
      image: gihyodocker/nginx:latest # 도커 이미지
      env: # 환경 변수
        - name: BACKEND_HOST
          value: localhost:8080
      ports: # 컨테이너가 노출시킬 포트 지정
        - containerPort: 80
    - name: echo
      image: gihyodocker/echo:latest
      ports:
        - containerPort: 8080
```

```bash
kubectl apply -f simple-pod.yaml
```

### 파드다루기

```bash
kubectl get pod # 파드의 상태 확인
```

```bas
kubectl exec -it simple-echo sh -c nginx
```

```bash
kubectl logs -f simple-echo -c echo
```

```bash
kubectl delete pod simple-echo
```

```bash
kubectl delete -f simple-pod.yaml
```

#### 파드와 파드 안에 든 컨테이너 주소

파드에는 각각 고유의 가상 IP 주소 할당됨

파드에 할당된 가상 IP 주소는 해당 파드에 속하는 모든 컨테이너가 공유

같은 파드 안의 모든 컨테이너의 가상 IP주소가 같기 때문에 컨테이너 간의 통신 가능

파드 내에서는 localhost/다른 파드와는 가상 IP

## 5.7 레플리카세트

똑같은 정의를 갖는 파드를 여러 개 생성하고 관리하기 위한 리소스

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo
  labels:
    app: echo # 동일해야함 1번
spec:
  replicas: 3 # 파드의 복제본 수
  selector:
    matchLabels: # 동일해야함 2번
      app: echo
  template: # 파드 정의와 동일
    metadata:
      labels: # 동일해야함 3번
        app: echo
    spec:
      containers:
        - name: nginx
          image: gihyodocker/nginx:latest
          env:
            - name: BACKEND_HOST
              value: localhost:8080
          ports:
            - containerPort: 80
        - name: echo1
          image: gihyodocker/echo:latest
          ports:
            - containerPort: 8080
```

레플리카세트를 조작해 파드의 수를 조절 가능

웹 애플리케이션 같은 무상태 파드를 사용하기에 유리함

## 5.8 디플로이먼트

애플리케이션 배포의 기본 단위가 되는 리소스

레플리카세트를 관리하고 다루기 위한 리소스

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  labels:
    app: echo # 동일해야함 1번
spec:
  replicas: 3 # 파드의 복제본 수
  selector:
    matchLabels: # 동일해야함 2번
      app: echo
  template: # 파드 정의와 동일
    metadata:
      labels: # 동일해야함 3번
        app: echo
    spec:
      containers:
        - name: nginx
          image: gihyodocker/nginx:latest
          env:
            - name: BACKEND_HOST
              value: localhost:8080
          ports:
            - containerPort: 80
        - name: echo1
          image: gihyodocker/echo:latest
          ports:
            - containerPort: 8080
```

디플로이먼트는 레플리카세트의 리비전 관리 가능

```bash
kubectl apply -f dip-test.yaml --record # 어떤 명령을 실행했는지 기록을 남김
kubectl rollout history deployment echo # 디플로이먼트의 리비전 확인
```

#### 레플리카세트의 생애주기

디플로이먼트를 수정하면 레플리카세트가 새로 생성되고 기존 레플리카세트와 교체됨

- 파드 개수만 수정하면 레플리카세트가 새로 생성되지 않음
- 컨테이너 정의 수정시 새로 생성

#### 롤백 실행하기

```bash
kubectl rollout history deployment echo --revision=1 # 특정 리비전의 내용 확인
kubectl rollout undo deployment echo # 이전 버전으로 롤백
kubectl delete -f dep-test.yaml # 삭제
```

## 5.9 서비스

쿠버네티스 클러스터 안에서 파드의 잡합에 대한 경로나 서비스 디스커버리(API 주소가 바뀌는 경우에도 접속 대상을 바꾸지 않고 하나의 이름으로 접근할 수 있도록 하는 기능)를 제공하는 리소스

서비스의 대상이 되는 파드는 서비스에서 정의하는 레이블 셀렉터로 정해짐

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  type: NodePort
  selector: # 레이블 셀렉터
    app: echo
    release: summer
  ports:
    - name: http
      port: 80
```

```bash
kubectl get svc echo # service 정보 보기
```

서비스는 기본적으로 쿠버네티스 클러스터 안에서만 접근 가능

#### 서비스의 네임 레졸루션

쿠버네티스의 클러스터 DNS는 서비스명.네임스페이스명.svc.local

svc.local은 생략 가능

같은 네임 스페이스 라면 서비스명만으로도 참조 가능

#### CluseterIp 서비스

서비스 종류 중 기본 값

쿠버네티스 클러스터의 내부 IP주소에 서비스 공개 가능

#### NodePort 서비스

각 노드에서 서비스 포트로 접속하기 위한 글로벌 포트를 개방

서비스를 쿠버네티스 클러스터 외부로 공개 가능

#### LoadBalancer 서비스

로컬에서는 불가능

로드밸런서와 연동하기 위해 사용

#### ExternalName 서비스

쿠버네티스 클러스터에서 외부 호스트를 네임 레졸루션하기 위한 별명을 제공

## 5.10 인그레스

NodePort로 노출시키는 방법은 L4까지만 다룰 수 있개 때문에 L7 레벨의 제어는 불가능하다.

서비스를 이용한 클러스터 외부에 대한 노출과 가상 호스트 및 경로 기반의 정교한 HTTP 라우팅을 양립 가능

로컬 쿠버네티스 환경에서는 인그레스를 사용해 서비스 노출 불가

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echo
spec:
  rules:
    - host: test.host
      http:
        paths:
          - path: /
            backend:
              serviceName: echo
              servicePort: 80
```

요청 제어도 처리 가능

#### freshpod

쿠버네티스로 배포된 컨테이너의 이미지가 업데이트됐는지 탐지해 파드를 자동으로 다시 배포하는 도구

freshpod으로 업데이트하려면 imagePullPolicy를 Always(파드를 재시작할 때마다 항상 최신 이미지를 가져옴)으로 설정

#### kube-prompt

kubectl 명령 및 리소스의 자동 완성 기능 제공

#### 쿠버네티스 API

쿠버네티스 리스소를 생성,수정,삭제하는 작업 수행

apiVersion은 해당 작업에 사용되는 API의 종류를 나타내는 값



## 참조

1. 도커/쿠버네티스를 활용한 컨테이너 개발 실전 입문