---
title: "TPS 개발 : 증강 시스템 (2) — GA와 이벤트 트리거"
date: 2026-07-11 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-GAS, UnrealEngine-GA, UnrealEngine-Augment, UnrealEngine-GameplayTag]
description: "증강 시스템의 GA 파트 — NKMAugmentAbility 상속과 GAS 이벤트 태그로 트리거 조건 구성하기"
---

# TPS 개발 - 증강 시스템 (2)

어제 GE와 EffectSet 연결까지 정리했다. 오늘은 "특정 조건에서 발동하는" 능동형 증강, **GA 파트**다.

---

## NKMAugmentAbility 상속

CPP의 `NKMAugmentAbility`를 상속받은 BP 또는 CPP 클래스를 하나 만든다. 이름은 **ID 이름으로 그냥 만들었다.** 어차피 내용을 디테일하게 적으면 가독성만 떨어질 것 같아서, 식별용 이름으로 충분하다고 판단했다.

트리거가 복잡한 경우, 혹은 트리거에 따른 또 다른 트리거가 필요한 경우에만 cpp에 내용을 추가로 적는다. **CPP로 로직을 만들었어도, 실제로는 BP로 다시 한 번 만들어서 트리거는 BP에서 부르는 식**으로 정리했다. 로직은 cpp가 갖고, 실행 트리거는 BP 레이어에서 관리하는 셈이다.

---

## 트리거 적용 — GAS 이벤트 태그

트리거는 GAS가 기본 제공하는 **이벤트 발생 태그**를 그대로 활용한다.

현재 정의된 이벤트는 이렇다.

| 어빌리티 | 이벤트 |
|---|---|
| GA_Fire | 사격 시 / 빗맞출 시 / 맞출 시 / WeakPoint 맞출 시 |
| GA_Dash | 대쉬 시작 / 대쉬 종료 |
| (전역) | HP 50% / 30% / 1% 미만일 때 태그 부착 |

각 어빌리티가 특정 시점에 이벤트를 `SendGameplayEvent`로 쏘고, 증강 GA는 그 이벤트 태그를 트리거로 받아서 반응한다.

```
GA_Fire::OnHit
  └─ SendGameplayEvent(Event.Weapon.Hit)
       └─ 증강 GA가 이 이벤트를 트리거로 등록해뒀다면 → 발동
```

---

## GA도 DataAsset에 연결해야 한다

GA를 만들었다고 끝이 아니다. GE와 마찬가지로 **`Data\DA_NKMAugmentEffectSet`에서 GA도 연결**해줘야 한다.

이렇게 연결해두면, **GA 트리거가 실행되는 순간 연결된 GE들이 전부 부여**된다.

```
DA_NKMAugmentEffectSet
  └─ EID (예: "AUG_Fire_OnKill_Heal")
       ├─ GA: GA_Augment_FireOnKillHeal
       └─ 연결된 GE 목록: [GE_Heal]
```

즉 GA는 "언제(트리거) 발동할지"를, DA에 연결된 GE는 "발동하면 무엇을 적용할지"를 담당하는 구조다.

---

## GA가 너무 복잡할 때

정 트리거 로직이 복잡해지면, **DA에는 GA 매칭만** 해두고 **GA cpp 내부에서 전부 해결**한 다음 호출만 잘 해주는 식으로 우회한다. DA ↔ GE 자동 연결 규칙에 억지로 맞추기보다, 복잡한 케이스는 GA가 알아서 처리하게 하는 편이 유지보수가 쉬웠다.

---

## 오늘 정리

- 증강 GA는 `NKMAugmentAbility` 상속, **ID 이름**으로 생성 (가독성보다 식별 우선)
- 로직은 cpp, **트리거 호출은 BP**에서 하도록 분리
- 트리거는 GAS 이벤트 태그 활용 (`GA_Fire`의 사격/빗맞음/명중/약점, `GA_Dash`의 시작/종료, HP 임계값 태그)
- GA도 `DA_NKMAugmentEffectSet`에 연결해야 트리거 시 GE가 부여됨
- 트리거가 너무 복잡하면 DA 매칭 + GA cpp 내부 처리로 우회

GE와 GA 각각의 역할은 잡혔으니, 다음은 이 둘을 하나로 묶는 전체 데이터 파이프라인이다.
