---
layout: post
title: "Git 브랜치 전략과 커밋 재정리 - Cherry-pick을 활용한 기능별 브랜치 분리"
date: 2025-10-29 23:00:00 +0900
categories: [Git, DevOps]
tags: [git, branch, cherry-pick, reset, stash, rebase]
---

## 들어가며

개발 중에 기능 분리가 명확하지 않은 상태에서 작업하다 보면, 하나의 브랜치에 여러 기능의 커밋이 섞여 있는 경우가 발생한다. 예를 들어:
```
feature/user-management:
  - 사용자 관리 기능 A
  - 사용자 관리 기능 B
  - 상품 관리 기능 C  ← 다른 기능이 섞임
  - UI 개선 D          ← 프론트엔드 작업도 섞임
```

이런 상황에서 커밋을 기능별로 깔끔하게 재정리하는 방법을 정리했다.

---

## 1. 문제 상황

### 초기 브랜치 상태
```bash
feature/module-a:
  09905cf (백엔드: B 모듈 구현)
  e03c0cb (프론트: A 모듈 UI 개선)
  95f5efa (프론트: B 모듈 테이블 추가)
  f81c8e5 (Merge from master)
  cfa6fd8 (백엔드: A 모듈 API 추가)
  117af7a (백엔드: A 모듈 검증 로직)
```

### 원하는 최종 상태
```bash
feature/module-a (A 모듈만):
  e03c0cb (프론트: A 모듈 UI 개선)
  f81c8e5 (Merge)
  cfa6fd8 (백엔드: A 모듈 API 추가)
  117af7a (백엔드: A 모듈 검증 로직)

feature/module-b (B 모듈만):
  09905cf (백엔드: B 모듈 구현)
  95f5efa (프론트: B 모듈 테이블 추가)
  + master의 모든 커밋 (최신 코드 기반)
```

---

## 2. Git 명령어 정리

### git branch: 브랜치 관리
```bash
# 브랜치 생성
git branch feature/new-feature

# 브랜치 목록 확인
git branch

# 브랜치 삭제
git branch -d feature/old-feature

# 강제 삭제
git branch -D feature/old-feature
```

**특징:**
- 브랜치를 생성하지만 이동하지는 않음
- 현재 커밋을 가리키는 새로운 포인터 생성

### git checkout: 브랜치 전환
```bash
# 브랜치 이동
git checkout feature/existing-branch

# 브랜치 생성과 동시에 이동
git checkout -b feature/new-branch

# 특정 커밋으로 이동
git checkout 09905cf
```

**checkout -b의 동작:**
```bash
# 이 두 명령어와 동일
git branch feature/new-branch
git checkout feature/new-branch
```

### git reset: 커밋 되돌리기
```bash
# 특정 커밋으로 되돌리기
git reset --hard f81c8e5
```

**--hard 옵션:**
- HEAD를 지정한 커밋으로 이동
- 작업 디렉토리의 파일도 그 시점으로 되돌림
- 이후 커밋들은 사라짐 (주의!)

**시각화:**
```
Before reset:
  C (HEAD)
  B
  A

git reset --hard A

After reset:
  A (HEAD)
  (B, C는 사라짐)
```

**다른 옵션:**
- `--soft`: HEAD만 이동, 파일은 유지
- `--mixed`: HEAD와 Staging Area 초기화, 파일은 유지

### git stash: 작업 임시 저장
```bash
# 현재 변경사항 저장
git stash

# 저장된 내용 확인
git stash list

# 가장 최근 stash 적용
git stash pop

# 특정 stash 적용
git stash apply stash@{0}

# stash 삭제
git stash drop
```

**사용 시나리오:**
```bash
# 작업 중 긴급 수정 필요
git stash                    # 현재 작업 저장
git checkout master          # 다른 브랜치로 이동
# 긴급 수정 작업...
git checkout feature/branch  # 원래 브랜치로 복귀
git stash pop               # 저장한 작업 복원
```

### git cherry-pick: 선택적 커밋 복사
```bash
# 특정 커밋만 현재 브랜치로 가져오기
git cherry-pick 09905cf

# 여러 커밋 연속으로 가져오기
git cherry-pick 95f5efa 09905cf

# 충돌 발생 시 중단
git cherry-pick --abort

# 충돌 해결 후 계속
git cherry-pick --continue
```

**동작 원리:**
```
source-branch:
  A - B - C - D

target-branch:
  X - Y

git cherry-pick C

target-branch:
  X - Y - C'  (C의 변경사항이 적용된 새 커밋)
```

