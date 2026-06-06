---
title: "언리얼 리플렉션 시스템"
date: 2026-06-05 09:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Basic]
tags: [UnrealEngine-Reflection]
description: "언리얼 리플렉션 시스템 기본"
---

---

# 언리얼 리플렉션 시스템

---

# 1. 리플렉션 시스템이란?

C++ 클래스, 변수, 함수를 언리얼 엔진이 인식할 수 있도록 등록하는 시스템이다.

리플렉션이 적용되면:

* 블루프린트 사용 가능
* 에디터 노출 가능
* 네트워크 복제 가능
* 직렬화 가능
* GC(Garbage Collection) 관리 가능

---

# 2. UCLASS

클래스를 엔진에 등록한다.

```cpp
UCLASS()
class TPSBASE_API AMyCharacter : public ACharacter
{
	GENERATED_BODY()
};
```

---

## 2-1. 자주 사용하는 태그

### Blueprintable

블루프린트 상속 가능

```cpp
UCLASS(Blueprintable)
class TPSBASE_API AMyCharacter : public ACharacter
{
	GENERATED_BODY()
};
```

---

### BlueprintType

블루프린트 변수 타입으로 사용 가능

```cpp
UCLASS(BlueprintType)
class TPSBASE_API UMyData : public UObject
{
	GENERATED_BODY()
};
```

---

### Abstract

직접 생성 불가

```cpp
UCLASS(Abstract)
class TPSBASE_API ACharacterBase : public ACharacter
{
	GENERATED_BODY()
};
```

---

### Config

ini 파일 사용

```cpp
UCLASS(Config=Game)
class TPSBASE_API UMySettings : public UObject
{
	GENERATED_BODY()
};
```

---

# 3. UPROPERTY

변수를 엔진에 등록한다.

```cpp
UPROPERTY()
float HP;
```

---

## 3-1. EditAnywhere

에디터 어디서든 수정 가능

```cpp
UPROPERTY(EditAnywhere)
float MoveSpeed = 600.f;
```

---

## 3-2. EditDefaultsOnly

블루프린트 기본값만 수정 가능

```cpp
UPROPERTY(EditDefaultsOnly)
float MaxHP = 100.f;
```

---

## 3-3. EditInstanceOnly

배치된 인스턴스만 수정 가능

```cpp
UPROPERTY(EditInstanceOnly)
float PatrolRadius;
```

---

## 3-4. VisibleAnywhere

읽기만 가능

```cpp
UPROPERTY(VisibleAnywhere)
UStaticMeshComponent* Mesh;
```

주로 컴포넌트에 사용

---

## 3-5. BlueprintReadOnly

블루프린트 읽기 가능

```cpp
UPROPERTY(BlueprintReadOnly)
float HP;
```

---

## 3-6. BlueprintReadWrite

블루프린트 읽기/쓰기 가능

```cpp
UPROPERTY(BlueprintReadWrite)
float HP;
```

---

## 3-7. Category

에디터 카테고리 분류

```cpp
UPROPERTY(EditAnywhere, Category="Character")
float HP;
```

---

## 3-8. Transient

저장되지 않음

```cpp
UPROPERTY(Transient)
float RuntimeDamage;
```

런타임 전용 데이터

---

## 3-9. Replicated

네트워크 복제

```cpp
UPROPERTY(Replicated)
float HP;
```

---

## 3-10. ReplicatedUsing

복제 시 콜백 호출

```cpp
UPROPERTY(ReplicatedUsing=OnRep_HP)
float HP;

UFUNCTION()
void OnRep_HP();
```

---

## 3-11. Meta

에디터 표시 방식 설정

```cpp
UPROPERTY(EditAnywhere, meta=(ClampMin="0"))
float HP;
```

---

### ClampMin

최소값 제한

```cpp
UPROPERTY(EditAnywhere, meta=(ClampMin="0"))
float HP;
```

---

### ClampMax

최대값 제한

```cpp
UPROPERTY(EditAnywhere, meta=(ClampMax="100"))
float HP;
```

---

### UIMin / UIMax

슬라이더 범위 제한

```cpp
UPROPERTY(EditAnywhere,
	meta=(UIMin="0", UIMax="100"))
float HP;
```

---

