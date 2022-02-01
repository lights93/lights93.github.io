---
title: "서비스 메시를 사용해 관찰 가능성 및 관리 편의성 개선"
date: 2021-07-05
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 18. 서비스 메시를 사용해 관찰 가능성 및 관리 편의성 개선

## 이스티오를 이용한 서비스 메시 소개

서비스 메시는 마이크로서비스와 같은 서비스 간의 통신을 제어하고 관찰하는 인프라 계층

서비스 메시는 마이크로서비스 사이의 전체 내부 통신을 제어하고 모니터링해 관찰 가능성, 보안, 정책 시행, 탄력성, 트래픽 관리 등의 기능을 구현

서비스 메시의 핵심 컴포넌트는 서비스 메시에 속한 모든 마이크로서비스에 삽입되는 경량 **프록시(proxy)**며, 마이크로서비스 안팎의 모든 트래픽은 프록시 컴포넌트를 통과하도록 구성됨.  



서비스 메시의 **컨트롤 플레인(control plane)**은 프록시의 공개API를 사용해 런타임에 프록시 컴포넌트를 구성하고, 원격 측정(telemetry) 데이터를 수집하며, 수집한 데이터를 이용해 서비스 메시의 트래픽 흐름을 시각화함.  



서비스 메시에는 **데이터 플레인(data plane)**도 있다. 데이터 플레인은 서비스 메시에 속한 전체 마이크로서비스의 프록시 컴포넌트와 외부에서 들어오는 트래픽, 서비스 메시에서 나가는 트래픽을 처리하는 별도의 컴포넌트로 구성됨.  



이스티오는 쿠버네티스 등의 다양한 환경에서 배포 가능

쿠버네티스에 이스티오를 배포하면 별도의 쿠버네티스 네임스페이스인 istio-system에 런타임 컴포넌트가 배포됨

쿠버네티스는 **CRD(Custom Resources Definition)**로 새 객체를 추가하면 쿠버네티스 API를 확장할 수 있는데, 이스티오는 자체 CRD 집합으로 이스티오 객체를 추가해 이스티오의 동작 방식을 구성한다,

또한 이스티오의 CLI 도구인 istioctl은 서비스 메시에 참가하는 마이크로서비스에 이스티오 프록시를 삽입.  



운영자는 원하는 상태를 정의하고자 라우팅 규칙 등을 선언한 이스티오 객체를 만들며, 이를 위해 쿠버네티스 API 서버를 이용

컨트롤 플레인은 이런 객체를 확인하고 라우팅 규칙 구성 등의 원하는 상태에 따른 작업 수행을 위해 데이터 플레인은 프록시로 커맨드를 전송함

프록시는 마이크로서비스 간의 통신을 처리하며, 원격 측정 데이터를 컨트롤 플레인으로 전송한다.

원격 측정 데이터는 서비스 메시에서 진행중인 작업을 시각화하는 다양한 컨트롤 플레인 컴포넌트에서 사용한다

### 기존 마이크로서비스에 이스티오 프록시 삽입

마이크로서비스 포드를 이스티오 기반의 서비스 메시에 합류시키려먼 이스티오 프록시를 실행하는 컨테이너를 포드에 추가해 이스티오 프록시를 삽입해야 함

> 이스티오 프록시처럼 기본 컨테이너를 지원할 목적으로 포드에 추가되는 컨테이너를 사이드카라고 함

다음 커맨드를 사용해 기존 디플로이먼트에 객체의 포드에 이스티오 프록시를 삽입

```bash
kubectl get deployment sample-deployment -o yaml | istioctl kube-inject -f - | kubectl apply -f -
```

1. kubectl get deployment: 쿠버네티스 API 서버에서 sample-deployment라는 이름의 디플로이먼트 정의를 가져와서 YAML 형식으로 반환
2. istioctl kube-inject: 디플로이먼트를 제어하는 포드에 이스티오 프록시 컨테이너를 추가하며, 디플로이먼트에 포함된 기존 컨테이너의 수신 및 발신 트래픽이 이스티오 프록시를 통과하도록 구성을 업데이트
3. kubectl apply: 업데이트한 구성 적용

### 이스티오의 API 객체 소개

- Gateway: 서비스 메시로 들어오는 트래픅과 나가는 트래픽의 처리 방법을 구성
- VirtualService: 서비스 메시의 라우팅 규칙을 정의
- DestinationRule: 버추얼 서비스에 의해 특정 서비스로 라우팅되는 트래픽에 대한 정책, 규칙을 정의
- Policy: 요청 인증 방법 정의

### 이스티오의 런타임 컴포넌트 소개

다음과 같은 런타임 컴포넌트로 이스티오 컨트롤 플레인 구성

