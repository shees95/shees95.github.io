---
title: "TPS 개발 : 라이라 Experience와 Game Feature 개념 정리"
date: 2026-07-14 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Lyra, UnrealEngine-Experience, UnrealEngine-GameFeature]
description: "라이라의 Experience와 Game Feature 개념 — 레벨 이동 없이 하나의 게임모드 안에서 모든 걸 처리하는 구조"
---

# TPS 개발 - 라이라 Experience 개념 정리

프로젝트가 라이라 위에서 굴러가다 보니, 라이라의 핵심 개념인 **Experience**와 **Game Feature**를 다시 짚고 넘어갈 필요가 있었다.

---

## Experience — 라이라가 만든 확장 게임모드

`Experience`는 라이라가 기본 `AGameModeBase` 위에 만든 **확장된 게임모드 개념**이다. 하나의 Experience 에셋이 아래를 전부 정의한다.

- **맵 / 에셋 로딩** — 어떤 맵과 에셋을 로드할지
- **게임모드 설정** — 어떤 룰로 게임을 진행할지 (스폰 규칙, 팀 구성 등)

즉 기존처럼 "맵 = 게임모드 하나"로 고정하는 게 아니라, **같은 맵 위에서도 다른 Experience를 얹으면 다른 게임이 되는** 구조다.

---

## Game Feature — 런타임 중 게임모드를 바꾸는 플러그인

`Game Feature`는 **게임 모드를 런타임 중에 변경**할 수 있게 해주는 플러그인 시스템이다.

핵심은 이거다.

> **레벨을 왔다갔다 하지 않고, 하나의 게임모드 안에서 모든 게 가능하도록 만든 것.**

전통적인 방식이라면 "웨이브 모드"와 "상점 모드"가 서로 다른 레벨/게임모드였을 수도 있는데, Game Feature를 쓰면 **레벨 전환 없이** 필요한 기능(전투, 상점, 크래프팅 등)을 게임 진행 중에 켜고 끌 수 있다.

```
기존 방식
  Level A (전투 게임모드) → Level 전환 → Level B (상점 게임모드)

Game Feature 방식
  단일 Experience 안에서
    └─ 전투 Feature ON/OFF
    └─ 상점 Feature ON/OFF
       (레벨 전환 없이 같은 자리에서 전환)
```

우리 프로젝트가 웨이브(전투) ↔ 상점을 반복하는 로그라이크 구조라, 이 개념이 정확히 들어맞는다. 상점 시스템과 전투 사이를 레벨 전환 없이 오갈 수 있는 게 이 Experience/Game Feature 구조 덕분이다.

---

## 오늘 정리

- **Experience** = 라이라가 만든 확장 게임모드. 맵/에셋 로딩 + 게임모드 설정을 하나로 관리
- **Game Feature** = 게임모드를 런타임 중 바꾸는 플러그인
- 핵심 가치: **레벨 전환 없이 하나의 게임모드 안에서 전투·상점 등 모든 걸 처리**
- 우리 게임의 웨이브 ↔ 상점 반복 구조가 이 개념 위에서 자연스럽게 성립함
