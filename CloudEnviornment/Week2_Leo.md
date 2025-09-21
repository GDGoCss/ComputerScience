# 2주차

# Docker가 왜 필요한가?

**전통적인 개발 환경의 문제점**

- 개발자마다 다른 환경: Java 8 vs 11 vs 17, Ubuntu vs Windows vs CentOS
- "내 컴퓨터에서는 잘 됐는데..." 와 같은 문제가 발생함.

**가상머신(VM)의 한계**

- 각 VM마다 완전한 OS 필요 → 무거움 (1-2GB)
- 부팅 시간 오래 걸림 → 느림
- 리소스 낭비 → 비효율적

## Container vs Virtual Machine

<a href="https://ibb.co/5gfV4YpF"><img src="https://i.ibb.co/Q31GQ8Sd/image.png" alt="image" border="0"></a>

## **반면 Docker Container의 특징**

- OS 커널 공유 → 가벼움 (10-100MB)
- 초 단위 시작 → 빠름
- 격리된 프로세스 → 안전함

---

## 1. Docker 핵심 원리

## 네임스페이스 (Namespace) - 격리 기술

**개념**: 프로세스를 독립적인 환경으로 격리하는 기술

- 각 컨테이너는 자신만의 독립적인 세계를 가짐

```java
Host System:
├── PID 1: systemd
├── PID 1234: docker daemon
└── PID 1235: container process

Container 내부에서는:
└── PID 1: my-application  *# 실제로는 Host의 PID 1235*
```

**격리 영역**

- **PID**: 프로세스 ID 격리
- **Network**: 네트워크 인터페이스 격리
- **Mount**: 파일시스템 격리
- **User**: 사용자 ID 격리

## Cgroups (Control Groups) - 리소스 제한

**개념**: 컨테이너가 사용할 수 있는 시스템 리소스를 제한

```java
*# CPU 사용량 제한*
docker run --cpus="1.5" my-app

*# 메모리 사용량 제한*  
docker run --memory="512m" my-app
```

**제어 가능한 리소스**

- CPU 사용률, 메모리 사용량, 네트워크 대역폭, 디스크 I/O

---

## 2. Docker 네트워크

### 네트워크가 왜 필요한가?

**문제 상황**

```java
docker run -d --name web-app my-web-app
docker run -d --name mysql-db mysql:8.0

*# 문제: web-app이 mysql-db에 어떻게 연결? # localhost:3306? → 컨테이너 내부에서는 접근 불가*
```

## Bridge Network 동작 원리

같은 네트워크 내에서는 컨테이너명으로 통신 가능

```java
*# Docker가 자동 생성하는 가상 브리지*
docker0: 172.17.0.1

Container 1: 172.17.0.2 ←→ docker0 ←→ Host Network  
Container 2: 172.17.0.3 ←→ docker0 ←→ Host Network
```

**실제 예시**

```java
*# 1. 사용자 정의 네트워크 생성*
docker network create my-app-network

*# 2. 같은 네트워크에 컨테이너 실행*
docker run -d --name mysql-db --network my-app-network mysql:8.0
docker run -d --name web-app --network my-app-network \
  -e DB_HOST=mysql-db my-web-app  
  
*# 컨테이너명으로 접근!
# 3. 통신 확인*
docker exec web-app ping mysql-db  *# 성공!*
```

## 보안 계층 구조

**3단계 방어선**

1. **서버 보안그룹/방화벽** (AWS Security Group, iptables)
2. **Docker Port Mapping** (-p 옵션으로 제어)
3. **Docker Network Isolation** (네트워크별 격리)

**접근 제어 예시**

```java
services:
  # 외부 접근 가능
  web:
    ports:
      - "80:80"
  
  # localhost만 접근  
  admin:
    ports:
      - "127.0.0.1:3000:3000"
  
  # 완전 내부 통신만
  database:
    # ports 설정 없음
```

---

## 3. Docker 볼륨

### 데이터 지속성 문제

> **문제 상황의 비유:** 이사할 때마다 모든 짐이 사라지는 원룸
> 

```java
docker run -d --name mysql-db mysql:8.0
*# 데이터 입력...*
docker rm -f mysql-db
*# 결과: 모든 데이터 손실! 😱*
```

**컨테이너 파일시스템의 특징**

- Container Layers (읽기/쓰기): 컨테이너 삭제 시 사라짐
- Image Layers (읽기 전용): 유지됨

## 볼륨 종류별 특징

### Named Volume (운영 환경 권장)

**개념**: Docker가 관리하는 전용 창고

```java
*# 볼륨 생성*
docker volume create mysql-data

*# 볼륨 사용*  
docker run -d --name mysql-db \
  -v mysql-data:/var/lib/mysql mysql:8.0

*# 컨테이너 삭제해도 데이터 유지*
docker rm -f mysql-db
docker volume ls  *# mysql-data 볼륨 존재*
```

