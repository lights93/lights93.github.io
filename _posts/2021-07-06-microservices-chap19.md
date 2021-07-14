---
title: "EFK 스택을 사용한 로깅 중앙화"
date: 2021-07-06
excerpt: "마이크로서비스"
categories:
  - microservice
tags:
  - microservice
---

# 19. EFK 스택을 사용한 로깅 중앙화

## 플루언티드 구성

### 플루언티드 소개

로그스태시는 자바VM에서 실행되기 때문에 비교적 많은 양의 메모리를 사용

로그스태시보다 메모리 소비가 훨씬 적은 오픈 소스 솔루션 중 하나가 플루언티드

플루언티드는 쿠버네티스 프로젝트를 곤리하는 클라우드 네이티브 컴퓨팅 재단에서 고나리

플루언티드는 C와 루비 언어를 함께 사용해 만들었다.  



플루언티드의 로그 레코드 이벤트 구성 요소

- time 필드: 로그 레코드의 작서 일시를 설명
- tag 필드: 로그 레코드 유형을 식별하며 플루언티드 라우팅 엔진에서 로그 레코드 처리 방법을 결정할 때 사용
- record 필드: JSON 객체 형태로 실제 로그 정보를 담는다

   

플루언티드 구성 파일 핵심 요소

- ```<source>```: 플루언티드가 어디에서 로그 레코드를 수집하는지 설명하고자 source요소 사용, ex. 태그를 지정해 쿠버네티스에서 실행중인 컨테이너에서 온 로그 레코드라는 것을 표시 가능
- ```<filter>```: 로그 레코드를 처리하고자 filter 요소 사용 ex. 관심 있는 레코드를 별도 필드로 추출하여 엘라스틱서치에서 따로 검색 가능
- ```<match>```: 출력 요소로는 두 가지 주요 작업을 수행
  - 처리한 로그 레코드를 엘라스틱서치 등의 대상으로 보냄
  - 라우팅으로 로그 레코드 처리 방법 결정, ```<rule>``` 요소로 라우팅 룰 표현

### 플루언티드 구성

kubernetes.conf

- source 요소: 컨테이너 로그 파일과 kubelet 및 도커 데몬처럼 쿠버네티스 외부에서 실행되는 프로세스의 컨테이너 로그를 테일링
- filter 요소: 컨테이너 이름이나 소속 네임스페이스와 같은 쿠버네티스 관련 정보로 쿠버네티스에서 실행 중인 컨테이너에서 수집한 로그 레코드를 보강

fluent.conf

- ```@include``` 문: kubernetes.conf 등의 다른 구성 파일을 포함시킴
- 출력 요소: 로그 레코드를 엘라스틱서치로 전송

자체 구성 파일에서 다루는 내용

- 마이크로서비스에서 온 스프링 부트 형식의 로그 레코드를 감지하고 구문 분석
- 여러 줄로 된 스택 추적을 처리
- 같은 포드에서 실행 중인 마이크로서비스에서 생성한 로그 레코드에서 istio-proxy 사이드카의 로그 레코드를 분리

fluentd-hands-on.conf

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-hands-on-config
  namespace: kube-system
data: # 플루언트 구성이 data 요소에 들어감
  fluentd-hands-on.conf: |

    <match kubernetes.**istio**> # kubernetes로 시작하고 태그 이름 중에 istio라는 문자열이 있는지 찾음
      @type rewrite_tag_filter # 태그 이름을 변경한 후 플루언티드 라우팅 엔진으로 로그 레코드를 재전송
      <rule>
        key log # 로그 레코드에서 log 필드를 찾도록 key 필드를 log로 설정
        pattern ^(.*)$ # 모든 문자열과 일치
        tag istio.${tag} # 태그 앞에 istio 문자열을 붙임
      </rule>
    </match>

    <match kubernetes.**hands-on**>
      @type rewrite_tag_filter
      <rule>
        key log
        pattern ^(.*)$
        tag spring-boot.${tag} # spring-boot 태그 붙임
      </rule>
    </match>

    <match spring-boot.**>
      @type rewrite_tag_filter
      <rule> # log 필드가 타임스탬프로 시작하는지 확인
        key log
        pattern /^\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}.*/
        tag parse.${tag}
      </rule>
      <rule> # log 필드가 타임스탬프로 시작하지 않으면 스택 추적 로그 레코드로 처리
        key log
        pattern /^.*/
        tag check.exception.${tag}
      </rule>
    </match>

    <match check.exception.spring-boot.**>
      @type detect_exceptions # 여러 줄로 된 예외를 스택 추적을 포함하는 한 줄의 로그 레코드로 취합하고자 사용
      languages java
      remove_tag_prefix check # 무한 루프를 제거하기 위해 check 접두어 제거
      message log
      multiline_flush_interval 5
    </match>

    <filter parse.spring-boot.**> # 로그 레코드르 재전송하지 않으며, 로그 레코드 태그와 일치하는 구성 파일의 다음 요소로 전달
      @type parser
      key_name log
      time_key time
      time_format %Y-%m-%d %H:%M:%S.%N
      reserve_data true
      # Sample log message: 2019-08-10 11:27:20.497 DEBUG [product-composite,677435df95da585a22d6d6a62db84082,59f32cef360398a4,true] 1 --- [or-http-epoll-1] s.m.u.h.GlobalControllerExceptionHandler : Returning HTTP status: 404 NOT_FOUND for path: /product-composite/13, message: Product Id: 13 not found 
      # <time> <spring.level> <log> 등을 추출
      format /^(?<time>\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3})\s+(?<spring.level>[^\s]+)\s+(\[(?<spring.service>[^,]*),(?<spring.trace>[^,]*),(?<spring.span>[^,]*),[^\]]*\])\s+(?<spring.pid>\d+)\s+---\s+\[\s*(?<spring.thread>[^\]]+)\]\s+(?<spring.class>[^\s]+)\s*:\s+(?<log>.*)$/
    </filter>
