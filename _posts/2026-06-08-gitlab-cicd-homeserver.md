---
layout: post
title: "홈서버에 GitLab CI/CD 파이프라인 구축하기"
date: 2026-06-08 23:00:00 +0900
categories: [Infrastructure, CI/CD]
tags: [gitlab, cicd, docker, nginx, runner, homeserver, devops]
---

## 들어가며

개인 프로젝트를 코드 수정할 때마다 SSH로 접속해서 직접 빌드하고 Docker 컨테이너를 재시작하는 방식으로 운영하고 있었다. 기능이 추가될수록 이 과정이 번거로워졌고, 자동화가 필요하다고 느꼈다.

이번에 GitLab CI/CD와 홈서버를 연결해서 `git push`만으로 빌드부터 배포까지 자동화하는 파이프라인을 구축했다. 이 글에서는 구축 과정과 맞닥뜨린 이슈들을 정리했다.

---

## 전체 구조

구축 목표는 아래 흐름을 자동화하는 것이었다.

```
git push (main 브랜치)
    ↓
GitLab 파이프라인 트리거
    ↓
[build] Docker 이미지 빌드
    ↓
[deploy] 기존 컨테이너 교체 → 새 컨테이너 실행
```

기술 스택은 다음과 같다.

- **GitLab.com**: 코드 저장소 + 파이프라인 관리
- **GitLab Runner**: 홈서버에 설치, 실제 빌드/배포 실행
- **Docker**: 애플리케이션 컨테이너화
- **Nginx**: 프론트엔드 정적 파일 서빙 + 백엔드 API 프록시

---

## 1. GitLab Runner란

GitLab Runner는 파이프라인 작업을 실제로 실행하는 에이전트다. GitLab.com은 파이프라인을 **지시**만 하고, Runner가 설치된 서버에서 빌드와 배포가 실행된다.

```
GitLab.com (파이프라인 지시)
    ↓
홈서버 Runner (실제 명령어 실행)
```

GitLab은 공용 Runner를 제공하지만, 홈서버에 직접 배포해야 하는 경우 Self-hosted Runner가 필요하다. 공용 Runner는 외부 서버라 내부 네트워크에 접근할 수 없기 때문이다.

### 설치 및 등록

macOS 기준으로 Homebrew로 설치했다.

```bash
brew install gitlab-runner
brew services start gitlab-runner
```

GitLab에서 Runner 토큰을 발급받아 등록했다. `shell` executor를 선택해 서버의 쉘에서 직접 명령어를 실행하게 했다.

```bash
gitlab-runner register \
  --url https://gitlab.com \
  --token <runner-token> \
  --executor shell \
  --non-interactive
```

### 서브그룹으로 Runner 공유 범위 제한

프로젝트가 여러 개인 경우 Runner 공유 범위를 제한할 수 있다. GitLab Runner는 등록된 레벨에서만 공유된다.

```
최상위 그룹 Runner   → 그룹 내 모든 프로젝트에서 사용
서브그룹 Runner      → 해당 서브그룹 하위 프로젝트에서만 사용
프로젝트 Runner      → 해당 프로젝트에서만 사용
```

서브그룹을 만들어 관련 레포를 묶고 서브그룹 레벨에 Runner를 등록하면, 다른 프로젝트와 Runner가 섞이지 않는다.

---

## 2. .gitlab-ci.yml 핵심 옵션

파이프라인 설정 파일이다. 레포 루트에 위치하면 GitLab이 자동으로 인식한다.

### tags — 어느 Runner에서 실행할지 지정

```yaml
build:
  stage: build
  tags:
    - my-server  # 이 태그를 가진 Runner에서만 실행
```

태그를 명시하지 않으면 GitLab이 사용 가능한 Runner 중 아무거나 선택한다. Self-hosted Runner를 쓰려면 반드시 태그로 지정해야 한다.

### only — 특정 브랜치에서만 트리거

```yaml
deploy:
  stage: deploy
  only:
    - main  # main 브랜치 push 시에만 실행
```

### 환경변수 관리 — docker run 직접 입력에서 CI/CD Variables로

자동화 이전에는 컨테이너를 실행할 때 환경변수를 직접 입력하고 있었다.

```bash
docker run -e DB_URL=jdbc:postgresql://db:5432/mydb \
           -e DB_USERNAME=<db-user> \
           -e DB_PASSWORD=<db-password> \
           my-app:latest
```

이 방식은 두 가지 문제가 있다. 첫째, 명령어를 어딘가에 따로 기록해두지 않으면 나중에 어떤 값으로 실행했는지 추적하기 어렵다. 둘째, CI/CD로 배포를 자동화하면 파이프라인이 컨테이너를 실행하는데, 이 값들을 코드에 넣을 수 없다.

