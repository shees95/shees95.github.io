---
title: "TPS 개발 : 무기 시스템 (2) — Definition 통일"
date: 2026-07-03 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Weapon, UnrealEngine-Definition, UnrealEngine-Equipment]
description: "아이콘·메쉬·몽타주까지 Weapon Definition에 통합해 하나의 정의에서 모든 작업이 이루어지게 개편"
---

# TPS 개발 - 무기 시스템 (2)

무기를 장착하는 것까진 됐는데, 문제가 있었다. 무기를 구성하는 데이터가 **여기저기 흩어져 있었다.**

- 전투 스탯은 Weapon Definition
- UI 아이콘은 상점 쪽
- 메쉬랑 몽타주는 또 다른 데서 참조

한 자루의 총을 화면에 올리는데 여러 군데를 뒤져야 했다. 오늘은 이걸 정리했다.

---

## Weapon Definition 개편

Weapon Definition이 **무기에 관한 모든 것**을 갖도록 확장했다.

```
UWeaponDefinition (개편 후)
  ├─ Origin           (신원 - ID, 이름)
  ├─ CombatAS 값       (데미지 / 연사 / 반동 / 탄퍼짐)
  ├─ UI 아이콘         (신규)
  ├─ 스켈레탈 메쉬      (신규)
  └─ 몽타주 세트        (신규 - Fire / Reload / Dash ...)
```

### 아이콘

UI에서 무기 슬롯을 그릴 때 쓰는 아이콘을 Definition에 직접 넣었다. 이제 UI는 상점을 거치지 않고 **장착된 무기의 Definition에서 바로** 아이콘을 가져온다.

### 메쉬 & 몽타주

무기를 손에 스폰할 때 필요한 스켈레탈 메쉬와, 발사/재장전/대시 시 재생할 몽타주 세트도 Definition에 포함시켰다.

```cpp
// 개편된 Weapon Definition (의사코드)
UCLASS()
class UWeaponDefinition : public UItemDefinition_Origin
{
    GENERATED_BODY()
public:
    // 전투 스탯
    UPROPERTY(EditDefaultsOnly) FWeaponCombatStats CombatStats;

    // UI
    UPROPERTY(EditDefaultsOnly) TObjectPtr<UTexture2D> Icon;

    // 표현
    UPROPERTY(EditDefaultsOnly) TObjectPtr<USkeletalMesh> WeaponMesh;

    // 몽타주 세트
    UPROPERTY(EditDefaultsOnly) TMap<FGameplayTag, TObjectPtr<UAnimMontage>> Montages;
};
```

---

## "하나의 Definition에서 모든 작업"

이번 개편의 핵심은 통일이다.

```
        [ 개편 전 ]                    [ 개편 후 ]

전투 스탯 → Weapon Def          
아이콘   → Shop 쪽        ───▶     모두 → Weapon Definition
메쉬     → 별도 참조                (단일 진입점)
몽타주   → 또 다른 곳
```

이제 무기와 관련된 어떤 작업이든 **Weapon Definition 하나만 보면 된다.**

- UI가 아이콘을 그릴 때 → Weapon Definition
- 캐릭터가 무기를 스폰할 때 → Weapon Definition
- GA가 몽타주를 재생할 때 → Weapon Definition
- 전투 계산이 스탯을 읽을 때 → Weapon Definition

데이터 진입점이 하나로 모이니, 무기를 추가하거나 수정할 때 헤맬 일이 없어졌다.

---

## 5일간의 흐름 돌아보기

이번 인벤토리·무기 시스템은 5일에 걸쳐 이렇게 굴러왔다.

| 날짜 | 한 일 | 배운 것 |
|---|---|---|
| 6/29 | Definition 뼈대 | CSV → Definition → Instance 분리 |
| 6/30 | 크래프팅 | 레시피리스 = 특성 이어붙이기, 제작용 Def 분리 |
| 7/01 | 상점 | 재료 N개 조합, 효율(RequireCount/ReturnCount) |
| 7/02 | 무기 장착 | Origin 재활용으로 Shop→Weapon 재빌드 |
| 7/03 | Definition 통일 | 무기의 모든 데이터를 한 곳으로 |

돌이켜보면 **첫날 잡은 "Origin + 타입별 재정의" 구조**가 크래프팅·상점·무기까지 전부 지탱해줬다. 초반 구조 설계에 시간을 쓴 게 결국 이득이었다.

---

## 오늘 정리

- Weapon Definition에 **아이콘·메쉬·몽타주**까지 통합
- 무기 관련 모든 작업이 **하나의 Definition**을 진입점으로 삼도록 통일
- 데이터가 흩어지지 않으니 무기 추가·수정 비용이 급감
