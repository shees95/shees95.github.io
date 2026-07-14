---
title: "TPS 개발 : 프로젝트 GAS 총정리 (2) — 구현편"
date: 2026-07-16 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-GAS, UnrealEngine-ExecCalculation, UnrealEngine-SetByCaller, UnrealEngine-GameplayTag]
description: "GE 부여/해제 핸들 관리, GA 트리거 델리게이트 수신, GiveAbility/ClearAbility, ExecCalculation까지 자주 쓰는 코드 모음"
---

# TPS 개발 - 프로젝트 GAS 총정리 (2) 구현편

[구조편]({% post_url NetKarma/2026-07-16-unreal-tps-wave-loguelike-gas-summary-structure %})에서 4개 계층의 역할을 정리했다. 이번 편은 실제로 반복해서 쓰는 코드들을 언제든 꺼내볼 수 있게 모아둔 것이다.

---

## 1. GE 부여 — SetByCaller + ApplyGameplayEffectSpecToSelf

증강 GA에서 GE를 붙이는 코드는 방어 코드를 빼면 사실상 이 세 줄이 전부다.

```cpp
const FGameplayEffectSpecHandle SpecHandle = CreateAugmentEffectSpec(EffectID, ASC);
SpecHandle.Data->SetSetByCallerMagnitude(NKMGameplayTags::Augment_Data, Value);
const FActiveGameplayEffectHandle Handle = ASC->ApplyGameplayEffectSpecToSelf(*SpecHandle.Data.Get());
```

- `CreateAugmentEffectSpec(EffectID, ASC)` — [EffectManagerSubsystem]({% post_url NetKarma/2026-07-16-unreal-tps-wave-loguelike-gas-summary-structure %})에 연결된 EID로 GE Spec을 만든다.
- `SetSetByCallerMagnitude` — CSV에서 읽어온 수치(`Value`)를 `Augment_Data` 태그에 실어 GE에 주입한다.
- `ApplyGameplayEffectSpecToSelf` — 실제로 적용하고, **`FActiveGameplayEffectHandle`을 반드시 받아둔다.** 이 핸들이 없으면 나중에 뗄 수 없다.

---

## 2. GE 해제 — 핸들 기반 RemoveActiveGameplayEffect

GE를 해제하려면 **핸들을 반드시 갖고 있어야 한다.**

```cpp
RemoveActiveGameplayEffect(핸들, 없앨 스택 수);
```

[구조편]({% post_url NetKarma/2026-07-16-unreal-tps-wave-loguelike-gas-summary-structure %})의 트리거형 증강 예시(권총 태그가 떨어지면 `EndAbility`에서 캐싱된 핸들을 전부 제거)가 바로 이 함수를 쓰는 자리다. `AugmentAbility`의 `EndAbility`가 항상 이 패턴으로 마무리된다.

---

## 3. GA 트리거 델리게이트 받는 법

### 단순한 경우 — BP 이벤트 트리거

조금 복잡한 GA는 트리거 델리게이트를 여러 번 받는 식으로 구성했다. 예를 들어 BP GA의 이벤트 트리거에 `Hit`, `Miss`를 각각 등록해두고, cpp에서는 **받은 EventTag에 따른 분기 처리**로 끝냈다.

```
BP: Event Trigger 노드에 [Event.Weapon.Hit], [Event.Weapon.Miss] 등록
cpp: OnEventReceived(EventTag)
       └─ EventTag == Event.Weapon.Hit  → 처리 A
       └─ EventTag == Event.Weapon.Miss → 처리 B
```

트리거 종류가 늘어나도 BP에 등록만 추가하고, cpp의 분기 하나만 늘리면 된다.

---

## 4. GA 추가/제거 — GiveAbility / ClearAbility

증강으로 GA를 부여하거나 회수할 때는 표준 GAS API를 그대로 쓴다.

- 추가: `GiveAbility(FGameplayAbilitySpec(...))`
- 제거: `ClearAbility(AbilitySpecHandle)`