- Pilot: 전체 사이드카로 업데이트된 서비스 메시 구성을 전달
- Mixer: 2개의 런타임 컴포넌트로 구성됨
  - Policy - 인증 권한 부여, 속도 제한, 할당량 등의 네트워크 정책을 시행
  - Telemetry - 원격 측정 정보를 수집해 프로메테우스 등으로 보냄
- Gallery: 구성 정보를 수집, 검증하고 컨트롤 플레인의 다른 이스티오 컴포넌트에게 배포
- Citaldel: 내부에서 사용하는 인증서의 발급, 교체르 담당
- Kiali: 서비스 메시에 관찰 가능성을 제공하고 서비스 메시에서 진행 중인 작업을 시각화함(오픈소스 프로젝트임)
- Prometheus: 성능 메트릭 등의 시계열 기반 데이터를 수집 및 저장(오픈소스 프로젝트임)
- Grafana: 프로메테우스에서 수집한 성능 메트릭 및 기타 시계열 관련 데이터를 시각화함(오픈소스 프로젝트임)
- Tracing: 분산 추적 정보를 처리하고 시각화함 (오픈소스 프로젝트인 예거(Jaeger) 기반)

다음과 같은 런타임 컴포넌트로 데이터 플레인 구성

- **인그레스(ingress) 게이트웨이**: 서비스 메시로 들어오는 트래픽 처리
- **이그레스(egress) 게이트웨이**: 서비스 메시로 나가는 트래픽 처리
- 모든 포드에 이스티오 프록시가 사이드카로 삽입됨

### 마이크로서비스 환경의 변경 사항

- 이스티오 인그레스 게이트웨이를 에지 서버로 사용해 쿠버네티스의 인그레스 리소스 대체 가능
- 이스티오에서 제공하는 예거 컴포넌트로 집킨을 대채헤 분산 추적 가능



#### 이스티오 인그레스 게이트웨이를 에지 서버로 사용해 쿠버네티스 인그레스 리소스 대체

인그레스 리스소로는 이스티오가 제공하는 세분화된 라우팅 규칙을 처리할 수 없음

필요 없어진 쿠버네티스 인그레스 리소스 정의 파일 제거

이스티오 인그레스 게이트웨이에 접근하기 위해선 쿠버네티스 인그레스 리소스에 접근할 때 사용한 IP 주소가 아닌 다른 IP 주소를 사용해야 함

#### 시스템 환경을 단순화하고 예거로 집킨 대체

집킨 서버를 예거로 대체하면 마이크로서비스 환경이 단순해짐

- org.springframework.cloud:spring-cloud-starter-zipkin에 대한 의존성 제거
- 도커 컴포즈 파일에서 집킨 서버 정의 제거
- 집킨을 위한 쿠버네티스 정의 파일을 제거

## 쿠버네티스 클러스터에 이스티오 배포

1. 이스티오 다운로드
2. 미니큐브 인스턴스 동작 확인
3. 이스티오가 제공하는 CRD를 쿠버네티스에 설치
4. 쿠버네티스에 이스티오 데모 구성 설치
5. 이스티오 디플로이먼트를 사용할 수 있을 때까지 기다림
6. 예거와 그라파나 URL이 있는 키알리 컨피그 맵 생성

### 이스티오 서비스에 대한 접근 설정

이스티오 인그레스 게이트웨이는 유형이 LoadBalancer인 쿠버네티스 서비스로 구성됨



이스티오의 HTTPS 기반 라우팅은 포트 번호를 허용하지 않음, 즉 기본 HTTPS(443) 포트로만 이스티오 인그레스 게이트웨이 접근 가능

미니큐브의 minikube tunnel로 로컬 로드 밸런서를 시뮬레이션 가능

DNS 이름으로 클러스터 내부의 쿠버네티스 서비스에 접근 가능

DNS 이름은 이름 지정 규칙([service-name].{namespace}.svc.cluster.local)을 따른다

ex. kiali.istio-system.svc.cluster.local   



미니큐브 터널 설정

1. 로컬에서 쿠버네티스 서비스에 접근하고자 별도의 터미널 창에서 다음 커맨드를 실행(minikube tunnel)
2. minikube.me를 이스티오 인그레스 게이트웨이의 IP 주소와 연결
   1. minikube tunnel 커맨드에 의해 접근이 가능해진 이스티오 인그레스 게이트웨이의 IP 주소를 INGRESS_HOST라는 이름의 환경 변수에 저장
   2. minikube.me가 이스티오 인그레스 게이트웨이를 가리키도록 /etc/hosts/ 파일을 업데이트
3. 키알리, 예거, 그라파나에 접근할 수 있는지 확인

## 서비스 메시 생성

### 소스 코드 변경

#### 이스티오 프록시를 삽입하도록 배포 스크립트 업데이트

