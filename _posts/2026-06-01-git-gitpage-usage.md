---
title: "GitPage 운영가이드"
date: 2026-06-01 16:50:00 +0900
categories: [Git, GitPages]
tags: [GitPage-CategorySetting]
---

# 깃허브 페이지(GitHub Pages) 운영 가이드

---

## 1. 머리말
* 파일 최상단에 카테고리 명시하여 자동 분류

```yaml
  ---
  layout: post
  date: yyyy-MM-dd HH:mm:ss +0900
  title: "포스트 제목"
  categories: [UnrealEngine, CPlusPlus]
  ---
```

## 2. 파일 관리
* **파일명 규칙 (Jekyll):** `YYYY-MM-DD-카테고리-주제-비고.md` 형식 준수 (예: `2026-06-01-gitpages-usage-1.md`)
* **확장자:** `.md` (Markdown) 필수

## 3. 이미지 업로드 전략
* **이미지**
  - 자주 사용 : `/assets/img/*.png` 형태로 업로드
  - 1회성 사용 : `git/issues/` 에 업로드 후 마크다운 링크 추출

## 4. 카테고리 분류
### Categories
- Unreal          : 언리얼 엔진 Blueprint, C++, 툴 이용 등
- CPP             : C++ only
- Algorithm       : 코테류, 알고리즘 풀기
- Project         : 프로젝트
- Git             : GitHub, GitPages 등 지식
- TroubleShooting : 오류, 디버깅
- Dev             : 컴퓨터 공학, 디자인 패턴 등등
