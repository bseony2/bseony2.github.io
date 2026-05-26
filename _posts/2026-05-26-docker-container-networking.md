---
layout: post
title: "Docker 컨테이너 네트워크 격리와 컨테이너 간 통신"
date: 2026-05-26 23:00:00 +0900
categories: [Infrastructure, Docker]
tags: [docker, container, network, kubernetes, devops]
---

## 들어가며

Spring Boot 백엔드를 Docker 컨테이너로 올리면서 예상치 못한 문제를 만났다. `application.yml`에 `localhost:5432`로 PostgreSQL 접속 설정을 해뒀는데, 컨테이너로 실행하자마자 DB 연결이 실패했다. 분명히 PostgreSQL은 잘 실행 중이었는데도.

원인은 컨테이너의 네트워크 격리였다. 이 글에서는 Docker 컨테이너가 왜 서로의 `localhost`에 접근할 수 없는지, 그리고 어떻게 컨테이너 간 통신을 구성하는지 정리했다.

---

## 1. 컨테이너는 왜 서로의 localhost에 접근할 수 없는가

Docker 컨테이너는 Linux의 **network namespace**를 이용해 네트워크를 완전히 격리한다.

```
호스트 머신
├── 호스트 네트워크 namespace (eth0: 192.168.1.100)
├── container-a network namespace (eth0: 172.17.0.2)
└── container-b network namespace (eth0: 172.17.0.3)
```

각 컨테이너는 독립된 네트워크 인터페이스, IP 주소, 라우팅 테이블을 가진다. 따라서 `container-a` 내부에서 `localhost`를 호출하면 `container-a` 자신을 가리키고, `container-b`는 전혀 다른 namespace에 있다.

```
# container-a 내부에서
localhost → container-a 자신 (172.17.0.2)
           ≠ container-b (172.17.0.3)
```

이것이 바로 `application.yml`에 `localhost:5432`를 적었을 때 PostgreSQL 컨테이너에 접근하지 못하는 이유다.

---

## 2. Docker 네트워크 종류

Docker는 목적에 따라 여러 네트워크 드라이버를 제공한다.

| 드라이버 | 설명 | 주 용도 |
|---|---|---|
| `bridge` | 가상 브릿지로 컨테이너 연결 (기본값) | 단일 호스트 컨테이너 간 통신 |
| `host` | 호스트 네트워크를 그대로 사용 | 성능 최우선 (Linux only) |
| `overlay` | 여러 Docker 호스트에 걸친 네트워크 | Docker Swarm, 멀티호스트 |
| `none` | 네트워크 없음 | 완전한 격리 필요 시 |

일반적인 로컬 개발 환경에서는 `bridge` 드라이버를 사용한다.

### bridge 네트워크의 동작 방식

```
Docker 호스트
└── bridge network (finmate-network)
    ├── postgres-global  (172.18.0.2)
    └── finmate-backend  (172.18.0.3)
         ↑
    같은 네트워크 → 컨테이너명으로 DNS 조회 가능
```

컨테이너를 사용자 정의 bridge 네트워크에 연결하면, Docker 내장 DNS가 **컨테이너명을 IP로 자동 변환**해준다. `finmate-backend`에서 `postgres-global`이라는 호스트명으로 PostgreSQL에 접근할 수 있는 이유가 바로 이 DNS 기능 덕분이다.

> **주의:** 기본 bridge 네트워크(`docker0`)에서는 컨테이너명 DNS가 동작하지 않는다. 사용자 정의 네트워크를 만들어야 한다.

---

## 3. 실전 구성

PostgreSQL과 Spring Boot 백엔드를 Docker 컨테이너로 구성하는 상황이었다.

### 네트워크 생성

```bash
docker network create finmate-network
```

### PostgreSQL 컨테이너 실행

```bash
docker run -d \
  --name postgres-global \
  --restart always \
  -e POSTGRES_USER=<db-user> \
  -e POSTGRES_PASSWORD=<db-password> \
  -p 5432:5432 \
  -v postgres_global_data:/var/lib/postgresql/data \
  postgres:17
```

이후 네트워크에 연결한다.

```bash
docker network connect finmate-network postgres-global
```

### 백엔드 컨테이너 실행

```bash
docker run -d \
  --name finmate-backend \
  --restart always \
  --network finmate-network \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-global:5432/finmate \
  -p 8080:8080 \
  finmate-backend:latest
```

