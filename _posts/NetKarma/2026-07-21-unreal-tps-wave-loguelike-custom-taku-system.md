---
title: "TPS 개발 : 커스텀 타쿠 — 종류별 적, 풀링, 코스메틱 버프"
date: 2026-07-21 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-AI, UnrealEngine-GAS, UnrealEngine-Pooling, UnrealEngine-DataAsset]
description: "ANKMBaseTakuCharacter 공통 뼈대 위에 종류별 타쿠를 얹고, 커스텀 타쿠 발탁 시 꾸미기+스펙버프를 함께 부여하는 구조"
---

# TPS 개발 - 커스텀 타쿠 시스템

웨이브에 등장하는 적("타쿠")을 여러 종류로 늘리면서, 공통 뼈대와 종류별 확장을 분리하는 작업을 했다. 여기에 "커스텀 타쿠"라는 특수 개체(꾸미기 + 스펙 버프가 붙은 강화 개체)도 얹었다.

---

## 종류별 타쿠 — 하나의 Base 위에 여러 확장

타쿠 종류는 `ENKMTakuPoolTypes`로 정의된다.

```cpp
enum class ENKMTakuPoolTypes : uint8
{
    NormalTaku, ADHDTaku, NarutoTaku, SprinterTaku,
    PaparazziTaku, PaoDancerTaku, LowAnglerTaku, HuggerTaku,
    PaoCloud, None,
};
```

전부 `ANKMBaseTakuCharacter`를 상속한다. Base가 갖고 있는 공통 기능:

- **HP/AS**: `UAS_NKMCharacterState`(StateAS)로 체력 관리, `HandleOutOfHealth`로 사망 처리
- **콜리전**: `BodyHitCollision`(캡슐) / `HeadHitCollision`(박스) — 헤드샷 판정을 몸통과 분리
- **풀링**: `ActivateFromPool` / `DeactivateToPool`
- **복귀 명령**: `ReturnBase` 메시지 구독 — 웨이브 종료 시 스폰 지점으로 되돌아가는 공통 로직
- **GAS 데이터 캐시**: `InitTakuGASData`로 종류별 GA/GE 매핑을 받아둠

종류별 서브클래스(`NKMADHDTakuCharacter`, `NKMHuggerTakuCharacter`, `NKMPaparazziTakuCharacter` 등)는 이 위에서 자기만의 행동(공격 패턴, 이동 방식)만 추가한다.

---

## GAS 데이터도 종류별로 테이블화

타쿠의 어빌리티/이펙트도 하드코딩하지 않고 `UNKMTakuGASSet`이라는 DataAsset으로 관리한다.

```cpp
enum class ENKMTakuGAId : uint8 { None, InstantAttack, LowAngler, ADHD, Hugger, PaoCloud, Naruto };
enum class ENKMTakuGEId : uint8 { None, InstantDamage, DamageIncrease, MoveSpeed, DamageReductionDebuff, ContinuousDamage };
enum class ENKMTakuGETarget : uint8 { None, Self, Player, NearbyTaku };

UCLASS()
class UNKMTakuGASSet : public UDataAsset
{
    TMap<ENKMTakuGAId, TSubclassOf<UGameplayAbility>> GAClasses;
    TMap<ENKMTakuGEId, FNKMTakuGEClassData> GEClasses;
};
```

`GEClasses`의 각 항목은 GE 클래스뿐 아니라 **`MagnitudeTag`(SetByCaller 데이터 태그)** 도 함께 들고 있어서, GA에서 `SetSetByCallerMagnitude` 호출 시 이 태그로 값을 실어보내면 된다. `ENKMTakuGETarget`(Self/Player/NearbyTaku)이 있는 게 특징인데, 예를 들어 "주변 타쿠에게 이동속도 버프를 준다" 같은 **범위 효과**를 종류 구분 없이 표현할 수 있다.

공격도 즉발형(`UNKMGA_TakuInstantAttack`)과 지속형(`UNKMGA_TakuContinuousAttack`)으로 나뉘어 있고, 데미지 계산은 `UNKMExecCalc_TakuDamage`가 전담한다.

---

## 죽음 처리 — 하늘로 발사하는 연출

`UNKMGA_TakuDeath`가 사망 연출을 담당한다.

