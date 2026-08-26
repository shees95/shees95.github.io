---
title: "언리얼 DeepRaiders - GameplayCue 기반 사운드 매니저"
date: 2026-08-19 19:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, GameplayCue, GameplayTag, Sound, DataAsset]
description: "GameplayCue 태그와 Sound Library를 이용한 공용 사운드 재생 구조 정리"
---

# GameplayCue 기반 사운드 매니저

DeepRaiders의 사운드는 `GameplayCue`와 `GameplayTag`를 이용해 공통된 흐름으로 재생하도록 구성했다.

각 기능에서 사운드 애셋을 직접 참조하지 않고 사운드 태그만 전달하면,
공용 GameplayCue가 태그에 맞는 사운드를 `Sound Library`에서 찾아 재생하는 방식이다.

![DeepRaiders 사운드 매니저 구성](/assets/img/deepraiders-sound-manager/sound-manager-overview.png)

## 전체 흐름

```text
게임플레이 코드
    ↓ 사운드 GameplayCue 태그 실행
GameplayCueManager
    ↓ 태그에 해당하는 Cue 호출
GC_Sound 공용 Blueprint
    ↓ GameplayCue 태그 Resolve
DRGameplayCueSound
    ↓ Sound Library 조회
Sound Library DataAsset
    ↓ 태그에 등록된 사운드 반환
사운드 재생
```

핵심 구성은 다음과 같다.

- `DRGameplayCueSound`: 사운드 GameplayCue의 공통 처리 부모 클래스
- `GC_Sound`: `DRGameplayCueSound`를 부모로 사용하는 공용 Blueprint
- `Sound Library`: 사운드 태그와 재생 설정을 보관하는 DataAsset
- GameplayCue 태그: 재생할 사운드를 식별하는 키

## Sound Library

사운드 애셋과 재생 옵션은 `DA_SoundLibrary`에 등록한다.

각 기능에서는 실제 사운드 애셋을 알 필요 없이 태그만 사용한다.
공용 GameplayCue가 전달받은 태그를 Resolve한 뒤 Sound Library에서 일치하는 설정을 찾는다.

사운드 설정에는 다음과 같은 값을 함께 둘 수 있다.

- 재생할 Sound
- 재생 위치를 결정하는 Playback Mode
- Attenuation
- Concurrency
- Sound Class
- Priority
- Volume 및 Pitch 배율
- Loop 여부
- Fade Out 시간

필요한 항목만 별도로 지정하고 나머지는 기본값을 사용하도록 구성하면,
사운드마다 반복해서 같은 설정을 입력하지 않아도 된다.

## 공용 GC_Sound

`GC_Sound`는 `DRGameplayCueSound`를 부모로 사용하는 공용 Blueprint다.

사운드 종류마다 GameplayCue Blueprint를 따로 만들지 않고,
하나의 `GC_Sound`에서 호출된 GameplayCue 태그를 Resolve해 재생할 사운드를 결정한다.

예를 들어 다음과 같은 태그를 Sound Library의 사운드 설정과 연결할 수 있다.

```text
GameplayCue.Sound.Ore.Dropped
GameplayCue.Sound.Item.PickedUp
GameplayCue.Sound.Weapon.Fired
```

새로운 사운드가 필요할 때는 기능 코드와 공용 Blueprint를 수정하기보다
태그를 선언하고 Sound Library에 해당 사운드를 등록하는 방식으로 확장할 수 있다.

## GameplayCue 실행

아이템 획득 사운드는 `GameplayCueParameters`에 재생에 필요한 문맥을 담은 뒤 실행한다.

```cpp
void ADRWorldItemActor::MulticastPlayPickupSound_Implementation(APawn* Interactor)
{
    if (!IsValid(Interactor))
    {
        return;
    }

    const UDRItemDefinition* Definition = ItemInstance.GetDefinition();

    FGameplayCueParameters CueParameters;
    CueParameters.Location = GetActorLocation();
    CueParameters.Instigator = Interactor;
    CueParameters.EffectCauser = this;
    CueParameters.SourceObject = Definition;

    if (UGameplayCueManager* CueManager = UAbilitySystemGlobals::Get().GetGameplayCueManager())
    {
        CueManager->HandleGameplayCue(
            Interactor,
            DRGameplayTags::GameplayCue_Sound_Item_PickedUp,
            EGameplayCueEvent::Executed,
            CueParameters);
    }
}
```

이 호출에서 기능 코드는 `USoundBase`나 Sound Cue 애셋을 직접 참조하지 않는다.
어떤 소리를 재생할지는 `GameplayCue_Sound_Item_PickedUp` 태그와 Sound Library의 설정이 결정한다.

`FGameplayCueParameters`에는 다음 정보를 전달한다.

| 값 | 용도 |
|---|---|
| `Location` | 월드 위치 기반 사운드 재생 위치 |
| `Instigator` | 이벤트를 발생시킨 Pawn |
| `EffectCauser` | 실제 이벤트가 발생한 액터 |
| `SourceObject` | 아이템 정의처럼 추가 판정에 사용할 원본 데이터 |

## 장점

- 게임플레이 코드가 사운드 애셋을 직접 참조하지 않는다.
- 공용 GameplayCue에서 사운드 재생 절차를 일관되게 처리한다.
- 태그와 DataAsset 설정만으로 사운드를 교체할 수 있다.
- Attenuation, Concurrency, Sound Class 같은 옵션을 한곳에서 관리할 수 있다.
- GameplayCue를 사용하므로 기존 GAS 및 GameplayCue 흐름과 자연스럽게 연결된다.
- 새로운 사운드를 추가할 때 중복 Blueprint 생성을 줄일 수 있다.

## 정리

DeepRaiders의 사운드 매니저는 **GameplayCue 태그와 Sound Library**를 중심으로 동작한다.

게임플레이 코드는 상황에 맞는 사운드 태그를 실행하고,
공용 `GC_Sound`가 태그를 Resolve해 Sound Library에 등록된 사운드와 재생 옵션을 선택한다.

이 구조를 사용하면 기능 코드와 사운드 애셋의 결합을 줄이면서
새로운 사운드 추가와 기존 사운드 설정 변경을 DataAsset 중심으로 관리할 수 있다.
