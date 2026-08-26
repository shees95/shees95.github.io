---
title: "언리얼 DeepRaiders - 복셀 프레임 드랍 네트워크 점검"
date: 2026-08-24 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, Voxel, Networking, RPC, Optimization, TIL]
description: "복셀 생성 중 발생한 프레임 드랍을 네트워크 문제로 가정하고 서버 권한 처리 구조를 점검한 과정"
---

# 복셀 프레임 드랍 원인 추적

DeepRaiders에서 눈을 발사해 복셀을 생성할 때 프레임이 크게 떨어지는 문제가 발생했다.

멀티플레이 환경에서 눈 생성 데이터를 서버와 클라이언트가 주고받고 있었기 때문에
처음에는 RPC 또는 복제 과정에서 발생하는 네트워크 문제라고 생각했다.

## 복셀 생성의 네트워크 흐름

눈 복셀 생성은 서버 권한을 기준으로 처리한다.

```text
플레이어 사격
    ↓
SnowProjectile이 월드와 충돌
    ↓ Server RPC
서버가 복셀 생성 요청 계산
    ├─ Voxel Tool
    ├─ Edit Mode
    ├─ 위치와 크기
    └─ 밀도와 팀 정보
    ↓
서버에서 복셀 생성
    ↓
생성 결과를 Operation 구조체로 기록
    ↓ Multicast 또는 복제 경로
클라이언트에 생성 데이터 전달
    ↓
각 클라이언트가 같은 데이터로 복셀 재생성
```

클라이언트가 충돌을 감지해 임의로 복셀을 생성하지 않도록 하고,
서버가 처리한 결과만 클라이언트가 적용하는 구조다.

실제 `SnowProjectile`도 Authority가 없는 인스턴스에서는 월드 충돌에 따른 눈 생성을 진행하지 않는다.

```cpp
void ADRSnowProjectile::HandleWorldImpact(const FHitResult& ImpactResult)
{
    if (!HasAuthority() || !IsValid(SnowAddComponent))
    {
        return;
    }

    SnowAddComponent->TryAddSnowFromHit(ImpactResult);
}
```

이를 통해 같은 충돌을 서버와 클라이언트가 각각 처리해 중복 생성하는 상황을 막았다.

## 클라이언트 생성 경로 제거

테스트 과정에서는 플레이어의 사격이 복셀에 닿더라도
클라이언트가 직접 생성하지 않고 서버에서만 생성 로직이 실행되도록 정리했다.

그 결과 클라이언트의 렉은 일부 줄었다.
처음에는 네트워크 처리량이나 중복 실행이 원인이라는 가설이 맞는 것처럼 보였다.

하지만 이것만으로 큰 프레임 드랍이 사라지지는 않았다.

## Standalone 비교

네트워크가 개입하지 않는 Standalone 환경에서도 같은 사격 프레임 드랍이 발생했다.

이 결과로 다음 사실을 확인할 수 있었다.

- 서버 권한 구조는 중복 실행을 막는 데 필요하다.
- 네트워크 경로를 정리하면 클라이언트 부하는 일부 줄어든다.
- 핵심 프레임 드랍은 네트워크가 없어도 재현된다.
- 실제 병목은 복셀 생성 연산 또는 렌더 갱신 경로에 있을 가능성이 크다.

![최적화 전 복셀 생성 테스트](/assets/img/deepraiders-voxel-performance/before-optimization-web.gif)

## 정리

처음 세운 가설은 네트워크 문제였지만 Standalone에서도 같은 현상이 발생했다.

서버 권한 처리와 클라이언트 중복 생성을 점검한 것은 의미가 있었지만,
프레임 드랍의 핵심 원인은 네트워크가 아니라 복셀 연산 자체에 더 가까웠다.

다음 단계에서는 복셀 생성 작업이 한 프레임에 몰리지 않도록 비동기 처리하고,
머티리얼 적용과 렌더 갱신 순서를 점검하기로 했다.
