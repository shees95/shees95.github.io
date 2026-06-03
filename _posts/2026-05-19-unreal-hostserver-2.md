---
title: "언리얼 호스트 서버 열기 2"
date: 2026-05-19 00:00:00 +0900
categories: [UnrealEngine, ListenServer]
tags: [Server-Operation]
description: "언리얼 Listen서버 내 데이터 통신 주고받기"
---

# 서버 내 연산 처리

---

## 1. 서버 권한 검증 (`HasAuthority`)

네트워크 환경에서 중요한 연산(데미지 계산, 아이템 획득 등)은 반드시 서버에서만 실행되어야 합니다.

* **`HasAuthority()`**: 현재 이 코드를 실행하는 주체가 서버(권한자)인지 확인하는 함수입니다.
* **역할**: 클라이언트의 임의 변조를 막고, 서버에서 안전하게 연산한 결과만을 클라이언트에 동기화(전달)하도록 강제합니다.

```cpp
if (GetOwner()->HasAuthority())
{
    // 서버에서만 실행되는 안전한 로직 (예: 체력 감소, 점수 획득)
}

```

---

## 2. 액터 복제 활성화 (기본 세팅)

액터의 변수나 움직임이 네트워크를 통해 클라이언트들에게 전달되려면, **생성자**에서 복제 기능을 켜주어야 합니다.

* **`bReplicates = true;`**: 이 액터를 네트워크를 통해 복제하겠다고 선언합니다.
* **`SetReplicateMovement(true);`**: 액터의 위치(Location), 회전(Rotation), 속도(Velocity) 등 움직임 정보도 자동으로 서버에서 클라이언트로 동기화합니다.

```cpp
AMyActor::AMyActor()
{
    PrimaryActorTick.bCanEverTick = true;

    // 네트워크 복제 활성화
    bReplicates = true;
    SetReplicateMovement(true); 
}

```

---

## 3. 멤버 변수 복제 (`Replicated`)

특정 변수의 값이 서버에서 변경되었을 때, 이를 클라이언트들에게 자동으로 전송(동기화)하고 싶다면 2가지 작업이 필요합니다.

### ① UPROPERTY 매크로 설정

변수 선언 위에 `Replicated` 지정자를 추가합니다. 이 변수는 **서버에서만 연산 및 수정**되어야 하며, 변경된 값이 클라이언트로 단방향 복제됩니다.

```cpp
protected:
    UPROPERTY(Replicated)
    int32 Health;

```

### ② `GetLifetimeReplicatedProps` 오버라이드 및 구현

`Replicated`로 지정된 변수는 **반드시** 이 함수를 통해 등록해 주어야 실제로 복제가 일어납니다.

* **헤더 파일 (`.h`) 선언**
```cpp
virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;

```


* **소스 파일 (`.cpp`) 구현**
* `Net/UnrealNetwork.h` 헤더를 반드시 포함해야 합니다.
* `DOREPLIFETIME(클래스이름, 변수이름);` 매크로를 사용해 등록합니다.



```cpp
#include "Net/UnrealNetwork.h" // 매크로 사용을 위해 필수 포함!

void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // Health 변수를 복제 대상으로 등록
    DOREPLIFETIME(AMyActor, Health);
}

```

---

## 정리

> **"생성자에서 `bReplicates`를 켜고, 변수에 `Replicated`를 붙인 뒤, `GetLifetimeReplicatedProps`에 `DOREPLIFETIME`으로 등록하면, 서버(`HasAuthority`)에서 바꾼 변수 값이 클라이언트에게 안전하게 전달된다!"**
