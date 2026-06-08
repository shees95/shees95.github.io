---
title: "언리얼 GAS - 기본 : 큰 틀 이해"
date: 2026-06-08 23:00:00 +0900
categories: [UnrealEngine, UnrealEngine-GAS]
tags: [UnrealEngine, UnrealEngine-GAS, UnrealEngine-ASC, UnrealEngine-AS, UnrealEngine-GA, UnrealEngine-GE, UnrealEngine-GameplayTag]
description: "GAS의 전체 구조와 핵심 컴포넌트 역할 이해 (기본)"
---

# GAS (Gameplay Ability System)

스킬, 버프/디버프, 스탯, 상태 등 복잡한 게임플레이 로직을  
**재사용 가능하고 확장 가능한 구조**로 관리하기 위한 언리얼 공식 프레임워크

> RPG, MOBA, 액션게임 전반에 사용 가능  
> 언리얼 Lyra, Fortnite 등 Epic 자체 게임도 GAS 기반

---

## 왜 GAS를 쓰는가

GAS 없이 스킬 시스템을 직접 짜면:
- 스킬마다 HP 계산 코드 중복
- 버프/디버프 상태 관리가 Character에 뒤엉킴
- 쿨타임, 코스트 조건 체크가 각 스킬 안에 흩어짐
- 스킬 추가할수록 의존성이 폭발적으로 증가

GAS는 이걸 역할별로 분리해서 해결한다

---

## 핵심 구성요소 6가지

```
┌─────────────────────────────────────────────┐
│           ASC (중앙 관리자)                  │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │    GA    │  │    GE    │  │    AS    │  │
│  │  (능력)  │─▶│  (효과)  │─▶│  (수치)  │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│  GameplayTag ──── 상태/조건 표현            │
│  GameplayCue ──── 시각/청각 피드백          │
└─────────────────────────────────────────────┘
```

---

### 1. ASC (Ability System Component)

**GAS의 중심. 모든 것이 여기를 통한다.**

- GA 부여 / 해제 / 실행 관리
- GE 적용 관리
- GameplayTag 보유 목록 관리
- AS 소유

```
액터에 컴포넌트로 붙인다 (Character, PlayerState 등)
```

> 어디에 붙이는지는 설계에 따라 다름  
> 싱글: Character에 직접  
> 멀티: PlayerState에 (리스폰 후에도 유지되도록)

---

### 2. GA (Gameplay Ability)

**"무엇을 할 수 있는가" 를 정의**

- 스킬, 구르기, 공격, 힐 등 하나의 능력 = 하나의 GA 클래스
- 발동 가능 조건 체크 (태그, 코스트, 쿨타임)
- 실제 로직 수행 후 GE를 ASC에 Apply 요청
- 반드시 `EndAbility()`로 종료해야 함

```
GA는 ASC에 "부여(Give)" 되어야 사용할 수 있다
부여 ≠ 사용   /   부여 후 → 조건 충족 시 → 활성화
```

---

### 3. GE (Gameplay Effect)

**"수치를 어떻게 바꿀 것인가" 를 정의**

GA가 "판단"한다면, GE는 "적용"한다

- AS의 속성값을 변경하는 유일한 공식 경로
- Duration(지속 타입)으로 버프/디버프 표현

| 타입 | 설명 | 예시 |
|---|---|---|
| `Instant` | 즉시 적용 후 종료 | 데미지, 힐 |
| `Duration` | 일정 시간 후 종료 | 3초 버프 |
| `Infinite` | 수동으로 제거할 때까지 유지 | 패시브, 장비 스탯 |

---

### 4. AS (Attribute Set)

**"어떤 수치를 가지는가" 를 정의**

- HP, MaxHP, MP, Attack, Defense 같은 게임 수치를 `FGameplayAttributeData` 타입으로 보유
- ASC가 소유하며 GE를 통해서만 변경

```c++
UPROPERTY(BlueprintReadOnly, Category="Attributes")
FGameplayAttributeData HP;
```

> AS를 직접 수정하면 안 됨 → 반드시 GE를 통해서

---

### 5. GameplayTag

**"지금 어떤 상태인가" 를 표현**

GAS 전반에서 조건/상태/식별을 태그 하나로 처리

```
Ability.Attack          → 공격 어빌리티 태그
State.Stunned           → 기절 상태 태그
Event.Hit               → 피격 이벤트 태그
Data.HealAmount         → 힐량 데이터 전달용 태그
```

사용 예시:
- GA 발동 조건: `State.Stunned` 태그 없을 때만 실행
- GE 적용 조건: 특정 태그 있을 때만 발동
- 상태 확인: `ASC->HasMatchingGameplayTag(FGameplayTag::RequestGameplayTag("State.Stunned"))`

---

### 6. GameplayCue

**"효과를 어떻게 보여줄 것인가" 를 정의**

- 파티클, 사운드, 애니메이션 등 시각/청각 피드백 담당
- 게임플레이 로직과 완전히 분리됨
- 멀티에서는 자동으로 클라이언트에 전파

```
GA → GE Apply → GameplayCue 발동 → 파티클/사운드 재생
```

> 현재 기본 단계에서는 "있다" 정도만 알면 됨

---

## 전체 흐름 (어빌리티 발동 시)

```
1. 입력 감지
      ↓
2. ASC->TryActivateAbilityByClass(GA클래스)
      ↓
3. GA 조건 체크 (태그, 쿨타임, 코스트)
   ├─ 실패 → 종료
   └─ 성공 → ActivateAbility() 실행
                ↓
4. GA 로직 수행
      ↓
5. GE를 ASC에 Apply
      ↓
6. GE → AS 속성값 변경
      ↓
7. GameplayCue 발동 (파티클, 사운드)
      ↓
8. EndAbility()
```

---

## 요약

| 컴포넌트 | 한 줄 역할 |
|---|---|
| **ASC** | 중앙 관리자. 모든 GAS 기능의 허브 |
| **GA** | 능력 정의. 조건 체크 + 로직 수행 |
| **GE** | 수치 변경 정의. AS에 반영하는 유일한 경로 |
| **AS** | 수치 보관. HP/MP/Attack 등 |
| **GameplayTag** | 상태/조건/식별을 태그로 표현 |
| **GameplayCue** | 시각/청각 피드백 (로직과 분리) |

---

## 다음 단계 (초급)

- [ ] ASC를 Character에 붙이고 AS 연결하기
- [ ] 간단한 GA 하나 만들어서 부여/활성화 해보기
- [ ] Instant GE로 HP 변경 해보기
- [ ] GameplayTag 조건으로 GA 제한해보기
