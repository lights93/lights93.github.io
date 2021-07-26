---
title: "JIT 컴파일의 세계로"
date: 2021-07-27
excerpt: "자바 최적화"
categories:
  - java
tags:
  - optimization
  - java
---

# 10. JIT 컴파일의 세계로



## 10.1 JITWatch란?

개발/데브옵스 담당자가 JITWatch를 이용하면, 애플리케이션 실행 중에 핫스팟이 실제로 바이트코드에 무슨 일을 했는지 이해하는 데 도움이 됨

JITWatch는 객관적인 비교에 필요한 측정값을 제공

JITWatch가 하는 일은, 실행 중인 자바 애플리케이션이 생성한 핫스팟 컴파일 상세 로그를 파싱/분석하여 그 결과를 GUI 형태로 보여주는 것

```
-XX:+UnlockDiagnosticVMOptions -XX:+TraceClassLoading -XX:+LogCompilation
```

### 10.1.1 기본적인 JITWatch 뷰

## 참조

1. Optimizing Java(자바 최적화)

2. https://github.com/AdoptOpenJDK/jitwatch
