---
title: "TPS 개발 : GAS 총기 히트 판정 — SphereCollision"
date: 2026-06-24 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, TroubleShooting, TroubleShooting-UnrealEngine, UnrealEngine-GAS, UnrealEngine-SphereCollision, UnrealEngine-SweepMultiChannel]
description: "라인트레이스의 한계를 극복한 총기 히트 판정 구현"
---

# TIL - GAS 총기 히트 판정 구현

---

## 1. 두께 없는 히트 판정

우리는 3인칭 슈팅 웨이브 로그라이크 게임을 만드려고 한다.  
웨이브가 발생하면 많은 적들이 나타날텐데, 과연 이 적들을 쉽게 맞출 수 있을까?  

**게임 자체가 매우 피로할 것이다.**

때문에 총알의 두께를 구현하여 피격 판정을 러프하게 하여 사용자가 게임을 좀 더 수월하게 할 수 있게 만드려고 했다.

하지만 우리가 배운 `LineTraceByChannel`로는 부족했다.  
두께가 없는 일직선 탄이 날아갈테니까.  

그렇다면 어떤걸 사용해야할까?

## 해결

![스크린샷 2026-06-26 111329.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20111329.png)  
공을 던지면 되잖아?

```c++
  // Sweep 체크할 구체 생성
  FCollisionShape CollisionSphere = FCollisionShape::MakeSphere(AttackRadius);
  
  // Sweep 호출 후
  GetWorld()->SweepMultiByChannel(HitResults, Start, End, FQuat::Identity, ECC_Visibility, CollisionSphere, CollisionParams);
```

라인트레이스 사용하는 것과 거의 일치한다.  
그냥 이렇게 공을 굴려 충돌을 알려주면 된다.  

Multi 채널로 세팅함으로써 여러 충돌 또한 고려했다.

---

## 구현

```c++
  // 히트 판정 수집
  for (const FHitResult& Hit : HitResults)
  {
  DrawDebugSphere(GetWorld(), Hit.ImpactPoint, AttackRadius, 12, FColor::Yellow, false, 2.f);
  }
```

---

## 2. 피격 안됨 이슈

### 증상
안맞는다..  
몬스터에게 쏘면 통과되는데, 벽이나 땅은 통과가 되지 않았다.  

### 해결
검색과 AI의 따끔한 첨언끝에 알아낸 것은 피격의 채널인 VIsibility가 꺼져있다는 사실이다.  

![스크린샷 2026-06-26 112543.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20112543.png)

왜냐하면 기본 캐릭터 비지빌리티 판정이 Ignore이기 때문이다.

Collision Presets을 Custom으로 변경 후, Camera 쪽은 스프링 암이 관여해야하니 Visibility를 Overlap으로 설정.  

이러면 피격이 된다!

---

## 3. 피격 판정이 이상함 이슈..

### 증상
머리를 쐈는데 공중에 피격 판정이 되면서 피격 Bone 값을 None으로 리턴했다.  

### 해결

충돌 가능한 모든 컴포넌트들의 콜리전 채널을 각각 세팅해줬다.

![스크린샷 2026-06-26 112914.png](/assets/netkarma/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7%202026-06-26%20112914.png)

```
Root : Visibility [Overlap]
CapsuleComponent : Visibility [Ignore]
Skeletal Mesh : Visibility [Overlap]
```

---

본이 None 값이 나온 이유도 CapsuleComponent를 맞췄기 때문!!

## 4. 근데도 Bone이 Head판정이 나지 않는다..

### 증상
분명 메쉬를 맞는데 헤드 판정이 나지 않는다.

### 해결
몬스터 피직스 에셋이 없으면 콜리전 체크가 안된다.  
피직스 에셋 생성 후 다시 쏘니까 헤드 판정 완성!!

---

## 정리

- CollisionSphere을 생성해 Sweep 하여 충돌되는 Hit들을 처리하면 피격 판정 완성  
- Collision 채널의 Visibility 세팅 잘해야 함
- 피격 대상의 피직스에셋 필수
