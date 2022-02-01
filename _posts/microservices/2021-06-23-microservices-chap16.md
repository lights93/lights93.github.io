---
title: "쿠버네티스에 마이크로서비스 배포"
date: 2021-06-23
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 16. 쿠버네티스에 마이크로서비스 배포

쿠버네티스는 서비스 객체와 kube-proxy 기반의 검색 기능을 내장

넷플릭스 유레카와 같은 별도의 검색 서비스는 필요 X

쿠버네티스의 검색 서비스를 사용하면 넷플릭스 유레카와 함께 사용한 넷플릭스 리본과 같은 클라이언트 라이브러리가 필요 X



넷플릭스 유레카 기반의 검색 서버를 쿠버네티스로 대체하려면 소스코드 변경 필요

- 넷플릭스 유레카와 리본과 관련된 클라이언트 및 서버 구성을 구성 저장소인 config-repo에서 제거

- 유레카 서버로의 라우팅 규칙을 게이트웨이 서비스의 config-repo/gateway.yml 파일에서 제거

- 유레카 서버 프로젝트 제거

- 도커 컴포즈 파일과 그래들 파일에서 유레카 서버 제거

- 모든 유레카 클아이언트의 빌드 파일에서 spring-cloud-starter-netfix-eureka-client 제거

- 모든 유레카 클라이언트 통합 테스트에서 eureka.client.enable=false 속성 제거

- lb 프로토콜을 사용하는 스프링 클라우드 방식의 클라이언트 측 로드 밸런서 기반 라우팅을 게이트웨이 서비스 구성에서 제거

- 마이크로서비스 및 권한 부여 서버에서 사용하는 HTTP 포트를 8080에서 기본 HTTP 포트인 80으로 변경 필요

  ```yaml
  spring.profiles: docker
  server.port: 80
  ```

## Kustomize 소개

쿠버네티스 정의 파일(YAML)을 개발, 테스트, 준비, 상용 환경 등의 다양한 환경에 맞춰 사용자 정의할 때 사용하는 도구

베이스(base) 폴더에 공통 정의 파일을 저장하고 환경별 오버레이(overlay) 폴더에 환경별 추가 정보를 저장

환경별 정보의 예

- 사용할 도커 이미지의 버전
- 실행할 복제본의 개수
- CPU, 메모리 등의 자원 할당량

### 베이스 폴더에 공통 정의 설정

베이스 폴더에는 자원 관리자와 마이크로서비스를 위한 정의 파일

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product
  template:
    metadata:
      labels:
        app: product
    spec:
      containers:
      - name: pro
        image: hands-on/product-service # 마이크로서비스를 빌드하면 생성됨
        imagePullPolicy: Never # 도커 이미지를 도커 레지스트리에서 다운로드하지 않음
        env: # docker 스프링 프로필을 사용하도록 설정
        - name: SPRING_PROFILES_ACTIVE
          value: "docker"
        envFrom:
        - secretRef:
            name: config-client-credentials
        ports: # 기본 HTTP 포트인 80 사용
        - containerPort: 80
        resources:
          limits:
            memory: 350Mi
        livenessProbe: # 액추에이터의 정보 엔드포인트(/actuator/info)로 보내는 HTTP 요청에 기반, 200응답이 오지 않으면 재시작
          httpGet:
            scheme: HTTP
            path: /actuator/info
            port: 80
          initialDelaySeconds: 10 # 컨테이너가 시작한 후 검사 시작 전까지 쿠버네티스가 대기하는 시간
          periodSeconds: 10 # 검사 요청 간격
          timeoutSeconds: 2 # 검사를 실패한 것으로 처리하기 전에 응답을 기다리는 시간
          failureThreshold: 20 # 검사에 몇 번 실패하면 포기할 것인지 지정(포드 다시 시작)
          successThreshold: 1 # 실패한 검사를 다시 성공으로 간주하게 하려면 검사에 몇 번 성공해야 하는지 지정(라이브니스는 무조건 1)
        readinessProbe: #  액추에이터의 상태 점검(health) 엔드포인트로 보내는 HTTP 요청에 기반, 200응답일 때만 요청 보냄
          httpGet:
            scheme: HTTP
            path: /actuator/health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3 # 검사에 몇 번 실패하면 포기할 것인지 지정(요청을 보내지 않음)
          successThreshold: 1
