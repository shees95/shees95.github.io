---
title: "TPS 개발 : 무기 시스템 (3) — 스페셜 탄 · 업그레이드 · UI 테스트"
date: 2026-07-04 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Weapon, UnrealEngine-Definition, UnrealEngine-Ammo, UnrealEngine-UI]
description: "웨폰 컴포넌트에 스페셜 탄 리스트·총기 업그레이드 Definition·탄 선택·장착을 붙이고 UI로 테스트"
---

# TPS 개발 - 무기 시스템 (3)

Weapon Definition을 하나로 통일했으니, 오늘은 그 위에 **실제로 굴러가는 무기 컴포넌트**를 완성했다. 스페셜 탄 관리부터 총기 업그레이드, 탄 선택, 장착, 그리고 UI 테스트까지 한 번에 붙였다.

---

## 스페셜 탄 리스트

크래프팅으로 만든 특수탄을, 무기에서 실제로 쓸 수 있게 **웨폰 컴포넌트에 스페셜 탄 리스트**를 만들었다.

핵심은 어떤 탄을 이 리스트에 **넣고 뺄지**를 런타임에 관리하는 것이다.

```cpp
// WeaponComponent (의사코드)
UPROPERTY()
TArray<UAmmoInstance*> SpecialAmmoList;   // 장전 가능한 스페셜 탄 목록

void AddSpecialAmmo(UAmmoInstance* Ammo);      // 리스트에 추가
void RemoveSpecialAmmo(UAmmoInstance* Ammo);   // 리스트에서 제거
```

크래프팅으로 얻은 특수탄을 리스트에 넣으면 그 무기로 쓸 수 있게 되고, 빼면 사라진다. 인벤토리와 무기 사이를 잇는 다리 역할이다.

---

## 총기 업그레이드 Definition

무기를 강화하는 **업그레이드 Definition**을 따로 만들었다. Definition 시스템의 확장선이다.

```
UWeaponUpgradeDefinition
  ├─ 대상 무기 (어떤 무기에 적용?)
  ├─ 스탯 변화량 (데미지 / 연사 / 탄퍼짐 ...)
  └─ 적용 조건 / 비용
```

업그레이드를 적용하면 Weapon Definition의 CombatAS 값이 갱신되는 방식이라, 통일해둔 Definition 구조 덕에 붙이기가 수월했다.

---

## 어떤 탄을 쓸지 — 탄 선택

무기에 스페셜 탄 리스트가 생겼으니, **지금 어떤 탄을 장전해서 쏠지** 고르는 로직도 필요했다.

```cpp
// 현재 선택된 탄
UPROPERTY()
UAmmoInstance* SelectedAmmo;

void SelectAmmo(int32 Index);   // 리스트에서 사용할 탄 선택
```

발사 시에는 `SelectedAmmo`가 가진 Ability/Effect가 적용된다. 화염탄을 고르면 화염 효과가, 관통탄을 고르면 관통이 나가는 식이다.

---

## 무기 장착

2편에서 잡아둔 장착 API를 실제 컴포넌트에 연결해, 무기 장착까지 전부 동작하게 했다.

```
가방(보유 무기 리스트)
  └─ EquipWeapon(Index)
       └─ Weapon Definition에서 메쉬·몽타주 로드
            └─ 소켓에 스폰 & CombatAS 반영
                 └─ 스페셜 탄 리스트 연결
```

장착과 동시에 그 무기의 스페셜 탄 리스트가 물려서, 장착 → 탄 선택 → 발사가 자연스럽게 이어진다.

---

## UI 만들고 테스트

마지막으로 이 모든 걸 조작할 **UI를 만들어서 직접 테스트**했다.

- 보유 무기 목록에서 무기 선택 → 장착
- 스페셜 탄 리스트에 탄 추가/제거
- 사용할 탄 선택
- 업그레이드 적용

UI에서 버튼을 누르는 대로 무기 컴포넌트가 반응하는 걸 확인했다. Definition을 하나로 통일해둔 게 여기서 제대로 값을 했다 — UI는 Definition만 바라보면 되니 연결이 깔끔했다.

![260704 1345.png](/assets/netkarma/260704%201345.png)

---

## 오늘 정리

- 웨폰 컴포넌트에 **스페셜 탄 리스트** + 추가/제거 관리
- **총기 업그레이드 Definition**으로 CombatAS 강화
- **탄 선택** 로직 (선택한 탄의 Ability/Effect가 발사에 반영)
- 장착 API 실연결 → 장착·탄 선택·발사가 하나로 이어짐
- **UI 제작 후 실제 테스트**로 전체 흐름 검증

인벤토리에서 시작해 상점, 무기까지 이어진 시스템이 드디어 손에 잡히는 형태로 굴러가기 시작했다.
