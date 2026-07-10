---
title: "TPS 개발 : 증강 시스템 (1) — GE와 EffectSet 연결"
date: 2026-07-10 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-GAS, UnrealEngine-GameplayEffect, UnrealEngine-Augment, UnrealEngine-DataAsset]
description: "증강 시스템의 GE 파트 — DA_NKMAugmentEffectSet에 이펙트 ID와 GE를 묶어 패시브 증강을 제공하는 구조"
---

# TPS 개발 - 증강 시스템 (1)

로그라이크의 핵심인 **증강(Augment) 시스템**을 만들기 시작했다. 이 시스템은 GE, GA, 데이터 파이프라인이 전부 얽혀 있어서 통째로 정리하면 너무 길어지니, GE → GA → 전체 흐름 순서로 나눠서 적는다. 이 시스템 하나 만드는 데 **일주일**이 걸렸다.

---

## 필요한 GE 먼저 만들기

증강 하나하나는 결국 GameplayEffect로 표현된다. 먼저 증강에 필요한 GE를 만든다. 데미지 증가, 이동속도 증가처럼 일반적인 어트리뷰트 변경 GE와 다를 게 없다.

---

## DA_NKMAugmentEffectSet — ID와 GE를 묶는 곳

`Data\DA_NKMAugmentEffectSet`이라는 DataAsset이 증강 시스템의 중심이다. 여기서 **이펙트 ID(EID)와 실제 GE 에셋을 묶어준다.**

```
DA_NKMAugmentEffectSet
  └─ EID (예: "AUG_Damage_Up_1")
       └─ 연결된 GE 에셋 (예: GE_Augment_DamageUp1)
```

CSV 테이블에서는 증강을 **문자열 ID**로만 다루고, 실제로 어떤 GE 에셋을 실행할지는 이 DataAsset이 매핑을 담당한다. CSV와 uasset 사이의 다리 역할이다.

---

## 정책은 게임 시작 시 자동 설정

이렇게 GE와 EID를 연결해두고 게임을 한 번 실행하면, `AugmentManagerSubsystem`의 `Initialize`에서 **EffectSet 데이터와 CSV 데이터 테이블을 읽어와 정책을 설정**해준다.

```
게임 시작
  └─ AugmentManagerSubsystem::Initialize
       └─ CSV 데이터 테이블 읽기
            └─ EffectSet에서 EID ↔ GE 매핑 읽기
                 └─ 각 GE의 정책(Duration, Stack 등) 자동 설정
```

수동으로 GE마다 세팅을 반복할 필요 없이, 정책이 CSV + DataAsset 조합으로 한 번에 잡힌다.

---

## Passive와 PID의 차이

- **Passive**: GE를 만들고 DA에 EID만 연결해두면 **"증강 줄 준비가 되었다"**고 판단해서 즉시 제공 가능한 상태가 된다. EID가 연결되어 있지 않으면 해당 패시브는 제공되지 않는다.
- **PID(패시브)**: 증강을 소유하는 **즉시 GE가 영구 발동**된다. 별도의 트리거 조건 없이 소유=발동이다.

```
Passive 증강
  └─ GE 준비 + EID 연결
       └─ 제공 가능 상태 (트리거는 나중에 GA가 담당)

PID (즉시발동 패시브)
  └─ 소유 즉시 GE 영구 적용
```

이 EID 연결 여부가 사실상 "이 증강이 실제로 게임에 나올 수 있느냐"를 결정하는 스위치다. 연결을 깜빡하면 증강이 목록에 있어도 절대 나오지 않는다.

---

## 오늘 정리

- 증강 하나 = GE 하나, `DA_NKMAugmentEffectSet`에서 **EID ↔ GE**로 매핑
- `AugmentManagerSubsystem::Initialize`가 CSV + EffectSet을 읽어 **정책을 자동 설정**
- **Passive**는 EID 연결만 해두면 제공 준비 완료 (연결 안 되면 증강 자체가 안 나옴)
- **PID**는 소유 즉시 GE가 영구 발동

다음은 트리거가 있는 능동형 증강, GA 파트다.
