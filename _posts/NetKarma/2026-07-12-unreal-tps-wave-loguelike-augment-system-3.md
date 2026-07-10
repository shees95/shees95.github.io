---
title: "TPS 개발 : 증강 시스템 (3) — 전체 파이프라인 정리"
date: 2026-07-12 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-GAS, UnrealEngine-Augment, UnrealEngine-CSV, UnrealEngine-Definition]
description: "CSV 파싱부터 증강 선택·제공까지, 일주일간 만든 증강 시스템의 전체 처리 흐름 회고"
---

# TPS 개발 - 증강 시스템 (3)

[GE]({% post_url NetKarma/2026-07-10-unreal-tps-wave-loguelike-augment-system-1 %})와 [GA]({% post_url NetKarma/2026-07-11-unreal-tps-wave-loguelike-augment-system-2 %}) 파트를 각각 정리했으니, 오늘은 이 둘이 어떻게 하나의 파이프라인으로 굴러가는지 전체 흐름을 정리한다. 증강 시스템 하나 만드는 데 정확히 **일주일**이 걸렸다.

---

## 처리 흐름 8단계

### 1. CSV 파서로 읽어오기

파서 자체적으로 약간의 데이터 가공이 들어간다. 거창한 건 아니고, `add → add(base)` 강제 컨버트 같은 텍스트 정규화 수준이다.

### 2. 게임 시작 시 정책 적용/초기화

`AugmentManagerSubsystem`의 `Initialize`에서 Infinite / Duration / Instant / 스택 여부 등을 조정한다.

| Duration Policy | 동작 |
|---|---|
| Infinite | Base 값 기반 **임시값**을 제공하는 형태 |
| Instant | 바로 AttributeSet 값을 바꿔치는 형태 (스택 안 됨) |
| Duration | 기간(Duration) 동안 유지, Period는 그 기간 중 반복 빈도 |

### 3. CSV 텍스트 → DA에 연결된 GA/GE로 해석

subsystem이 CSV에서 읽어온 문자열들을 DataAsset에 연결된 실제 GA/GE로 해석한다. **해석할 GE 데이터 연결이 없으면(DA 미작성) 해당 증강은 제공되지 않는다.** [1편]({% post_url NetKarma/2026-07-10-unreal-tps-wave-loguelike-augment-system-1 %})에서 언급한 그 스위치다.

### 4. 증강 선택 (Draw)

```
{증강 제공자} → subsystem에 증강 draw 요청
  └─ 랜덤 증강 N개 제공
       └─ 플레이어 선택
            └─ AugmentComponent의 owned(소유) 리스트에 add
                 └─ 이후 draw는 owned 제외하고 제공
```

이미 가진 증강은 다시 뽑히지 않도록 owned 리스트를 필터로 쓴다.

### 5. 증강 제공

subsystem이 증강 ID에 맞는 DataAsset을 제공한다.

- **GA 형태**면 `GiveAbility` 및 핸들 제공
- **PID**면 Infinite 적용 및 핸들 제공

컴포넌트(AugmentComponent)는 이 핸들을 **보유하고 관리만** 한다. 실제 부여/적용 판단은 subsystem이 하고, 컴포넌트는 결과물을 들고 있는 쪽이다.

### 6. GA 발동

증강 ID와 트리거 태그를 부여해두면, [2편]({% post_url NetKarma/2026-07-11-unreal-tps-wave-loguelike-augment-system-2 %})에서 본 대로 DA 값에 따라 GE가 실행된다.

### 7. GE 적용 시 디테일

DA에 묶어만 두면 알아서 값이 따라 들어온다. 몇 가지 세부 규칙이 있다.

- 디버깅용으로 시전 시 **EID 값을 태그로 달아둔다.** (단, Instant는 태그가 안 달림 — 즉시 적용되고 사라지니 태그를 붙일 틈이 없다)
- 흡혈처럼 복잡한 계산이 필요하면 **ExecCalculation으로 빼서 Rule을 적용**하고, Stat 값은 비워둔다.
- **Stat도 적고 Rule도 적으면 둘 다 적용되거나, 하나가 다른 하나에 침식당할 수 있다.** 반드시 둘 중 하나만 채워야 한다.

### 8. 스택

스택 여부도 CSV에서 읽어오는 값이라 **언제든 조정 가능**하다.

---

## 일주일 걸려 정리한 핵심 파일

AI에게 맥락을 줄 때 참고하려고 핵심 파일을 정리해봤다.

**핵심 (반드시 필요)**

| 파일 | 역할 |
|---|---|
| AugmentManagerSubsystem | CSV → 가공 → Definition, 컴포넌트에 제공. EID 연결 안 되면 증강 제공 안 됨 |
| AugmentComponent | Definition 받으면 GiveAbility 요청, 핸들 관리 |
| AugmentEffectSet | CSV 문자열 값을 실제 GA/GE uasset과 매칭 |
| AugmentAbility | GA 기반 베이스 |

**참고하면 좋은 파일**

- `GA_NKMTID0` — 트리거 조건이 복잡하지 않으면 다소 과한 예시. 쉬운 케이스는 그냥 BP에 트리거만 넣어도 충분
- 이벤트 send 클래스 — `GA_Fire`의 `Event.Weapon.Hit`, `GA_Dash`의 `Event.Dash.End` 같은 이벤트 발송지

---

## 오늘 정리

- CSV 파싱 → 정책 초기화(Infinite/Instant/Duration) → DA 해석 → 선택(Draw) → 제공 → GA 발동 → GE 적용 → 스택, 총 **8단계** 파이프라인
- **subsystem이 판단, component는 보유·관리**로 역할 분리
- Instant는 태그 없음, ExecCalc 쓸 땐 Stat 비우기 — 안 지키면 중복/침식 발생
- 핵심 4파일(Subsystem/Component/EffectSet/Ability)만 잡으면 전체 구조 파악 가능

일주일치 삽질을 정리하고 나니, 결국 "CSV는 데이터, DA는 매핑, Subsystem은 해석과 정책, Component는 보관"이라는 역할 분리가 전부였다.
