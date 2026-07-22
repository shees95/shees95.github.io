---
title: "TPS 개발 : 이번 세션 TIL 모음 — HitResult, 스테일 스테이트, 리다이렉트, 콤보/리워드 설계"
date: 2026-07-17 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, TroubleShooting, UnrealEngine-HitResult, UnrealEngine-Niagara, UnrealEngine-CoreRedirects, UnrealEngine-MVVM]
description: "ImpactPoint 오버랩 함정, 스테일 스테이트 버그, 나이아가라 완료 이벤트 신뢰성, CoreRedirects 재발, 콤보/리워드 데이터 설계, MVVM 경계"
---

# TPS 개발 - 이번 세션 TIL 모음

이번 세션에 겪은 것들을 종류별로 묶어서 정리한다. 버그 세 개, 설정 파일 트러블 하나, 설계 결정 두 개다.

---

## 1. `FHitResult::ImpactPoint`는 Overlap 히트에 유효하지 않다

### 증상

사격 시 가끔 탄착 이펙트/트레일이 `(0,0,0)`(월드 원점)으로 날아가는 것처럼 보였다. 원점 방향을 보고 있을 때 더 자주 관측됐다.

### 원인

언리얼 공식 문서에 `ImpactPoint`는 **"Overlap 히트엔 유효하지 않다"** 고 명시돼 있다. [적 히트박스(`TakuHitBox` 채널)]({% post_url NetKarma/2026-06-24-unreal-tps-wave-loguelike-til-hitresult %})가 Overlap 응답이라, 근접/좁은 반경 스윕이 시작부터 겹쳐있으면(`bStartPenetrating`) `ImpactPoint`가 `(0,0,0)`으로 남는 경우가 있었다.

이 값이 임팩트 VFX 위치, 트레일 끝점, 심지어 2단계 트레이스(카메라→총구)의 **트레이스 목적지 좌표 자체**로까지 흘러 들어가서, 조준 방향과 무관하게 실제로 원점을 향해 트레이스가 날아가는 결과를 만들었다.

### 교훈

Overlap 판정에서 위치가 필요하면 `ImpactPoint` 대신 **`Hit.Location`(스윕 셰이프 중심)** 을 써야 한다. 이건 오버랩이든 블록이든 항상 유효하다.

"가끔 좌표가 이상하다"는 버그는 조사할 때 방향성/재현조건("원점 방향을 보면 더 자주")을 꼭 물어봐야 진짜 원인(트레이스 목적지 자체가 오염됨)까지 도달할 수 있었다.

---

## 2. 루프 밖에 선언된 상태 변수가 "이전 반복의 값"을 새 반복에 새어들게 함

### 증상

위와 별개로, 샷건/연사 라이플처럼 **펠릿이 여러 개인 무기**로 쏠 때만 가끔 탄착점이 원점에 박혔다.

### 원인

`bHasHit`이 펠릿 루프 **바깥**에 선언되어 있었고, `LastHit`은 매 펠릿마다 **루프 안에서 새로** 선언됐다(기본생성 = zero). 펠릿 A가 맞고 펠릿 B가 완전히 빗나가면, `bHasHit`은 A 때의 `true`가 그대로 남아있는데 `LastHit`은 B 이터레이션에서 한 번도 안 채워진 zero 상태 — "맞았다는 플래그"와 "맞은 위치"가 서로 다른 스코프 생명주기를 가져서 어긋난 전형적인 stale-state 버그였다.

### 교훈

"이 값이 true/false냐"와 "그 값이 가리키는 데이터"는 반드시 **같은 스코프, 같은 생명주기**로 묶어서 관리해야 한다. 특히 루프 안에서 "이번 반복 전용" 변수와 "루프 전체를 관통하는" 변수를 섞어 쓸 때 이런 실수가 잘 난다.

---

## 3. 나이아가라 완료 이벤트(`OnSystemFinished`)는 신뢰 100%가 아니다

### 증상

