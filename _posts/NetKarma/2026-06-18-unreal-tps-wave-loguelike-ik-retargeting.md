---
title: "TPS 개발 : 블렌더 FBX → 언리얼 IK 리타게팅 이슈 정리"
date: 2026-06-18 00:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, TroubleShooting, TroubleShooting-UnrealEngine, UnrealEngine-IKRig, UnrealEngine-IKRetargeting, UnrealEngine-SkeletalMesh, UnrealEngine-RootMotion, Blender-FBX, NetKarma]
description: "블렌더 FBX 임포트 후 IK 리타게팅 시 팔다리 꺾임, 시퀀스에서 하늘로 날아가는 이슈 해결"
---

# TIL - 블렌더 FBX → 언리얼 IK 리타게팅 이슈

블렌더에서 만든 FBX 스켈레탈 메시를 언리얼에 임포트하고 IK Rig, IK Retargeting을 설정하는 과정에서 두 가지 이슈가 있었다.

---

## 이슈 1 — A포즈 / T포즈 차이로 팔다리가 기괴하게 꺾임

### 원인

소스 스켈레톤과 타겟 스켈레톤의 기준 포즈가 달라서 발생한다.
리타게터가 포즈 기준으로 회전값을 매핑하는데, 기준 포즈가 다르면 팔다리가 의도와 다른 방향으로 꺾인다.

![스크린샷 2026-06-26 101110.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20101110.png)
<왼쪽 : A포즈 / 오른쪽 : T포즈>

### 해결

IK Retargeter 에디터에서 소스/타겟 각각 포즈를 맞춰준다.

```
IK Retargeter → Edit Pose
소스와 타겟의 포즈를 동일한 기준(A포즈 or T포즈)으로 맞추기
```

![스크린샷 2026-06-26 101214.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20101214.png)
<포즈 매칭>

---

## 이슈 2 — 시퀀스로 내보내면 캐릭터가 하늘로 날아감

### 증상

- 시퀀스 재생 시 캐릭터가 엄청난 속도로 위로 날아다님
- 실제 루트 본의 이동량은 작은데 메시만 크게 휘적거림

![녹음 2026-06-26 101943.gif](/assets/netkarma/%EB%85%B9%EC%9D%8C%202026-06-26%20101943.gif)

### 원인

스켈레탈 메시의 루트 본 스케일이 **100**으로 설정되어 있었다.
루트 본 스케일이 100이면 루트의 미세한 이동도 100배로 증폭되어 적용된다.

![스크린샷 2026-06-26 102148.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20102148.png)

### 시도한 방법 (실패)

루트 본 스케일을 에디터에서 직접 1로 바꾸려 했으나, 값을 수정해도 자동으로 `99999.87` 같은 값으로 되돌아갔다.

### 해결

스켈레탈 메시 에디터에서 **Skeletal Edit 모드**로 진입한 뒤 모든 본의 스케일을 직접 `1, 1, 1`로 변경했다.

```
스켈레탈 메시 에디터 → 상단 Skeleton Tree 우클릭 → Edit Selected Bone
또는 Skeletal Edit 모드에서 전체 본 스케일 1, 1, 1로 설정
```

이후 리타게팅이 정상적으로 동작했다.

![녹음 2026-06-26 102420.gif](/assets/netkarma/%EB%85%B9%EC%9D%8C%202026-06-26%20102420.gif)
<무슨 사내아이처럼 당차고 힘차게 달린다..>

---

## 주의 — Enable Root Motion 반드시 끄기



리타게팅된 애니메이션을 시퀀스에서 사용할 때, **Enable Root Motion이 켜져 있으면 이슈 2와 동일한 현상이 다시 발생**한다.

![스크린샷 2026-06-26 102513.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20102513.png)

---

## 정리

| 이슈 | 원인 | 해결 |
|---|---|---|
| 팔다리 기괴하게 꺾임 | A포즈 / T포즈 기준 불일치 | IK Retargeter에서 포즈 직접 맞추기 |
| 시퀀스에서 하늘로 날아감 | 루트 본 스케일 100 | Skeletal Edit 모드에서 전체 본 스케일 1,1,1로 변경 |
| 날아가는 현상 재발 | Enable Root Motion 활성화 | 애니메이션 에셋에서 Enable Root Motion 체크 해제 |


## 후일담

클러드야~ Unreal Engine의 기본 프로젝트 마네퀸처럼 본 재생성 부탁해~ ^^

애니메이션은 정상적으로 들어왔지만, 리타게팅 자체가 마네퀸과 매칭되지 않아 치마가 흔들리지 않는다거나, 달리는 자세가 어정쩡하다거나 하는 이슈가 사라지지 않았다.