```bash
kubectl get deployment auth-server product product-composite recommendation review -o yaml | istioctl kube-inject -f - | kubectl apply -f -
waitForPods 5 'version=latest' # 완료될 때까지 기다림
kubectl wait --timeout=120s --for=condition=Ready pod --all
```

#### 쿠버네티스 정의 파일의 구조 변경

개발 환경에선 단일 버전의 마이크로서비스만 실행하므로 세 폴더 모두가 기본 폴더가 되도록 Kustomization 파일을 업데이트함

```yaml
bases:
- ../../base/deployments
- ../../base/services
- ../../base/istio
```

#### 이스티오를 위한 쿠버네티스 정의 파일 추가

이스티오 게이트웨이 파일 설정

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata: # 게이트웨이 이름
  name: hands-on-gw
spec:
  selector: # 내장된 인그레스 게이트웨이가 게이트웨이 리소스를 처리하도록 지정
    istio: ingressgateway
  servers:
  - hosts:
    - "minikube.me"
    port:
      number: 443
      name: https
      protocol: HTTPS
    tls: # 인증서 및 개인 키의 위치를 지정
      mode: SIMPLE
      serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
      privateKey: /etc/istio/ingressgateway-certs/tls.key
```

버추얼 서비스 객체

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-composite-vs
spec:
  hosts: # minikube.me로 전송된 요청을 라우팅하도록 지정
  - "minikube.me"
  gateways:
  - hands-on-gw
  http: # /product-composite로 시작하는 요청을 product-composite 쿠버네티스 서비스로 라우팅
  - match:
    - uri:
        prefix: /product-composite
    route:
    - destination:
        port:
          number: 80
        host: product-composite
```

### 커맨드를 실행해 서비스 메시 생성

1. 도커 이미지 빌드

   ```bash
   eval $(minikube docker-env)
   ./gradlew build && docker-compose build
   ```

2. hands-on 네임스페이스를 다시 생성하고 기본 네임스페이스로 지정

   ```bash
   kubectl delete namespace hands-on
   kubectl create namespace hands-on
   kubectl config set-context $(kubectl config current-context) --namespace=hands-on
   ```

3. 스크립트 실행

   ```bash
   ./kubernetes/scripts/deploy-dev-env.bash
   ```

4. 배포가 완료되면 각 마이크로서비스 포드에 속한 컨테이너가 2개인지 확인

   ```bash
   kubectl get pods
   ```

5. 테스트 실행

   ```bash
   ./test-em-all.bash
   ```

6. 수동으로 API 실행 가능

## 서비스 메시 관찰

마이크로서비스 구성을 변경하면 상태 점검 엔드포인트와 같은 액추에이터 엔드포인트의 포트 변경 가능

모든 마이크로서비스를 위한 공통 구성 파일인 config-repo/application.yml 파일에 다음 행 추가

```yaml
management.server.port: 4004
```



도커 컴포즈 테스트를 위해 유지하고 있는 스프링 클라우드 게이트웨이는 같은 포트를 계속해서 사용해 상태 점검과 API 요청을 처리

같은 포트를 사용하도록 config-repo/gateway.yml 구성 파일에 다음 행 추가

```yaml
management.server.port: 8443
```

## 서비스 메시 보안

### HTTPS와 인증서로 외부 엔드포인트 보호

인그레스 게이트웨이의 구성 확인

### OAuth 2.0/OIDC 접근 토큰을 사용한 외부 요청 인증

인증을 사용하려면 이스티오 policy 객체를 만들어서 보호 대상과 신뢰할 수 있는 접근 토큰 발급자를 지정해야 함

```yaml
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "jwt-authentication-policy"
spec:
  targets: # product-composite 마이크로서비스에 대한 용청에 인증 검사를 수행하도록 지정 
  - name: product-composite
  peers:
  - mtls:
      mode: PERMISSIVE
  origins: # origins 목록은 OAuth 2.0/OIDC 공급자를 지정하며, 각 공급자의 이름과 JSON 웹 키 세트의 URL이 입력됨
  - jwt:
      issuer: "http://auth-server.local"
      jwksUri: "http://auth-server.hands-on.svc.cluster.local/.well-known/jwks.json"
  principalBinding: USE_ORIGIN

```



product-composite 마이크로서비스와 이스티오 인그레스 게이트웨이 중 어디에서 유효하지 않은 요청을 기버후는지 구분하는 가장 쉬운 방법은 접근 토큰 없이 요청을 수행한 후 반환된 오류 메시지를 확인하는 것

### 상호 인증을 사용한 내부 통신 보호

생략

## 서비스 메시의 탄력성 확보

이스티오로 서비스 메시의 일시적인 결함 처리

