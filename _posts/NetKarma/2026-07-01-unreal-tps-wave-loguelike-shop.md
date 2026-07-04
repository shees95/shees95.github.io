---
title: "TPS 개발 : 상점 시스템 — 구매·판매·크래프팅"
date: 2026-07-01 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Shop, UnrealEngine-Crafting, UnrealEngine-Definition, UnrealEngine-CSV]
description: "상점 골조 구축, 크래프팅을 재료 N개 조합으로 확장하고 재료별 효율(개수)을 CSV로 관리"
---

# TPS 개발 - 상점 시스템

상점의 기본 골조가 나왔다. **구매·판매·크래프팅** 세 기능이 동작한다. 만들다 보니 크래프팅 설계가 또 한 번 크게 바뀌었다.

---

## 상점 골조

상점에서 다루는 아이템은 인벤토리 아이템과 관심사가 다르다. 가격, 재고, 판매가 같은 **상점 전용 데이터**가 필요하다.

그래서 아이템 Definition을 상점용으로 따로 구현했다.

```
UItemDefinition_Shop
  ├─ Origin 참조 (어떤 아이템인지)
  ├─ 구매 가격
  ├─ 판매 가격
  └─ 재고 / 노출 조건
```

이걸 기반으로 세 가지 상호작용을 붙였다.

| 기능 | 동작 |
|---|---|
| 구매 | 재화 차감 → ItemManager로 인스턴스 Build → 인벤 추가 |
| 판매 | 인벤에서 제거 → 판매가만큼 재화 지급 |
| 크래프팅 | 재료 소모 → 특수탄 인스턴스 생성 |

크래프팅을 위해 어제 만든 제작 전용 Definition을 활용하는 **Craft Definition**도 만들었다.

---

## 크래프팅, 또 갈아엎다

어제는 "재료 2개"를 가정했다. 그런데 상점 UI로 실제로 만들어보니, **재료를 굳이 2개로 제한할 이유가 없었다.**

특성을 이어붙이는 구조라, 재료를 3개든 4개든 막 갖다 붙여도 되는 설계였다. 그래서 그렇게 열어버렸다.

### 이름과 효과가 "이어붙는" 방식

결과물에 고정된 이름을 줄 수가 없다. 조합이 매번 다르니까. 그래서 **재료명을 이어붙여** 이름을 만들고, 효과도 이어붙인다.

```
재료: Fire + Pierce + Water
결과물 이름: "Fire Pierce Water" (재료명 연결)
결과물 효과: [화염] + [관통] + [기본] (효과 연결)
```

특정 결과물을 미리 정의하지 않아도, 재료만 넣으면 이름과 효과가 자동으로 합쳐진다.

---

## 재료마다 효율이 있어야 하잖아?

여기서 문제가 하나 보였다. 재료를 그냥 "1개씩 넣으면 1개 나온다"로 하니 밸런스가 밋밋했다.

**재료마다 효율이 달라야** 재미가 있다. 귀한 재료는 조금만 넣어도 많이 나오고, 흔한 재료는 잔뜩 넣어야 조금 나오는 식.

그래서 CSV에 **열을 두 개 더 팠다.**

| 컬럼 | 의미 |
|---|---|
| `RequireCount` | 제작에 필요한 재료 개수 |
| `ReturnCount` | 크래프트 결과물 개수 |

### 제작 로직

```cpp
// 의사코드
bool UShopComponent::TryCraft(const TArray<UItemInstance*>& Materials)
{
    // 1. 각 재료가 요구 개수만큼 있는지 전부 체크
    for (const auto& Mat : Materials)
    {
        if (GetOwnedCount(Mat) < Mat->GetDef()->RequireCount)
            return false;   // 하나라도 모자라면 제작 불가
    }

    // 2. 재료 소모
    ConsumeMaterials(Materials);

    // 3. ReturnCount 만큼 결과물 지급
    const int32 ResultCount = CalcReturnCount(Materials);
    GiveCraftedItem(BuildCraftedDef(Materials), ResultCount);
    return true;
}
```

예를 들어 기본탄 재료인 **Water**를 섞으면, Water의 `ReturnCount`만큼 결과물을 제공하는 식으로 동작한다. 재료가 요구 개수만큼 있는지 전부 검사한 뒤, 통과하면 정해진 개수만큼 결과물을 준다.

---

## 오늘 정리

- 상점 전용 Definition으로 **구매·판매·크래프팅** 골조 완성
- 크래프팅을 재료 2개 → **N개 조합**으로 확장
- 결과물 이름은 재료명 연결, 효과는 효과 연결 (레시피리스 유지)
- CSV에 `RequireCount` / `ReturnCount` 열 추가로 **재료별 효율** 반영
- 요구 개수 전부 체크 → 소모 → 결과 개수만큼 지급

상점까지 왔으니 이제 진짜 문제, **무기 시스템**이 남았다.