```cpp
UPROPERTY(EditDefaultsOnly) float SkyLaunchDuration = 10.f;
UPROPERTY(EditDefaultsOnly) float SkyLaunchSpeed = 3500.f;
UPROPERTY(EditDefaultsOnly) float SkyLaunchRotationSpeed = 720000.f;
```

애니메이션을 멈추고(`StopAnimationBeforeRagdoll`) 래그돌을 시작한 뒤(`StartDeathRagdoll`), 위 방향으로 쏘아 올리는(`StartSkyLaunch`) 연출이다. 죽었을 때 `BroadCastDeadMessage`로 킬 정보를 방송하는데, 이때 가해자 정보는 `HandleOutOfHealth`에서 미리 캐싱해둔 `LastDamageInstigator`를 사용한다 — 사망 GA 자체는 **태그 기반으로 활성화**되기 때문에 이벤트 데이터로 가해자를 못 받아서, Base 클래스가 캐싱을 대신 책임진다.

---

## 커스텀 타쿠 — 발탁 시 꾸미기 + 스펙 버프 세트

일반 타쿠 풀에서 가끔 "커스텀 타쿠"가 발탁된다. `FNKMCustomTakuData`(DataTable 행) 하나가 꾸미기와 버프를 함께 정의한다.

```cpp
enum class ENKMTakuCosmeticType : uint8 { None, Hair, Glasses, Accessory, Slim };

struct FNKMCustomTakuData : FTableRowBase
{
    ENKMTakuCosmeticType CosmeticType;
    ENKMTakuCosmeticType RequiredCosmeticType; // 이 꾸미기가 뽑히려면 먼저 뽑혀 있어야 하는 부모 분류
    TArray<ENKMTakuPoolTypes> AllowedTakuTypes; // 비어있으면 전체 적용 가능
    FName ComponentTag;
    FName ParentSocketName;
    TSoftObjectPtr<UStaticMesh> StaticMesh;
    TArray<FNKMCustomTakuBuffEntry> SpecBuffs;  // 발탁 시 적용되는 GE 세트
    float StylishScale = 0.f;                    // 처치 시 코인 보너스에 더해질 배율
};
```

몇 가지 눈여겨볼 설계:

- **`Slim` 타입은 메시 부착이 아니라 모프타겟**이다. `MorphValueMin/Max` 범위 안에서 정규분포로 뽑아 적용한다 — 살찐/마른 변형처럼 메시 교체 없이 표현 가능한 꾸미기는 별도 분류로 뺐다.
- **`RequiredCosmeticType`으로 꾸미기 간 의존관계**를 표현한다(예: 안경을 뽑으려면 특정 헤어가 먼저 뽑혀 있어야 하는 식).
- **`SpecBuffs`는 GE + SetByCaller 태그 + 값**의 3종 세트로, 연산 방식(가산/곱연산/오버라이드)은 GE의 Modifier Op에 맡기고 이 데이터는 값만 책임진다.
- **`StylishScale`은 [리워드 시스템]({% post_url NetKarma/2026-07-16-unreal-tps-wave-loguelike-gas-summary-structure %})과 바로 연결**된다. `ANKMBaseTakuCharacter::GetStylishScale()`이 이 값을 들고 있다가, 처치 시 코인 보상 계산의 `CustomTaku` 카테고리로 들어간다.

일반 타쿠는 이 값이 항상 0이라, 커스텀 타쿠만 처치 보상이 더 높아지는 구조가 자연스럽게 성립한다.

---

## 정리

- 모든 타쿠는 `ANKMBaseTakuCharacter` 공통 뼈대(HP/콜리전/풀링/복귀) 위에서 종류별로 확장
- GA/GE도 `ENKMTakuGAId`/`ENKMTakuGEId` + `UNKMTakuGASSet`으로 테이블화, `ENKMTakuGETarget`으로 대상 범위까지 데이터로 표현
- 사망 GA는 태그 기반 활성화라 가해자 정보를 이벤트로 못 받음 — Base가 미리 캐싱해서 넘겨줌
- 커스텀 타쿠는 "꾸미기(DataTable 행) = 스펙 버프 + 스타일리쉬 배율"로 묶어서, 발탁되는 순간 강함과 보상이 함께 따라옴
