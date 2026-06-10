---
title: "트러블슈팅 : 언리얼 착지 애니메이션 시 캐릭터 크기 변형 문제"
date: 2026-06-05 17:00:00 +0900
categories: [TroubleShooting, TroubleShooting-UnrealEngine]
tags: [TroubleShooting, UnrealEngine-Animation]
description: "랜드 애니메이션 시 캐릭터 작아짐 이슈"
---

# 언리얼 애니메이션 디버깅

---

## 오류

점프 착륙을 할 때마다 캐릭터가 작아졌다가 커지는 오류가 발생
![land issue.png](/assets/img/troubleshooting-unreal-land-animiation/land%20issue.png)
```<하체 비만인가보다>```

## Locomotion

![locomotion.png](/assets/img/troubleshooting-unreal-land-animiation/locomotion.png)


## Additive 란?

`Additive Animation`은 애니메이션 자체를 재생하는 것이 아니라

```txt~~~~
현재 포즈 + 애니메이션 변화량(Delta)
```

을 적용하는 방식

주로

* 에임 오프셋(Aim Offset)
* 상체만 움직이는 애니메이션
* 반동(Recoil)

등에 사용

이번 문제에서는 `MM_Land` 애니메이션이 `Additive Anim Type : Local Space` 로 설정됨

착지 애니메이션 재생 시 캐릭터가 순간적으로 작아졌다가 다시 원래 크기로 돌아오는 현상이 발생하였으며,

`Additive Anim Type : No Additive` 로 변경하자 해당 현상이 사라졌다.

따라서 이번 이슈는 **Land 애니메이션이 Additive 방식으로 적용되면서 발생한 문제**로 판단하였다.

## Automatic Rule Based on Sequence Player

애니메이션이 다 끝나기도 전에 착지를 하게 되어 Land 애니메이션이 일절 보이지 않았다.

`Automatic Rule Based on Sequence Player: false -> true`

설정 시

`Land 재생 완료 후 Idle 전환`

이 정상적으로 수행되는 것을 확인하였다.

---

## 결론

이번 문제는 두 가지 설정이 복합적으로 작용한 결과였다.

### 1. Additive 설정 문제

```txt
MM_Land
Additive Anim Type : Local Space
```

↓

착지 시 캐릭터가 순간적으로 작아지는 현상 발생

↓

```txt
No Additive
```

로 변경하여 해결

### 2. State 전환 설정 문제

```txt
Land → Idle
```

전환이 정상적으로 수행되지 않음

↓

```txt
Automatic Rule Based on Sequence Player
```

활성화

↓

Land 애니메이션 재생 후 Idle 상태로 자연스럽게 전환

### 최종적으로

```txt
Additive Anim Type : No Additive
Automatic Rule Based on Sequence Player : True
```

설정을 적용하여

* 캐릭터가 작아지는 현상 제거
* Land 애니메이션 정상 재생
* Idle 상태 자연 전환

을 모두 해결할 수 있었다.
