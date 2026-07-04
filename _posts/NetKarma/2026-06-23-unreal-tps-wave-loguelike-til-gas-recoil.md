---
title: "TPS 개발 : GAS 총기 반동 구현 — PlayerController 책임 분리"
date: 2026-06-23 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, TroubleShooting, TroubleShooting-UnrealEngine, UnrealEngine-GAS, UnrealEngine-PlayerController, UnrealEngine-Recoil]
description: "GAS에서 반동 직접 제어가 안 되는 이유와 PlayerController PlayerTick으로 분리한 구조"
---

# TIL - GAS 총기 반동 구현

---

## 문제

GAS 어빌리티 안에서 총기 반동을 구현하려고 했으나, 카메라 회전을 직접 제어할 방법이 없었다.

`AddControllerPitchInput` 같은 함수는 `PlayerController` 또는 `Pawn`에 종속되어 있어서 GAS 내부에서 자연스럽게 호출하기 어렵고, GAS 레이어에서 카메라를 직접 건드리는 건 구조적으로도 맞지 않는다.

---

### 구조 — 책임 분리

GAS가 반동의 **목표 위치(Target)만 결정**하고, 실제 화면을 올리는 건 **PlayerController**가 담당하도록 분리했다.



```
GA (발사 어빌리티)
  └─ 반동 목표 Pitch 계산 → PlayerController에 전달

PlayerController::PlayerTick(float DeltaTime)
  └─ 현재 Pitch → 목표 Pitch 방향으로 틱마다 보간하여 적용
       └─ AddControllerPitchInput(...)
```

---

## 구현

### GA (발사 어빌리티)

발사 시 반동 목표값만 PlayerController에 넘긴다.

![스크린샷 2026-06-26 105854.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20105854.png)  
<GAFire::AddRecoil() : 최대 이만큼만 마시렴>

```cpp
void UGA_Fire::ApplyRecoil(float RecoilAmount)
{
    APlayerController* PC = Cast<APlayerController>(GetActorInfo().PlayerController.Get());
    if (AMyPlayerController* MyPC = Cast<AMyPlayerController>(PC))
    {
        MyPC->RecoilTargetPitch += RecoilAmount;
    }
}
```

---

### PlayerController

반동 목표값을 보관하고, `PlayerTick`에서 매 틱 처리한다.

![스크린샷 2026-06-26 110050.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20110050.png)  
<Controller::Tick() : 꼴깍꼴깍>  

```cpp
// MyPlayerController.h

UPROPERTY()
float RecoilTargetPitch = 0.f;

float RecoilSpeed = 10.f;
```

```cpp
// MyPlayerController.cpp

void AMyPlayerController::PlayerTick(float DeltaTime)
{
    Super::PlayerTick(DeltaTime);

    if (!FMath::IsNearlyZero(RecoilTargetPitch))
    {
        float Step = RecoilSpeed * DeltaTime;
        float Applied = FMath::Sign(RecoilTargetPitch) * FMath::Min(FMath::Abs(RecoilTargetPitch), Step);

        AddPitchInput(Applied);
        RecoilTargetPitch -= Applied;
    }
}
```

---

## 핵심 요약

| 역할 | 담당 |
|---|---|
| 반동 크기/목표 결정 | GA (GAS 어빌리티) |
| 실제 카메라 이동 | PlayerController `PlayerTick` |