**핵심:**
- 커밋을 "복사"하는 것 (이동이 아님)
- 원본 브랜치는 그대로 유지
- 새로운 커밋 해시가 생성됨

### git merge: 브랜치 병합
```bash
# feature 브랜치를 현재 브랜치로 병합
git merge feature/branch

# Fast-forward 방지 (병합 커밋 생성)
git merge --no-ff feature/branch

# 충돌 발생 시 중단
git merge --abort
```

**Fast-forward vs Merge Commit:**
```
Fast-forward:
  master:  A - B
                \
  feature:       C - D

  git merge feature
  master:  A - B - C - D (일직선)

Non-fast-forward:
  master:  A - B ----------- M (merge commit)
                \           /
  feature:       C - D ----
```

---

## 3. 실전 시나리오: 브랜치 재정리

### 상황 분석
```bash
$ git log --oneline -6
09905cf (HEAD -> feature/module-a) B 모듈 백엔드 구현
e03c0cb A 모듈 프론트 개선
95f5efa B 모듈 프론트 테이블
f81c8e5 Merge branch 'master'
cfa6fd8 A 모듈 API 추가
117af7a A 모듈 검증 로직
```

**분석:**
- `117af7a`, `cfa6fd8`: A 모듈 백엔드
- `e03c0cb`: A 모듈 프론트엔드
- `95f5efa`, `09905cf`: B 모듈 (백엔드 + 프론트)

### Step 1: 백업 생성
```bash
# 안전을 위한 백업
git branch backup/module-a-before-cleanup
```

**이유:**
- reset --hard는 커밋을 날리므로 위험
- 문제 발생 시 복구 가능

### Step 2: module-a 브랜치 정리
```bash
# feature/module-a에서 B 모듈 커밋 제거
git checkout feature/module-a

# Merge 커밋까지만 남기고 되돌리기
git reset --hard f81c8e5

# 로그 확인
git log --oneline -4
# f81c8e5 (HEAD) Merge
# cfa6fd8 A 모듈 API
# 117af7a A 모듈 검증
```

**결과:**
- B 모듈 관련 커밋 제거됨
- A 모듈 백엔드만 남음

### Step 3: A 모듈 프론트엔드 추가

작업 중이던 A 모듈 프론트 변경사항이 있다면:
```bash
# 변경사항 임시 저장
git stash

# 다시 적용
git stash pop

# 커밋
git add .
git commit -m "feat: A 모듈 UI 개선 및 카테고리 연동"
```

### Step 4: master에 A 모듈 병합
```bash
# master로 이동
git checkout master

# 최신 코드 받기 (리모트 연결 시)
git pull origin master

# A 모듈 병합
git merge feature/module-a
```

**결과:**
- master에 A 모듈 기능 추가됨
- 완성된 기능만 master에 포함

### Step 5: B 모듈 브랜치 생성
```bash
# master 기반으로 새 브랜치 생성
git checkout -b feature/module-b

# 백업 브랜치에서 B 모듈 커밋만 가져오기
git cherry-pick 95f5efa  # B 프론트
git cherry-pick 09905cf  # B 백엔드

# 로그 확인
git log --oneline -4
# 09905cf' (새 해시) B 백엔드
# 95f5efa' (새 해시) B 프론트
# (master의 모든 커밋들)
```

**핵심:**
- master를 기반으로 생성했으므로 A 모듈 포함
- cherry-pick으로 B 모듈만 추가
- 깔끔하게 분리된 브랜치

---

## 4. Cherry-pick 활용 패턴

### 패턴 1: 잘못된 브랜치에 커밋한 경우
```bash
# 실수로 master에 커밋
git checkout master
git commit -m "새 기능"

# feature 브랜치로 옮기기
git checkout -b feature/new-feature
git cherry-pick master  # 방금 커밋 가져오기

# master에서 커밋 제거
git checkout master
git reset --hard HEAD~1
```

### 패턴 2: 특정 버그 수정만 다른 브랜치에 적용
```bash
# develop 브랜치의 버그 수정 커밋
develop:
  a1b2c3d (버그 수정)

# hotfix 브랜치에 적용
git checkout release/v1.0
git checkout -b hotfix/critical-bug
git cherry-pick a1b2c3d
```

### 패턴 3: 여러 커밋 중 일부만 선택
```bash
# feature 브랜치
feature/complex:
  A (기능 1)
  B (기능 2)
  C (기능 1)
  D (기능 3)

# 기능 1만 가져오기
git checkout -b feature/function-1
git cherry-pick A
git cherry-pick C
```

---

## 5. 충돌 해결

