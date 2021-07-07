---
title: "쿠버네티스로 기존 인프라 대체"
date: 2021-06-24
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 17. 쿠버네티스로 기존 인프라 대체

서비스 검색 디자인 패턴을 구현한 넷플릭스 유레카를 쿠버네티스에서 제공하는 검색 서비스로 대체

스프링 클라우드 컨피그 서버는 쿠버네티스 컨피그 맵과 시크릿으로 대체

스프링 클라우드 게이트웨이는 동일한 방식의 에지 서버로 동작하는 쿠버네티스 인그레스 리소스로 대체   



인증서를 수동으로 관리하면 시간이 많이 걸리고 오류가 발생하기 쉬운데 Cert Mananger를 사용하면 이런 문제가 해결 됨

Cert Manager는 인그레스에 의해 외부로 노출되는 HTTPS 엔드포인트의 인증서가 만료됐을 때 이를 대체할 새 인증서를 자동으로 프로비저닌ㅇ함

Cert Manager는 Let's Encrpyt를 사용해 인증서를 발급하도록 구성됨   



쿠버네티스와 같은 플랫폼의 기능을 많이 사용한다고 해서 마이크로서비스 소스 코드가 플랫폼 의존적이어선 안 됨

## 스프링 클라우드 컨피그 서버 대체

쿠버네티스 컨피그 맵과 시크릿을 사용하면 지연 없이 더 빠르게 자동화된 통합 테스트를 실행 가능

### 스프링 클라우드 컨피그 서버를 대체하기 위한 소스 코드 변경

- spring-cloud/config-server 프로젝트 제거

  - settings.gradle 빌드 파일에서 프로젝트 제거
  - base와 overlays/prod 폴더에 있는 구성 서버 YAML 파일을 제거학서 kustomization.yaml에 있는 모든 선언 삭제

- 모든 마이크로서비스에서 구성 제거

  - build.gradle 빌드 파일에서 spring-cloude-stater-config 의존성 제거
  - 각 프로젝트의 src/main/resource 폴더에서 bootstrap.yml 파일 제거
  - 통합 테스트 설정에서 spring,cloud.config.enabled=false 속성 삭제

- config-repo 폴더의 구성 파일 변경

  - 쿠버네티스 시크릿으로 대체해야 하는 MongDB, MySQL, RabbitMQ 자격 증명이나 TLS 인증서 암호 등의 민감한 정보 삭제
  - 에지 서버 구성에서 구성 서버 API 경로 삭제

