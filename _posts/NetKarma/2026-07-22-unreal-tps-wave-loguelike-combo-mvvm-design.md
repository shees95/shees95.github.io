---
title: "TPS 개발 : 콤보 카운트 정규화와 MVVM 경계"
date: 2026-07-22 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-MVVM, UnrealEngine-UMG, UnrealEngine-GameplayMessage]
description: "행동당 1콤보만 세도록 이벤트에 대표 플래그를 싣는 설계, 그리고 ViewModel이 게임로직 구조체를 몰라야 하는 이유"
---

# TPS 개발 - 콤보 카운트 정규화와 MVVM 경계

오늘은 설계 결정 두 개를 정리한다. 하나는 콤보 카운트를 "타격 수"가 아니라 "공격 행동 수"로 바꾼 것, 다른 하나는 UI 레이어 경계를 다시 그은 것이다.

---

## 1. 콤보 카운트를 "타격 수"가 아니라 "공격 행동 수"로 정규화

### 요청

관통/펠릿(샷건)/근접 클리브/폭발 스플래시처럼 **한 번의 공격 행동이 여러 개별 히트를 만드는 경우**에도 콤보는 행동당 1회만 올라야 한다.

### 설계

`FNKMDamageApplicationParams`/`FNKMWeaponHitMessage`에 `bCountsForCombo` 플래그를 추가하고, 각 공격 GA(Fire/Melee/Throwable/PistolExplosion)가 **"이번 행동에서 이미 카운트했는가"** 를 자체적으로 추적(멤버 변수 or 로컬 변수)해서 첫 성공 히트에만 `true`를 실어 보냈다.

```
Fire (펠릿 3발 관통)
  펠릿1 히트 → bCountsForCombo = true   (이번 행동 첫 히트)
  펠릿1 관통 후 2번째 대상 → bCountsForCombo = false
  펠릿2 히트 → bCountsForCombo = false
  ...
```

리셋 타이밍은 행동 종류에 맞춰 스코프를 잡았다 — Fire는 `ProcessFireHits()` 1회(펠릿+관통 전체) 스코프, Melee는 스윙 1회 스코프.

### 교훈

"이벤트가 여러 번 발생하지만 논리적으로는 1회"인 상황은 이벤트 자체에 "이게 대표 이벤트인지" 플래그를 실어 보내는 게, 구독자마다 각자 중복제거 로직을 짜는 것보다 훨씬 안전하고 일관됐다. 리스너가 여러 개(리워드 컴포넌트, 스탯 컴포넌트)여도 다 같은 판단을 공유하기 때문이다.

---

## 2. MVVM ViewModel은 게임로직 쪽 메시지 구조체를 직접 알면 안 됨

### 포인트

콤보 리워드 UI를 만들면서 `SetLiveState(const FNKMPendingCoinChangedMessage&, ...)`처럼 ViewModel이 리워드 컴포넌트의 메시지 구조체를 직접 받게 짰다가, "뷰모델이 리워드 컴포넌트 구조체를 이용한 게 맞나?"라는 질문에 걸렸다.

기존 코드를 다시 보니 전부 **원시값만** 받고 있었다.

```cpp
FloatingDamageViewModel::SetPresentation(float, bool, bool)
WaveCoinViewModel::SetRewardState(int32, float, ...)
```

메시지 언패킹은 항상 **Widget(View)의 책임**이었다.

```
게임로직 메시지 구조체
  └─ Widget(View)이 구독, 필드를 꺼냄
       └─ ViewModel엔 원시값만 넘김 (float, int32, bool ...)
            └─ ViewModel은 그 값을 표시용으로만 가공
```

### 교훈

ViewModel은 "표시할 값"만 알아야 하고, 그 값이 어디서/어떤 구조체로 왔는지는 몰라야 한다. 그래야 게임로직 쪽 구조체가 나중에 바뀌어도(flat 필드 → entry 배열로 대격변해도) ViewModel/UI는 전혀 안 건드려도 된다 — 실제로 리워드 구조체를 통째로 갈아엎었는데 ViewModel은 코드 한 줄도 안 바뀌었다.

---

## 정리

- 여러 번 발생하는 이벤트 중 "대표 1건"을 표시해야 하면, 판단 로직을 구독자마다 복제하지 말고 **이벤트에 플래그를 실어서** 발신 측에서 한 번만 결정
- ViewModel은 **원시값만** 받고, 메시지 구조체 언패킹은 View(Widget)가 담당 — 게임로직 구조체 변경에 UI가 영향받지 않게 하는 경계