```

검사 설정을 최적화하는 것은 어려운 작업

포드에 과도한 검사 요청을 보내지 않으면서도 포드의 가용성 변경에 쿠버네티스가 신속하게 반응할 수 있도록 균형을 맞추는 건 쉽지 않음

서비스 객체는 거의 동일

게이트웨이 마이크로서비스는 서비스 유형이 NodePort로 지정돼 있기 때문에 호스트의 31443 포트로 외부에 공개됨

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gateway
spec:
  type: NodePort
  selector:
    app: gateway
  ports:
    - port: 443
      nodePort: 31443
      targetPort: 8443
```

베이스 폴더에 있는 모든 것을 Kustomize 파일로 한데 묶음

```yaml
resources:
- auth-server.yml
- config-server.yml
- gateway.yml
- product-composite.yml
- product.yml
- recommendation.yml
- review.yml
- zipkin-server.yml
```

## 개발 및 테스트 환경을 위한 쿠버네티스 배포

개발 환경을 위한 자원 관리자 정의 파일

```yaml
bases:
- ../../base
resources:
- mongodb-dev.yml
- rabbitmq-dev.yml
- mysql-dev.yml
```

### 도커 이미지 빌드

일반적으로 도커 레지스트리에 이미지를 올리고 레지스트리에서 이미지를 가져오도록 쿠버네티스를 구성

하지만 로컬 단일 노드 클러스터를 사용하는 경우에는 도커 클라이언트가 미니큐브의 도커 엔진을 가리키도록 설정한 후 docker-compose build로 실행하는 게 간편함

### 쿠버네티스에 배포

마이크로서비스를 쿠버네티스에 배포하려면 먼저 네임스페이스, 컨피그 맵, 시크릿을 만들어야 함   



hand-on 네임스페이스를 생성하고 kubectl의 기본 네임스페이스로 설정   

```bash
kubectl create namespace hands-on
kubectl config set-context $(kubectl config current-context) --namespace=hands-on
```

모든 애플리케이션 구성은 구성 서버가 관리하는 구성 저장소에 보관되며, 구성 서버 접속을 위한 자격 증명과 암호화 키만 구성 저장소 밖에 저장됨

구성 서버는 암호화 키를 사용해 구성 저장소의 주요 정보를 암호화한 후 안전하게 디스크에 보관

구성 저장소와 암호화한 민감한 정보는 컨피그 맵에 저장

1. config-repo 폴더의 파일을 기반으로 구성 저장소를 위한 컨피그 맵을 생성

   ```bash
   kubectl create configmap config-repo --from-file=config-repo/ --save-config
   ```

2. 구성 서브를 위한 시크릿 생성

   ```bash
   kubectl create secret generic config-server-secrets \
   --from-literal=ENCRYPT_KEY=my-very-secure-encrypt-key \
   --from-literal=SPRING_SECURITY_USER_NAME=dev-usr \
   --from-literal=SPRING_SECURITY_USER_PASSWORD=dev-pwd \
   --save-config
   ```

3. 구성 서버 클라이언트를 위한 시크릿 생성

   ```bash
   kubectl create secret generic config-client-credentials \
   --from-literal=CONFIG_SERVER_USR=dev-usr \
   --from-literal=CONFIG_SERVER_PWD=dev-pwd \
   --save-config
   ```

4. -k 스위치로 Kustomize 활성화하고 dev 오버레이를 사용해 개발 환경에 마이크로서비스 배포

   ```bash
   kubectl apply -k kubernetes/services/overlays/dev
   ```

5. 디플로이먼트 및 포드가 동작할 때까지 기다림

   ```bash
   kubectl wait --timeout=600s --for=condition=ready pod --all
   ```

6. 개발 환경의 도커 이미지를 확인

   ```bash
   kubectl get pods -o json | jq .items[].spec.containers[].image
   ```