이스티오는 시간 초과, 재시도 등의 메커니즘과 서킷 브레이커의 일종인 **이상 탐지(outlier detection)** 기능을 제공

이스티오가 제공하는 탄력성과 관련된 기능 중에는 기존 서비스 메시에 결함이나 지연을 삽입하는 기능도 있음

마이크로서비스의 복원 기능이 예상대로 작동하는지 확인할 때는 필요에 따라 지연을 삽입하는 기능이 중요

### 결함을 삽입해 탄력성 테스트

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-vs
spec:
  hosts:
    - product
  http:
  - route:
    - destination:
        host: product
    fault: # 서비스로 전송된 요청의 20%가 HTTP 상태 코드 500에 의해 중단되도록 정의
      abort:
        httpStatus: 500
        percent: 20
```

### 지연을 삽입해 탄력성 테스트

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-vs
spec:
  hosts:
    - product
  http:
  - route:
    - destination:
        host: product
    fault: # 모든 요청을 3초간 지연 시킴
      delay:
        fixedDelay: 3s
        percent: 100
```

## 비가동 시간 없이 배포 수행

쿠버네티스의 롤링 업그레이드 메커니즘을 사용하면 전체 프로세스를 자동화할 수는 있지만, 모든 사용자를 새 버전으로 라우팅하지 않고는 새 버전을 테스트 불가능

이스티오는 새 버전을 배포하면 처음에는 기존 버전으로 모든 사용자를 라우팅하지만, 이후로는 이스티오의 세분화된 라우팅 메커니즘을 사용해 사용자를 새 버전과 이전 버전으로 라우팅하는 방식을 제어할 수 있다.

- **카나리아 배포(canary deploy)**: 선택된 테스트 사용자 그룹만 새 버전으로 라우팅하고 대다수의 사용자를 이전 버전으로 라우팅하는 배포 방식, 테스트에 참여한 사용자가 새 버전을 승인하면 블루/그린 배포를 사용해 일반 사용자도 새 버전으로 라우팅
- **블루/그린 배포(blue/green deploy)**: 일반적으로 블루/그린 배포는 모든 사용자를 블루 버전이나 그린 버전으로 전환하는 것을 의미하는데, 두 버전 중 하나는 새 버전이고 하나는 이전 버전. 새 버전으로 전환한 후 문제가 발생했을 때 이전 버전으로 다시 전환하는 것이 매우 간단함

이런 유형의 업그레이드 전략에는 이전 버전과 업그레이드 버전 사이의 호환성이 전제 조건이라는 점을 명심해야 함

이런 전략을 사용할 때는 다른 서비스와 통신할 때 사용하는 API 및 메시지 형식과 데이터베이스 구조가 서로 호환돼야 함

### 소스 코드 변경

#### 여러 버전의 마이크로서비스를 위한 서비스 및 디플로이먼트 객체

여러 버전의 마이크로서비스를 동시에 실행하려면 각 디플로이먼트 객체 및 포드 이름이 달라야 함

그러나 서비스 객체는 마이크로서비스별로 하나씩만 있어야 함.  



디플로이먼트 객체 및 포드 버전에 따른 이름 지정을 위해 kustomization.yml 파일에 namesuffix 필드를 추가

```yaml
nameSuffix: -v1 # 생성된 모든 객체의 이름에 -v1 접미어가 붙게 됨
bases:
- ../../../base/deployments
patchesStrategicMerge:
- auth-server-prod.yml
- product-composite-prod.yml
- product-prod.yml
- recommendation-prod.yml
- review-prod.yml
```

#### 이스티오를 위한 쿠버네티스 정의 파일 추가

각 마이크로서비스에는 이전 버전과 새 버전으로 라우팅되는 트래픽 비율을 정하는 버추얼 서비스 객체가 있다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: product-vs
spec:
  hosts:
    - product
  http: # X-group 헤더 값이 test인 요청을 new 하위 그룹으로 라우팅하도록 지정
  - match:
    - headers:
        X-group:
          exact: test
    route:
    - destination:
        host: product
        subset: new
  - route:
    - destination:
        host: product
        subset: old
      weight: 100
    - destination:
        host: product
        subset: new
      weight: 0
```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: product-dr
spec:
  host: product
  subsets:
  - name: old
    labels:
      version: v1 # 특정 버전으로 트래픽을 라우팅하려면 버전을 식별할 수 있어야 하는데, 이스티오 설명문에선 포드에 version 레이블을 붙이는 것을 권장
  - name: new
    labels:
      version: v2
```

### v1 및 v2 버전의 마이크로서비스 배포

### 모든 트래픽이 v1 버전의 마이크로서비스로 전달되는지 확인

### 카나리아 테스트 실행

### 블루/그린 테스트 실행

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)

  