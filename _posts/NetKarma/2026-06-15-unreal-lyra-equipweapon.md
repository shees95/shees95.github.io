---
title: "언리얼 라이라 - 총기 획득 및 사용"
date: 2026-06-15 15:00:00 +0900
categories: [UnrealEngine, UnrealEngine-StarterProject, UnrealEngine-Lyra]
tags: [UnrealEngine, UnrealEngine-Lyra, UnrealEngine-GAS, UnrealEngine-ASC, UnrealEngine-GA, UnrealEngine-AbilitySet, UnrealEngine-Inventory, UnrealEngine-Equipment]
description: "라이라 총기 획득부터 GAS 어빌리티 부여, 입력 처리까지 전체 흐름 정리"
---

# 라이라 총기 획득 및 사용

라이라의 총기 시스템은 **Inventory → Equipment → GAS** 세 레이어가 순서대로 연결되는 구조다.

---

## 전체 흐름

```
ALyraWeaponPickup (월드 액터)
  └─ Interact → ULyraInventoryManagerComponent::AddItemDefinition()
       └─ ULyraInventoryItemInstance 생성 (서버)
            └─ UInventoryFragment_EquippableItem → ULyraEquipmentDefinition 참조
                 └─ ULyraEquipmentManagerComponent::EquipItem()
                      └─ ULyraEquipmentInstance (or ULyraWeaponInstance) 생성
                           └─ ULyraAbilitySet::GiveToAbilitySystem(ASC, &Handle)
                                └─ InputTag ↔ AbilityTag 바인딩 완료
```

---

## 1. Inventory (아이템 획득)

픽업 시 `ULyraInventoryManagerComponent::AddItemDefinition()` 호출 (서버 전용).

`ULyraInventoryItemInstance`가 생성되며, `StatTags`에 탄약 수 등 초기 수치가 복사된다.

```cpp
// Inventory/LyraInventoryManagerComponent.cpp

ULyraInventoryItemInstance* ULyraInventoryManagerComponent::AddItemDefinition(
    TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 StackCount)
{
    ULyraInventoryItemInstance* Result = nullptr;
    if (ItemDef != nullptr)
    {
        Result = NewObject<ULyraInventoryItemInstance>(GetOwner());
        // Fragment들 초기화 (SetStats, EquippableItem 등)
        Result->AddStatTagStack(...);
        InventoryList.AddEntry(Result);
    }
    return Result;
}
```

### ItemDefinition Fragment 구조

`ULyraInventoryItemDefinition`에는 `Fragments` 배열이 있다.
각 Fragment가 모듈식 데이터를 담고, 런타임에 `FindFragmentByClass<T>()`로 읽는다.

```
ULyraInventoryItemFragment (base)
  ├─ UInventoryFragment_EquippableItem    → EquipmentDefinition 참조
  ├─ UInventoryFragment_SetStats          → 초기 StatTag 설정 (탄약 등)
  ├─ UInventoryFragment_QuickBarIcon      → HUD 아이콘
  └─ UInventoryFragment_ReticleConfig     → 조준선 설정
```

> Fragment는 Definition에 붙는다. Instance에 붙지 않는다.

### 탄약 데이터 흐름

```
UInventoryFragment_SetStats (Def)
  └─ InitialItemStats: { Lyra.ShooterGame.Weapon.MagazineAmmo: 30 }
       └─ AddItemDefinition() 시 Instance::StatTags에 복사
            └─ GA_Hero_Fire::CommitAbilityCost()
                 → StatTags.AddStack(Lyra.ShooterGame.Weapon.MagazineAmmo, -1)
```

---

## 2. Equipment (장착)

`UInventoryFragment_EquippableItem`이 참조하는 `ULyraEquipmentDefinition`을 기반으로
`ULyraEquipmentManagerComponent::EquipItem()`이 호출된다.

```cpp
// Equipment/LyraEquipmentDefinition.h

UPROPERTY(EditDefaultsOnly, Category=Equipment)
TSubclassOf<ULyraEquipmentInstance> InstanceType;   // 생성할 인스턴스 타입

UPROPERTY(EditDefaultsOnly, Category=Equipment)
TArray<FLyraEquipmentActorToSpawn> ActorsToSpawn;   // 스폰할 무기 Actor

UPROPERTY(EditDefaultsOnly, Category=Equipment)
TArray<TObjectPtr<const ULyraAbilitySet>> AbilitySetsToGrant; // 부여할 어빌리티셋
```