### 쿠버네티스 환경에 맞게 테스트 스크립트 수정

액추에이터 엔드포인트는 외부로 노출되지 않으므로 도커 컴포즈나 쿠버네티스 환경에선 다른 방식을 사용해 테스트 스크립트가  내부 엔드포인트에 접근해야 함

- 도커 컴포즈를 사용할 때 테스트 스크립트는 docker run 커맨드로 도커 컨테이너를 실행하며, 도커 컴포즈가 생성한 네트워크 내붕네서 actuator 엔드포인트를 호출
- 쿠버네티스를 사용할 때 테스트 스크립트는 적절한 쿠버네티스 내부 커맨드로 쿠버네티스 포드를 실행

#### 도커 컴포즈 환경에서의 내부 액추에이터 엔드포인트 접근 방법

```bash
EXEC="docker run --rm -it --network=my-network alpine"
```

#### 쿠버네티스 환경에서의 내부 액추에이터 엔드포인트 접근 방법

함수의 시작 부분에서 포드를 시작

```bash
echo "Restarting alpine-client..."
local ns=$NAMESPACE
if kubectl -n $ns get pod alpine-client > /dev/null ; then
    kubectl -n $ns delete pod alpine-client --grace-period=1
fi
kubectl -n $ns run --restart=Never alpine-client --image=alpine --command -- sleep 600
echo "Waiting for alpine-client to be ready..."
kubectl -n $ns wait --for=condition=Ready pod/alpine-client

EXEC="kubectl -n $ns exec alpine-client --"
```

```bash
kubectl -n $ns delete pod alpine-client --grace-period=1 # 종료시 포드 삭제
```

#### 도커 컴포즈 및 쿠버네티스 환경 구별 방법

```bash
if [ "$HOST" = "localhost" ] # localhost인지 확인하여 구별
then
    EXEC="docker run --rm -it --network=my-network alpine"
else
		...
```

### 디플로이먼트 테스트

테스트 실행 

네임스페이스를 삭제하면 네임스페이스 엤는 리소스가 재귀적으로 삭제 됨

```bash
kubectl delete namespace hands-on
```

## 준비 및 사용 환경을 위한 쿠버네티스 배포

- 쿠버네티스 클러스터 외부에서 자원 관리자를 실행해야 함. 
- 제한
  - 상용 환경에선 보안상의 이유로 actuator 엔드포인트 및 로그 레벨 등을 제한해야 함
  - 보안 관점에서 외부로 노출된 엔드포인트를 점검해야 함
  - 배포한 마이크로서비스의 버전을 추적할 수 있도록 도커 이미지에 태그를 지정해야 함
- 가용 리소스 확장: 고가용성 및 고부하 요구 사항을 충족하려면 디플로이먼트당 최소 2개의 포드를 실행해야 함.  또한 포드에서 시용할 수 있는 메모리와 CPU의 양을 늘리는 것도 고려해야 함
- 상용화 준비가 완료된 쿠버네티스 클러스터

### 소스 코드 수정

- prod라는 이름의 스프링 프로필을 config-repo 구성 저장소의 구성 파일에 추가

  ```yaml
  spring.profiles: prod
  ```

- prod 프로필에 다음 내용 추가

  - 일반 도커 컨테이너로 실행한 자원 관리자의 URL

    ```yaml
    spring.data.mongodb.host: 172.17.0.1
    spring.rabbitmq.host: 172.17.0.1
    spring.datasource.url: jdbc:mysql://172.17.0.1:3306/review-db
    ```

  - 로그 레벨을 warning 이상으로 설정

    ```yaml
    logging.level.root: WARN
    ```

  - HTTP를 통해 노출되는 actuator 엔드포인트는 쿠버네티스의 레디니스 프로브가 사용하는 info 엔드포인트와 health 엔드포인트뿐, 테스트 스크립트에서 사용하는 curcuitbreakerevents도 노출

    ```yaml
    management.endpoints.web.exposure.include: health,info,circuitbreakerevents
    ```