- kubernetes/services/base 폴더에 있는 쿠버네티스 디플로이먼트 리소스 정의 파일 변경

  - 볼륨으로 마운트된 컨피그 맵은 컨테이너 파일시스템의 폴더와 연결되며, 마이크로서비스는 이 폴더를 통해 컨피그 맵에 있는 구성 파일을 가져옴
  - 볼륨에 있는 궁성 파일 경로를 지정하는 SPRING_CONFIG_LOCATION 환경 변수 정의
  - 시크릿을 사용해 자원 관리자 접근을 위한 자격 증명 정의

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
          image: hands-on/product-service
          imagePullPolicy: Never
          env:
          - name: SPRING_PROFILES_ACTIVE
            value: "docker"
          - name: SPRING_CONFIG_LOCATION # 환경 변수에 속성 파일 경로 저장
            value: file:/config-repo/application.yml,file:/config-repo/product.yml
          envFrom: # 시크릿의 내용을 바탕으로 환경 변수로 설정
          - secretRef:
              name: rabbitmq-credentials
          - secretRef:
              name: mongodb-credentials
          ports:
          - containerPort: 80
          resources:
            limits:
              memory: 350Mi
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /actuator/info
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 20
            successThreshold: 1
          readinessProbe:
            httpGet:
              scheme: HTTP
              path: /actuator/health
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
            successThreshold: 1
          volumeMounts: # config-repo-volume 볼륨을 파일시스템의 /config-repo에 마운트
          - name: config-repo-volume
            mountPath: /config-repo
        volumes: # config-repo-product 컨피그 맵을 cofnig-repo-volume 볼륨에 매핑
        - name: config-repo-volume
          configMap:
            name: config-repo-product
  ```

## 스프링 클라우드 게이트웨이 대체

스프링 클라우드 게이트웨이를 쿠버네티스에서 제공하는 인그레스 리소스로 대체해 배포가 필요한 자원 서비스의 수를 줄임



스프링 클라우드 게이트웨이가 인그레스 리소스에 비해 풍부한 라우팅 기능을 제공하긴 하지만, 인그레스는 쿠버네티스 플랫폼에서 기본 제공하는 기능이며, Cert Manager를 사용해 인증서를 자동으로 프로비저닝하도록 확장 가능.  



인그레스 리소스는 OAuth나 OIDC를 지원하는 인그레스 컨트롤러 구현도 있음.  



게이트웨이에서 추가한 복합 상태 점검은 각 마이크로서비스의 디플로이먼트 리소스에 라이브니스 프로브와 레디니스 프로브를 정의해 대체 가능

###  스프링 클라우드 게이트웨이를 대체하기 위한 소스 코드 변경

- 쿠버네티스 디플로이먼트 리소스 정의 파일 변경

  - base와 overlays/prod 폴더에 있는 gateway.yml 파일을 제거하고 kustomization.yml 파일에서 gateway.yaml 선언 삭제

  - 인그레스 리소스 정의 파일 base/ingress-edge-server.yaml을 추가하고 base/kustomization.yml 파일에 참조 추가

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata: # 이름
      name: edge
    spec:
      tls: # 인그레스가 HTTPS를 사용하며, minikibe.me 호스트 이름에 대해 발급한 인증서를 사용한다고 지정
        - hosts:
          - minikube.me
          secretName: tls-certificate # 인증서는 tls-certificate라는 이름의 시크릿에 저장
      rules:
      - host: minikube.me # 호스트 이름에 대한 라우팅 규칙 설정
        http:
          paths: # 라우트 경로 정의
          - path: /oauth
            backend:
              serviceName: auth-server
              servicePort: 80
          - path: /product-composite
            backend:
              serviceName: product-composite
              servicePort: 80        
          - path: /actuator/health
            backend:
              serviceName: product-composite
              servicePort: 80
    ```

## 쿠버네티스 컨피그 맵, 시크릿, 인그레스 리소스를 사용한 테스트

### 배포 스크립트 분석

1. 마이크로서비스별로 컨피그 맵을 생성

   ```bash
   kubectl create configmap config-repo-product --from-file=config-repo/application.yml --from-file=config-repo/product.yml --save-config
   ```

2. 필수 시크릿 생성

   ```bash
   kubectl create secret generic rabbitmq-server-credentials \
       --from-literal=RABBITMQ_DEFAULT_USER=rabbit-user-dev \
       --from-literal=RABBITMQ_DEFAULT_PASS=rabbit-pwd-dev \
       --save-config
   ```

3. 자원 관리자를 위한 시크릿도 생성

   ```bash
   kubectl create secret generic rabbitmq-credentials \
       --from-literal=SPRING_RABBITMQ_USERNAME=rabbit-user-dev \
       --from-literal=SPRING_RABBITMQ_PASSWORD=rabbit-pwd-dev \
       --save-config
   ```

4. tls-certificate 시크릿에 인그레스를 위한 TLS 인증서를 저장하며, kubernetes/cert 폴더에 있는 자체 서명 인증서를 사용

   ```bash
   kubectl create secret tls tls-certificate --key kubernetes/cert/tls.key --cert kubernetes/cert/tls.crt
   ```

5. -k 스위치로 Kustomize를 활성화해 dev 오버레이 기반 개발 환경에 마이크로서비스를 배포

   ```bash
   kubectl apply -f kubernetes/services/overlays/dev
   ```