장착 시 `ULyraEquipmentInstance::OnEquipped()`가 호출되며 GAS에 어빌리티가 부여된다.

```cpp
// Equipment/LyraEquipmentInstance.cpp

void ULyraEquipmentInstance::OnEquipped()
{
    // 무기 Actor 스폰
    SpawnEquipmentActors(GetEquipmentDefinition()->ActorsToSpawn);

    // GAS 어빌리티 부여
    for (const ULyraAbilitySet* AbilitySet : GetEquipmentDefinition()->AbilitySetsToGrant)
    {
        AbilitySet->GiveToAbilitySystem(LyraASC, &GrantedHandles);
    }
}
```

### 해제 (Unequip) 경로

```
EquipmentManager::UnequipItem(Instance)
  └─ Instance::OnUnequipped()
       └─ GrantedHandles.TakeFromAbilitySystem(ASC)
            └─ FGameplayAbilitySpecHandle들 ClearAbility()
                 └─ 스폰된 Actor들 Destroy()
```

---

## 3. GAS 어빌리티 부여

### AbilitySet

`ULyraAbilitySet`은 어빌리티, 이펙트, 어트리뷰트셋을 묶어서 한 번에 부여/회수하는 DataAsset이다.

```cpp
// AbilitySystem/LyraAbilitySet.h

UPROPERTY(EditDefaultsOnly, Category=GameplayAbilities)
TArray<FLyraAbilitySet_GameplayAbility> GrantedGameplayAbilities;

UPROPERTY(EditDefaultsOnly, Category=GameplayEffects)
TArray<FLyraAbilitySet_GameplayEffect> GrantedGameplayEffects;

UPROPERTY(EditDefaultsOnly, Category=AttributeSets)
TArray<FLyraAbilitySet_AttributeSet> GrantedAttributes;

struct FLyraAbilitySet_GameplayAbility
{
    TSubclassOf<ULyraGameplayAbility> Ability;
    int32 AbilityLevel = 1;
    FGameplayTag InputTag;   // ← 여기서 InputTag 지정
};
```

### Grant 시 InputTag 주입

`GiveToAbilitySystem()` 내부에서 `DynamicAbilityTags`에 InputTag를 주입한다.
나중에 `AbilityInputTagPressed()`가 이 태그를 키로 어빌리티를 탐색한다.

```cpp
// AbilitySystem/LyraAbilitySet.cpp

FGameplayAbilitySpec AbilitySpec(AbilityToGrant.Ability, AbilityToGrant.AbilityLevel);
AbilitySpec.SourceObject = SourceObject;
AbilitySpec.DynamicAbilityTags.AddTag(AbilityToGrant.InputTag); // InputTag 주입

FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(AbilitySpec);
GrantedHandles->AbilitySpecHandles.Add(Handle);  // Handle 보관 필수
```

### Grant 경로 두 가지

#### 경로 A — PawnData (베이스 어빌리티)

Experience 로드 시 항상 부여. 이동, 점프, 인터랙션 등.

```
ULyraExperienceDefinition
  └─ DefaultPawnData: ULyraPawnData
       └─ AbilitySets: TArray<ULyraAbilitySet*>
            └─ GiveToAbilitySystem(ASC, &PawnHandles)
```

#### 경로 B — Equipment (무기 어빌리티)

Equip 시 부여, Unequip 시 회수. 무기별 독립 핸들 관리.

---

## 4. 입력 처리

### 바인딩 초기화

Pawn이 Possess될 때 `ULyraHeroComponent::InitializePlayerInput()`이 호출된다.

```
APawn::PossessedBy()   [서버]
APawn::OnRep_Controller()  [클라이언트]
  └─ ULyraHeroComponent::InitializePlayerInput()
       ├─ EnhancedInputSubsystem->AddMappingContext(IMC, Priority)
       ├─ LyraInputComponent->BindNativeAction(...)   // Move, Look, Jump
       └─ LyraInputComponent->BindAbilityActions(...) // Fire, ADS, Reload
```

### Ability Action 바인딩

