---
title: "DeepRaiders 개발 : 복셀 월드 바운드와 최대 깊이"
date: 2026-08-10 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, UnrealEngine-Project-DeepRaiders]
tags: [UnrealEngine, DeepRaiders, Voxel, WorldBounds]
description: "Render Octree Depth와 Voxel Size로 결정되는 복셀 월드 범위를 확인하고 확장한 과정"
---

# DeepRaiders 개발 - 복셀 월드 바운드와 최대 깊이

[이전 기록]({% post_url DeepRaiders/2026-08-07-unreal-deepraiders-voxel-world-test %})에서 약 5만 cm 아래로 내려가자 복셀이 생성되지 않았다. 원인은 굴착 함수가 아니라 복셀 월드의 최대 바운드였다.

---

## 현재 월드 크기 계산

테스트 당시 설정은 다음과 같았다.

```text
RenderOctreeDepth = 5
WorldSizeInVoxel = 32 × 32 = 1,024
VoxelSize = 100 cm
```

월드 전체 길이는 약 `102,400 cm`이고 원점을 중심으로 생성되므로, 실제 범위는 대략 다음과 같다.

```text
-51,200 cm ~ +51,200 cm
```

따라서 Z축으로 약 `-50,000 cm`까지 내려가면 월드 바운드 끝에 가까워지고, 그 아래에는 복셀 데이터가 생성되지 않는다.

## 바운드 확장

월드의 바운드 값을 늘리자 더 넓고 깊은 복셀 맵을 생성할 수 있었다. DeepRaiders는 아래로 계속 개척하는 게임이므로 예상 플레이 깊이보다 충분히 큰 범위를 확보해야 한다.

다만 범위를 크게 잡는 것만으로 끝낼 수는 없다. 월드 크기가 커질수록 청크 관리와 생성 비용도 함께 고려해야 하므로, 실제로 필요한 영역만 로드하는 구조와 같이 사용해야 한다.

---

## 정리

- 복셀 월드는 무한하지 않고 설정에 따른 최대 바운드를 가진다.
- 현재 설정에서는 원점 기준 약 `-51,200 ~ +51,200 cm`가 생성 범위였다.
- 바운드를 확장해 더 깊은 맵을 만들되 청크 로딩 비용도 함께 관리해야 한다.
