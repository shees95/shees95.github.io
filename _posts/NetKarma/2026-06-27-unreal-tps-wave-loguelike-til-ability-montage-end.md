---
title: "TPS 개발 : 몽타주 기준 어빌리티 종료 — Reload / Dash / Fire"
date: 2026-06-27 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, TroubleShooting, TroubleShooting-UnrealEngine, UnrealEngine-GAS, UnrealEngine-AnimMontage, UnrealEngine-SetByCaller, UnrealEngine-GameplayTag]
description: "SetByCaller 태그 규칙과 몽타주 종료 델리게이트 기준으로 어빌리티 종료 처리"
---

# TIL - 몽타주 기준 어빌리티 종료

---

## 1. SetByCaller의 태그 필터

`SetByCaller`는 자체 태그 필터가 적용되어 있다.

`Data.*` 형태의 태그로 넣어줘야 경고 문구가 뜨지 않는다.

```cpp
// 권장 - Data. 접두사
SpecHandle.Data->SetSetByCallerMagnitude(
    FGameplayTag::RequestGameplayTag("Data.Damage"), DamageValue);
```

> 필수는 아니다. `Data.` 접두사를 안 써도 동작은 하지만 경고가 출력된다.

---

## 2. 몽타주 종료 기준으로 어빌리티 끝내기

세 어빌리티 모두 **몽타주 재생이 끝나는 시점**을 기준으로 종료되어야 한다.

공통 구조:
~~~~
```
GA Activate
  └─ PlayMontageAndWait (몽타주 재생)
       └─ OnMontageEnded / OnBlendOut 델리게이트
            └─ EndAbility + StateTag 정리
```

각 몽타주의 길이는 어빌리티 특성에 맞는 값에 기반한다.

| 어빌리티 | 몽타주 길이 기준 | 추가 처리 |
|---|---|---|
| Reload | 장전 속도 | 몽타주 종료 델리게이트에서 EndAbility, StateTag 반영 |
| Dash | 대시 Duration | EndAbility, StateTag 반영 + 대시 스택은 별도 GE로 구분 |
| Fire | Fire Speed | 탄 소모는 GE의 SetByCaller로 반영, 0 체크는 cpp |

---

### Reload

- 몽타주 = 장전 속도 기반
- 몽타주 종료 델리게이트 기준으로 `EndAbility` 호출, StateTag 정리

### Dash

- 몽타주 = 대시 Duration 기반
- 몽타주 종료 델리게이트 기준 `EndAbility`, StateTag 반영
- **대시 스택은 GE로 따로 구분**해서 관리

#### 연속 대시

연속 대시가 필요해서, 델리게이트를 이용해 몽타주가 끝나기 전에 대시를 하면 연속 대시가 가능하도록 변경했다.

태그를 이용해 **Dash로 Dash를 캔슬**한 뒤 다시 Dash를 재사용하도록 유도했다.

```
Dash 실행 중 (State.Dash)
  └─ 다시 Dash 입력
       └─ 태그로 기존 Dash 캔슬
            └─ Dash 재활성화 → 연속 대시
```

### Fire

- 몽타주 = Fire Speed 기반
- 탄 소모는 GE의 `SetByCaller`로 반영
- 탄 소모 체크는 cpp에서 `0 > CurAmmo` 로 직접 체크

---

## 정리

- `SetByCaller`는 `Data.*` 태그로 넣어야 경고가 없음 (필수 아님)
- Reload / Dash / Fire 모두 몽타주 종료 델리게이트 기준으로 EndAbility + StateTag 정리
- 대시 스택은 GE로 분리, 연속 대시는 태그로 Dash→Dash 캔슬 후 재사용
- 탄 소모는 GE SetByCaller + cpp 잔탄 체크
