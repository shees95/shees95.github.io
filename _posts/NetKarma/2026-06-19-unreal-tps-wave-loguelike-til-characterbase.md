---
title: "TPS 개발 : 캐릭터 반짝임, 고개 방향 이슈 외"
date: 2026-06-19 00:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, TroubleShooting, TroubleShooting-UnrealEngine, UnrealEngine-Bounds, UnrealEngine-Culling, UnrealEngine-Animation, UnrealEngine-Lyra]
description: "캐릭터 반짝임(Bounds Culling), 고개 방향 역전, 라이라 폴더 분류"
---

# TIL - 언리얼 개발 중 오류 트러블슈팅

---

## 1. 캐릭터가 특정 각도에서 반짝이는 이슈

### 증상

- 그림자는 정상이고 캐릭터 메시만 깜빡임
- 카메라의 Pitch, Yaw를 돌리다 보면 특정 각도에서만 발생

![녹음 2026-06-26 103437.gif](../../assets/netkarma/%EB%85%B9%EC%9D%8C%202026-06-26%20103437.gif)

### 원인

캐릭터의 **Bound Scale**이 너무 작아서 발생.

렌더러가 Bounds를 기준으로 카메라 Frustum 안/밖을 판단하는데, Bounds가 실제 메시보다 작으면 캐릭터가 화면 안에 있어도 밖이라고 판단해 렌더링을 생략한다. 다음 틱에는 다시 안쪽이라고 판단해 렌더링하는 것을 반복하면서 반짝임이 생긴다.

### 해결

```
캐릭터 BP → Details → Rendering → Bounds Scale
기본값 1.0 → 1.8 로 설정
```

![스크린샷 2026-06-26 103601.png](../../assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20103601.png)

1.8 정도부터 해당 이슈가 미발생했다.
모든 각도에서 잘 보일 때 까지 올려야한다.

---

## 2. 캐릭터가 앞으로 달리는데 고개가 왼쪽으로 계속 돌아가는 이슈

### 증상

직진 중에 캐릭터 고개가 왼쪽으로 고정되거나 계속 틀어짐

![스크린샷 2026-06-26 103813.png](../../assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20103813.png)

### 원인

애니메이션 블렌드 스페이스의 **파라미터 축이 역전**되어 있었다.

입력 방향과 블렌드 스페이스의 X/Y 축 매핑이 반대로 연결되어, 앞으로 달리는 입력이 회전 방향으로 해석됐다.

### 해결

블렌드 스페이스 파라미터 연결을 확인해서 축을 올바르게 맞춰줬다.

은근 X축 Y축이 보기 헷깔리게 되어있다..

---

## 3. 라이라 기반 프로젝트 폴더와 격리

### 문제점

라이라를 베이스로 개발하지만 라이라 게임 자체를 쓰지 않기 때문에 폴더 구조가 모호해지는 문제가 있었다.

기존엔 `LyraGame`, `LyraEngine` 폴더만 있어서 우리 게임 전용 코드를 둘 곳이 없었다.

### 해결

**→ 커스텀 게임 전용 cpp 프로젝트 폴더를 별도로 생성해서 분리했다.**

| 폴더             | 용도 |
|----------------|---|
| `LyraGame`     | 라이라 게임 원본 |
| `LyraEngine`   | 라이라 엔진 원본 |
| `NetKarmaGame` | 우리 게임 전용 코드 |

![스크린샷 2026-06-26 104001.png](../../assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20104001.png)

내부에 새 프로젝트 만들어 빌드파일을 만들어 디렉토리를 생성했다.

![스크린샷 2026-06-26 104136.png](../../assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20104136.png)
