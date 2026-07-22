---
title: "TPS 개발 : 프로젝트 GAS 총정리 (1) — 구조편"
date: 2026-07-16 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-GAS, UnrealEngine-Subsystem, UnrealEngine-Augment, UnrealEngine-GameplayEffect]
description: "EffectManagerSubsystem, AugmentManagerSubsystem, AugmentComponent, AugmentAbility 네 계층의 역할과 구조 정리"
---

# TPS 개발 - 프로젝트 GAS 총정리 (1) 구조편

[증강 시스템 삼부작]({% post_url NetKarma/2026-07-10-unreal-tps-wave-loguelike-augment-system-1 %})과 [이펙트 매니저 서브시스템]({% post_url NetKarma/2026-07-14-unreal-tps-wave-loguelike-effect-subsystem %})까지 만들고 나니, 이 프로젝트에서 GAS를 어떻게 쓰고 있는지 한 번은 통째로 정리해둘 필요가 있었다. 언제든 다시 찾아볼 수 있게 **구조편(이 글)** 과 **구현편**으로 나눠서 적는다.

전체는 4개 계층으로 나뉜다.

```
EffectManagerSubsystem   ← GE 해석 / 정책 초기화 (공용)
        ↓
AugmentManagerSubsystem  ← 증강 데이터 해석 / 제공
        ↓
AugmentComponent         ← 부여받은 GA/GE 보유·관리
        ↓
AugmentAbility           ← 실제 GE를 붙이고 떼는 GA 부모 클래스
```

---

## 1. EffectManagerSubsystem — CSV 테이블 읽기 및 증강 해석

GE를 재사용하기 위해 CSV를 짠 뒤, **정책이 통일된 GE들을 재사용**하는 구조다. 증강뿐 아니라 [아이템(크래프팅 특수탄)]({% post_url NetKarma/2026-06-30-unreal-tps-wave-loguelike-item-definition-2 %})과 [무기 업그레이드]({% post_url NetKarma/2026-07-08-unreal-tps-wave-loguelike-weapon-upgrade-ge %})도 전부 GE로 구현되어 있다.

증강만 CSV+GE로 컨트롤하기엔 구조가 아쉬워서 전부 통합했는데, 그러다 보니 **게임 시작 전 전체 GE 정책을 초기화하는 작업**이 필요해졌다. 이 초기화는 게임 시작 전에 모든 세팅이 끝나 있어야 하는 일이라, 이를 전담하는 서브시스템을 하나 구축했다.

- EffectManagerSubsystem이 `Initialize`될 때 **한 번에 초기화**한다.
- **GE BP와 CSV 테이블 ID를 DataAsset에서 연결**만 해주면, CSV 테이블의 정책 값들이 GE BP에 자동으로 적용된다.
- **Effects와 Effects ID가 연동되어 있는지도 체크**한다. 연동되지 않은 GE를 소유한 GA나 GE는 **증강 분배에서 전부 제외**된다.

즉 이 서브시스템의 존재 이유는 "CSV 문자열 하나로 GE 하나를 완성시키는" 반복 작업을, 증강/아이템/업글이 각자 하지 않고 한 곳에서 끝내는 것이다.

---

## 2. AugmentManagerSubsystem — 증강 제공

CSV에서 글자와 숫자로 받아온 데이터들을 **증강 관점에서 해석**하기 위한 서브시스템이다. EffectManagerSubsystem이 "GE를 어떻게 만들지"를 담당한다면, 이쪽은 "그 GE를 증강으로서 언제·어떻게 제공할지"를 담당한다.

증강은 CSV 테이블 상에서 크게 두 갈래로 나뉜다.

| 유형 | 설명 |
|---|---|
| 트리거형 | 특정 행동이 트리거가 되어 효과가 들어옴 (GA로 구현) |
| 일반 패시브 | 조건 없이 즉시 적용됨 (GE로 구현) |

### 트리거형 증강 — GA + Event.Trigger 델리게이트 + GE

특정 행동이 트리거되는 경우는 **GA로 `Event.Trigger`를 델리게이트로 받아 `ActivateAbility`** 한다. 
![스크린샷 2026-07-15 103028.png](../../assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-07-15%20103028.png)

트리거가 제거된 후에 GE를 떼어내야 하는 경우에만 **`EndAbility`에서 명시적으로 제거**한다.

예를 들어 총기별 증강이 있다면, 권총을 들었을 때 `Player.Weapon.Pistol` 같은 태그가 달린다. 해당 태그가 붙어있는 동안은 `ActivateAbility`가 1회 실행되고 대기 상태에 들어가며, 태그가 떨어지는 순간 `EndAbility`가 돌면서 `RemoveActiveGameplayEffect`로 **캐싱해둔 핸들들을 전부 제거**한다.

```
Player.Weapon.Pistol 태그 부착
  └─ ActivateAbility (1회) → GE 적용, 핸들 캐싱, 대기
       └─ 무기 스왑 (태그 소멸)
            └─ EndAbility → RemoveActiveGameplayEffect(핸들들)
```

### 패시브형 증강 — 조건 없이 즉시 GE 적용

패시브 GE는 조건이 필요 없기 때문에, 증강을 선택하는 즉시 **`EffectToActor`로 바로 붙여준다.** 트리거를 기다릴 필요가 없는 패시브 증강은 이 경로다.

---

## 3. AugmentComponent — 증강 관리

부여, 해제, 보유 중인 GA·GE 등을 여기서 관리한다. 나중에 (2회차 이후) 증강을 이어받는 **계승** 기능도 이 컴포넌트에서 관리될 예정이다.

컴포넌트는 **관리만** 하면 되도록 역할을 좁혀뒀기 때문에, 당장은 크게 작성된 로직이 없다. 판단(무엇을 줄지, 언제 뗄지)은 위쪽 Subsystem들이 하고, 이 컴포넌트는 그 결과(핸들, 목록)를 들고 있는 저장소에 가깝다.

---

## 4. AugmentAbility — GA 부여용 부모 클래스

GA 부여를 위해 `AugmentAbility`라는 부모 클래스를 파서, 증강 GA 실행의 공통 뼈대로 사용했다.

- `ActivateAbility`하면 **바로 GE를 부여할 수 있게** 세팅해두고, 트리거가 떨어지면 `EndAbility`로 전환되게 만들었다.
- `EndAbility` 시 **모든 GE를 떨어뜨리도록** 했다.

트리거형이든 일반이든, 증강 GA는 결국 이 부모 클래스의 "부여 → (대기) → 트리거 → 전부 제거" 골격 위에서 움직인다.

---

## 구조 요약

| 계층 | 역할 |
|---|---|
| EffectManagerSubsystem | CSV → GE 해석, 정책 초기화 (증강/아이템/업글 공용) |
| AugmentManagerSubsystem | CSV 데이터를 증강 관점에서 해석, 트리거형/패시브 분기 처리 |
| AugmentComponent | 부여받은 GA/GE 보유·관리, 추후 계승 담당 |
| AugmentAbility | 증강 GA의 공통 부모 — 부여/트리거/전체 해제 골격 |

구조를 잡았으니, 다음 편에서는 실제로 자주 쓰는 코드들 — GE 연결/해제, GA 트리거 델리게이트 수신, GiveAbility/ClearAbility, ExecCalculation — 을 정리한다.
