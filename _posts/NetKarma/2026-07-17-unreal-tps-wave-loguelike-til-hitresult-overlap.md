---
title: "TPS 개발 : TIL — ImpactPoint 오버랩 함정과 스테일 스테이트 버그"
date: 2026-07-17 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, TroubleShooting, UnrealEngine-HitResult, UnrealEngine-Collision]
description: "탄착이 가끔 원점으로 날아가던 버그 — ImpactPoint의 오버랩 함정과 루프 스코프가 어긋난 stale state"
---

# TPS 개발 - TIL: ImpactPoint 오버랩 함정

사격 시 가끔 탄착 이펙트/트레일이 `(0,0,0)`(월드 원점)으로 날아가는 버그를 잡았다. 원인이 두 개 겹쳐 있었다.

---

## 1. `FHitResult::ImpactPoint`는 Overlap 히트에 유효하지 않다

### 증상

원점 방향을 보고 있을 때 더 자주 관측됐다.

### 원인

언리얼 공식 문서에 `ImpactPoint`는 **"Overlap 히트엔 유효하지 않다"** 고 명시돼 있다. [적 히트박스(`TakuHitBox` 채널)]({% post_url NetKarma/2026-06-24-unreal-tps-wave-loguelike-til-hitresult %})가 Overlap 응답이라, 근접/좁은 반경 스윕이 시작부터 겹쳐있으면(`bStartPenetrating`) `ImpactPoint`가 `(0,0,0)`으로 남는 경우가 있었다.

이 값이 임팩트 VFX 위치, 트레일 끝점, 심지어 2단계 트레이스(카메라→총구)의 **트레이스 목적지 좌표 자체**로까지 흘러 들어가서, 조준 방향과 무관하게 실제로 원점을 향해 트레이스가 날아가는 결과를 만들었다.

### 교훈

Overlap 판정에서 위치가 필요하면 `ImpactPoint` 대신 **`Hit.Location`(스윕 셰이프 중심)** 을 써야 한다. 이건 오버랩이든 블록이든 항상 유효하다.

"가끔 좌표가 이상하다"는 버그는 조사할 때 방향성/재현조건("원점 방향을 보면 더 자주")을 꼭 물어봐야 진짜 원인(트레이스 목적지 자체가 오염됨)까지 도달할 수 있었다.

---

## 2. 루프 밖에 선언된 상태 변수가 "이전 반복의 값"을 새어들게 함

### 증상

위와 별개로, 샷건/연사 라이플처럼 **펠릿이 여러 개인 무기**로 쏠 때만 가끔 탄착점이 원점에 박혔다.

### 원인

`bHasHit`이 펠릿 루프 **바깥**에 선언되어 있었고, `LastHit`은 매 펠릿마다 **루프 안에서 새로** 선언됐다(기본생성 = zero). 펠릿 A가 맞고 펠릿 B가 완전히 빗나가면, `bHasHit`은 A 때의 `true`가 그대로 남아있는데 `LastHit`은 B 이터레이션에서 한 번도 안 채워진 zero 상태 — "맞았다는 플래그"와 "맞은 위치"가 서로 다른 스코프 생명주기를 가져서 어긋난 전형적인 stale-state 버그였다.

```
bHasHit (루프 바깥, 펠릿 전체 공유)
LastHit (루프 안, 펠릿마다 새로 생성)

펠릿 A: 명중 → bHasHit = true, LastHit = A위치
펠릿 B: 빗나감 → bHasHit는 그대로 true, LastHit는 새로 zero로 생성됨

결과: "맞았다"고 판단하는데 위치는 zero
```

### 교훈

"이 값이 true/false냐"와 "그 값이 가리키는 데이터"는 반드시 **같은 스코프, 같은 생명주기**로 묶어서 관리해야 한다. 특히 루프 안에서 "이번 반복 전용" 변수와 "루프 전체를 관통하는" 변수를 섞어 쓸 때 이런 실수가 잘 난다.

---

## 정리

- Overlap 히트에서 위치가 필요하면 `ImpactPoint` 대신 `Hit.Location`
- 상태 플래그와 그 플래그가 가리키는 데이터는 같은 스코프에 묶어서 선언
- 두 버그 모두 같은 증상(탄착 원점행)으로 나타났지만 원인은 완전히 별개였다 — 증상이 같다고 원인도 하나라고 단정하지 말 것
