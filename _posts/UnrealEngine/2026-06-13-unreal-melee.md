---
title: "언리얼 밀리 공격 구현"
date: 2026-06-13 15:00:00 +0900
categories: [UnrealEngine, UnrealEngine-GAS]
tags: [UnrealEngine]
description: "밀리 공격 구현"
---

# 언리얼에서 근접 공격은 어떻게 구현되어있을까?

---

## 기본프로젝트 까보기

### 공격 입력은 어떻게 받는가?

#### 1. 기본 ThirdPerson 의 컨트롤러 상속 및 자산이용
Move와 Look은 기본 ThirdPerson을 상속받았고, Input Action들도 기본 자산을 이용    
Combat 관련 입력 (Combo, Charge) 같은건 Combat Input에서 따로 관리    

캐릭터에서 InputAction 관리 및 바인딩  
컨트롤러에서 IMC 및 매핑 관리  




### Third Person Base 에서 Combat Map의 근접공격 구현 내용 찾아보기

#### 공격
클릭 시 주는 공격 명령어

`CombatCharacter.cpp`
```c++
void ACombatCharacter::ComboAttack()
{
	// raise the attacking flag
	bIsAttacking = true;

	// reset the combo count
	ComboCount = 0;

	// notify enemies they are about to be attacked
	NotifyEnemiesOfIncomingAttack();

	// play the attack montage
	if (UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance())
	{
		const float MontageLength = AnimInstance->Montage_Play(ComboAttackMontage, 1.0f, EMontagePlayReturnType::MontageLength, 0.0f, true);

		// subscribe to montage completed and interrupted events
		if (MontageLength > 0.0f)
		{
			// set the end delegate for the montage
			AnimInstance->Montage_SetEndDelegate(OnAttackMontageEnded, ComboAttackMontage);
		}
	}

}
```

#### 전방 적 탐지

`CombatCharacter.cpp`
```c++
void ACombatCharacter::NotifyEnemiesOfIncomingAttack()
{
  // 탐지 리스트
	TArray<FHitResult> OutHits;
  // 공격 범위 지정
	const FVector TraceStart = GetActorLocation();
	const FVector TraceEnd = TraceStart + (GetActorForwardVector() * DangerTraceDistance);

  // 충돌 범위 지정
	FCollisionObjectQueryParams ObjectParams;
	ObjectParams.AddObjectTypesToQuery(ECC_Pawn);

  // Sphere Collision 으로 체크
	FCollisionShape CollisionShape;
	CollisionShape.SetSphere(DangerTraceRadius);
  // 본인 제외
	FCollisionQueryParams QueryParams;
	QueryParams.AddIgnoredActor(this);

  // World의 콜리전 함수사용
	if (GetWorld()->SweepMultiByObjectType(OutHits, TraceStart, TraceEnd, FQuat::Identity, ObjectParams, CollisionShape, QueryParams))
	{
		// 맞은 놈들 반복
		for (const FHitResult& CurrentHit : OutHits)
		{
			// 때릴 수 있는 놈인지 인터페이스 보유 체크
			ICombatDamageable* Damageable = Cast<ICombatDamageable>(CurrentHit.GetActor());
      
      // 때릴 수 있으면 데미지 부여
			if (Damageable)
			{
				Damageable->NotifyDanger(GetActorLocation(), this);
			}
		}
	}
}
```

---

## AnimNotify로 함수 호출하기

---

몽타주에서 함수를 직접 호출하는 게 아니다.  
`UAnimNotify`를 상속한 C++ 클래스를 만들고, `Notify()` 안에서 호출하면  
컴파일 후 몽타주 노티파이 목록에 자동으로 등장한다.

```c++
// ANS_DoAttackTrace.h
UCLASS()
class UANS_DoAttackTrace : public UAnimNotify
{
    GENERATED_BODY()
public:
    virtual void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation) override;
};
```

```c++
// ANS_DoAttackTrace.cpp
void UANS_DoAttackTrace::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation)
{
    AActor* Owner = MeshComp->GetOwner();
    if (ICombatAttacker* Attacker = Cast<ICombatAttacker>(Owner))
    {
        Attacker->DoAttackTrace();
    }
}
```

### 인터페이스와 조합하는 이유

`DoAttackTrace()`를 `ICombatAttacker` 인터페이스에 선언하고,  
노티파이에서 `Cast<ICombatAttacker>` 후 호출한다.

```yaml
장점:
  노티파이 클래스 하나로 캐릭터, 적 등 구현체가 달라도 공유 가능
  노티파이가 구체 클래스를 알 필요 없음
```

---

## 몽타주 연결 방식

---

경로 문자열로 에셋을 찾는 게 아니라 `UPROPERTY`로 변수를 선언하고,  
블루프린트 디테일 패널에서 직접 할당한다.

```c++
// CombatCharacter.h
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
UAnimMontage* ComboAttackMontage;
```

```c++
// 호출
PlayAnimMontage(ComboAttackMontage);
```

---

## 데미지 흐름

---

```
AnimNotify (공격 판정 타이밍)
    └→ DoAttackTrace()      구체 스윕으로 히트 감지
        └→ ApplyDamage()    ICombatDamageable 인터페이스로 호출
            └→ TakeDamage() HP 차감
                └→ HandleDeath()  HP <= 0
```

### CombatDamageable 인터페이스

피격 가능한 모든 액터가 구현하는 인터페이스.  
공격자는 대상이 캐릭터인지 오브젝트인지 몰라도 데미지를 전달할 수 있다.

```c++
UINTERFACE()
class UCombatDamageable : public UInterface { GENERATED_BODY() };

class ICombatDamageable
{
    GENERATED_BODY()
public:
    virtual void ApplyDamage(float DamageAmount, AActor* DamageCauser) = 0;
};
```

### CombatAttacker 인터페이스

공격 판정 로직을 인터페이스로 분리.  
노티파이가 이 인터페이스만 알면 어떤 액터든 공격 판정을 위임할 수 있다.

```c++
UINTERFACE()
class UCombatAttacker : public UInterface { GENERATED_BODY() };

class ICombatAttacker
{
    GENERATED_BODY()
public:
    virtual void DoAttackTrace() = 0;
};
```
