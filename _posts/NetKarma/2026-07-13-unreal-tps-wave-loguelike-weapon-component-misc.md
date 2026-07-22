---
title: "TPS 개발 : 웨폰 컴포넌트 잡다한 개선 + 소프트 포인터 TIL"
date: 2026-07-13 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Weapon, UnrealEngine-Inventory, UnrealEngine-Delegate, UnrealEngine-SoftPointer]
description: "인벤 캐싱 최적화, 탄 레이아웃 swap, 피격 정보 델리게이트, 소프트 포인터/레퍼런스/클래스 개념 정리"
---

# TPS 개발 - 웨폰 컴포넌트 잡다한 개선

증강 시스템 삼부작을 끝내고, 오늘은 그동안 밀린 자잘한 개선들을 처리했다.

---

## 인벤토리 캐싱 처리

인벤토리 아이템 개수를 매번 순회해서 계산하던 걸 캐싱으로 바꿨다.

| 대상 | 처리 |
|---|---|
| 변경된 배열(array) | 변경분만 갱신 |
| 변경된 개별 아이템 | 해당 아이템만 갱신 |
| 모든 아이템 개수 | 병합해서 **캐싱** 처리 |

전체 개수를 물어볼 때마다 순회하지 않고, 변경이 일어난 시점에만 캐시를 갱신하도록 했다.

---

## WeaponComponent — 탄 레이아웃

스페셜 탄 리스트에 이어서, 탄 배열 순서를 다루는 기능을 추가했다.

- 탄 레이아웃 **인덱스 Get**
- 탄 레이아웃 **변경(Swap)**

UI에서 탄 순서를 드래그해서 바꾸는 것 같은 조작을 위한 기반 함수들이다.

---

## 타쿠 피격 정보 델리게이트 → GA_Fire

타쿠(적)가 맞았을 때의 정보를 델리게이트로 묶어서 `GA_Fire`에 전달하도록 정리했다.

```
피격 정보 델리게이트
  ├─ 데미지
  ├─ WeakPoint 여부
  └─ 전달한 GE
```

히트 판정에서 얻은 결과를 그냥 흩뿌리지 않고, 데미지·약점 여부·적용된 GE를 한 묶음으로 만들어 `GA_Fire`가 필요한 시점에 받아 쓰게 했다. 증강 시스템의 이벤트 트리거도 결국 이 정보를 근거로 발동하는 것이라, 여기서 정보가 깔끔하게 정리돼야 뒷단이 편해진다.

---

## TIL — 소프트 포인터 / 소프트 레퍼런스 / 소프트 클래스

언리얼의 소프트 참조 계열 개념을 다시 짚었다.

| 타입 | 예시 | 특징 |
|---|---|---|
| Soft Pointer / Soft Object Ptr | `TSoftObjectPtr<UTexture2D>` | 에셋 경로만 들고 있고, 실제 로드는 명시적으로(`LoadSynchronous`, 비동기 로드 등) 해야 함 |
| Soft Reference | 에셋 경로(FSoftObjectPath) | 참조만 하고 강제로 메모리에 들고 있지 않음 |
| Soft Class | `TSoftClassPtr<AActor>` | 클래스 자체를 소프트하게 참조, 필요할 때 로드 |

Hard Reference(`UPROPERTY()`로 직접 포인터를 들고 있는 것)와 달리, 소프트 계열은 **참조하는 순간 바로 메모리에 로드되지 않는다.** 그래서 무기·업그레이드·증강처럼 종류가 많고 한꺼번에 다 메모리에 있을 필요 없는 데이터를 다룰 때, 필요한 시점에만 로드하도록 설계할 수 있다.

---

## 오늘 정리

- 인벤토리 아이템 개수는 **변경 시점에만 캐시 갱신**하도록 최적화
- WeaponComponent에 **탄 레이아웃 인덱스 조회/스왑** 추가
- 피격 정보(데미지/WeakPoint/GE)를 델리게이트로 묶어 `GA_Fire`에 전달
- 소프트 포인터/레퍼런스/클래스 = **참조는 하되 로드는 필요할 때만** 하는 계열
