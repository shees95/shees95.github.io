---
title: "TPS 개발 : 이펙트 매니저 서브시스템 — GE 해석/적용 일원화"
date: 2026-07-14 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-GAS, UnrealEngine-GameplayEffect, UnrealEngine-Subsystem, UnrealEngine-Augment]
description: "증강 전용으로 GE 테이블을 만들다가, 아이템·업글도 GE를 쓴다는 걸 깨닫고 GE 해석/정책 적용을 일원화한 이펙트 매니저 서브시스템으로 리팩터링"
---

# TPS 개발 - 이펙트 매니저 서브시스템

[증강 시스템]({% post_url NetKarma/2026-07-10-unreal-tps-wave-loguelike-augment-system-1 %})의 GE 테이블을 만들다가, 구조를 한 단계 더 정리하게 됐다.

---

## 계기 — 증강 전용으로 만들기엔 GE 재활용도가 아깝다

Augment 이펙트 테이블을 만들었다. 그런데 완성하고 보니 **이걸 증강용으로만 쓰기엔 GE 재활용도가 너무 낮다**는 생각이 들었다.

곰곰이 보니, GE를 쓰는 게 증강만이 아니었다.

- [크래프팅 특수탄]({% post_url NetKarma/2026-06-30-unreal-tps-wave-loguelike-item-definition-2 %})도 GE로 효과를 적용 중
- [무기 업그레이드]({% post_url NetKarma/2026-07-08-unreal-tps-wave-loguelike-weapon-upgrade-ge %})도 GE로 어트리뷰트를 올리는 중

세 시스템(증강 / 아이템 / 업글)이 전부 GE를 쓰는데, 각자 자기 도메인 안에서 **"CSV 문자열을 GE로 해석하고, Duration/Stack 같은 정책을 적용하는"** 로직을 따로 들고 있었다. [증강 파이프라인]({% post_url NetKarma/2026-07-12-unreal-tps-wave-loguelike-augment-system-3 %})에서 정리했던 "CSV → 해석 → 정책 적용" 단계가, 사실 증강만의 것이 아니라 세 시스템 공통의 문제였던 것이다.

---

## 해결 — 이펙트 매니저 서브시스템

그래서 **GE 해석, 정책 적용 등을 일괄 관리하는 이펙트 매니저 서브시스템**을 만들었다.

```
EffectManagerSubsystem
  ├─ CSV 문자열 → 실제 GE 에셋 해석
  └─ Duration / Instant / Infinite / Stack 등 정책 적용
```

이 서브시스템이 하는 일은 [증강 시스템 1편]({% post_url NetKarma/2026-07-10-unreal-tps-wave-loguelike-augment-system-1 %})에서 `AugmentManagerSubsystem`이 혼자 하던 GE 해석·정책 적용 부분을 **공용 레이어로 뽑아낸 것**이다.

---

## 기존 시스템은 그대로, "경유"만 추가

여기서 중요한 건 **기존 시스템을 갈아엎지 않았다**는 점이다.

- 증강, 아이템, 업글 각 도메인 서브시스템(AugmentManagerSubsystem 등)은 **그대로 유지**된다.
- 각 도메인은 GE를 실제로 해석하고 정책을 적용해야 할 때, 그 부분만 **이펙트 매니저 서브시스템을 한 번 경유**해서 처리한다.
- 컴포넌트가 핸들을 보유하고 관리하는 방식, 도메인별 Definition 구조 등은 [기존 그대로]({% post_url NetKarma/2026-07-12-unreal-tps-wave-loguelike-augment-system-3 %}) 유지된다.

```
Before
  Augment  → (자체) CSV 해석 + 정책 적용 → GE
  Item     → (자체) CSV 해석 + 정책 적용 → GE
  Upgrade  → (자체) CSV 해석 + 정책 적용 → GE
                (셋 다 같은 로직을 각자 구현)

After
  Augment  ┐
  Item     ├─→ EffectManagerSubsystem (해석 + 정책 적용) → GE
  Upgrade  ┘
```

즉 "이 증강/아이템/업글이 무엇을 하는가"를 결정하는 도메인 로직은 각자 갖고 있되, **"그 결과를 실제 GE로 바꾸고 정책을 매기는" 공통 절차만** 서브시스템 하나로 모은 것이다. 데이터 적용 방식만 서브시스템을 한 번 거치도록 바꿨을 뿐, 각 시스템의 소유 구조나 흐름 자체는 건드리지 않았다.

---

## 오늘 정리

- Augment 이펙트 테이블을 만들다가 **GE가 증강 전용이 아니라는 것**을 깨달음
- 아이템(크래프팅 특수탄), 업그레이드도 이미 GE를 쓰고 있었음
- **GE 해석 + 정책 적용을 일괄 관리하는 EffectManagerSubsystem** 신설
- 기존 도메인 서브시스템(Augment/Item/Upgrade)은 유지, **GE 처리 단계만 공용 서브시스템을 경유**하도록 변경

각 도메인이 따로 구현하던 "CSV → GE 해석 → 정책 적용" 로직이 한 곳에 모이니, 앞으로 새 GE 소비처(예: 상태이상, 환경 효과 등)가 생겨도 이 서브시스템만 경유시키면 되는 구조가 됐다.
