---
title: "NAS에 Gitea 설치하기"
date: 2026-06-16 10:00:00 +0900
categories: [Git, Git-NAS]
tags: [Git, Git-NAS, Git-Gitea, Git-LFS, Git-MariaDB]
description: "NAS에 Gitea를 설치하고 MariaDB 연결, 포트포워딩, LFS까지 설정한 과정"
---

# NAS에 Gitea 설치하기

개인 Git 서버를 NAS에 구축하기로 했다. 외부 서비스에 의존하지 않고 직접 관리할 수 있는 Git 서버가 필요했고, NAS에서 패키지로 지원하는 Gitea를 사용하기로 결정했다.

---

## 1. Gitea 다운로드

NAS 패키지 센터에서 Gitea를 다운로드했다.

---

## 2. MariaDB에 DB 생성

Gitea는 SQLite로도 동작하지만, 추후 안정성과 동시성을 고려해 MariaDB를 DB로 사용하기로 했다.

MariaDB 패키지에서 Gitea가 사용할 **DB만 미리 생성**해두었다. Gitea 설치 마법사에서 이 DB에 연결하는 방식이다.

* MariaDB 접속 후 Gitea 전용 데이터베이스 생성
* 계정/권한은 Gitea가 해당 DB에만 접근 가능하도록 설정

---

## 3. Gitea 연결 및 설치 실행

Gitea 설치 마법사에서 DB 연결 정보를 입력했다.

* DB 종류: MariaDB
* Host / Port
* DB 이름 (2번에서 생성한 DB)
* 계정 / 비밀번호

연결 확인 후 설치를 진행해 Gitea 인스턴스를 실행했다.

---

## 4. 포트포워딩

외부에서 NAS의 Gitea에 접근하기 위해 라우터에서 포트포워딩을 설정했다.

* Gitea 웹 UI 포트 : 8418
* Git SSH 포트 : 22

내부 NAS IP로 트래픽이 전달되도록 설정했다.

---

## 5. 대용량 파일 업로드 설정 (LFS)

언리얼 프로젝트처럼 `.uasset`, `.umap` 등 대용량 바이너리 파일을 다룰 일이 많기 때문에 Git LFS 설정도 함께 진행했다.

> LFS 동작 원리는 [GitHub LFS 원리]({% post_url Git/2026-05-12-git_lfs %})에 정리해둔 내용과 동일하다. Gitea도 LFS를 지원하므로 같은 방식으로 포인터 파일과 실제 파일이 분리되어 저장된다.

* Gitea 설정에서 LFS 활성화
* 저장소에서 `git lfs track` 으로 대상 확장자 등록

---

## 정리

> Gitea 다운로드 → MariaDB에 DB 생성 → Gitea 설치 시 DB 연결 → 포트포워딩 → LFS 설정

여기까지 진행해서 NAS 안에서 개인 Git 서버가 동작하는 상태까지 만들었다.

---

## 다음 할 일

* 미러링 설정 (외부 저장소 ↔ NAS Gitea 동기화)
