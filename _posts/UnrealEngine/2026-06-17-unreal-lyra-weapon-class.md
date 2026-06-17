---
title: "언리얼 라이라 - 무기 시스템 클래스 구조"
date: 2026-06-17 00:00:00 +0900
categories: [UnrealEngine, UnrealEngine-StarterProject, UnrealEngine-Lyra]
tags: [UnrealEngine, UnrealEngine-Lyra, UnrealEngine-GAS, UnrealEngine-GA, UnrealEngine-AbilitySet, UnrealEngine-Inventory, UnrealEngine-Equipment, UnrealEngine-Fragment]
description: "GiveWeapon 흐름, 인벤토리/장비 클래스 구조, AbilitySet, 커스텀 무기 설정까지 정리"
---

# 라이라 무기 시스템 클래스 구조

---

## GiveWeapon 함수 흐름

GiveWeapon은 **퀵바 슬롯, 인벤토리 매니저, 무기 인스턴스** 세 가지를 다룬다.

```
GiveWeapon(QuickBarSlot, Inventory, WeaponInstance)
```

인벤토리에 해당 무기가 이미 있는지 여부에 따라 분기:

```
인벤에 무기가 있으면
  └─ Stack만 처리

없으면
  └─ Cue Tag 전달
       └─ 인벤에 ItemDef 추가
            └─ 퀵바에 Item 추가
                 └─ 퀵바 슬롯 Activate
```

---

## 인벤토리 클래스 구조

### InventoryManager

인벤토리 전체를 관리하는 컴포넌트.

| 구성 | 설명 |
|---|---|
| `Struct 인벤 리스트` | 전체 아이템 목록, 변경사항 반영 함수 포함 |
| `AddEntry()` | 아이템 배열 추가, 인스턴스 생성, Fragment 주입 |

### ItemInstance

런타임 아이템 상태를 담는 클래스.

- 네트워크 복제 여부 설정
- 태그 보유 / 추가 / 제거
- 정의(Definition) Set/Get
- `OnEquip (BP)` → 애니메이션 실행
- `OnUnEquip (BP)` → 애니메이션 실행

### ItemDefinition

아이템 설계 데이터.

- 이름
- Fragment 리스트

### Fragment

ItemDefinition에 붙는 모듈식 데이터 조각. 상속 구조로 확장.

| Fragment 종류 | 역할 |
|---|---|
| `Stat` | 태그 기반 수치 Get/Set (탄약 등) |
| `IconDrawer` | HUD 아이콘 표시용 Brush |
| `EquippableItem` | EquipDefinition 참조 |
| `Reticle` | 조준선 설정 (Common Widget) |

---

## 장비 클래스 구조

### EquipmentDefinition

장비 설계 데이터.

- 스폰할 Actor
- 붙일 소켓
- Transform 세팅
- 어떤 어빌리티(AbilitySet)를 줄지

### EquipmentInstance

런타임 장비 상태.

- 네트워크 복제
- 생성 리스트
- 장비한 Actor 리스트
- 장비한 Actor들에게 소켓 및 Transform 기준으로 Mesh Component Attach

---

## AbilitySet (라이라용)

어빌리티, 이펙트, 어트리뷰트셋을 묶어 한 번에 부여/회수하는 DataAsset.

```
AbilitySet
  ├─ AS List
  ├─ GA List  →  LyraGameplayAbility (라이라용 베이스)
  ├─ GE List
  └─ GiveToAbilitySystem()
```

### LyraGameplayAbility

라이라 전용 GA 베이스 클래스. 일반 `UGameplayAbility`를 확장한다.

- 사용 트리거 정책
- 사용 코스트
- 카메라 모드 설정
- 능력 추가 (BP 오버라이드)
- 능력 삭제 (BP 오버라이드)

---

## InventoryDefinition 실제 구성

커스텀 무기를 만들 때 `InventoryItemDefinition` 기반으로 ID 타입을 만들고, Fragments에 아래 항목들을 추가한다.

```
ID 타입 (InventoryItemDefinition 서브클래스/에셋)
  └─ Fragments
       ├─ EquipDefinition → WID BP (WeaponInventoryDefinition 에셋)
       │     ├─ InstanceType
       │     ├─ 데이터 에셋
       │     ├─ 생성할 BP (무기 Actor)
       │     ├─ 붙일 소켓
       │     └─ Transform 세팅
       ├─ Icons           → Brush
       ├─ Stat            → Map<Tag, Value>  (탄약 초기값 등)
       └─ Reticle         → Common Widget
```

---

## 커스텀 무기 예시 — Rifle

### B_Rifle (무기 Actor BP)

- 에셋 (스켈레탈 메시)
- 애니메이션

### B_WeaponInstance_Rifle (WeaponInstance)

총기 고유 수치를 설정하는 인스턴스.

| 항목 | 설명 |
|---|---|
| 사격당 총알 수 | |
| 최대거리 | |
| Sweep Radius | |
| 거리 데미지 | |
| 정밀도 배율 | |
| Montage 설정 | |
| 탄 퍼짐 | |
| 탄 퍼짐 회복 | |
| 상태별 탄퍼짐 이동속도 | 이동/정지/점프 등 상태에 따라 다른 탄퍼짐 |

### AbilitySet_Rifle

Rifle에 부여하는 어빌리티 묶음.

```
AbilitySet_Rifle
  ├─ GA_FireAuto_Rifle
  ├─ GA_Reload_Rifle
  └─ GA_AutoReload
```
