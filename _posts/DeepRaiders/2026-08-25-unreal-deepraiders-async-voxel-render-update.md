---
title: "언리얼 DeepRaiders - 복셀 생성 비동기화와 렌더 갱신"
date: 2026-08-25 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, Voxel, Async, Rendering, Optimization, TIL]
description: "복셀 생성 연산을 비동기로 분리하고 머티리얼 처리 이후 렌더 갱신을 실행하도록 개선한 과정"
---

# 복셀 생성 작업 비동기화

Standalone에서도 프레임 드랍이 발생했으므로 네트워크가 아닌 로컬 복셀 처리 경로를 확인했다.

복셀 값 변경, 팀 머티리얼 적용, 월드 렌더 갱신이 한 흐름에서 실행되면서
Game Thread가 작업 완료를 오래 기다리는 것이 문제였다.

## Unreal Insights 확인

최적화 전 캡처에서는 한 프레임이 약 `35.3ms`까지 증가했다.
Game Thread의 `World_Tick` 구간에는 약 `28.8ms`의 `WaitForTasks`가 나타났다.

![복셀 최적화 전 Unreal Insights](/assets/img/deepraiders-voxel-performance/insights-before.png)

단순히 네트워크 패킷을 줄이는 것으로 해결할 수 있는 형태가 아니었다.
복셀 생성 작업을 요청한 뒤 연산 완료를 기다리는 시간이 한 프레임에 집중되고 있었다.

## 비동기 편집 큐

방향성 복셀 추가 요청은 큐에 넣고 한 번에 하나씩 비동기 처리하도록 변경했다.

```text
AddSnow 요청
    ↓
DirectionalAddQueue에 추가
    ↓
현재 작업이 없을 때 비동기 복셀 편집 시작
    ↓
복셀 값 변경 완료
    ↓
머티리얼 적용 완료
    ↓
렌더 갱신 예약
    ↓
다음 요청 처리
```

`AddSnow()`은 방향성 편집 요청을 즉시 계산하지 않고 큐에 등록한다.

```cpp
DirectionalAddQueue.Enqueue(Request);
ProcessNextDirectionalAdd();
```

`ProcessNextDirectionalAdd()`는 이전 작업이 끝난 후 다음 요청을 시작한다.
동시에 여러 쓰기 작업이 같은 Voxel World에 몰리는 상황을 피하면서 Game Thread의 긴 점유를 줄였다.

## 머티리얼 처리 이후 렌더 갱신

처리 순서도 다음과 같이 정리했다.

1. 복셀 값을 비동기로 수정한다.
2. 실제로 변경된 복셀 정보를 수집한다.
3. 새로 채워진 표면에 팀 머티리얼을 적용한다.
4. 머티리얼 작업이 완료된 후 렌더 갱신을 큐에 넣는다.
5. 완료 콜백에서 다음 복셀 요청을 실행한다.

복셀 값과 머티리얼이 완성되기 전에 렌더러를 갱신하지 않으므로
하나의 생성 요청에서 불필요한 중간 렌더 업데이트가 발생하는 것을 줄일 수 있다.

## 렌더 업데이트 병합

같은 Voxel World에서 서로 겹치는 편집 영역은 하나의 Bounds로 합친 뒤 갱신한다.
렌더 요청은 즉시 실행하지 않고 짧은 시간 동안 모아 `FlushRenderUpdates()`에서 처리한다.

```text
편집 A Bounds ─┐
편집 B Bounds ─┼─ 겹치는 영역 병합 → 한 번의 렌더 갱신
편집 C Bounds ─┘
```

이를 통해 연속 사격 중 같은 영역의 메시와 렌더 상태를 반복해서 갱신하는 비용을 줄였다.

## 결과

비동기 편집과 렌더 큐 조정만으로도 사격 시 발생하던 프레임 드랍이 크게 감소했다.

다만 복셀을 실제로 계산하는 작업 자체는 여전히 무거웠다.
작업을 분산해 순간적인 멈춤은 줄였지만 CPU에서 수행되는 연산 비용까지 제거된 것은 아니었다.

다음 단계에서는 복셀 계산 경로를 CPU에서 GPU로 전환해 순수 연산 시간을 줄이기로 했다.

## 정리

이번 개선의 핵심은 작업량을 줄이는 것뿐 아니라 실행 순서를 분리하는 것이었다.

복셀 값 변경과 머티리얼 처리를 비동기로 완료한 뒤 렌더 갱신을 예약하고,
겹치는 렌더 영역을 병합해 한 프레임에 몰리던 대기와 갱신 비용을 줄였다.
