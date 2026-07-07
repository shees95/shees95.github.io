---
title: "TPS 개발 : 샷건 스프레드 — 카메라 기준 각도 퍼짐"
date: 2026-07-08 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Weapon, UnrealEngine-Shotgun, UnrealEngine-Camera, TroubleShooting]
description: "랜덤 지점→랜덤 지점으로 쏘던 샷건탄을 카메라 기준 각도 퍼짐(tan)으로 바꿔 샷건다운 느낌 만들기"
---

# TPS 개발 - 샷건 스프레드

샷건도 테스트해봤다. 기존 구현을 보니 **카메라에서 랜덤 지점으로 시작해 랜덤 지점으로 날아가는** 방식이었다.

---

## 문제 — 거리와 무관하게 그냥 퍼진다

랜덤 시작점 → 랜덤 끝점 방식이면, 탄이 퍼지는 정도가 **거리와 상관없이 일정한 범위**로 흩어진다.

```
기존 방식
  카메라 근처 랜덤 지점 A
    └─ 화면 상 랜덤 지점 B로 발사
```

이러면 적이 가까워도 멀어도 퍼지는 반경이 비슷해져서, **샷건다운 느낌이 안 났다.** 원래 샷건은 가까이서 쏘면 탄이 뭉쳐 맞고, 멀리서 쏘면 넓게 퍼져서 빗나가는 게 정상이다. 거리에 따라 탄착군이 달라져야 "산탄"이라는 무기 특성이 산다.

---

## 해결 — 카메라 기준 각도 퍼짐

그래서 **카메라를 발사 원점**으로 삼고, 퍼짐을 거리가 아니라 **각도(spread)** 로 정의했다.

- Spread 값을 세타(θ) 각도로 구한다.
- `tan(θ)`를 이용해 카메라 기준 방향 벡터를 퍼뜨린다.

```cpp
// 의사코드
float Theta = FMath::DegreesToRadians(SpreadAngleDeg);

// 카메라 forward 기준으로 랜덤 각도만큼 벌어진 방향 계산
float RandomYaw   = FMath::FRandRange(-Theta, Theta);
float RandomPitch = FMath::FRandRange(-Theta, Theta);

FVector SpreadDir = CameraForward.RotateAngleAxis(RandomYaw,   CameraUp);
SpreadDir         = SpreadDir.RotateAngleAxis(RandomPitch, CameraRight);

// tan(θ)로 인한 퍼짐 반경은 거리에 비례해서 자연히 커짐
```

각도 기반이라, **가까우면 좁게 뭉치고 멀면 넓게 퍼지는** 게 자동으로 성립한다. `tan(θ) × 거리`가 실제 퍼짐 반경이 되니, 거리에 비례해서 커지는 게 물리적으로도 자연스럽다.

---

## VFX는 머즐 기준, 히트는 카메라 기준

여기서 한 가지 신경 쓴 부분이 있다. **히트 판정은 카메라 기준**으로 계산하지만, **VFX 시작점은 머즐(총구)** 로 뒀다.

```
히트 판정 (탄 방향/착탄점) : 카메라 기준
VFX 시작 위치            : 머즐(총구)
```

카메라와 머즐 위치가 다르다 보니 둘이 어긋나 보일까 걱정했는데, [VFX 탄 자체 속도를 빠르게]({% post_url NetKarma/2026-07-07-unreal-tps-wave-loguelike-weapon-vfx %}) 만들어둔 덕에 어색함이 크게 없었다. 눈에 보이는 시간이 워낙 짧아서 시작점 차이가 티가 안 났다.

히트 판정 자체는 **사용자가 바라보는 카메라 기준**으로 되니, 조준하고 쏘는 조작감에도 불편함이 없었다. 화면 중앙을 보고 쏘면 그 방향을 기준으로 퍼지니 직관적이다.

---

## 오늘 정리

- 기존 랜덤→랜덤 방식은 **거리와 무관하게 퍼짐** → 샷건답지 않음
- **카메라 기준 spread 각도(θ) + tan(θ)** 로 거리 비례 퍼짐 구현
- 히트 판정은 **카메라 기준**, VFX 시작점은 **머즐 기준** — 탄속이 빨라 어긋남 체감 안 됨
- 조준은 화면 중앙(카메라) 기준이라 조작감도 자연스러움