**장점**: Docker 자동 관리, 백업/복원 용이, 여러 컨테이너 간 공유

### Bind Mount (개발 환경 유용)

**개념**: 호스트 컴퓨터의 폴더를 직접 연결

```java
*# 실시간 코드 반영*
docker run -d --name dev-app \
  -v /home/user/project:/app node:16

*# Host에서 수정 → Container에 즉시 반영*
```

**장점**: 실시간 파일 동기화, 개발 시 매우 유용

### tmpfs Mount (임시 데이터)

**개념**: RAM에 임시 저장

```java
*# 빠른 임시 저장소*
docker run -d --name cache-app \
  --tmpfs /tmp:rw,size=100m redis:alpine
```

**특징**: 매우 빠름, 재시작 시 소실, 보안 데이터용

---

## 4. Docker Compose

> **여러 컨테이너를 하나로 묶어서 관리하는 도구**로 만약에 아래와 같이 개별 Docker를 사용하는 복잡한 경우를 Docker Compose를 통해서 한번에 **통합으로 관리**가 가능하다.
> 

### 주요 기능으로는...

1. **통합 관리**: 여러 서비스를 한 번에 시작/중지
2. **자동 네트워크**: 서비스 간 자동 연결
3. **의존성 제어**: 실행 순서 자동 관리
4. **설정 통합**: 포트, 환경변수, 볼륨을 파일 하나에 정의

```java
# 기존: 복잡한 명령어 수십 줄
docker network create...
docker run --name mysql...
docker run --name redis...

# Compose: 간단한 한 줄
docker-compose up -d
```

---

## 5. Dockerfile 최적화

> Dockerfile 최적화를 진행하면 빌드 시간을 단축시킬 수 있는데 그 방법은 아래와 같습니다.
> 

### 1. Multi-stage Build

**기존 방식의 문제점**

```yaml
FROM openjdk:17
COPY . .
RUN ./gradlew build
CMD ["java", "-jar", "app.jar"]

# 문제: 소스코드, Gradle, 빌드 도구 모두 포함 → 600MB
```

**Multi-stage Build 해결책**

```yaml
# 빌드 단계
FROM openjdk:17-jdk AS build
COPY . .
RUN ./gradlew bootJar

# 실행 단계
FROM openjdk:17-jre
COPY --from=build build/libs/*.jar app.jar
CMD ["java", "-jar", "app.jar"]

# 결과: JRE만 포함 → 200MB (67% 감소)
```

위와 같이 **jdk**는 빌드에 필요한 것만 포함하고, **jdk**는 실행에 필요한 것만 포함해서 용량이 절반으로 줄어서 빌드 시간을 줄일 수 있습니다.

---

### Copy-on-Write (COW) 방식

**레이어 구조 이해**

**이미지 레이어 *(읽기 전용)***

```yaml
Ubuntu 20.04         ← Layer 1 (100MB)
OpenJDK 17            ← Layer 2 (200MB)  
Spring Boot JAR     ← Layer 3 (50MB)
설정 파일                   ← Layer 4 (1MB)
```

이미지 레이어는 위와 같이 ***읽기 전용***으로 이루어져있지만

**컨테이너 레이어 *(읽기/쓰기)***

```yaml
Container A: 로그 파일 생성      ← +5MB
Container B: 임시 파일 생성     ← +3MB
Container C: 캐시 데이터 생성   ← +8MB

이미지 레이어는 공유, 변경사항만 개별 저장
```

컨테이너 레이어는 ***읽기와 쓰기 모두 가능***한 점을 이용해서 **COW 방식**을 사용할 수 있습니다.

### → COW 동작 방식은 다음과 같습니다.

**파일 읽기**: 이미지 레이어에서 직접 읽음

**파일 수정**: 컨테이너 레이어로 복사 후 수정

**결과**: 원본은 보존, 변경사항만 별도 관리

---

## 레이어 캐싱 방식

### 잘못된 순서 (매번 재빌드)

```yaml
FROM openjdk:17-jdk
COPY . .                    # 소스 변경 시 아래 모든 레이어 재빌드
RUN ./gradlew dependencies   
RUN ./gradlew bootJar
```

### 최적화된 순서 (캐싱 활용)

```yaml
FROM openjdk:17-jdk

# 1. 의존성 파일만 먼저 복사
COPY build.gradle settings.gradle ./
COPY gradle/ gradle/

# 2. 의존성 다운로드 (자주 변경되지 않아 캐시됨)
RUN ./gradlew dependencies

# 3. 소스 코드는 마지막에 복사
COPY src/ src/
RUN ./gradlew bootJar
```

위와 같이 기존에 배포할 때마다 매번 재빌드하는 방식보다는 그 아래와 같이 최적화된 순서로 캐싱 방식을 활용하면 **최대 90%까지 빌드 시간을 단축**시킬 수 있습니다.
