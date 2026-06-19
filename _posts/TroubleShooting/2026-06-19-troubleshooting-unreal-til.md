---
title: "TIL : 캐릭터 반짝임, 고개 방향 이슈 외"
date: 2026-06-19 00:00:00 +0900
categories: [TroubleShooting, TroubleShooting-UnrealEngine]
tags: [TroubleShooting, TroubleShooting-UnrealEngine, UnrealEngine-Bounds, UnrealEngine-Culling, UnrealEngine-Animation, UnrealEngine-Lyra]
description: "캐릭터 반짝임(Bounds Culling), 고개 방향 역전, 라이라 폴더 분류"
---

# TIL

---

## 1. 캐릭터가 특정 각도에서 반짝이는 이슈

### 증상

- 그림자는 정상이고 캐릭터 메시만 깜빡임
- 카메라의 Pitch, Yaw를 돌리다 보면 특정 각도에서만 발생

### 원인

캐릭터의 **Bound Scale**이 너무 작아서 발생.

렌더러가 Bounds를 기준으로 카메라 Frustum 안/밖을 판단하는데, Bounds가 실제 메시보다 작으면 캐릭터가 화면 안에 있어도 밖이라고 판단해 렌더링을 생략한다. 다음 틱에는 다시 안쪽이라고 판단해 렌더링하는 것을 반복하면서 반짝임이 생긴다.

### 해결

```
캐릭터 BP → Details → Rendering → Bounds Scale
기본값 1.0 → 1.8 이상으로 설정
```

1.8 정도부터 해당 이슈가 미발생했다.

---

## 2. 캐릭터가 앞으로 달리는데 고개가 왼쪽으로 계속 돌아가는 이슈

### 증상

직진 중에 캐릭터 고개가 왼쪽으로 고정되거나 계속 틀어짐

### 원인

애니메이션 블렌드 스페이스의 **파라미터 축이 역전**되어 있었다.

입력 방향과 블렌드 스페이스의 X/Y 축 매핑이 반대로 연결되어, 앞으로 달리는 입력이 회전 방향으로 해석됐다.

### 해결

블렌드 스페이스 파라미터 연결을 확인해서 축을 올바르게 맞춰줬다.

---

## 3. 라이라 기반 프로젝트 폴더 분류

라이라를 베이스로 개발하지만 라이라 게임 자체를 쓰지 않기 때문에 폴더 구조가 모호해지는 문제가 있었다.

기존엔 `LyraGame`, `LyraEngine` 폴더만 있어서 우리 게임 전용 코드를 둘 곳이 없었다.

**→ 커스텀 게임 전용 cpp 프로젝트 폴더를 별도로 생성해서 분리했다.**

| 폴더 | 용도 |
|---|---|
| `LyraGame` | 라이라 게임 원본 |
| `LyraEngine` | 라이라 엔진 원본 |
| `(커스텀 폴더)` | 우리 게임 전용 코드 |