### AllowPrivateAccess

Private 변수 BP 노출

```cpp
UPROPERTY(
	VisibleAnywhere,
	BlueprintReadOnly,
	meta=(AllowPrivateAccess="true"))
UCameraComponent* Camera;
```

---

# 4. UFUNCTION

함수를 엔진에 등록한다.

```cpp
UFUNCTION()
void Heal();
```

---

## 4-1. BlueprintCallable

블루프린트에서 호출 가능

```cpp
UFUNCTION(BlueprintCallable)
void Heal();
```

---

## 4-2. BlueprintPure

실행 핀 없는 함수

```cpp
UFUNCTION(BlueprintPure)
float GetHP() const;
```

Getter 함수에 사용

---

## 4-3. BlueprintImplementableEvent

블루프린트에서 구현

```cpp
UFUNCTION(BlueprintImplementableEvent)
void OnDeath();
```

CPP 구현 없음

---

## 4-4. BlueprintNativeEvent

CPP 기본 구현 + BP 확장 가능

```cpp
UFUNCTION(BlueprintNativeEvent)
void OnDeath();
```

```cpp
void OnDeath_Implementation();
```

---

## 4-5. CallInEditor

에디터 버튼 생성

```cpp
UFUNCTION(CallInEditor)
void GeneratePoints();
```

---

# 5. RPC

멀티플레이 함수 호출

---

## 5-1. Server

클라이언트 → 서버

```cpp
UFUNCTION(Server, Reliable)
void ServerAttack();
```

---

## 5-2. Client

서버 → 특정 클라이언트

```cpp
UFUNCTION(Client, Reliable)
void ClientNotify();
```

---

## 5-3. NetMulticast

서버 → 모든 클라이언트

```cpp
UFUNCTION(NetMulticast, Reliable)
void MulticastExplosion();
```

---

## 5-4. Reliable

전송 보장

```cpp
UFUNCTION(Server, Reliable)
void ServerInteract();
```

중요한 이벤트

* 공격
* 아이템 획득
* 문 열기

---

## 5-5. Unreliable

전송 보장 안함

```cpp
UFUNCTION(Server, Unreliable)
void ServerMove();
```

빈번한 데이터

* 이동
* 카메라
* 조준 방향

---

# 6. USTRUCT

구조체를 엔진에 등록한다.

```cpp
USTRUCT(BlueprintType)
struct FCharacterData
{
	GENERATED_BODY()

	UPROPERTY(EditAnywhere)
	float HP;

	UPROPERTY(EditAnywhere)
	float MP;
};
```

---

# 7. UENUM

Enum을 엔진에 등록한다.

```cpp
UENUM(BlueprintType)
enum class ECharacterState : uint8
{
	Idle,
	Run,
	Jump,
	Dead
};
```

---

# 8. 실무에서 가장 많이 쓰는 조합

### 컴포넌트

```cpp
UPROPERTY(
	VisibleAnywhere,
	BlueprintReadOnly,
	Category="Components")
USpringArmComponent* SpringArm;
```

---

### 에디터 설정값

```cpp
UPROPERTY(
	EditDefaultsOnly,
	BlueprintReadOnly,
	Category="Character")
float MaxHP = 100.f;
```

---

### 레벨 배치 설정값

```cpp
UPROPERTY(
	EditInstanceOnly,
	Category="AI")
float PatrolRadius;
```

---

### Getter

```cpp
UFUNCTION(BlueprintPure)
float GetHP() const;
```

---

### 액션 함수

```cpp
UFUNCTION(BlueprintCallable)
void Attack();
```

---

### 서버 RPC

```cpp
UFUNCTION(Server, Reliable)
void ServerInteract();
```

---

# 9. 우선 암기할 것

초반에 가장 많이 보는 태그는 아래 10개다.

```cpp
UCLASS()

UPROPERTY(EditAnywhere)
UPROPERTY(EditDefaultsOnly)
UPROPERTY(VisibleAnywhere)
UPROPERTY(BlueprintReadOnly)
UPROPERTY(BlueprintReadWrite)

UFUNCTION(BlueprintCallable)
UFUNCTION(BlueprintPure)

UFUNCTION(Server, Reliable)

USTRUCT(BlueprintType)
```
