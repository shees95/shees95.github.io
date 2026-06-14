# Lyra Sample Game — 총기·GAS 시스템 레퍼런스

> UE 5.x Lyra Starter Game 기준. 소스 경로는 `LyraStarterGame/Source/LyraGame/` 기준.

---

## 목차

1. [총기 획득 전체 흐름](#1-총기-획득-전체-흐름)
2. [GAS 어빌리티 부여 흐름](#2-gas-어빌리티-부여-흐름)
3. [데이터 에셋 계층 구조](#3-데이터-에셋-계층-구조)
4. [입력 처리 흐름](#4-입력-처리-흐름)
5. [클래스 퀵 레퍼런스](#5-클래스-퀵-레퍼런스)
6. [구현 체크리스트](#6-구현-체크리스트)

---

## 1. 총기 획득 전체 흐름

![Weapon Acquisition Flow](images/01_weapon_acquisition_flow.svg)

### 개요

Lyra의 총기 획득은 **Inventory → Equipment → GAS** 세 레이어가 순서대로 연결되는 구조다.
각 레이어는 독립적으로 동작하며 서로 참조만 한다.

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

### 단계별 설명

| 단계 | 클래스 | 핵심 메서드 | 비고 |
|------|--------|-------------|------|
| ① Pickup | `ULyraInventoryManagerComponent` | `AddItemDefinition(TSubclassOf<ULyraInventoryItemDefinition>)` | 서버 전용 |
| ② Instance 생성 | `ULyraInventoryItemInstance` | 생성자, `AddStatTagStack()` | COND_OwnerOnly 복제 |
| ③ Equip | `ULyraEquipmentManagerComponent` | `EquipItem(const ULyraEquipmentDefinition*)` | `FLyraEquipmentList`에 추가 |
| ④ Actor 스폰 | `ULyraEquipmentInstance` | `OnEquipped()`, `SpawnEquipmentActors()` | `ActorsToSpawn[]` 기준 |
| ⑤ AbilitySet Grant | `ULyraAbilitySet` | `GiveToAbilitySystem(ASC, OutGrantedHandles)` | Handle 보관 필수 |
| ⑥ 사용 가능 | `ULyraGameplayAbility_RangedWeapon` | `ActivateAbility()` | InputTag로 트리거 |

### Unequip 경로

```
EquipmentManager::UnequipItem(Instance)
  └─ Instance::OnUnequipped()
       └─ GrantedHandles.TakeFromAbilitySystem(ASC)
            └─ FGameplayAbilitySpecHandle들 ClearAbility()
                 └─ 스폰된 Actor들 Destroy()
```

---

## 2. GAS 어빌리티 부여 흐름

![GAS Ability Grant Sequence](images/02_gas_ability_grant_sequence.svg)

### 두 가지 Grant 경로

#### 경로 A — PawnData (베이스 어빌리티)

게임 Experience 로드 시 Pawn 고유 베이스 어빌리티 부여.
이동, 점프, 기본 인터랙션 등.

```
ULyraExperienceDefinition
  └─ DefaultPawnData: ULyraPawnData
       └─ AbilitySets: TArray<ULyraAbilitySet*>
            └─ GiveToAbilitySystem(ASC, &PawnHandles)
```

#### 경로 B — Equipment (무기 어빌리티)

Equip 시 추가, Unequip 시 회수. 무기별 독립 핸들 관리.

```cpp
// Equipment/LyraEquipmentDefinition.h
UPROPERTY(EditDefaultsOnly, Category=Equipment)
TArray<TObjectPtr<const ULyraAbilitySet>> AbilitySetsToGrant;

// Equipment/LyraEquipmentInstance.cpp
void ULyraEquipmentInstance::OnEquipped()
{
    // AbilitySetsToGrant 순회하며 각각 Grant
    for (const ULyraAbilitySet* AbilitySet : GetEquipmentDefinition()->AbilitySetsToGrant)
    {
        AbilitySet->GiveToAbilitySystem(LyraASC, &GrantedHandles);
    }
}
```

### AbilitySet 내부 구조

```cpp
// AbilitySystem/LyraAbilitySet.h

// 에디터에서 설정하는 Grant 목록
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
    FGameplayTag InputTag;   // ← InputTag 여기서 지정
};
```

### Grant 후 InputTag 주입 메커니즘

```cpp
// AbilitySystem/LyraAbilitySet.cpp (GiveToAbilitySystem 내부)

FGameplayAbilitySpec AbilitySpec(AbilityToGrant.Ability, AbilityToGrant.AbilityLevel);
AbilitySpec.SourceObject = SourceObject;
AbilitySpec.DynamicAbilityTags.AddTag(AbilityToGrant.InputTag); // ← InputTag 주입

FGameplayAbilitySpecHandle Handle = ASC->GiveAbility(AbilitySpec);
GrantedHandles->AbilitySpecHandles.Add(Handle);
```

> `DynamicAbilityTags`에 주입된 InputTag가 나중에 `AbilityInputTagPressed()` 탐색의 키가 된다.

---

## 3. 데이터 에셋 계층 구조

![Data Asset Hierarchy](images/03_data_asset_hierarchy.svg)

### Definition (불변) vs Instance (런타임)

| 구분 | 클래스 | 생존 범위 | 복제 |
|------|--------|-----------|------|
| **Inventory Def** | `ULyraInventoryItemDefinition` | DataAsset (에디터) | ✗ |
| **Inventory Instance** | `ULyraInventoryItemInstance` | 런타임 UObject | OwnerOnly |
| **Equipment Def** | `ULyraEquipmentDefinition` | DataAsset (에디터) | ✗ |
| **Equipment Instance** | `ULyraEquipmentInstance` | 런타임 UObject | All |
| **WeaponInstance** | `ULyraWeaponInstance` | Equipment Instance 서브클래스 | All |
| **AbilitySet** | `ULyraAbilitySet` | DataAsset (에디터) | ✗ |
| **GrantedHandles** | `FLyraAbilitySet_GrantedHandles` | EquipmentInstance 내 struct | (Instance와 함께) |

### Fragment 시스템

`ULyraInventoryItemDefinition`에는 `Fragments` 배열이 있고,
각 Fragment가 모듈식 데이터 조각을 담는다.

```
ULyraInventoryItemFragment (base)
  ├─ UInventoryFragment_EquippableItem    → EquipmentDefinition 참조
  ├─ UInventoryFragment_SetStats          → 초기 StatTag 설정 (탄약 수 등)
  ├─ UInventoryFragment_QuickBarIcon      → HUD 아이콘 정보
  └─ UInventoryFragment_ReticleConfig     → 조준선 설정
```

> Fragment는 Instance가 아닌 Definition에 붙는다.
> 런타임에 `ItemInstance->FindFragmentByClass<T>()` 로 읽는다.

### 탄약 데이터 흐름

```
UInventoryFragment_SetStats (Def)
  └─ InitialItemStats: { Lyra.ShooterGame.Weapon.MagazineAmmo: 30 }
       └─ AddItemDefinition() 시 Instance::StatTags에 복사
            └─ GA_Hero_Fire::CommitAbilityCost()
                 → StatTags.AddStack(Lyra.ShooterGame.Weapon.MagazineAmmo, -1)
```

---

## 4. 입력 처리 흐름

![Input Handling Flow](images/04_input_handling_flow.svg)

### 바인딩 초기화 경로

```
APawn::PossessedBy(AController*)  [서버]
APawn::OnRep_Controller()         [클라이언트]
  └─ ULyraHeroComponent::InitializePlayerInput()
       ├─ EnhancedInputSubsystem->AddMappingContext(IMC, Priority)
       ├─ LyraInputComponent->BindNativeAction(...)   // Move, Look, Jump
       └─ LyraInputComponent->BindAbilityActions(...) // Fire, ADS, Reload
```

### Native Action 바인딩 (이동/시점)

```cpp
// Input/LyraHeroComponent.cpp

InputComponent->BindNativeAction(
    InputConfig,
    LyraGameplayTags::InputTag_Move,
    ETriggerEvent::Triggered,
    this,
    &ThisClass::Input_Move,
    /*bLogIfNotFound=*/false
);
```

```cpp
void ULyraHeroComponent::Input_Move(const FInputActionValue& InputActionValue)
{
    // Pawn → Controller forward/right 기준 이동
    const FVector2D Value = InputActionValue.Get<FVector2D>();
    Pawn->AddMovementInput(ForwardDir, Value.Y);
    Pawn->AddMovementInput(RightDir, Value.X);
}
```

### Ability Action 바인딩 (발사/재장전)

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
        BindAction(Action.InputAction, ETriggerEvent::Triggered, Object,
                   PressedFunc, Action.InputTag);
        BindAction(Action.InputAction, ETriggerEvent::Completed, Object,
                   ReleasedFunc, Action.InputTag);
    }
}
```

### InputTag → Ability 연결 (런타임)

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
}

// ProcessAbilityInput() 에서 실제 TryActivateAbility() 호출
```

### 입력 태그 구조 (GameplayTag 계층)

```
InputTag (root)
  ├─ InputTag.Move
  ├─ InputTag.Look.Mouse
  ├─ InputTag.Look.Stick
  ├─ InputTag.Jump
  ├─ InputTag.Crouch
  └─ InputTag.Weapon
       ├─ InputTag.Weapon.Fire
       ├─ InputTag.Weapon.FireAuto
       ├─ InputTag.Weapon.Reload
       └─ InputTag.Weapon.ADS
```

---

## 5. 클래스 퀵 레퍼런스

### Inventory

| 클래스 | 파일 | 역할 |
|--------|------|------|
| `ULyraInventoryManagerComponent` | `Inventory/LyraInventoryManagerComponent.h` | 인벤토리 CRUD, 서버 권한 |
| `ULyraInventoryItemDefinition` | `Inventory/LyraInventoryItemDefinition.h` | 아이템 설계 데이터 (DA) |
| `ULyraInventoryItemInstance` | `Inventory/LyraInventoryItemInstance.h` | 런타임 인스턴스, StatTag 보관 |
| `ULyraInventoryItemFragment` | `Inventory/LyraInventoryItemDefinition.h` | Fragment 베이스 |
| `UInventoryFragment_EquippableItem` | `Equipment/LyraPickupDefinition.h` | EquipmentDef 참조 |

### Equipment

| 클래스 | 파일 | 역할 |
|--------|------|------|
| `ULyraEquipmentManagerComponent` | `Equipment/LyraEquipmentManagerComponent.h` | Equip/Unequip 관리 |
| `ULyraEquipmentDefinition` | `Equipment/LyraEquipmentDefinition.h` | 장비 설계 (AbilitySet, 스폰Actor) |
| `ULyraEquipmentInstance` | `Equipment/LyraEquipmentInstance.h` | 런타임 장비 상태 |
| `ULyraWeaponInstance` | `Weapons/LyraWeaponInstance.h` | 총기 특화 (HeatTracker, Spread) |
| `ULyraRangedWeaponInstance` | `Weapons/LyraRangedWeaponInstance.h` | 사거리/정확도 계산 |

### GAS / Ability

| 클래스 | 파일 | 역할 |
|--------|------|------|
| `ULyraAbilitySystemComponent` | `AbilitySystem/LyraAbilitySystemComponent.h` | ASC + InputTag 처리 |
| `ULyraAbilitySet` | `AbilitySystem/LyraAbilitySet.h` | Grant/Revoke 묶음 DA |
| `ULyraGameplayAbility` | `AbilitySystem/LyraGameplayAbility.h` | 베이스 GA |
| `ULyraGameplayAbility_RangedWeapon` | `Weapons/LyraGameplayAbility_RangedWeapon.h` | 발사 로직 |
| `FLyraAbilitySet_GrantedHandles` | `AbilitySystem/LyraAbilitySet.h` | Grant 핸들 묶음 struct |

### Input

| 클래스 | 파일 | 역할 |
|--------|------|------|
| `ULyraInputConfig` | `Input/LyraInputConfig.h` | Tag→InputAction 매핑 DA |
| `ULyraInputComponent` | `Input/LyraInputComponent.h` | BindAbilityActions() 확장 |
| `ULyraHeroComponent` | `Character/LyraHeroComponent.h` | 입력 초기화, Native 바인딩 |
| `ULyraMappableConfigPair` | `Input/LyraMappableConfigPair.h` | IMC 래퍼 |

---

## 6. 구현 체크리스트

### 커스텀 무기 추가 시

```
[ ] ULyraInventoryItemDefinition 서브클래스 또는 에셋 생성
      └─ Fragments에 UInventoryFragment_EquippableItem 추가
           └─ EquipmentDefinition 지정
[ ] ULyraEquipmentDefinition 에셋 생성
      ├─ InstanceType: ULyraWeaponInstance (or 커스텀 서브클래스)
      ├─ AbilitySetsToGrant: [DA_AbilitySet_Weapon_X]
      └─ ActorsToSpawn: [BP_WeaponActor]
[ ] ULyraAbilitySet 에셋 생성
      └─ GrantedGameplayAbilities:
           ├─ Ability: GA_Hero_Fire, InputTag: InputTag.Weapon.Fire
           ├─ Ability: GA_Hero_Reload, InputTag: InputTag.Weapon.Reload
           └─ Ability: GA_Hero_ADS, InputTag: InputTag.Weapon.ADS
[ ] ULyraInputConfig 에셋에 AbilityInputActions 항목 확인
[ ] InputMappingContext에 IA_Fire / IA_Reload / IA_ADS 키 매핑 확인
[ ] ALyraWeaponPickup 배치 또는 기존 Pickup 클래스 활용
```

### 커스텀 어빌리티 추가 시 (GAS-only, 무기 무관)

```
[ ] ULyraGameplayAbility 서브클래스 생성
[ ] ULyraAbilitySet 에셋에 추가
      └─ InputTag 지정 (없으면 코드에서 TryActivateAbilitiesByTag)
[ ] Grant 경로 결정:
      A) PawnData.AbilitySets → 항상 Grant (베이스)
      B) Equipment.AbilitySetsToGrant → Equip 시 Grant
      C) 코드에서 직접 AbilitySet->GiveToAbilitySystem()
```

### 주의 사항

- `GiveToAbilitySystem()`의 `OutGrantedHandles`는 반드시 보관.
  Unequip 시 `TakeFromAbilitySystem()` 호출 필수.
- `InputTag`는 AbilitySet의 `FLyraAbilitySet_GameplayAbility.InputTag`에서 지정.
  Ability CDO에서 지정하는 것이 아님.
- `ULyraEquipmentDefinition`은 `UObject`이지만 에디터에서
  Blueprint로 만들어 DataAsset처럼 사용함 (실제 DA 파생은 아님).
- `ULyraWeaponInstance::GetBulletSpreadAngle()`은
  `HeatTracker`와 `ULyraRangedWeaponInstance`의 StandingSpreadAngle을 조합.
  사격 후 `AddSpread()`, 시간 경과 시 `UpdateSpread()` 호출.
