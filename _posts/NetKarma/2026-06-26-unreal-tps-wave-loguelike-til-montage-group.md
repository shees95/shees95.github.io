---
title: "TPS 개발 : 몽타주 그룹 분리 — 동시 실행 충돌 해결"
date: 2026-06-26 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, TroubleShooting, TroubleShooting-UnrealEngine, UnrealEngine-AnimMontage, UnrealEngine-SlotGroup, UnrealEngine-GAS]
description: "여러 몽타주 동시 실행 시 막히는 문제를 슬롯 그룹 분리로 해결"
---

# TIL - 몽타주 그룹 분리

---

## 증상

몽타주가 한 번에 여러 개 실행되면 서로 막혀서 재생되지 않았다.

대시 중에 발사를 하거나, 발사 중에 재장전을 하면 한쪽 몽타주가 무시되는 식이었다.

---

## 원인

기본적으로 몽타주는 같은 **슬롯 그룹**에 속하면 동시에 재생될 수 없다.

하나의 슬롯에서는 하나의 몽타주만 재생되기 때문에, 같은 그룹에 묶여 있으면 나중에 실행된 몽타주가 앞의 것을 밀어내거나 막힌다.

---

## 해결

신체 부위 기준으로 슬롯 그룹을 쪼갰다.

| 슬롯 그룹 | 할당 몽타주 |
|---|---|
| FullBody | Dash |
| UpperBody | Fire, Reload |

- **Dash**는 전신 동작이라 `FullBody` 슬롯에 배치
- **Fire / Reload**는 상체만 움직이면 되므로 `UpperBody` 슬롯에 배치

이렇게 분리하니 대시를 하면서 동시에 발사/재장전 몽타주가 충돌 없이 재생되는 것을 확인했다.

---

## 정리

- 같은 슬롯 그룹의 몽타주는 동시 재생 불가
- 동작 부위에 따라 슬롯 그룹을 나누면(FullBody / UpperBody) 동시 실행 가능
- AnimGraph에서 각 슬롯을 블렌딩해 최종 포즈를 합성