[총알/임팩트 풀링]({% post_url NetKarma/2026-07-07-unreal-tps-wave-loguelike-weapon-vfx %}) 액터가 가끔 반납이 안 돼서 계속 새로 스폰됐다(누수 의심).

### 원인

풀 반납이 전적으로 `UNiagaraComponent::OnSystemFinished` 델리게이트에 의존했는데, 이 이벤트는 나이아가라 에셋 자체의 Completion 설정(Life Cycle Mode, Loop Behavior, Inactive Response 등)에 따라 **아예 안 쏠 수도 있다.**

### 교훈/조치

외부 이벤트에 전적으로 의존하는 리소스 반납 로직에는 **타임아웃 안전장치**를 이중으로 걸어두는 게 안전하다(이번엔 3초 강제 반납 타이머 추가). 나이아가라 쪽에서도 `Life Cycle Mode: Self` + `Loop Behavior: Once` + `Inactive Response: Continue`로 "자체적으로도 언젠가는 끝나는 경로"를 보장해주는 게 좋다 — 단, Loop Duration이 코드가 계산하는 실제 지속시간(`Distance/Speed`)보다 짧으면 이펙트가 조기 종료될 수 있으니 여유값으로 잡아야 한다.

---

## 4. CoreRedirects는 "머지로 되살아날 수 있다" — 두 번이나 겪음

### 증상

- **이전 세션**: [패키징/쿡 실패]({% post_url NetKarma/2026-07-15-unreal-tps-wave-loguelike-packaging-build-issues %}), `LogCoreRedirects: Error: AddRedirect(...) found conflicting redirects`.
- **이번 세션**: `GA_NKMTID0`를 부모로 하는 BP가 에디터를 껐다 켤 때마다 부모가 `GA_NKMTID1`로 자동으로 바뀌어 있었다.

### 원인

둘 다 `Config/DefaultEngine.ini`의 `+ClassRedirects` 항목이 원인이었다. 특히 이번 건은 git 히스토리를 보니 **팀원이 이미 한 번 삭제했던 줄이 이후 머지 과정에서 부활**한 것 — 리다이렉트가 남아있으면 로드/컴파일 때마다 조용히 참조를 다시 써버려서, 저장까지 하면 잘못된 참조가 영구적으로 박힌다.

### 교훈

텍스트 기반 설정 파일(.ini)이라도 머지 충돌 해결 시 "삭제됐던 줄이 되살아나는" 실수는 흔하다. 이상한 자동 변경(부모 클래스가 저절로 바뀐다, 특정 클래스가 자꾸 딴 걸로 로드된다) 증상을 보면 `DefaultEngine.ini`의 Redirects부터, `git log -S'키워드'`로 히스토리를 훑어보는 게 빠르다.

그리고 리다이렉트를 지운 뒤에도 **이미 저장된 에셋엔 잘못된 참조가 그대로 남아있으므로** 수동으로 한 번 재지정+저장이 필요하다. 설정 수정이 과거 데이터를 소급 치유하진 않는다.

---

## 5. 콤보 카운트를 "타격 수"가 아니라 "공격 행동 수"로 정규화하기

### 요청

관통/펠릿(샷건)/근접 클리브/폭발 스플래시처럼 **한 번의 공격 행동이 여러 개별 히트를 만드는 경우**에도 콤보는 행동당 1회만 올라야 한다.

### 설계

`FNKMDamageApplicationParams`/`FNKMWeaponHitMessage`에 `bCountsForCombo` 플래그를 추가하고, 각 공격 GA(Fire/Melee/Throwable/PistolExplosion)가 **"이번 행동에서 이미 카운트했는가"** 를 자체적으로 추적(멤버 변수 or 로컬 변수)해서 첫 성공 히트에만 `true`를 실어 보냈다. Fire는 `ProcessFireHits()` 1회(펠릿+관통 전체) 스코프, Melee는 스윙 1회 스코프로 리셋 타이밍을 맞췄다.

