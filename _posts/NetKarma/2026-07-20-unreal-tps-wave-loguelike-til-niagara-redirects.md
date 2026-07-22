---
title: "TPS 개발 : TIL — 나이아가라 완료 이벤트와 되살아난 CoreRedirects"
date: 2026-07-20 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, TroubleShooting, UnrealEngine-Niagara, UnrealEngine-CoreRedirects]
description: "OnSystemFinished가 안 쏘일 수 있다는 것, 그리고 지웠던 ClassRedirects가 머지로 되살아난 사고"
---

# TPS 개발 - TIL: 나이아가라 완료 이벤트와 되살아난 리다이렉트

풀링 리소스 반납과 설정 파일, 서로 무관해 보이는 두 트러블을 겪었다.

---

## 1. 나이아가라 완료 이벤트(`OnSystemFinished`)는 신뢰 100%가 아니다

### 증상

[총알/임팩트 풀링]({% post_url NetKarma/2026-07-07-unreal-tps-wave-loguelike-weapon-vfx %}) 액터가 가끔 반납이 안 돼서 계속 새로 스폰됐다(누수 의심).

### 원인

풀 반납이 전적으로 `UNiagaraComponent::OnSystemFinished` 델리게이트에 의존했는데, 이 이벤트는 나이아가라 에셋 자체의 Completion 설정(Life Cycle Mode, Loop Behavior, Inactive Response 등)에 따라 **아예 안 쏠 수도 있다.**

### 교훈/조치

외부 이벤트에 전적으로 의존하는 리소스 반납 로직에는 **타임아웃 안전장치**를 이중으로 걸어두는 게 안전하다(이번엔 3초 강제 반납 타이머 추가).

나이아가라 쪽에서도 `Life Cycle Mode: Self` + `Loop Behavior: Once` + `Inactive Response: Continue`로 "자체적으로도 언젠가는 끝나는 경로"를 보장해주는 게 좋다 — 단, Loop Duration이 코드가 계산하는 실제 지속시간(`Distance/Speed`)보다 짧으면 이펙트가 조기 종료될 수 있으니 여유값으로 잡아야 한다.

---

## 2. CoreRedirects는 "머지로 되살아날 수 있다" — 두 번이나 겪음

### 증상

- **이전 세션**: [패키징/쿡 실패]({% post_url NetKarma/2026-07-15-unreal-tps-wave-loguelike-packaging-build-issues %}), `LogCoreRedirects: Error: AddRedirect(...) found conflicting redirects`.
- **이번 세션**: `GA_NKMTID0`를 부모로 하는 BP가 에디터를 껐다 켤 때마다 부모가 `GA_NKMTID1`로 자동으로 바뀌어 있었다.

### 원인

둘 다 `Config/DefaultEngine.ini`의 `+ClassRedirects` 항목이 원인이었다. 특히 이번 건은 git 히스토리를 보니 **팀원이 이미 한 번 삭제했던 줄이 이후 머지 과정에서 부활**한 것 — 리다이렉트가 남아있으면 로드/컴파일 때마다 조용히 참조를 다시 써버려서, 저장까지 하면 잘못된 참조가 영구적으로 박힌다.

```
팀원 A: +ClassRedirects 줄 삭제, 커밋
팀원 B: 오래된 브랜치에서 머지 → 삭제됐던 줄이 다시 합쳐짐
결과: 리다이렉트 부활 → 로드할 때마다 참조가 조용히 재작성됨
```

### 교훈

텍스트 기반 설정 파일(.ini)이라도 머지 충돌 해결 시 "삭제됐던 줄이 되살아나는" 실수는 흔하다. 이상한 자동 변경(부모 클래스가 저절로 바뀐다, 특정 클래스가 자꾸 딴 걸로 로드된다) 증상을 보면 `DefaultEngine.ini`의 Redirects부터, `git log -S'키워드'`로 히스토리를 훑어보는 게 빠르다.

그리고 리다이렉트를 지운 뒤에도 **이미 저장된 에셋엔 잘못된 참조가 그대로 남아있으므로** 수동으로 한 번 재지정+저장이 필요하다. 설정 수정이 과거 데이터를 소급 치유하진 않는다.

---

## 정리

- 나이아가라 `OnSystemFinished`만 믿지 말고 타임아웃 안전장치를 이중으로
- CoreRedirects는 머지로 부활할 수 있음 — 이상한 자동 변경을 보면 DefaultEngine.ini부터 의심
- 리다이렉트를 지워도 이미 저장된 에셋은 소급 치유되지 않으니 수동 재지정 필요