핵심은 `-e SPRING_DATASOURCE_URL`로 환경변수를 주입하는 것이다. Spring Boot는 환경변수 `SPRING_DATASOURCE_URL`이 있으면 `application.yml`의 `spring.datasource.url`을 덮어쓴다. 덕분에 로컬 개발(`bootRun`)에서는 `localhost:5432`를 그대로 쓰고, 컨테이너 실행 시에만 `postgres-global:5432`로 오버라이드된다.

```
로컬 bootRun:   application.yml의 localhost:5432 사용
컨테이너 실행:  환경변수 SPRING_DATASOURCE_URL=postgres-global:5432 우선 적용
```

---

## 4. --restart 옵션

컨테이너를 장기 운영할 때 `--restart` 옵션이 중요하다.

| 옵션 | 동작 |
|---|---|
| `no` (기본값) | 자동 재시작 안 함 |
| `always` | 항상 재시작 (수동 정지 후에도) |
| `unless-stopped` | 수동으로 정지한 경우 제외하고 재시작 |
| `on-failure` | 비정상 종료 시에만 재시작 |

`always`와 `unless-stopped`의 차이는 수동으로 `docker stop`했을 때 나타난다.

```bash
docker stop postgres-global

# --restart always: Docker 재시작 시 postgres-global 자동 시작
# --restart unless-stopped: Docker 재시작 후에도 정지 상태 유지
```

운영 환경에서는 `always`, 개발 환경에서는 `unless-stopped`가 더 적합하다. 개발 중에 의도적으로 특정 컨테이너를 내렸는데 재시작할 때마다 다시 올라오면 불편하기 때문이다.

---

## 5. Docker standalone vs docker-compose

### docker-compose 방식

```yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: finmate
  backend:
    image: finmate-backend:latest
    depends_on:
      - postgres
```

`docker-compose.yml` 파일로 여러 컨테이너를 선언적으로 관리한다. compose 내 서비스들은 자동으로 같은 네트워크에 속한다.

### standalone 방식

```bash
docker run -d --name postgres-global --restart always postgres:17
docker network create finmate-network
docker network connect finmate-network postgres-global
docker run -d --name finmate-backend --network finmate-network finmate-backend:latest
```

명령어로 직접 컨테이너를 관리한다. docker-compose.yml 파일 없이 컨테이너가 독립적으로 존재한다.

### 선택 기준

| | docker-compose | standalone |
|---|---|---|
| 설정 관리 | 파일로 선언적 관리 | 명령어 히스토리 의존 |
| 생명주기 | 프로젝트 단위 시작/종료 | 컨테이너 단위 독립 관리 |
| 재사용성 | 특정 프로젝트에 종속 | 여러 프로젝트가 공유 가능 |
| 적합한 상황 | 앱과 DB가 항상 같이 기동 | DB를 시스템 전역으로 운영 |

PostgreSQL처럼 여러 프로젝트가 공유하거나 시스템 수준에서 항상 떠 있어야 하는 서비스는 standalone이 더 적합했다. Kubernetes에서도 각 서비스는 독립 Pod로 배포되므로 standalone 방식이 그 사고방식에 더 가깝다.

---

## 6. Kubernetes와의 연관성

Docker network의 컨테이너 간 통신 방식은 Kubernetes의 Service 통신과 거의 동일한 개념이다.

```
Docker network                    Kubernetes
──────────────                    ──────────
컨테이너명으로 DNS 조회       →   Service명으로 DNS 조회
finmate-network 내 통신       →   같은 Namespace 내 Pod 통신
환경변수로 접속 정보 주입     →   ConfigMap / Secret으로 주입
```

로컬에서 Docker network로 구성하는 습관을 들이면, Kubernetes 전환 시 개념적 연결이 훨씬 자연스럽다.

---

## 정리

- 컨테이너는 각자 독립된 **network namespace**를 가지므로 서로의 `localhost`에 접근할 수 없다
- 사용자 정의 **bridge 네트워크**를 만들면 같은 네트워크 내 컨테이너끼리 **컨테이너명으로 통신** 가능하다
- 접속 설정은 **환경변수로 오버라이드**해 로컬 개발(`localhost`)과 컨테이너 실행(`컨테이너명`) 환경을 분리한다
- 시스템 전역으로 운영하는 서비스는 **standalone 컨테이너** + `--restart always`가 적합하다
- Docker network의 컨테이너 간 통신 패턴은 **Kubernetes Service** 통신과 동일한 개념이다
