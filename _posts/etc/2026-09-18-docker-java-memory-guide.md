---
title: Docker에서 Java 애플리케이션 메모리 설정하기
author: hkh7670
date: 2026-09-18 00:00:00 +0900
categories: [etc]
tags: [etc]
---


Java/Spring Boot 애플리케이션을 Docker 컨테이너로 운영할 때 메모리 설정은 단순히 `-Xmx` 하나만 지정한다고 끝나지 않는다.

컨테이너에는 전체 메모리 한도가 있고, JVM은 그 안에서 Heap뿐 아니라 Metaspace, Thread Stack, Direct Buffer, Code Cache, GC 및 Native Memory 등을 함께 사용한다.

따라서 핵심은 다음 두 가지를 분리해서 생각하는 것이다.

- **Docker 컨테이너 전체 메모리 제한**
- **JVM이 사용할 Heap 메모리 제한**

이 글에서는 Java 21 + Spring Boot + Docker 환경을 기준으로, 실무에서 어떻게 메모리를 설정하면 좋은지 예시와 함께 정리한다.

---

## 1. 가장 중요한 개념: Container Memory와 Heap Memory는 다르다

예를 들어 Docker 컨테이너에 1GB 메모리를 할당했다고 가정하자.

```text
Docker Container
┌──────────────────────────┐
│        1024 MB           │
│                          │
│  Java Heap    ~650 MB    │
│                          │
│  Metaspace               │
│  Thread Stack            │
│  Direct Buffer           │
│  Code Cache              │
│  GC / JVM Native Memory  │
│                          │
└──────────────────────────┘
```

JVM 프로세스가 사용하는 전체 메모리는 대략 다음과 같이 생각할 수 있다.

```text
Container Memory
= Heap
+ Metaspace
+ Thread Stack
+ Direct Memory
+ Code Cache
+ GC Memory
+ JVM Native Memory
+ 기타 Native Memory
```

따라서 다음과 같은 설정은 피하는 것이 좋다.

```yaml
services:
  app:
    mem_limit: 1g

    environment:
      JAVA_TOOL_OPTIONS: "-Xmx1g"
```

컨테이너 메모리 한도도 1GB이고 최대 Heap도 1GB이기 때문이다.

Heap 이외의 메모리가 추가로 필요하므로 실제 프로세스 메모리 사용량이 1GB를 초과할 수 있고, 이 경우 컨테이너가 OOM Kill될 가능성이 있다.

---

## 2. Java는 Docker의 메모리 제한을 인식할 수 있다

현대 JVM은 컨테이너 환경을 인식한다.

Java 21 기준으로 `UseContainerSupport`가 기본 활성화되어 있으며, JVM은 cgroup의 CPU 및 메모리 제한을 참고한다.

다만 별도의 Heap 크기를 지정하지 않는 경우 `MaxRAMPercentage` 기본값은 일반적으로 25%다.

예를 들어 컨테이너가 1GB라면 최대 Heap이 대략 다음 정도가 될 수 있다.

```text
1024MB × 25%
≈ 256MB
```

Spring Boot 애플리케이션에서는 이 값이 너무 작을 수도 있기 때문에, 운영 환경에서는 직접 비율이나 최대 Heap을 지정하는 경우가 많다.

---

## 3. 추천 방식 1: `MaxRAMPercentage` 사용

Docker, Kubernetes, ECS처럼 컨테이너마다 메모리 크기가 달라질 수 있는 환경에서는 `-Xmx`를 고정하는 대신 `MaxRAMPercentage`를 사용하는 방법이 편리하다.

예를 들어 다음과 같이 설정할 수 있다.

```yaml
services:
  app:
    image: my-spring-app:latest

    mem_limit: 1g

    environment:
      JAVA_TOOL_OPTIONS: >-
        -XX:InitialRAMPercentage=25
        -XX:MaxRAMPercentage=65
        -XX:+ExitOnOutOfMemoryError

    ports:
      - "8080:8080"
```

이 경우 최대 Heap은 대략 다음과 같다.

```text
Container Memory = 1024MB

Max Heap
≈ 1024 × 0.65
≈ 666MB
```

즉, 전체 컨테이너 메모리의 약 65% 정도를 Heap으로 사용하게 된다.

### 장점