6. 디플로이먼트와 포드가 실행될 때까지 기다림

   ```bash
   kubectl wait --timeout=600s --for=condition=ready pod --all
   ```

### 배포 및 테스트 커맨드 실행

1. /etc/hosts 파일에 행을 추가해 minikube.me DNS 이름을 미니큐브 인스턴스의 IP 주소와 연결

   ```bash
   sudo bash -c "echo $(minikube ip) minikube.me | tee -a /etc/hosts"
   ```

2. 소스 코드로 도커 이미지 빌드

   ```bash
   eval $(minikube docker-env)
   ./gradlew build && docker-compose build
   ```

3. hands-on 네임스페이스 다시 생성하고 기본 네임스페이스로 설정

   ```bash
   kubectl create namespace hands-on
   kubectl config set-context $(kubectl config current-context) --namespace=hands-on
   ```

4. 배포 시작

   ```bash
   ./kubernetes/scripts/deploy-dev-env.bash
   ```

5. 테스트 시작

   ```bash
   HOST=minikube.me PORT=443 ./test-em-all.bas
   ```

6. OAuth/OIDC 접근 토큰을 가져오고 API 호출

   ```bash
   ACCESS_TOKEN=$(curl -k https://writer:secret@minikube.me/oauth/token -d grant_type=password -d username=magnus -d password=password -s | jq .access_token -r)
   curl -ks https://minikube.me/product-composite/2 -H "Authorization: Bearer $ACCESS_TOKEN" | jq .productId
   ```

## 인증서 프로비저닝 자동화

### Cert Manager 배포 및 Let's Encrypt 발급자 정의

cert-manager 설치

```bash
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.yaml
```

Let's Encrypt에는 다음과 같은 유형의 발급자가 있다.

- 준비환경: 많은 양의 단기 인증서가 필요한 개발 및 테스트 단계에서 사용한다. 준비 환경에서는 다량의 인증서를 만들 수 있지만 신뢰할 수 없는 root CA를 사용한다. 따라서 준비 환경 인증서는 웹 브라우저에서 사용하는 API나 웹 페이지를 보호할 수 없다. 웹 브라우저가 root CA를 신뢰하지 않기 때문에 준비 환경 인증서로 보호하고 있는 웹 페이지를 열면 오류가 발생한다.
- 상용 환경: 신뢰할 수 있는 root CA를 사용해 인증서를 발급한다. 따라서 웹 브라우저에서 신뢰하는 인증서를 발급할 때 사용한다.

Cert Manager와 Let's Encrypt가 인증서 프로비저닝을 위해 통신할 때는 ACME v2(Automated Certificate Management v2) 기반의 표준 프로토콜 사용

Let's Encrypt가 CA 역할을 하고 Cert Manager는 ACME 클라이언트 역할을 함

- http-01: CA는 임의의 명명한 파일에 다음과 같은 경로로 접근할 수 있게 해 달라고 ACME 클라이언트에게 요청

  URL: http://<도메인 이름>/.well-known/acme-challenge/<임의의 파일 이름> CA가 이 URL로 파일에 접근하면 도메인 소유권이 확인된다.

- dns-01: CA는 DNS 서버에 _acme-challenge.<도메인 이름>이라는 TXT 레코드를 생성하고 지정한 값을 넣어 달라고 ACME 클라이언트에게 요청한다. 보통은 DNS 제공 업체의 API를 사용해 레코드를 생성함

dns-01 방식의 자동화가 http-01 방식에 비해 어렵지만 인터넷에서 HTTP 엔드포인트에 연결할 수 없는 경우에 유용함

Let's Encrypt 준비 환경을 위한 발급자 정의

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-issuer-staging # 발급자 이름, 인증서에 프로비저닝할 때 인그레스에서 발급자를 참조하고자 사용
spec:
  acme:
    email: <your email address> # 인증서 만료나 계정과 관련된 문제가 있는 경우에 이메일로 연락
    server: https://acme-staging-v02.api.letsencrypt.org/directory # 준비 환경의 URL
    privateKeySecretRef: # 시크릿 이름
      name: letsencrypt-issuer-staging-account-key
    solvers: # http-01 방식으로 도메인 소유권을 확인한다고 선언
    - http01:
        ingress:
          class: nginx
