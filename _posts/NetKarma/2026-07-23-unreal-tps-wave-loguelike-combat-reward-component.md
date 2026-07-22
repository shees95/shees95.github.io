---
title: "TPS 개발 : 전투 리워드 컴포넌트 — 카테고리·연산방식 분리 설계"
date: 2026-07-23 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-GameplayMessage, UnrealEngine-DataDriven]
description: "코인 보상 로직을 카테고리(무엇)와 스케일모드(어떻게)로 분리하고, 타격마다 코인을 캐싱했다가 콤보 종료 시 배율을 곱해 지급하는 구조"
---

# TPS 개발 - 전투 리워드 컴포넌트

킬/데미지/콤보/커스텀 타쿠 처치 등 여러 출처의 코인 보상을 하나로 계산하는 `UNKMCombatRewardComponent`를 정리했다.

---

## 배경 — 필드가 반복되던 문제

기획자가 "스케일 합할 때 연산에 영향 끼치는 종류들을 이넘으로 정의해서 줘야 한다"고 요청했다. 기존엔 `bKillBonus/DealtCoin`, `bSandevistanBonus/SandevistanScale` 식으로 카테고리마다 필드가 반복되고, "이게 곱연산(스케일)에 들어가는지 그냥 가산인지"가 필드명(`XxxCoin` vs `XxxScale`)에만 암묵적으로 담겨 있었다.

## 리팩터 — 카테고리(무엇) + 스케일모드(어떻게)

```cpp
enum class ENKMCoinRewardCategory : uint8
{
    Dealt, Kill, Sandevistan, Dash, Penetration,
    Performance, CustomTaku, Combo, HeadCombo,
};

enum class ENKMCoinScaleMode : uint8
{
    Additive,           // BaseCoin에 그대로 더해짐 (Value = 코인량)
    InstantScale,       // 이번 타격의 ScaleSum에 즉시 더해짐 (Value = 배율)
    DeferredMultiplier, // 스트릭 최댓값만 추적하다 스트릭 종료 시 한 번에 곱해짐
};

struct FNKMCoinRewardEntry
{
    ENKMCoinRewardCategory Category;
    ENKMCoinScaleMode Mode;
    float Value;
    float ContributedCoin; // 최종 배분 후 채워지는 값 (breakdown/로그용)
};
```

새 보너스 종류를 추가할 때 이제 **enum 값 + `Entries.Add(...)` 한 줄**이면 끝난다. "이게 곱연산에 들어가는지"를 필드명에서 추측할 필요가 없다.

---

## 계산 흐름 — 타격마다 캐싱, 스트릭 종료 시 확정

타격 1회당(콤보/헤드콤보 제외):

```
(Damage * CoinPerDamageDealt + (죽였으면 CoinPerKill))
  * (산데비스탄배율 + 대쉬배율 + 관통배율 + 처치율성과배율 + 커스텀타쿠배율)
```

- 산데비스탄/대쉬 배율은 타격 순간 해당 상태 태그가 켜져있을 때만 더해진다(꺼져있으면 0)
- 관통배율은 Fire 전용 — `CoinScaleOnPenetration[관통순번]` 배열, 범위를 넘으면 마지막 값 고정. Melee/Grenade/PistolExplosion은 0

이 결과는 매 타격 `PendingCoin`에 그대로 누적된다.

콤보/헤드콤보 배율은 **타격마다 반영하지 않는다.** 대신 콤보 스트릭이 이어지는 동안 `CoinScaleOnCombo[콤보수]` / `CoinScaleOnHeadCombo[최고헤드샷콤보수]`의 **최댓값만 계속 추적**하다가, 스트릭이 끝나는 순간(콤보 끊김 또는 웨이브 종료) `ApplyComboMultiplierToPendingCoin()`이 `PendingCoin` 전체에 (최대콤보배율 + 최대헤드콤보배율)을 곱해 최종 지급액을 확정한다.

```
타격1 (콤보1) → PendingCoin += 기본코인, ComboScale 최댓값 갱신
타격2 (콤보2) → PendingCoin += 기본코인, ComboScale 최댓값 갱신
...
콤보 끊김/웨이브 종료
  └─ PendingCoin *= (1 + 최대ComboScale + 최대HeadComboScale)
       └─ FlushPendingCoinToInventory()
```

배율표는 CSV(`UNKMCoinRewardTableParser`)에서 로드한다.

---

## 콤보 카운트를 컴포넌트가 독립적으로 추적하는 이유

`UNKMCombatRewardComponent`는 `LocalShotCombo`/`LocalHeadshotCombo`를 **`UNKMCombatStatComponent`의 콤보 카운트와 별도로** 자체 추적한다. 이유는 주석에 명시돼 있다:

> `Message_Weapon_Hit/Miss`만으로 독립 추적하는 로컬 콤보 카운터 — `UNKMCombatStatComponent`의 `ShotCombo`와 갱신 순서가 보장되지 않아(같은 메시지의 리스너 호출 순서는 비결정) 여기서 따로 들고 간다.

같은 게임플레이 메시지를 두 컴포넌트가 각각 구독하는데, **리스너 호출 순서가 언어/엔진 차원에서 보장되지 않는다.** 그래서 "다른 컴포넌트가 먼저 갱신했겠지"라고 가정하지 않고, 코인 계산에 필요한 콤보 카운트를 이 컴포넌트 안에서 완결시켰다.

또한 `LocalComboBestHeadshotCombo`(일반 콤보 스트릭 동안의 헤드샷 콤보 최고치)는 몸샷이 껴서 `LocalHeadshotCombo`가 0으로 리셋돼도 유지되고, `LocalShotCombo` 자체가 끊길 때(Miss)만 함께 리셋된다 — "순간의 헤드샷 여부"가 아니라 "이 콤보 동안의 최고 기록"을 배율 계산 기준으로 삼기 위해서다.

---

## Breakdown — 사후 집계용 역산 구조

`FNKMCoinBreakdown`은 "이번 웨이브 코인 중 몇 %가 어디서 나왔는지"를 역산하기 위한 구조다.

```cpp
struct FNKMCoinBreakdown
{
    TMap<ENKMCoinRewardCategory, float> CoinByCategory;
    int32 TotalCoin;
    int32 HitCount;
};
```

타격의 `ScaleSum`이 5.5이고 그중 관통배율이 1.2였다면, 그 타격 CoinGain의 `(1.2/5.5)`만큼을 `CoinByCategory[Penetration]`에 더하는 식으로 **비례 배분**한다. 반올림 오차를 빼면 전체 합이 `TotalCoin`과 거의 일치한다. 웨이브별로 스냅샷(`CoinBreakdownPerWave`)해두기 때문에, 나중에 "이 웨이브는 커스텀 타쿠 처치 비중이 높았다" 같은 걸 UI/로그로 보여줄 수 있다.

---

## 정리

- 리워드 데이터를 **카테고리(무엇) + 스케일모드(어떻게)** 로 분리해 필드 반복과 암묵적 규칙을 제거
- 타격마다 코인을 계산해 `PendingCoin`에 누적, 콤보/헤드콤보 배율만 스트릭 종료 시 한 번에 곱해서 확정
- 콤보 카운트는 다른 컴포넌트에 의존하지 않고 **이 컴포넌트 안에서 완결** — 메시지 리스너 순서가 보장 안 되기 때문
- `FNKMCoinBreakdown`으로 카테고리별 기여도를 비례 배분해 사후 집계 가능하게 함