컨테이너 메모리만 변경해도 Heap 크기가 함께 변경된다.

예를 들어 같은 이미지를 다음과 같이 배포할 수 있다.

```text
DEV     512MB
STAGE     1GB
PROD      2GB
```

`MaxRAMPercentage=65`라면 각각의 Heap 최대 크기도 컨테이너 크기에 맞춰 자동으로 달라진다.

2GB 컨테이너라면:

```text
2048MB × 65%
≈ 1331MB
```

정도의 최대 Heap을 사용할 수 있다.

---

## 4. 추천 방식 2: `-Xms`, `-Xmx` 직접 지정

전통적인 방식처럼 Heap 크기를 직접 지정하는 것도 문제없다.

예를 들어:

```yaml
services:
  app:
    image: my-spring-app:latest

    mem_limit: 1g

    environment:
      JAVA_TOOL_OPTIONS: >-
        -Xms256m
        -Xmx640m
        -XX:+ExitOnOutOfMemoryError
```

메모리 구조는 대략 다음과 같다.

```text
Container    1024MB
Xms           256MB
Xmx           640MB
Remaining    ~384MB
```

이 방식의 장점은 최대 Heap 크기가 명확하고 예측 가능하다는 것이다.

### 언제 적합한가?

서버 스펙이 고정되어 있고 애플리케이션별 메모리 요구량을 이미 알고 있다면 좋은 선택이다.

예를 들어:

```text
Container = 2GB
-Xms512m
-Xmx1400m
```

처럼 명확하게 제한할 수 있다.

반대로 Kubernetes나 ECS처럼 컨테이너 메모리 제한이 환경마다 자주 바뀐다면 다음과 같은 비율 기반 설정이 관리하기 편하다.

```text
-XX:MaxRAMPercentage=65
```

---

## 5. Heap은 컨테이너 메모리의 몇 %가 적당할까?

정답은 애플리케이션에 따라 달라진다.

다만 일반적인 Spring Boot REST API 서버라면 초기값으로 컨테이너 메모리의 약 **60~70% 정도를 Heap**으로 두고 관찰하는 방법이 실용적이다.

예시는 다음과 같다.

| Container Memory | Heap 시작점 예시 |
| ---: | ---: |
| 512MB | 250~300MB |
| 1GB | 600~700MB |
| 2GB | 1.2~1.4GB |
| 4GB | 2.5~2.8GB |
| 8GB | 5~6GB |

예를 들어:

```text
Container = 2GB
Heap Max  ≈ 1.3GB
Remaining ≈ 700MB
```

정도로 시작할 수 있다.

### Heap 비율을 더 낮게 잡아야 하는 경우

다음과 같은 애플리케이션은 Native/Direct Memory 비중이 커질 수 있다.

- Spring WebFlux / Netty
- Kafka Client 사용량이 많은 애플리케이션
- NIO 기반 처리
- 대용량 파일 업로드/다운로드
- 이미지 처리
- Direct Buffer 사용량이 큰 서비스
- 많은 Thread를 생성하는 서비스

이런 경우에는 Heap을 컨테이너 메모리의 50~60% 정도부터 시작하는 것이 더 안전할 수 있다.

---

## 6. `Xms = Xmx`로 맞춰야 할까?

전통적인 서버 튜닝에서는 다음과 같은 설정을 자주 볼 수 있다.

```text
-Xms2g
-Xmx2g
```

Heap resize를 줄이고 메모리 사용량을 예측하기 쉬운 장점이 있다.

하지만 Docker 환경에서는 항상 최선의 선택이라고 보기는 어렵다.

예를 들어:

```text
Container = 2GB
-Xms1300m
-Xmx1300m
```

처럼 설정하면 애플리케이션의 실제 Heap 요구량이 적더라도 큰 초기 Heap을 기준으로 JVM이 동작하게 된다.

일반적인 Spring Boot REST API라면 다음처럼 시작할 수 있다.

```text
-Xms256m
-Xmx1300m
```

또는 비율 기반으로:

```text
-XX:InitialRAMPercentage=20
-XX:MaxRAMPercentage=65
```

처럼 설정할 수 있다.

반대로 매우 일정한 트래픽을 처리하고 서버 메모리가 충분하며, GC 동작과 메모리 사용량을 최대한 예측 가능하게 만들고 싶다면 `Xms`와 `Xmx`를 동일하게 두는 전략도 여전히 유효하다.