```

정의파일 적용

```bash
kubectl apply -f kubernetes/services/base/letsencrypt-issuer-staging.yaml
kubectl apply -f kubernetes/services/base/letsencrypt-issuer-prod.yaml
```

### ngrok으로 HTTP 터널 생성

HTTP 터널 생성을 위해 다음 단계를 수행

1. HTTP 터널 생성
2. 확인한 호스트 이름을 환경 변수에 저장

### Cert Manager와 Let'S Encrypt를 사용한 인증서 프로비저닝

1. 인그레스 생성
2. 인증서를 인그레스에 프로비저닝
3. 프로비저닝이 완료되면 Cert Manager가 DNS 이름을 소유하고 있는지 http-01 방식으로 확인
4. Cert Manager는 인증서를 쿠버네티스에 저장하고 인그레스에서 지정한 이름으로 시크릿을 생성

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: edge-ngrok
  annotations:
    certmanager.k8s.io/issuer: "letsencrypt-issuer-staging" # Cert Manager에게 프로비저닝 요청
spec:
  tls:
  - hosts: # 사용할 HTTP 터널의 호스트 이름
    - xxxxxxxx.ngrok.io
    secretName: tls-ngrok-letsencrypt-certificate # 프로비저닝이 완료되면 시크릿에 인증서 저장
  rules:
  - host: xxxxxxxx.ngrok.io
    http:
      paths:
      - path: /oauth
        backend:
          serviceName: auth-server
          servicePort: 80
      - path: /product-composite
        backend:
          serviceName: product-composite
          servicePort: 80        
      - path: /actuator/health
        backend:
          serviceName: product-composite
          servicePort: 80
```

#### Let's Encrypt 준비 환경 사용

1. kubernetes/services/base/ingress-edge-server-ngrok.yaml 파일을 편집해 호스트 이름을 사용할 HTTP 터널의 호스트 이름으로 변경

2. 프로비저닝을 시작하기 전에 인증서의 프로비저닝을 모니터링하고자 별도의 터미널 창에서 watch 커맨드를 실행

   ```bash
   kubectl get cert --watch
   ```

3. 새 인그레스 정의를 적용해 프로비저닝을 시작한다.

   ```bash
   kubectl apply -f kubernetes/services/base/ingress-edge-server-ngrok.yml 
   ```

4. Cert Manager가 새 인그레스를 감지하고 Let's Encrypt 준비 환경을 발급자로 하는 인증서를 프로비저닝하며, 발급자는 ngrok으로 설정한 HTTP 터널을 통해 ACME v2 프로토콜을 사용

5. 잠시 후에 HTTP 터널을 실행하고 있는 터미널 창에서 http-01 방식의 확인 요청을 볼 수 있다

6. tls-ngrok-letsencrypt-certificate 인증서가 생성돼 인그레스에 명시된 tls-ngrok-letsencrypt-certificate라는 이름의 시크릿에 저장됨

7. 잠시 후 인증서의 READY 상태가 True로 변경되는데, 이는 인증서 프로비저닝이 완료돼 사용할 수 있다는 뜻

8. Let's Encrypt에서 프로비저닝한 인증서를 사용하려면 미니큐브 IP를 가리켜도록 ngrok 호스트 이름을 리디렉션해야 함

9. /etc/hosts 파일을 편집해 앞 절에서 추가한 줄의 minikube.me 뒤에 HTTP 터널의 호스트 이름을 추가

10. keytool 커맨드를 사용해 HTTP 터널의 호스트 이름에 대한 인증서를 확인

11. 인증서는 HTTPS 터널의 호스트 이름에 대해 발급

12. 테스트 스크립트 실행

생략...

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)
- https://cert-manager.io/docs/installation/kubernetes/
- https://ngrok.com/