```cpp
// Input/LyraInputComponent.h

template<class UserClass, typename PressedFuncType, typename ReleasedFuncType>
void BindAbilityActions(
    const ULyraInputConfig* InputConfig,
    UserClass* Object,
    PressedFuncType PressedFunc,
    ReleasedFuncType ReleasedFunc,
    TArray<uint32>& BindHandles)
{
    for (const FLyraInputAction& Action : InputConfig->AbilityInputActions)
    {
        BindAction(Action.InputAction, ETriggerEvent::Triggered,  Object, PressedFunc,  Action.InputTag);
        BindAction(Action.InputAction, ETriggerEvent::Completed,  Object, ReleasedFunc, Action.InputTag);
    }
}
```

### InputTag → Ability 연결

```cpp
// AbilitySystem/LyraAbilitySystemComponent.cpp

void ULyraAbilitySystemComponent::AbilityInputTagPressed(const FGameplayTag& InputTag)
{
    if (InputTag.IsValid())
    {
        for (const FGameplayAbilitySpec& AbilitySpec : GetActivatableAbilities())
        {
            if (AbilitySpec.Ability
                && AbilitySpec.DynamicAbilityTags.HasTagExact(InputTag))
            {
                InputPressedSpecHandles.AddUnique(AbilitySpec.Handle);
                InputHeldSpecHandles.AddUnique(AbilitySpec.Handle);
            }
        }
    }
    // ProcessAbilityInput()에서 실제 TryActivateAbility() 호출
}
```

### 입력 태그 계층

```
InputTag
  ├─ InputTag.Move
  ├─ InputTag.Jump
  └─ InputTag.Weapon
       ├─ InputTag.Weapon.Fire
       ├─ InputTag.Weapon.FireAuto
       ├─ InputTag.Weapon.Reload
       └─ InputTag.Weapon.ADS
```

---

## 클래스 구조 요약

| 레이어 | 클래스 | 역할 |
|---|---|---|
| Inventory | `ULyraInventoryManagerComponent` | 아이템 획득/보관 (서버 전용) |
| Inventory | `ULyraInventoryItemDefinition` | 아이템 설계 DataAsset |
| Inventory | `ULyraInventoryItemInstance` | 런타임 인스턴스 (StatTag 보관) |
| Equipment | `ULyraEquipmentManagerComponent` | Equip/Unequip 관리 |
| Equipment | `ULyraEquipmentDefinition` | 장비 설계 (AbilitySet, 스폰Actor) |
| Equipment | `ULyraEquipmentInstance` | 런타임 장비 상태, Handle 보관 |
| Equipment | `ULyraWeaponInstance` | 총기 특화 (HeatTracker, Spread) |
| GAS | `ULyraAbilitySet` | Grant/Revoke 묶음 DataAsset |
| GAS | `ULyraAbilitySystemComponent` | ASC + InputTag 처리 |
| GAS | `ULyraGameplayAbility_RangedWeapon` | 발사 로직 |
| Input | `ULyraHeroComponent` | 입력 초기화, 바인딩 |
| Input | `ULyraInputConfig` | Tag → InputAction 매핑 DataAsset |

---

## 커스텀 무기 추가 체크리스트

```
[ ] ULyraInventoryItemDefinition 에셋 생성
      └─ Fragments에 UInventoryFragment_EquippableItem 추가
           └─ EquipmentDefinition 지정
[ ] ULyraEquipmentDefinition 에셋 생성
      ├─ InstanceType: ULyraWeaponInstance (또는 서브클래스)
      ├─ AbilitySetsToGrant: [DA_AbilitySet_Weapon_X]
      └─ ActorsToSpawn: [BP_WeaponActor]
[ ] ULyraAbilitySet 에셋 생성
      └─ GrantedGameplayAbilities:
           ├─ Ability: GA_Hero_Fire,   InputTag: InputTag.Weapon.Fire
           ├─ Ability: GA_Hero_Reload, InputTag: InputTag.Weapon.Reload
           └─ Ability: GA_Hero_ADS,    InputTag: InputTag.Weapon.ADS
[ ] ULyraInputConfig 에셋에 AbilityInputActions 항목 확인
[ ] InputMappingContext에 IA_Fire / IA_Reload / IA_ADS 키 매핑 확인
```

### 주의

- `GiveToAbilitySystem()`의 `OutGrantedHandles`는 반드시 보관해야 한다.
  Unequip 시 `TakeFromAbilitySystem()` 호출이 필수.
- `InputTag`는 AbilitySet의 `FLyraAbilitySet_GameplayAbility.InputTag`에서 지정.
  GA CDO에서 지정하는 게 아님.