---

## 7. 운영 환경에서 추천하는 JVM 옵션

운영 환경에서는 단순 Heap 제한 외에도 OOM 대응 옵션을 함께 설정하는 것이 좋다.

예를 들어:

```yaml
services:
  app:
    image: my-app:latest

    mem_limit: 2g

    environment:
      JAVA_TOOL_OPTIONS: >-
        -XX:InitialRAMPercentage=20
        -XX:MaxRAMPercentage=65
        -XX:+HeapDumpOnOutOfMemoryError
        -XX:HeapDumpPath=/heapdump
        -XX:+ExitOnOutOfMemoryError

    volumes:
      - ./heapdump:/heapdump

    restart: unless-stopped
```

각 옵션의 의미는 다음과 같다.

### `InitialRAMPercentage`

초기 Heap 크기를 컨테이너 메모리 대비 비율로 지정한다.

```text
-XX:InitialRAMPercentage=20
```

### `MaxRAMPercentage`

최대 Heap 크기를 컨테이너 메모리 대비 비율로 지정한다.

```text
-XX:MaxRAMPercentage=65
```

### `HeapDumpOnOutOfMemoryError`

Java Heap OOM 발생 시 Heap Dump를 생성한다.

```text
-XX:+HeapDumpOnOutOfMemoryError
```

### `HeapDumpPath`

Heap Dump 저장 위치를 지정한다.

```text
-XX:HeapDumpPath=/heapdump
```

Docker 컨테이너 내부 파일은 컨테이너 제거 시 함께 사라질 수 있으므로, 운영 환경에서는 volume으로 외부에 보존하는 것이 좋다.

### `ExitOnOutOfMemoryError`

OOM 발생 시 JVM을 종료시킨다.

```text
-XX:+ExitOnOutOfMemoryError
```

컨테이너 환경에서는 보통 다음과 같은 구조가 운영하기 편하다.

```text
OOM 발생
  ↓
JVM 종료
  ↓
Container 종료
  ↓
Docker / Kubernetes가 재시작
```

---

## 8. Java OOM과 Docker OOM Kill은 다르다

메모리 문제를 분석할 때 반드시 구분해야 한다.

### 8.1 JVM Heap OOM

예를 들어 다음 오류가 발생하는 경우다.

```text
java.lang.OutOfMemoryError: Java heap space
```

JVM의 Heap 제한에 도달한 것이다.

예를 들어:

```text
-Xmx640m
```

인데 Heap이 640MB를 모두 사용하면 JVM이 직접 `OutOfMemoryError`를 발생시킨다.

이 경우 JVM 프로세스가 OOM을 인식하므로 `HeapDumpOnOutOfMemoryError`가 활성화되어 있다면 Heap Dump를 남길 수 있다.

---

### 8.2 Container OOM Kill

반면 다음과 같은 상황을 생각해보자.

```text
Container = 1GB

Heap        650MB
Direct      200MB
Metaspace   120MB
Thread       80MB
Native       70MB
----------------
Total      1120MB
```

JVM Heap 자체는 제한을 초과하지 않았지만, 프로세스 전체 메모리가 컨테이너의 1GB 한도를 초과했다.

이 경우 Linux cgroup의 메모리 제한에 의해 프로세스가 강제로 종료될 수 있다.

Docker에서는 종종 다음과 같은 형태로 확인할 수 있다.

```text
Exit Code: 137
```

또는:

```bash
docker inspect <container>
```

결과에서:

```text
OOMKilled = true
```

로 확인할 수 있다.

이 경우 JVM이 직접 `OutOfMemoryError`를 처리한 것이 아니므로 Heap Dump가 남지 않을 수도 있다.

그래서 다음 설정이 위험하다.

```text
-Xmx = Container Memory
```

JVM Heap 외의 메모리를 위한 공간이 전혀 남지 않기 때문이다.

---

## 9. 실제 JVM이 컨테이너 메모리를 어떻게 인식했는지 확인하기

컨테이너 내부에서 JVM 옵션을 확인할 수 있다.

```bash
java -XX:+PrintFlagsFinal -version | grep RAMPercentage
```

다음과 같은 값을 확인할 수 있다.