```

## 쿠버네티스에 EFK 스택 배포

### 마이크로서비스 빌드 및 배포

1. 도커 이미지 빌드

   ```bash
   ./gradlew build && docker-compose build
   ```

   

2. 네임스페이스 다시 생성 및 기본 네임스페이스로 변경

3. deploy-dev-env.bash로 배포 시작

4. minikube tunnel 실행

5. 테스트 실행

### 엘라스틱서치와 키바나 배포

logging 네임 스페이스에 배포

#### 정의 파일 분석

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  selector:
    matchLabels:
      component: elasticsearch
  template:
    metadata:
      labels:
        component: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch-oss:7.3.0 # 엘라스틱 서치 공식 오픈소스 7.3.0버전
        env:
        - name: discovery.type
          value: single-node
        ports:
        - containerPort: 9200
          name: http
          protocol: TCP
        resources: # 많은 양의 메모리 할당 -> 성능 향상
          limits:
            cpu: 500m
            memory: 2Gi
          requests:
            cpu: 500m
            memory: 2Gi
```

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kibana
spec:
  selector:
    matchLabels:
      run: kibana
  template:
    metadata:
      labels:
        run: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana-oss:7.3.0
        env: # 엘라스틱서치 주소 할당
        - name: ELASTICSEARCH_URL
          value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
          name: http
          protocol: TCP
```

#### 배포 커맨드 실행

1. 네임스페이스 생성
2. 도커 이미지 끌어옴
3. 일라스틱서치 배포 후 포드 준비되길 기다림
4. 일라스틱 서치 작동 중인지 확인
5. 키바나 배포 후 포드가 준비되길 기다림
6. 키바나 작동 중인지 확인

### 플루언티드 배포

플루언티드 프로젝트에서 도커 허브에 올린 도커이미지화 깃허브의 플루언티드 프로젝트에 있는 구성 파일을 사용해 배포

각 플루언티드 포드는 동작 중인 노드에서 실행되는 프로세스 및 컨테이너에서 로그 출력을 수집

#### 정의 파일 분석

```dockerfile
FROM fluent/fluentd-kubernetes-daemonset:v1.4.2-debian-elasticsearch-1.1

RUN gem install fluent-plugin-detect-exceptions -v 0.0.12 \ # fluent-plugin-detect-exceptions 설치
 && gem sources --clear-all \
 && rm -rf /var/lib/apt/lists/* \
           /home/fluent/.gem/ruby/2.3.0/cache/*.gem
```

데몬 셋 정의 파일

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet # 데몬셋 지정
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    app: fluentd
    version: v1
spec:
  template:
    metadata:
      labels:
        app: fluentd
        version: v1
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: hands-on/fluentd:v1
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST # 엘라스틱서치 호스트 지정
            value: "elasticsearch.logging"
          - name:  FLUENT_ELASTICSEARCH_PORT # 엘라스틱서치 포트 지정
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          - name: FLUENT_ELASTICSEARCH_SED_DISABLE
            value: "true"
        resources:
          limits:
            cpu: 500m
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts: # 여러 개의 볼륨, 즉 파일 시스템을 포드에 매핑, 테일링 및 수집할 로그 파일들
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: journal
          mountPath: /var/log/journal
          readOnly: true
        - name: fluentd-extra-config
          mountPath: /fluentd/etc/conf.d
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: journal
        hostPath:
          path: /run/log/journal
      - name: fluentd-extra-config
        configMap: # 로그 레코드를 처리하는 방법을 지정한 구성 파일 매핑
          name: "fluentd-hands-on-config"
      terminationGracePeriodSeconds: 30

```

## EFK 스택 실험

생략

## 참고

- 마이크로서비스(http://www.acornpub.co.kr/book/microservices-spring)

- https://github.com/fluent/fluentd-kubernetes-daemonset/blob/master/fluentd-daemonset-elasticsearch.yaml

  