### Cherry-pick 충돌
```bash
$ git cherry-pick 09905cf
error: could not apply 09905cf... B 모듈 구현
hint: after resolving the conflicts, mark them with
hint: 'git add <paths>' and run 'git cherry-pick --continue'
```

**해결 방법:**
```bash
# 1. 충돌 파일 확인
git status

# 2. 충돌 파일 수정
# <<<<<<< HEAD
# 현재 브랜치 내용
# =======
# cherry-pick하려는 내용
# >>>>>>> 09905cf

# 3. 충돌 해결 후 add
git add .

# 4. cherry-pick 계속
git cherry-pick --continue

# 또는 중단
git cherry-pick --abort
```

### Merge 충돌
```bash
$ git merge feature/branch
Auto-merging file.txt
CONFLICT (content): Merge conflict in file.txt
```

**해결 방법:**
```bash
# 1. 충돌 파일 수정

# 2. add
git add .

# 3. 커밋
git commit  # 자동으로 merge commit 메시지 생성

# 또는 중단
git merge --abort
```

---

## 6. 실무 브랜치 전략

### Git Flow
```
master (프로덕션)
  ↑
release/v1.0 (배포 준비)
  ↑
develop (개발 통합)
  ↑
feature/user-management (기능 개발)
```

**브랜치 역할:**
- `master`: 프로덕션 배포 코드
- `develop`: 개발 통합 브랜치
- `feature/*`: 기능 개발
- `release/*`: 배포 준비
- `hotfix/*`: 긴급 수정

### Feature Branch 전략
```bash
# 새 기능 시작
git checkout develop
git checkout -b feature/user-profile

# 개발...

# develop에 병합
git checkout develop
git merge --no-ff feature/user-profile

# 브랜치 삭제
git branch -d feature/user-profile
```

### 커밋 메시지 컨벤션
```bash
feat: 새로운 기능 추가
fix: 버그 수정
refactor: 코드 리팩토링
docs: 문서 수정
test: 테스트 코드 추가
chore: 빌드 설정 등

예시:
feat: 사용자 프로필 조회 API 구현
fix: 로그인 시 세션 만료 버그 수정
refactor: 카테고리 조회 로직 개선
```

---

## 7. 유용한 Git 명령어

### 커밋 히스토리 조회
```bash
# 간단한 로그
git log --oneline -10

# 그래프 형태
git log --oneline --graph --all

# 특정 파일의 히스토리
git log --oneline -- src/main.java

# 특정 기간
git log --since="2025-10-01" --until="2025-10-31"
```

### 변경사항 확인
```bash
# 작업 디렉토리 변경사항
git diff

# Staged 변경사항
git diff --staged

# 특정 커밋 간 비교
git diff 117af7a..cfa6fd8

# 특정 파일만
git diff HEAD -- src/main.java
```

### 커밋 수정
```bash
# 마지막 커밋 메시지 수정
git commit --amend -m "새로운 메시지"

# 마지막 커밋에 파일 추가
git add forgotten-file.txt
git commit --amend --no-edit
```

---

## 8. 주의사항

### Reset의 위험성
```bash
# ⚠️ 위험: 커밋이 사라짐
git reset --hard HEAD~3

# ✅ 안전: 백업 후 작업
git branch backup/before-reset
git reset --hard HEAD~3
```

### Force Push 주의
```bash
# ⚠️ 위험: 다른 사람의 작업 덮어씀
git push --force

# ✅ 조금 더 안전: 원격이 변경되지 않았을 때만
git push --force-with-lease
```

**원칙:**
- 공유 브랜치(master, develop)에는 force push 금지
- 개인 feature 브랜치에서만 사용
- 팀원과 소통 후 진행

### Cherry-pick 남용 주의

**좋은 사용:**
```bash
# hotfix를 release 브랜치에 적용
git cherry-pick hotfix-commit
```

**나쁜 사용:**
```bash
# 여러 브랜치에서 무분별하게 복사
# → 같은 변경사항이 여러 곳에 중복
# → 나중에 충돌과 혼란 발생
```

---

## 정리

- **git branch**: 브랜치 생성 및 관리
- **git checkout -b**: 브랜치 생성과 동시에 이동
- **git reset --hard**: 특정 커밋으로 되돌리기 (주의 필요)
- **git stash**: 작업 중인 변경사항 임시 저장
- **git cherry-pick**: 특정 커밋만 선택적으로 복사
- **git merge**: 브랜치 전체 병합
- 커밋 재정리 시 백업 브랜치 필수 생성
- 기능별로 브랜치를 명확히 분리하여 관리
- cherry-pick은 특정 커밋만 필요할 때 사용
- 공유 브랜치에서는 reset, force push 지양