```text
InitialRAMPercentage
MaxRAMPercentage
MinRAMPercentage
```

컨테이너 환경 감지 로그를 확인하려면 Java 21에서 다음 옵션도 유용하다.

```bash
java -Xlog:os+container=trace -version
```

JVM이 cgroup 관련 정보를 어떻게 감지했는지 확인할 수 있다.

---

## 10. 실행 중인 JVM의 Heap 확인하기

Spring Boot 애플리케이션이 실행 중이라면 `jcmd`를 통해 확인할 수 있다.

```bash
jcmd <PID> GC.heap_info
```

JVM에 실제 적용된 옵션은 다음과 같이 확인할 수 있다.

```bash
jcmd <PID> VM.flags
```

컨테이너의 PID가 1이라면:

```bash
jcmd 1 GC.heap_info
```

처럼 사용할 수도 있다.

단, 사용하는 JRE/JDK 이미지에 `jcmd`가 포함되어 있어야 한다.

---

## 11. Native Memory까지 확인하려면 NMT 사용

Heap은 충분한데 컨테이너 메모리가 계속 증가한다면 Native Memory를 확인할 필요가 있다.

이럴 때 JVM의 Native Memory Tracking(NMT)을 사용할 수 있다.

애플리케이션 시작 시:

```text
-XX:NativeMemoryTracking=summary
```

옵션을 추가한다.

그 다음 실행 중인 JVM에서:

```bash
jcmd <PID> VM.native_memory summary
```

를 실행하면 다음과 같은 항목을 확인할 수 있다.

```text
Java Heap
Class
Thread
Code
GC
Compiler
Internal
Symbol
NMT
```

예를 들어:

```text
-Xmx1g
```

인데 `docker stats`에서는 컨테이너가 1.5GB를 사용하고 있다면 Heap 외 메모리 사용량을 조사해야 한다.

NMT는 이런 문제를 분석할 때 특히 유용하다.

단, NMT 자체도 약간의 오버헤드가 있으므로 운영 환경에서는 필요성과 비용을 함께 고려해야 한다.

---

## 12. `docker stats`와 JVM 지표를 함께 봐야 한다

Docker 레벨에서는 다음 명령어로 메모리 사용량을 확인할 수 있다.

```bash
docker stats
```

하지만 `docker stats`만으로는 Heap과 Native Memory를 구분할 수 없다.

따라서 운영에서는 다음 정보를 같이 보는 것이 좋다.

```text
docker stats
      +
JVM Heap Metrics
      +
GC Metrics
      +
Native Memory
```

Spring Boot Actuator와 Micrometer를 사용하고 있다면 Prometheus + Grafana 같은 모니터링 환경을 통해 다음과 같은 JVM 지표도 관찰할 수 있다.

- JVM Heap 사용량
- Non-Heap 사용량
- GC 횟수 및 시간
- Thread 수
- Buffer Memory
- Process Memory

---

## 13. 실전 추천 설정

Java 21 + Spring Boot + Docker로 일반적인 REST API 서버를 운영한다면 다음 정도에서 시작할 수 있다.

```yaml
services:
  app:
    image: my-app:latest

    mem_limit: 2g

    environment:
      JAVA_TOOL_OPTIONS: >-
        -XX:InitialRAMPercentage=20
        -XX:MaxRAMPercentage=65
        -XX:+HeapDumpOnOutOfMemoryError
        -XX:HeapDumpPath=/heapdump
        -XX:+ExitOnOutOfMemoryError

    volumes:
      - ./heapdump:/heapdump

    restart: unless-stopped
```

메모리 구조는 대략 다음과 같이 생각할 수 있다.

```text
Docker Container  2048MB
        │
        ├── Heap Max        ~1331MB (65%)
        │
        └── Remaining        ~717MB
             ├── Metaspace
             ├── Thread Stack
             ├── Direct Memory
             ├── Code Cache
             ├── GC
             └── JVM / Native Memory
```

이 값은 정답이 아니라 **관찰을 시작하기 위한 초기값**이다.

실제 운영에서는 애플리케이션 특성에 따라 반드시 조정해야 한다.

---

## 14. Blue/Green 배포에서는 동시에 실행되는 메모리까지 계산해야 한다

Blue/Green 배포에서는 기존 버전과 신규 버전이 일정 시간 동시에 실행된다.

