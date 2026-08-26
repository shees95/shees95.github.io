---
title: "언리얼 DeepRaiders - 복셀 연산 GPU 전환과 프레임 최적화"
date: 2026-08-26 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, Voxel, GPU, UnrealInsights, Profiling, Optimization, TIL]
description: "CPU 복셀 연산을 GPU 경로로 전환하고 Unreal Insights로 프레임 개선을 확인한 과정"
---

# 복셀 연산 GPU 전환

비동기 처리와 렌더 갱신 순서 변경으로 프레임 드랍은 줄었지만,
CPU에서 실행되는 복셀 계산 비용은 여전히 남아 있었다.

팀원이 복셀 연산을 CPU에서 GPU로 옮겨 측정한 결과,
약 `18ms`가 걸리던 작업이 `1.8ms` 수준까지 감소했다.
이를 프로젝트의 복셀 생성 경로에 적용하자 사격 중 프레임 드랍이 눈에 띄게 줄었다.

## 최적화 전후 비교

최적화 전에는 사격과 복셀 생성이 겹칠 때 FPS가 40대까지 떨어졌다.
Insights에서도 긴 Task 대기와 30ms 이상의 프레임을 확인할 수 있었다.

![최적화 전 Unreal Insights](/assets/img/deepraiders-voxel-performance/insights-before.png)

GPU 연산 경로로 전환한 뒤 캡처에서는 대체로 약 `6~13ms` 범위의 프레임을 확인할 수 있었다.
이전 캡처에서 크게 보이던 Game Thread의 긴 복셀 작업 대기도 눈에 띄게 줄었다.

![최적화 후 Unreal Insights](/assets/img/deepraiders-voxel-performance/insights-after.png)

## CPU에서 GPU로 옮긴 이유

복셀 생성은 일정 영역의 많은 샘플을 반복 계산하는 작업이다.
각 위치의 값을 독립적으로 계산할 수 있는 구간은 병렬 처리에 적합하다.

CPU에서 긴 작업 하나로 처리하면 해당 프레임의 Game Thread나 작업 스레드 대기가 길어진다.
같은 계산을 GPU 병렬 연산으로 전환하면 다수의 복셀 값을 동시에 처리해 전체 소요 시간을 줄일 수 있다.

이번 측정에서는 다음과 같은 차이가 있었다.

| 구분 | 복셀 연산 시간 |
|---|---:|
| CPU 연산 | 약 18ms |
| GPU 연산 | 약 1.8ms |

약 10배 수준으로 연산 시간이 줄면서 실제 플레이의 순간적인 끊김도 크게 완화됐다.

## 플레이 결과

최적화 이후에는 복셀을 생성하지 않을 때 약 120 FPS를 유지했다.
사격으로 복셀을 계속 생성하는 상황에서도 평균 100 FPS대까지 유지되는 모습을 확인했다.

![최적화 후 복셀 생성 테스트](/assets/img/deepraiders-voxel-performance/after-optimization-web.gif)

기존에는 사격할 때 40 FPS대까지 떨어졌으므로 체감 차이가 확실했다.

## 아직 남은 병목

GPU 연산 전환으로 모든 프레임 드랍이 해결된 것은 아니다.

- Join Snapshot 생성과 적용
- 거점 탈환 비율 계산
- 복셀 머티리얼 스캔
- 넓은 영역의 렌더 갱신
- 연속된 복셀 Operation 처리

특히 거점 점유율을 계산하기 위한 스캔은 많은 복셀을 확인할 수 있으므로
실행 주기, 한 번에 처리하는 범위, 결과 캐싱 여부를 추가로 점검해야 한다.

Snapshot 또한 생성과 직렬화, 압축, 클라이언트 적용 단계가 있으므로
각 단계를 별도의 Insights 구간으로 측정할 필요가 있다.

## 이번 최적화에서 얻은 점

처음에는 멀티플레이에서 발생한 현상이라는 이유로 네트워크를 의심했다.
하지만 Standalone 비교를 통해 네트워크 가설을 제외했고,
Insights로 실제 대기 구간을 확인하면서 복셀 연산과 렌더 갱신이 핵심 병목임을 찾았다.

```text
네트워크 구조 점검
    ↓
Standalone에서도 재현 확인
    ↓
비동기 복셀 편집
    ↓
머티리얼 완료 후 렌더 갱신
    ↓
렌더 요청 병합
    ↓
CPU 연산을 GPU로 전환
    ↓
Insights와 실제 FPS로 결과 확인
```

## 정리

이번 최적화로 사격 시 40대까지 떨어지던 FPS가 평균 100대까지 개선됐고,
복셀 생성이 없을 때는 120 FPS를 유지했다.

핵심은 네트워크 문제라고 단정하지 않고 Standalone으로 재현 조건을 분리한 것,
그리고 비동기화와 GPU 전환을 각각 적용해 대기 시간과 순수 연산 비용을 함께 줄인 것이다.

남은 Snapshot과 거점 비율 계산도 같은 방식으로 측정 구간을 분리해 하나씩 개선할 예정이다.
