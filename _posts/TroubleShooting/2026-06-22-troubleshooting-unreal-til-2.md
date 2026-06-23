---
title: "TIL : 카와이 피직스 떨림, ASC Raw 포인터, 머테리얼 원근"
date: 2026-06-22 12:00:00 +0900
categories: [TroubleShooting, TroubleShooting-UnrealEngine]
tags: [TroubleShooting, TroubleShooting-UnrealEngine, UnrealEngine-KawaiiPhysics, UnrealEngine-GAS, UnrealEngine-ASC, UnrealEngine-Material]
description: "카와이 피직스 떨림 이슈, ASC TObjectPtr 강제, 카와이 머테리얼 원근 이슈"
---

# TIL

---

## 1. 카와이 피직스 머리카락 덜덜 떨리는 이슈

### 증상

머리카락에 카와이 피직스를 적용했는데 자연스럽게 흔들리지 않고 덜덜 떨리는 현상 발생.

### 원인 및 해결

피직스 시뮬레이션 기준 공간이 **World** 로 설정되어 있어서, 캐릭터가 이동할 때마다 월드 기준으로 힘을 재계산하며 진동이 생겼다.

```
Kawaii Physics → Simulation Space
World → Component 로 변경
```

Component 기준으로 바꾸니 캐릭터 이동에 상관없이 부드럽게 동작했다.

---

## 2. ASC를 Raw 포인터로 선언하면 고장남

### 증상

`AbilitySystemComponent`를 Raw 포인터로 선언했더니 GAS가 제대로 동작하지 않음.

### 원인

언리얼의 GC(가비지 컬렉터)는 `UPROPERTY`로 등록된 포인터만 추적한다. Raw 포인터는 GC 대상이 아니라서 오브젝트가 수집되거나 댕글링 포인터가 될 수 있다.

`AbilitySystemComponent`는 내부적으로 `TObjectPtr` 사용이 강제되어 있다.

### 해결

```cpp
// ❌
UAbilitySystemComponent* AbilitySystemComponent;

// ✅
UPROPERTY()
TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;
```

---

## 3. 카와이 피직스 — 무기/부품 머테리얼 원근이 안 먹히는 이슈

### 증상

카와이 피직스를 쓰는 캐릭터에서 무기나 부품에 적용한 머테리얼의 원근(거리 기반 효과)이 제대로 동작하지 않음.

### 원인 및 해결

카와이 피직스의 **아웃라이너 / 바디 전용 머테리얼**을 부모로 상속받아 머테리얼을 만들어야 원근이 올바르게 적용된다.

무기·부품의 머테리얼을 해당 베이스 머테리얼을 상속받아 생성하고 적용하니 정상 동작했다.