두 경우 모두 **핸들 관리**가 핵심이다. [AugmentComponent]({% post_url NetKarma/2026-07-16-unreal-tps-wave-loguelike-gas-summary-structure %})가 이 핸들들을 보유하고 있다가, 증강 해제나 계승 시점에 핸들 기준으로 정리한다.

---

## 5. ExecCalculation — 흡혈처럼 복잡한 계산

Stat 값을 단순 가감으로 표현할 수 없는 경우(예: 흡혈 — 가한 데미지의 일정 비율만큼 회복)는 **ExecCalculation**을 하나 만들어서 처리한다.

```cpp
const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();

const float Ratio = Spec.GetSetByCallerMagnitude(NKMGameplayTags::Augment_Data, false, 0.f);
const float DamageDealt = Spec.GetSetByCallerMagnitude(NKMGameplayTags::Data_Damage, false, 0.f);

const float HealAmount = Ratio * DamageDealt;

UE_LOG(LogTemp, Log, TEXT("[Augment] NKMExecCalc_AugmentLifesteal — GEClass=%s, Ratio=%.4f, DamageDealt=%.4f, HealAmount=%.4f"),
    *GetNameSafe(Spec.Def), Ratio, DamageDealt, HealAmount);

if (HealAmount <= 0.f) return;

OutExecutionOutput.AddOutputModifier(FGameplayModifierEvaluatedData(UAS_NKMCharacterState::GetHPAttribute(), EGameplayModOp::Additive, HealAmount));
```

- `GetSetByCallerMagnitude`로 GE에 실려있던 **비율(Ratio)** 과 **데미지량(DamageDealt)** 을 꺼낸다.
- 두 값을 곱해 회복량을 계산하고, `AddOutputModifier`로 HP 어트리뷰트에 가산 적용한다.
- ExecCalculation도 GA에 연결되어 있으므로, **해당 GA에 연동해둔 태그들을 이 안에서 매칭만 하면** 필요한 데이터를 전부 끌어올 수 있다.

> [1편]({% post_url NetKarma/2026-07-12-unreal-tps-wave-loguelike-augment-system-3 %})에서 언급했듯, Stat과 Rule(ExecCalc)을 둘 다 채우면 중복 적용되거나 서로 침식할 수 있다. ExecCalc을 쓸 거면 Stat 쪽은 비워둔다.

---

## 6. BP에서 세팅하는 법

cpp 쪽 로직이 갖춰지면, 실제 사용은 BP 레이어에서 이루어진다.

- GA는 `NKMAugmentAbility`(또는 `AugmentAbility`)를 상속한 BP로 만들고, ID 이름을 그대로 붙인다.
- `DA_NKMAugmentEffectSet`에서 **EID ↔ GA/GE**를 연결한다. 이 연결이 없으면 증강이 아예 분배되지 않는다.
- 트리거가 필요한 GA는 BP의 Event Trigger 노드에 필요한 이벤트 태그(`Event.Weapon.Hit` 등)를 등록하고, 실제 판단 로직은 cpp에 맡긴다.

---

## 정리

이번 프로젝트에서 GAS를 다루며 반복적으로 쓴 코드는 결국 이 여섯 가지로 수렴한다.

1. **GE 연결** — `SetSetByCallerMagnitude` + `ApplyGameplayEffectSpecToSelf`
2. **GE 해제** — 핸들 기반 `RemoveActiveGameplayEffect`
3. **GA 트리거 델리게이트 수신** — BP 이벤트 태그 등록 + cpp 분기
4. **GA 추가/제거** — `GiveAbility` / `ClearAbility`, 핸들 관리
5. **ExecCalculation** — `GetSetByCallerMagnitude`로 값을 끌어와 `AddOutputModifier`로 적용
6. **BP 세팅** — DataAsset에서 EID ↔ GA/GE 연결, 트리거 이벤트 등록

[구조편]({% post_url NetKarma/2026-07-16-unreal-tps-wave-loguelike-gas-summary-structure %})의 4계층(EffectManagerSubsystem / AugmentManagerSubsystem / AugmentComponent / AugmentAbility)과 이 여섯 가지 코드 패턴만 기억해두면, 이 프로젝트의 GAS 관련 작업 대부분을 다시 짤 수 있다.