### 교훈

"이벤트가 여러 번 발생하지만 논리적으로는 1회"인 상황은 이벤트 자체에 "이게 대표 이벤트인지" 플래그를 실어 보내는 게, 구독자마다 각자 중복제거 로직을 짜는 것보다 훨씬 안전하고 일관됐다(리스너가 여러 개 — 리워드 컴포넌트, 스탯 컴포넌트 — 여도 다 같은 판단을 공유).

---

## 6. 리워드 데이터는 "카테고리 + 연산방식(enum)"으로 분리하는 게 협업에 필수

### 배경

기획자(용수님)가 "스케일 합할 때 연산에 영향 끼치는 종류들을 이넘으로 정의해서 줘야 한다"고 명확히 요청했다. 기존엔 `bKillBonus/DealtCoin`, `bSandevistanBonus/SandevistanScale` 식으로 카테고리마다 필드가 반복되고, "이게 곱연산(스케일)에 들어가는지 그냥 가산인지"가 필드명(`XxxCoin` vs `XxxScale`)에만 암묵적으로 담겨 있었다.

### 리팩터

`ENKMCoinRewardCategory`(무엇) + `ENKMCoinScaleMode`(어떻게: Additive/InstantScale/DeferredMultiplier) + `FNKMCoinRewardEntry{Category, Mode, Value, ContributedCoin}` 하나로 통합했다. 새 보너스 종류 추가 시 enum 값 + `Entries.Add(...)` 한 줄이면 끝나는 구조다.

### 교훈

협업하는 기획자/디자이너 입장에서 "코드를 까보지 않고도 연산 방식을 알 수 있게" 만드는 것 자체가 요구사항이 될 수 있다 — enum으로 의도를 명시하는 게 단순 리팩터링을 넘어서 **커뮤니케이션 비용을 줄이는 설계**였다.

---

## 7. MVVM ViewModel은 게임로직 쪽 메시지 구조체를 직접 알면 안 됨

### 포인트

콤보 리워드 UI를 만들면서 `SetLiveState(const FNKMPendingCoinChangedMessage&, ...)`처럼 ViewModel이 리워드 컴포넌트의 메시지 구조체를 직접 받게 짰다가, "뷰모델이 리워드 컴포넌트 구조체를 이용한 게 맞나?"라는 질문에 걸렸다.

기존 코드(`FloatingDamageViewModel::SetPresentation(float, bool, bool)`, `WaveCoinViewModel::SetRewardState(int32, float, ...)`)를 보면 전부 **원시값만** 받는다 — 메시지 언패킹은 항상 Widget(View)의 책임이었다.

### 교훈

ViewModel은 "표시할 값"만 알아야 하고, 그 값이 어디서/어떤 구조체로 왔는지는 몰라야 한다. 그래야 게임로직 쪽 구조체가 나중에 바뀌어도(이번처럼 flat 필드 → entry 배열로 대격변해도) ViewModel/UI는 전혀 안 건드려도 된다 — 실제로 리워드 구조체를 통째로 갈아엎었는데 ViewModel은 코드 한 줄도 안 바뀌었다.

---

## 정리

| 항목 | 한 줄 요약 |
|---|---|
| ImpactPoint | Overlap 히트엔 무효 — `Hit.Location` 쓸 것 |
| Stale State | 플래그와 데이터는 같은 스코프·생명주기로 묶기 |
| 나이아가라 완료 이벤트 | 안 쏠 수 있음 — 타임아웃 안전장치 이중화 |
| CoreRedirects | 머지로 되살아남 — 지운 뒤엔 에셋도 수동 재지정 필요 |
| 콤보 카운트 | 이벤트에 "대표 여부" 플래그를 실어서 구독자 간 판단 통일 |
| 리워드 데이터 | 카테고리 + 연산방식 enum으로 분리 — 협업 비용 절감 |
| MVVM 경계 | ViewModel은 원시값만, 메시지 언패킹은 View 책임 |
