---
title: "TPS 개발 : 에임 오프셋 이슈 정리"
date: 2026-06-28 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, TroubleShooting, TroubleShooting-UnrealEngine, UnrealEngine-AimOffset, UnrealEngine-AdditiveAnim, UnrealEngine-LayeredBlendPerBone]
description: "에임 오프셋 시퀀스 깨짐, additive settings 일치, layered blend per bone 상체 회전 이슈"
---

# TIL - 에임 오프셋 이슈

---

## 1. 에임 오프셋 시퀀스 깨짐

### 증상

에임 오프셋에 들어가는 시퀀스가 깨졌다.

### 해결

`Additive Anim Type`을 **Mesh → Local**로 변경했다가 다시 **Mesh**로 되돌리니 멀쩡해졌다.

> 캐시 꼬임으로 추정. 값을 한 번 토글해주면 정상화됐다.

---

## 2. Additive Settings는 완전히 일치해야 함

에임 오프셋에 시퀀스를 넣으려면 **additive settings가 완전히 일치**해야 들어간다.

```
Additive Anim Type
Base Pose Type
Base Pose Animation
```

하나라도 다르면 에임 오프셋에 들어가지 않는다. 전부 동일하게 맞춰야 한다.

---

## 3. Layered Blend Per Bone과 섞을 때 상체만 회전

### 증상

에임 오프셋에 `Layered Blend Per Bone`을 섞으면 상체만 돌아간다.

### 원인 / 해결

몸 전체가 같이 돌아야 한다면 `Use Controller Rotation Yaw`를 설정해줘야 한다.

```
Character → Use Controller Rotation Yaw : true
```

- 상체만 회전 → 의도된 동작 (Layered Blend Per Bone이 상체 본부터 블렌드)
- 몸 전체 회전 필요 → `Use Controller Rotation Yaw` 활성화

---

## 정리

- 에임 오프셋 시퀀스 깨지면 Additive Anim Type을 Local↔Mesh 토글
- Additive settings는 하나라도 다르면 에임 오프셋에 안 들어감 — 완전 일치 필요
- Layered Blend Per Bone은 상체만 회전. 몸 전체 회전은 Use Controller Rotation Yaw로 처리
