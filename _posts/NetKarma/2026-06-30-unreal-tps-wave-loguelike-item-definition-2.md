---
title: "TPS 개발 : 아이템 Definition 시스템 (2) — 크래프팅이 깨뜨린 구조"
date: 2026-06-30 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Inventory, UnrealEngine-Definition, UnrealEngine-Crafting, UnrealEngine-GAS]
description: "구매 전용으로 만든 Definition이 크래프팅에서 무너진 이유와 제작 전용 Definition 분리"
---

# TPS 개발 - 아이템 Definition 시스템 (2)

어제만든 Definition 구조는 사실 **구매(상점 획득)만 염두에 둔 설계**였다. 오늘 크래프팅을 붙이려다 이 구조가 무너졌다.

---

## 문제 — 크래프팅은 Definition을 미리 만들 수 없다

우리 게임의 크래프팅은 이렇다.

> **재료 2개 이상 섞어서 특수탄을 만든다.**

처음엔 여느 게임처럼 "레시피 테이블"을 생각했다. `재료A + 재료B = 결과물C` 같은 표를 만들고, 결과물 C의 Definition을 미리 정의해두는 방식.

그런데 우리 크래프팅은 **레시피가 없다.** 결과물이 고정돼 있지 않고, **재료의 특성을 연결해서** 그때그때 결과가 결정된다.

즉, 결과물 Definition을 CSV에 미리 박아둘 수가 없다. 어제 만든 "CSV 행 = Definition" 구조와 정면충돌한 것이다.

```
기존 (구매용)
CSV 행 → Definition → Instance          (결과물이 이미 정해져 있음)

크래프팅
재료A 특성 + 재료B 특성 → ??? → 특수탄     (결과물이 조합으로 결정됨)
```

---

## 해결 — 제작 전용 Definition 분리

구매용 Definition에 크래프팅을 욱여넣는 대신, **특수탄 제작용 Definition을 따로** 만들었다.

핵심은 이 제작용 Definition이 **효과를 동적으로 담을 수 있어야** 한다는 것이다. 재료가 무엇이냐에 따라 붙는 능력과 효과가 달라지니까.

그래서 `TArray`로 Ability와 Effect를 붙일 수 있게 만들었다.

```cpp
// 특수탄 제작용 Definition
UCLASS()
class UItemDefinition_CraftedAmmo : public UItemDefinition_Origin
{
    GENERATED_BODY()
public:
    // 재료 특성에 따라 동적으로 채워지는 능력/효과
    UPROPERTY(EditDefaultsOnly)
    TArray<TSubclassOf<UGameplayAbility>> GrantedAbilities;

    UPROPERTY(EditDefaultsOnly)
    TArray<TSubclassOf<UGameplayEffect>> GrantedEffects;
};
```

재료 A가 "추가데미지" 특성을 갖고 재료 B가 "경직" 특성을 가지면, 두 특성의 Ability/Effect가 이 배열에 **누적**되어 하나의 특수탄이 만들어진다.

```
재료A (화염 Effect) ┐
                   ├─→ CraftedAmmo Definition
재료B (관통 Ability)┘     ├─ GrantedEffects:   [추가데미지]
                         └─ GrantedAbilities: [경직]
```

레시피 없이 **특성을 이어붙이는** 방식이라, 재료 조합이 늘어나도 테이블을 새로 팔 필요가 없다.

---

## 아이템 매니저 대거 수정

Definition 구조가 바뀌니 이걸 인스턴스로 만드는 `ItemManager`도 크게 손봐야 했다.

기존 `Build`는 "CSV에서 온 완성된 Definition"만 가정했는데, 이제는 **런타임에 조합으로 생성된 Definition**도 처리해야 한다.

- 구매 경로: CSV Definition → Build → Instance (기존)
- 제작 경로: 재료 특성 조합 → CraftedAmmo Definition **동적 생성** → Build → Instance (신규)

두 경로가 같은 `Build` 출구로 모이도록 매니저를 정리했다. 결과적으로 인스턴스를 만드는 최종 관문은 하나로 유지했다.

---

## 오늘 정리

- 구매 전용으로 짠 Definition은 **레시피 없는 크래프팅**을 감당 못 함
- 특수탄 **제작 전용 Definition** 분리, `TArray`로 Ability·Effect를 동적으로 부착
- 크래프팅 = 재료 특성(능력/효과)을 **이어붙이는** 방식
- ItemManager를 구매/제작 두 경로 모두 지원하도록 대거 수정

구조는 잡혔지만 아직 "재료 몇 개가 필요한가", "결과물이 몇 개 나오는가"는 손도 못 댔다. 이건 상점을 만들면서 자연스럽게 튀어나온다.
