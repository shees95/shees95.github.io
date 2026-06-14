---
title: "언리얼 밀리 공격 구현"
date: 2026-06-13 15:00:00 +0900
categories: [UnrealEngine, UnrealEngine-StarterProject]
tags: [UnrealEngine, UnrealEngine-Combat, UnrealEngine-AnimNotify, UnrealEngine-Interface, UnrealEngine-Montage, UnrealEngine-SphereTrace]
description: "AnimNotify + Interface 조합으로 근접 공격 구현"
---

# 언리얼 밀리 공격 구현

---

## 공격 흐름

근접 공격은 아래 파이프라인으로 동작한다.

```
입력
  └→ ComboAttack()        공격 플래그 설정 + 몽타주 재생
      └→ AnimNotify       몽타주의 특정 프레임에서 발동
          └→ DoAttackTrace()      구체 스윕으로 히트 감지
              └→ ApplyDamage()    ICombatDamageable 인터페이스로 호출
                  └→ TakeDamage() HP 차감
                      └→ HandleDeath()  HP <= 0
```

---

## 입력 처리

---

Move/Look은 기본 ThirdPerson을 상속받고, 공격 관련 입력(Combo, Charge)은 별도 InputAction으로 분리한다.

- 캐릭터: InputAction 바인딩 관리
- 컨트롤러: IMC(Input Mapping Context) 및 매핑 관리

---

## 몽타주 연결 방식

---

`UPROPERTY`로 선언하고 BP 디테일 패널에서 직접 할당한다.

```c++
// CombatCharacter.h
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
UAnimMontage* ComboAttackMontage;
```

할당된 몽타주 애니메이션을 호출만 하는 식.

```c++
// CombatCharacter.cpp
void ACombatCharacter::ComboAttack()
{
    bIsAttacking = true;
    ComboCount = 0;

    NotifyEnemiesOfIncomingAttack();

    if (UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance())
    {
        const float MontageLength = AnimInstance->Montage_Play(
            ComboAttackMontage, 1.0f,
            EMontagePlayReturnType::MontageLength, 0.0f, true);

        if (MontageLength > 0.0f)
        {
            AnimInstance->Montage_SetEndDelegate(OnAttackMontageEnded, ComboAttackMontage);
        }
    }
}
```

---

## AnimNotify로 함수 호출하기

---

몽타주에서 함수를 직접 호출할 수 없다.  
`UAnimNotify`를 상속한 C++ 클래스를 만들고 `Notify()`를 오버라이드하면  
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

노티파이가 `ICombatAttacker`만 알면 되기 때문에,  
캐릭터든 적이든 인터페이스를 구현한 액터라면 이 노티파이 하나로 공유할 수 있다.

---

## 인터페이스 설계

---

### ICombatAttacker

공격 판정 로직을 인터페이스로 분리한다.  
AnimNotify가 이 인터페이스만 알면 어떤 액터든 공격 판정을 위임할 수 있다.

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

### ICombatDamageable

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

---

## 공격 판정 구현

---

### 전방 적 사전 탐지

공격 시작 시점에 전방의 적을 미리 탐지해 대응할 수 있게 알린다.

```c++
void ACombatCharacter::NotifyEnemiesOfIncomingAttack()
{
    TArray<FHitResult> OutHits;

    const FVector TraceStart = GetActorLocation();
    const FVector TraceEnd = TraceStart + (GetActorForwardVector() * DangerTraceDistance);

    FCollisionObjectQueryParams ObjectParams;
    ObjectParams.AddObjectTypesToQuery(ECC_Pawn);

    FCollisionShape CollisionShape;
    CollisionShape.SetSphere(DangerTraceRadius);

    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(this);

    if (GetWorld()->SweepMultiByObjectType(OutHits, TraceStart, TraceEnd,
        FQuat::Identity, ObjectParams, CollisionShape, QueryParams))
    {
        for (const FHitResult& CurrentHit : OutHits)
        {
            ICombatDamageable* Damageable = Cast<ICombatDamageable>(CurrentHit.GetActor());
            if (Damageable)
            {
                // 피격자 들에게 Notify 제공
                Damageable->NotifyDanger(GetActorLocation(), this);
                
                // 이러면 Shoot RayTrace 할 때도, 미리 Notify를 한 뒤에 데미지 Apply를 하는 식으로 구현하면
                // AI에게 회피기능을 제공할 수도 있을 것 같다
            }
        }
    }
}
```

### DoAttackTrace

AnimNotify 발동 시 실제 히트 판정을 수행한다.  
구체 스윕으로 범위 내 적을 감지하고 `ApplyDamage()`로 데미지를 전달한다.

```c++
void ACombatCharacter::DoAttackTrace()
{
    TArray<FHitResult> OutHits;

    const FVector TraceStart = GetActorLocation();
    const FVector TraceEnd = TraceStart + (GetActorForwardVector() * AttackTraceDistance);

    FCollisionShape CollisionShape;
    CollisionShape.SetSphere(AttackTraceRadius);

    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(this);

    if (GetWorld()->SweepMultiByChannel(OutHits, TraceStart, TraceEnd,
        FQuat::Identity, ECC_Pawn, CollisionShape, QueryParams))
    {
        for (const FHitResult& Hit : OutHits)
        {
            if (ICombatDamageable* Damageable = Cast<ICombatDamageable>(Hit.GetActor()))
            {
                Damageable->ApplyDamage(AttackDamage, this);
            }
        }
    }
}
```
