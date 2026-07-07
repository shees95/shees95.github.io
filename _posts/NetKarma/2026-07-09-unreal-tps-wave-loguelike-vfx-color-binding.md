---
title: "TPS 개발 : VFX 색상 — Material Attribute Binding"
date: 2026-07-09 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-VFX, UnrealEngine-Niagara, UnrealEngine-Material, TroubleShooting]
description: "재료별 VFX 색상을 RGBA 테이블 값으로 런타임에 바꾸기까지 — 유저 파라미터와 머테리얼 변수를 지나 Attribute Binding에 도달한 과정"
---

# TPS 개발 - VFX 색상 바인딩

[특수탄 VFX]({% post_url NetKarma/2026-07-07-unreal-tps-wave-loguelike-weapon-vfx %})까지는 만들었는데, 효과에 따라 이펙트 색상도 달라야 했다. 재료가 다르면 특수탄 색깔도 달라야 한다는 건데, 이걸 붙이는 접근법 자체가 까다로웠다.

---

## 데이터 준비 — item 테이블에 RGBA 통합

먼저 데이터 쪽부터 정리했다. item 테이블에 **RGBA 값을 통합**시켜서, 재료마다 자기 색상을 갖게 했다.

```
Item 테이블
  ├─ ID, 이름, ...
  └─ Color (R, G, B, A)   ← 신규
```

이제 재료에 따라 VFX의 `LinearColor`를 머테리얼에 적용하기만 하면 되는 줄 알았다. 실제로는 접근법을 세 번 갈아엎었다.

---

## 시도 1 — 이니셜라이즈에 유저 파라미터

나이아가라 이미터의 **이니셜라이즈 모듈**에 유저 파라미터를 하나 만들어서 붙여봤다.

```
Niagara Emitter → Initialize Particle
  └─ User Parameter (Color) 연결
```

결과: **아무 효과가 없었다.** 파라미터는 만들어졌는데 실제 머테리얼 렌더링에 반영되지 않았다.

---

## 시도 2 — 머테리얼 변수로 런타임 변경

다음엔 머테리얼 자체에 변수를 만들고, 그걸 런타임에 바꾸는 방향으로 갔다.

문제는 **다른 효과들도 함께 변경**돼버린 것이다. 머테리얼 인스턴스를 공유하는 다른 파티클/이펙트가 같이 물들어버렸다. 머테리얼 자체를 건드리는 방식이라 범위가 너무 넓었다.

---

## 해결 — Material Attribute Binding

좀 더 찾아보니, 나이아가라 렌더러 쪽에 **Material Attribute Binding**이라는 항목이 있었다. 이곳에서 **머테리얼에서 생성된 벡터 컬러 파라미터를 파티클 단위로 바인딩**할 수 있었다.

```
Niagara Renderer
  └─ Material Attribute Binding
       └─ 머테리얼의 Vector(Color) 파라미터
            ↔ 유저 파라미터(VFX Color) 연결
```

여기에 유저 파라미터로 만든 VFX 컬러를 연결하니, **색상이 파티클 단위로 정확히 반영**됐다. 다른 이펙트에 번지지도 않고, 이니셜라이즈 모듈처럼 무시되지도 않는, 딱 원하던 지점이었다.

---

## 왜 두 번 헛발질했는지

돌이켜보면 각 계층의 역할이 달랐다.

| 시도 | 계층 | 문제 |
|---|---|---|
| 이니셜라이즈 유저 파라미터 | 파티클 스폰 시점 값 | 머테리얼 렌더링과 연결 안 됨 |
| 머테리얼 변수 직접 수정 | 머테리얼 에셋 자체 | 이 머테리얼을 쓰는 모든 것에 영향 |
| Material Attribute Binding | 렌더러 ↔ 머테리얼 파라미터 연결 지점 | 파티클별로 독립적으로 반영 ✅ |

**파티클마다 다른 색을 내려면, 파티클 데이터와 머테리얼 파라미터를 잇는 바인딩 지점**을 써야 했다. 앞의 두 시도는 각각 "머테리얼과 무관한 값"과 "전역에 영향을 주는 값"이라 원하는 결과가 안 나온 것이다.

---

## 오늘 정리

- item 테이블에 **RGBA 컬러**를 통합해 재료별 색상 데이터 확보
- 이니셜라이즈 유저 파라미터 → 머테리얼에 미반영
- 머테리얼 변수 직접 수정 → 다른 이펙트까지 오염
- **Material Attribute Binding**으로 유저 파라미터를 머테리얼 벡터 파라미터에 연결 → 파티클별 색상 정상 반영
