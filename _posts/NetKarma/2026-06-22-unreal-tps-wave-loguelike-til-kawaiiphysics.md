---
title: "TPS 개발 : 카와이 피직스 떨림, ASC Raw 포인터, 머테리얼 원근"
date: 2026-06-22 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, TroubleShooting, TroubleShooting-UnrealEngine, UnrealEngine-KawaiiPhysics, UnrealEngine-GAS, UnrealEngine-ASC, UnrealEngine-Material]
description: "카와이 피직스 떨림 이슈, ASC TObjectPtr 강제, 카와이 머테리얼 원근 이슈"
---

# TIL - 언리얼 개발 중 오류 트러블슈팅

---

## 1. 카와이 피직스 머리카락 덜덜 떨리는 이슈

### 증상

머리카락에 카와이 피직스를 적용했는데 자연스럽게 흔들리지 않고 덜덜 떨리는 현상 발생.

![녹음 2026-06-26 104428.gif](/assets/netkarma/%EB%85%B9%EC%9D%8C%202026-06-26%20104428.gif)

### 원인 및 해결

피직스 시뮬레이션 기준 공간이 **World** 로 설정되어 있어서, 캐릭터가 이동할 때마다 월드 기준으로 힘을 재계산하며 진동이 생겼다.

```
Kawaii Physics → Simulation Space
World → Component 로 변경
```

![스크린샷 2026-06-26 104509.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20104509.png)

Component 기준으로 바꾸니 캐릭터 이동에 상관없이 부드럽게 동작했다.

![녹음 2026-06-26 104548.gif](/assets/netkarma/%EB%85%B9%EC%9D%8C%202026-06-26%20104548.gif)
하야카짱이 살랑살랑 귀엽게 뛰기 시작했다..

---

## 2. ASC를 Raw 포인터로 선언하면 고장남

### 증상

`AbilitySystemComponent`를 Raw 포인터로 선언했더니 GAS가 제대로 동작하지 않음.

### 원인

GAS `AbilitySystemComponent`는 언리얼 GC 추적 가능한, `UPROPERTY`로 리플렉션 등록된 포인터로만 사용 가능하다.   
그리고 내부적으로 `TObjectPtr` 사용이 강제되어 있다.

### 해결

```cpp
// ❌ Raw 포인터는 UPROPERYT() 등록을 해도 오류남.
UAbilitySystemComponent* AbilitySystemComponent;

// ✅
UPROPERTY()
TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;
```

---

## 3. 카와이 피직스 — 무기/부품 머테리얼 원근이 안 먹히는 이슈

### 증상

카와이 피직스를 쓰는 캐릭터에서 무기나 부품에 적용한 머테리얼의 원근(거리 기반 효과)이 제대로 동작하지 않음.

![녹음 2026-06-26 105434.gif](/assets/netkarma/%EB%85%B9%EC%9D%8C%202026-06-26%20105434.gif)

### 원인 및 해결

카와이 피직스의 **아웃라이너 / 바디 전용 머테리얼**을 부모로 상속받아 머테리얼을 만들어야 원근이 올바르게 적용된다.

![스크린샷 2026-06-26 104929.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20104929.png)

무기·부품의 머테리얼을 해당 베이스 머테리얼을 상속받아 생성하고 적용하니 정상 동작했다.