예를 들어 서버 메모리가 4GB인데 각각의 컨테이너에 다음과 같이 설정했다고 가정하자.

```text
Blue  Container = 2GB
Green Container = 2GB
```

배포 순간에는 두 컨테이너가 모두 실행될 수 있다.

```text
Blue    2GB
Green   2GB
-------------
Total   4GB
```

여기에 운영체제, Docker daemon, Nginx, SSH, 모니터링 에이전트 등도 메모리를 사용한다.

따라서 실제 서버 메모리가 4GB라고 해서 Java 컨테이너 두 개에 각각 2GB씩 할당하는 것은 위험할 수 있다.

예를 들어 4GB 서버라면 다음처럼 여유 공간까지 고려해야 한다.

```text
Host Memory = 4GB

OS / Docker / Nginx / Agent
≈ 500~1000MB

Remaining
≈ 3~3.5GB
```

Blue/Green 두 개가 동시에 올라와야 한다면 개별 컨테이너 한도를 그보다 낮게 잡거나 서버 메모리를 늘리는 것이 안전하다.

특히 배포 시점에만 OOM Kill이 발생한다면 Blue/Green 컨테이너가 동시에 실행되는 순간의 Peak Memory를 확인해볼 필요가 있다.

---

## 15. 메모리 문제가 발생했을 때 확인 순서

개인적으로는 다음 순서로 확인하는 것이 편하다.

### 1. 컨테이너가 OOM Kill되었는지 확인

```bash
docker inspect <container>
```

`OOMKilled` 여부를 확인한다.

### 2. 전체 컨테이너 메모리 확인

```bash
docker stats
```

### 3. JVM 최대 Heap 확인

```bash
jcmd <PID> GC.heap_info
```

### 4. JVM 옵션 확인

```bash
jcmd <PID> VM.flags
```

### 5. Heap OOM이라면 Heap Dump 분석

```text
*.hprof
```

파일을 Eclipse MAT, VisualVM, IntelliJ Profiler 등의 도구로 분석한다.

### 6. Heap은 여유 있는데 RSS가 높다면 Native Memory 확인

```bash
jcmd <PID> VM.native_memory summary
```

Thread, Direct Buffer, Metaspace, Code Cache 등의 사용량을 확인한다.

---

## 16. 정리

Docker에서 Java 애플리케이션의 메모리를 설정할 때 가장 중요한 원칙은 다음과 같다.

### 1. `Container Memory != Java Heap`

Heap 외에도 JVM은 많은 메모리를 사용한다.

### 2. `-Xmx`를 컨테이너 한도와 동일하게 설정하지 않는다

```text
Bad

Container = 1GB
-Xmx      = 1GB
```

대신 여유 공간을 남겨야 한다.

```text
Better

Container = 1GB
-Xmx      ≈ 600~700MB
```

### 3. 일반적인 Spring Boot API 서버라면 60~70%부터 시작

```text
MaxRAMPercentage=60~70
```

정도를 초기값으로 두고 실제 메트릭을 보면서 조정한다.

### 4. Native Memory 사용량이 큰 서비스라면 더 보수적으로 설정

Netty, Kafka, NIO, 많은 Thread를 사용하는 서비스라면 Heap 비율을 50~60% 정도부터 시작하는 것도 좋다.

### 5. OOM 종류를 구분한다

```text
Java Heap OOM
≠
Container OOM Kill
```

원인이 완전히 다를 수 있다.

### 6. 운영에서는 측정 후 조정한다

초기 설정만으로 끝내기보다는 다음을 지속적으로 관찰하는 것이 중요하다.

```text
Container Memory
Heap Usage
GC
Thread
Direct Buffer
Native Memory
```

결국 좋은 JVM 메모리 설정은 고정된 공식이 아니라,

> **컨테이너 전체 메모리 안에서 Heap과 Non-Heap/Native Memory가 안전하게 공존하도록 설정하고 실제 사용량을 관찰하면서 조정하는 것**

이라고 볼 수 있다.

---

## 참고 문서

- Oracle Java 21 `java` command documentation  
  <https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html>

- Docker Resource Constraints  
  <https://docs.docker.com/engine/containers/resource_constraints/>

- Docker Compose Services Reference  
  <https://docs.docker.com/reference/compose-file/services/>
