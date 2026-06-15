---
title: "언리얼 GAS 사용법 3"
date: 2026-06-10 9:24:00 +0900
categories: [UnrealEngine, UnrealEngine-GAS]
tags: [UnrealEngine, UnrealEngine-GAS, UnrealEngine-GameplayAbility, UnrealEngine-GameplayTag, UnrealEngine-TryActivateAbility, UnrealEngine-ActivationBlockedTags]
description: "태그 부여 방식"
---

바로 그릴게. 3개 다이어그램 순서대로 렌더링함.이제 순서대로 3개 렌더링한다. 각 다이어그램 사이에 설명 포함.

---

## 1. Ability Activation Flow

`UAbilitySystemComponent::TryActivateAbility` 진입부터 `EndAbility` 까지의 흐름.---

## 2. GameplayEffect & Attribute Flow

`ApplyGameplayEffectSpecToSelf` 진입부터 `UAttributeSet::PostGameplayEffectExecute` 까지. Instant / Duration / Infinite GE 분기 포함.---

## 3. Gameplay Cue & Tag Flow

태그 부착/제거 → `GameplayCueNotify` 트리거 → 네트워크 복제 흐름.---

**3개 다이어그램 요약**

| # | 다이어그램 | 핵심 진입점 | 종료 조건 |
|---|-----------|------------|---------|
| 1 | Ability Activation | `TryActivateAbility` | `NotifyAbilityEnded` + delegate |
| 2 | GE & Attribute | `MakeOutgoingSpec` → `ApplyGESpecToSelf` | `OnExpired` / handle 반환 |
| 3 | Cue & Tag | `AddLooseGameplayTag` / `AddGameplayCue` | `NetMulticast` client reconcile |

추가로 **Prediction / Rollback 흐름**, **AbilityTask 내부 tick 루프**, **GE Stacking 정책** 등 특정 서브플로우가 필요하면 말해줘.
