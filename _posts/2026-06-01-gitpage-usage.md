---
title: "GitPage 운영가이드"
date: 2026-06-01 16:50:00
categories: [Git,GitPages]
tags: [Category]
---

# 깃허브 페이지(GitHub Pages) 운영 가이드

## 1. 카테고리
* **머리말(Front Matter):** 파일 최상단에 카테고리 명시하여 자동 분류

```yaml
  ---
  layout: post
  title: "포스트 제목"
  categories: [UnrealEngine, CPlusPlus]
  ---
```

## 2. 파일 관리
* **파일명 규칙 (Jekyll):** 반드시 `YYYY-MM-DD-제목.md` 형식 준수 (예: `2026-06-01-git-pages.md`)
* **확장자:** `.md` (Markdown) 필수, `.txt`는 적용 불가

## 3. 이미지 업로드 전략
* **이미지**
  - 자주 사용 : `/assets/img/*.png` 형태로 업로드
  - 1회성 사용 : `git/issues/` 에 업로드 후 마크다운 링크 추출