GitLab의 **CI/CD Variables**에 값을 등록하면 파이프라인 실행 시 환경변수로 자동 주입된다.

```yaml
# .gitlab-ci.yml
- docker run -e DB_URL=$DB_URL \
             -e DB_USERNAME=$DB_USERNAME \
             -e DB_PASSWORD=$DB_PASSWORD \
             my-app:latest
```

`$DB_PASSWORD` 같은 민감한 값은 **Mask** 옵션을 활성화하면 파이프라인 로그에 노출되지 않는다. 코드에 민감정보를 하드코딩하지 않으면서, 배포에 필요한 값을 안전하게 관리할 수 있다.

### `|| true` 패턴

```yaml
- docker stop my-container || true
- docker rm my-container || true
```

처음 배포할 때는 컨테이너가 없어서 `docker stop`이 실패한다. `|| true`를 붙이면 실패해도 파이프라인이 계속 진행된다.

---

## 3. 트러블슈팅

### 공용 Runner가 실행되어 docker 명령어를 찾지 못함

**증상**
```
docker: command not found
```

파이프라인을 처음 실행했을 때 홈서버 Runner가 아닌 GitLab 공용 Runner가 잡았다. 공용 Runner는 별도 컨테이너 환경에서 실행되기 때문에 docker 명령어가 없었다.

**원인 및 해결**

Runner에 태그를 설정하지 않으면 GitLab이 아무 Runner나 선택한다. Runner에 태그를 추가하고 `.gitlab-ci.yml`에 `tags`를 명시해서 홈서버 Runner만 실행하도록 고정했다.

### 파이프라인이 stuck 상태

**증상**
```
This job is stuck because there are no runners that match all of the job's tags
```

**원인 및 해결**

Runner 등록 시 UI에서 태그를 잘못 입력했다. `.gitlab-ci.yml`에 명시한 태그와 Runner에 등록된 태그가 정확히 일치해야 한다. GitLab API로 Runner 태그를 수정해서 해결했다.

---

## 4. Nginx 리버스 프록시로 프론트-백엔드 연결

### 문제

프론트엔드 React 앱이 백엔드 API를 `http://localhost:8080/api`로 호출하고 있었다. 로컬 개발 환경에서는 잘 됐지만 서버에 배포하면 동작하지 않았다.

React 앱은 **브라우저에서 실행**된다. 브라우저 입장에서 `localhost`는 서버가 아니라 사용자의 PC를 가리킨다.

```
브라우저에서 localhost:8080 요청
    → 사용자 PC에서 찾음 (실패)
```

서버 IP를 하드코딩하는 방법도 있지만, IP가 바뀔 때마다 재빌드가 필요하다.

### 해결: Nginx 리버스 프록시

Nginx가 `/api` 경로의 요청을 백엔드 컨테이너로 대신 전달하게 했다.

```
브라우저 → http://서버주소/api/...
                    ↓ Nginx 프록시
            http://backend-container:8080/api/...
```

```nginx
location /api {
    proxy_pass http://backend-container:8080;
}
```

프론트엔드 API base URL은 `/api`(상대 경로)로 변경했다. 이제 어느 IP에서 접속하든 같은 호스트의 Nginx를 통해 백엔드로 연결되고, IP 변경에도 코드 수정이 불필요하다.

### Dockerfile 인라인 주석으로 빌드 실패

**증상**
```
ERROR: failed to calculate checksum of ref: "/프록시": not found
```

`COPY` 명령어 뒤에 한글 주석을 달았더니 경로로 인식해서 빌드가 실패했다.

```dockerfile
# 실패
COPY nginx.conf /etc/nginx/conf.d/default.conf  # 프록시 설정 적용

# 수정
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

Dockerfile에서 인라인 주석은 지원하지 않는다. 주석은 반드시 별도 줄에 써야 한다.

---

## 마치며

단순히 파이프라인 파일 하나 작성하는 게 아니라, Runner 등록 → 태그 설정 → 네트워크 구성까지 연결이 돼야 제대로 동작했다. 특히 GitLab 공용 Runner와 Self-hosted Runner의 차이, 그리고 브라우저에서 실행되는 프론트엔드의 통신 방식이 서버 사이드와 다르다는 점이 핵심이었다.

---

## 정리

| 항목 | 내용 |
|------|------|
| CI/CD 도구 | GitLab CI/CD |
| Runner | Self-hosted (shell executor) |
| 파이프라인 | build → deploy, main 브랜치 push 시 트리거 |
| 환경변수 관리 | GitLab CI/CD Variables |
| 프론트 배포 | Nginx (정적 파일 서빙 + API 프록시) |
| 주요 이슈 | 공용 Runner 실행, Runner 태그 불일치, Dockerfile 인라인 주석 |