- prod overlay 폴더에는 각 마이크로서비스를 위해 다음과 같은 내용 적용

  - 모든 마이크로서비스의 도커 이미지 태그를 v1으로 지정하고 활성 스프링 프로필에 prod 추가

    ```yaml
    image: hands-on/product-composite-service:v1
    env:
    - name: SPRING_PROFILES_ACTIVE
      value: "docker,prod"
    ```

  - 구성 서버와 집킨은 구성을 구성 저장소에 저장하지 않으며, 구성 정보를 환경 변수로 정의해 디플로이먼트에 추가

    ```yaml
    env:
      - name: LOGGING_LEVEL_ROOT
        value: WARN
      - name: MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE
        value: "health,info"
      - name: RABBIT_ADDRESSES
        value: 172.17.0.1
    ```

  - kustomization.yml 파일에서는 prod overlay 폴더에 있는 파일과 base 폴더에 있는 대응 파일을 patchStrategicMerge 메커니즘을 사용해 병합

    ```yaml
    bases:
    - ../../base
    patchesStrategicMerge:
    - auth-server-prod.yml
    - config-server-prod.yml
    - gateway-prod.yml
    - product-composite-prod.yml
    - product-prod.yml
    - recommendation-prod.yml
    - review-prod.yml
    - zipkin-server-prod.yml
    ```

### 쿠버네티스에 배포

1. config-repo 폴더의 파일을 기반으로 구성 저장소를 위한 컨피그 맵을 생성

   ```bash
   kubectl create configmap config-repo --from-file=config-repo/ --save-config
   ```

2. 구성 서브를 위한 시크릿 생성

   ```bash
   kubectl create secret generic config-server-secrets \
   --from-literal=ENCRYPT_KEY=my-very-secure-encrypt-key \
   --from-literal=SPRING_SECURITY_USER_NAME=dev-usr \
   --from-literal=SPRING_SECURITY_USER_PASSWORD=dev-pwd \
   --save-config
   ```

3. 구성 서버 클라이언트를 위한 시크릿 생성

   ```bash
   kubectl create secret generic config-client-credentials \
   --from-literal=CONFIG_SERVER_USR=dev-usr \
   --from-literal=CONFIG_SERVER_PWD=dev-pwd \
   --save-config
   ```

4. 커맨드 기록에서 암호화 키와 암호를 삭제

   ```bash
   history -c;history -w
   ```

5. -k 스위치로 Kustomize 활성화하고 dev 오버레이를 사용해 개발 환경에 마이크로서비스 배포

   ```bash
   kubectl apply -k kubernetes/services/overlays/dev
   ```

6. 디플로이먼트 및 포드가 동작할 때까지 기다림

   ```bash
   kubectl wait --timeout=600s --for=condition=ready pod --all
   ```

7. 개발 환경의 도커 이미지를 확인

   ```bash
   kubectl get pods -o json | jq .items[].spec.containers[].image
   ```

## 롤링 업그레이드 수행

롤링 업그레이드는 새 쿠버네티스 포드로 새 버전의 마이크로서비스를 시작하는 것

새 포드에 문제가 없으면 쿠버네티스는 이전 버전을 종료함

롤링 업그레이드가 제대로 작동하려면 다른 서비스와의 통신에 사용하는 메시지 형식 및 API나 데이터베이스 구조가 이전 버전과의 호환성을 유지하고 있어야 함

### 롤링 업그레이드 준비

현재 버전 확인

```bash
kubectl get pod -l app=product -o jsonpath='{.items[*].spec.containers[*].image}'
```

새로 태그를 붙인다

### product 서비스를  v1에서 v2로 업그레이드

일정 시간동안 두 포드가 모두 요청 처리

기존 포드는 잠시 후 종료

기존 포드가 종료될 때 간헐적으로 503 발생

## 실패한 디플로이먼트 롤백

디플로이먼트 히스토리 확인 및 롤백

```bash
kubectl rollout history deploymeny product # 전체 확인
kubectl rollout history deplyment product --revision=2 # 상세 확인
kubectl rollout undo deployment product --to-revision=2 # 롤백
```



## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)