---
title: "TPS 개발 : 무기 시스템 (4) — Equip을 GA로 통일"
date: 2026-07-05 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Weapon, UnrealEngine-GAS, UnrealEngine-GA, UnrealEngine-GameplayTag, TroubleShooting]
description: "컨트롤러에 붙인 Equip이 스왑 중 태그를 못 떼는 버그, Equip을 GA로 만들어 태그로 통일"
---

# TPS 개발 - 무기 시스템 (4)

무기 장착이 동작은 했지만, 구조에 문제가 있었다. **Equip을 그냥 컨트롤러에 붙여둔** 게 화근이었다.

---

## 문제 — 스왑 중 태그가 안 떨어진다

무기 장착(Equip)을 `PlayerController`에 직접 붙여서 1번 2번 누르면 스왑되도록 처리했다. 평소엔 잘 됐다.

그런데 **사격 중 / 재장전 중에 무기를 스왑**하면 버그가 났다.

```
Fire 진행 중 (State.Fire 태그 부여됨)
  └─ 무기 스왑
       └─ Fire 어빌리티는 중단됐는데...
            └─ State.Fire 태그가 안 떨어짐  ← 버그
```

Fire나 Reload는 GA라서 상태 태그를 붙이는데, 스왑은 컨트롤러가 GAS 바깥에서 처리하다 보니 **진행 중인 어빌리티를 제대로 캔슬/정리하지 못했다.** 태그만 덩그러니 남아서, 이후 입력이 막히는 식이었다.

GAS 밖에서 GAS 상태를 건드리려니 생긴 문제다.

---

## 해결 — Equip을 GA로 만들기

Equip 자체를 **GameplayAbility로** 만들었다. 이러면 장착도 GAS 판 위에서 놀게 되어, 태그로 다른 어빌리티와 상호작용할 수 있다.

```cpp
// GA_EquipWeapon (의사코드)
UGA_EquipWeapon::UGA_EquipWeapon()
{
    // 스왑 어빌리티 식별 태그
    AbilityTags.AddTag(TAG_Ability_Weapon_Swap);

    // 스왑이 시작되면 진행 중인 사격/재장전을 캔슬
    CancelAbilitiesWithTag.AddTag(TAG_State_Fire);
    CancelAbilitiesWithTag.AddTag(TAG_State_Reload);
}
```

핵심은 **스왑도 태그로 관리**하는 것이다.

- 스왑이 시작되면 `CancelAbilitiesWithTag`로 진행 중인 Fire / Reload를 **정식으로 캔슬**한다.
- 캔슬된 어빌리티는 `EndAbility` 경로를 타면서 자기 태그(`State.Fire` 등)를 **스스로 정리**한다.

GAS가 어빌리티 종료와 태그 정리를 책임지니, 태그가 남는 버그가 사라졌다.

```
스왑 GA 발동
  └─ CancelAbilitiesWithTag(State.Fire / State.Reload)
       └─ Fire/Reload GA EndAbility → 태그 정리 ✓
            └─ 장착 진행
```

---

## 전반적인 Equip 통일

마침 팀에 이미 개발돼 있던 기능이 하나 있었다. **다이얼로 노즐을 선택하는 옵션**인데, 이것도 장착 계열 동작이라 같은 문제를 안고 있었다.

그래서 팀원에게 이 노즐 선택도 **GA로 분리해달라고 요청**했다. 장착과 관련된 동작을 전부 GA로 맞춰서, **Equip 계열을 하나의 방식으로 통일**한 것이다.

| 동작 | 이전 | 이후 |
|---|---|---|
| 무기 장착 | Controller 직접 | GA |
| 무기 스왑 | Controller 직접 | GA (태그로 Fire/Reload 캔슬) |
| 노즐 선택(다이얼) | 별도 처리 | GA |

장착과 관련된 모든 게 GAS 위에서 태그로 조율되니, "무기 관련 동작 중엔 무엇을 막고 무엇을 캔슬한다"는 규칙을 태그 하나로 일관되게 표현할 수 있게 됐다.

---

## 오늘 정리

- 컨트롤러에 붙인 Equip은 **GAS 바깥**이라 스왑 시 진행 중 어빌리티 태그를 못 뗌
- **Equip을 GA로** 전환, 스왑도 태그(`CancelAbilitiesWithTag`)로 Fire/Reload 정식 캔슬
- 팀원의 **노즐 선택(다이얼)도 GA로 분리** 요청 → Equip 계열 전면 통일
- 장착·스왑·노즐 선택이 모두 GAS 태그 체계 안으로 들어와 상태 관리가 일관됨
