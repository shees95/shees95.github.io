---
title: "언리얼 TIL - Gameplay Message Router 사용법"
date: 2026-08-18 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-System]
tags: [UnrealEngine, GameplayMessageRouter, GameplayTag, TIL]
description: "Gameplay Tag 채널로 구조체 메시지를 발송하고 구독하는 방법과 팀 학습용 예시"
---

# Gameplay Message Router

`GameplayMessageRouter`는 송신자와 수신자가 서로를 직접 참조하지 않고,
`GameplayTag` 채널을 통해 구조체 데이터를 전달하는 플러그인이다.

델리게이트처럼 메시지를 방송하고 구독하지만 송신자가 수신자를 알 필요가 없어,
게임플레이 로직과 UI처럼 결합도를 낮춰야 하는 시스템 사이에 사용하기 좋다.

```text
송신자 → BroadcastMessage(채널 태그, 페이로드)
                         ↓
             GameplayMessageSubsystem
                         ↓
수신자 ← RegisterListener(채널 태그, 콜백)
```

## 사용 순서

1. `GameplayMessageRouter` 플러그인을 활성화한다.
2. 전달할 데이터를 `USTRUCT` 페이로드로 선언한다.
3. 송신자는 `UGameplayMessageSubsystem::BroadcastMessage()`를 호출한다.
4. 수신자는 `RegisterListener()`로 채널을 구독한다.
5. 수신이 끝나면 리스너 핸들을 해제한다.

태그는 발신자를 한정하는 값이 아니라 **메시지 채널을 구분하는 값**이다.
특정 발신자를 구분해야 한다면 페이로드에 `Sender`나 `Instigator`를 포함한다.

## 팀 학습 예시: 아이템 획득 알림

플레이어가 아이템을 획득하면 획득 시스템이 메시지를 발송하고,
UI가 메시지를 받아 아이템 이름과 수량을 표시하는 예시다.

### 1. 페이로드와 채널 선언

```cpp
#pragma once

#include "CoreMinimal.h"
#include "GameplayTagContainer.h"
#include "ItemAcquiredMessage.generated.h"

UE_DECLARE_GAMEPLAY_TAG_EXTERN(TAG_Message_Item_Acquired);

USTRUCT(BlueprintType)
struct FItemAcquiredMessage
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    TObjectPtr<AActor> Sender = nullptr;

    UPROPERTY(BlueprintReadOnly)
    FName ItemName;

    UPROPERTY(BlueprintReadOnly)
    int32 Quantity = 0;
};
```

```cpp
#include "ItemAcquiredMessage.h"

UE_DEFINE_GAMEPLAY_TAG(TAG_Message_Item_Acquired, "Message.Item.Acquired");
```

### 2. 메시지 발송

아이템 획득 처리가 끝난 시점에 페이로드를 만들어 방송한다.

```cpp
#include "GameFramework/GameplayMessageSubsystem.h"
#include "ItemAcquiredMessage.h"

void UInventoryComponent::NotifyItemAcquired(const FName ItemName, const int32 Quantity)
{
    FItemAcquiredMessage Message;
    Message.Sender = GetOwner();
    Message.ItemName = ItemName;
    Message.Quantity = Quantity;

    UGameplayMessageSubsystem& MessageSubsystem = UGameplayMessageSubsystem::Get(this);
    MessageSubsystem.BroadcastMessage(TAG_Message_Item_Acquired, Message);
}
```

### 3. 메시지 구독

UI 위젯이 생성될 때 리스너를 등록하고 제거될 때 해제한다.

```cpp
#include "GameFramework/GameplayMessageSubsystem.h"
#include "ItemAcquiredMessage.h"

void UItemNotificationWidget::NativeConstruct()
{
    Super::NativeConstruct();

    UGameplayMessageSubsystem& MessageSubsystem = UGameplayMessageSubsystem::Get(this);
    ListenerHandle = MessageSubsystem.RegisterListener(
        TAG_Message_Item_Acquired,
        this,
        &ThisClass::OnItemAcquired);
}

void UItemNotificationWidget::NativeDestruct()
{
    ListenerHandle.Unregister();
    Super::NativeDestruct();
}

void UItemNotificationWidget::OnItemAcquired(
    FGameplayTag Channel,
    const FItemAcquiredMessage& Message)
{
    // 실제 프로젝트에서는 여기서 알림 위젯의 텍스트와 애니메이션을 갱신한다.
    UE_LOG(
        LogTemp,
        Log,
        TEXT("Acquired %s x%d"),
        *Message.ItemName.ToString(),
        Message.Quantity);
}
```

`ListenerHandle`은 위젯의 멤버 변수로 보관한다.

```cpp
FGameplayMessageListenerHandle ListenerHandle;
```

## 태그 매칭

- `ExactMatch`: 등록한 태그와 완전히 같은 채널만 수신한다.
- `PartialMatch`: 등록한 태그와 그 하위 채널까지 수신한다.

예를 들어 `Message.Item`을 부분 매칭으로 구독하면
`Message.Item.Acquired`와 `Message.Item.Removed`를 함께 받을 수 있다.

## 주의할 점

- 메시지는 일회성이다. 구독 전에 발송된 과거 메시지는 받을 수 없다.
- 메시지를 저장해야 한다면 별도의 상태 보관 객체가 필요하다.
- 서버와 클라이언트 사이를 자동으로 복제하지 않는다.
- 멀티플레이에서는 RPC나 복제로 전달한 뒤 각 로컬 환경에서 방송해야 한다.
- 하나의 채널에는 하나의 페이로드 타입을 일관되게 사용하는 편이 안전하다.
- 실행 순서나 처리가 반드시 보장되어야 하는 핵심 로직에는 직접 호출을 우선한다.

## 팀 실습

1. `Message.Item.Removed` 채널과 제거 페이로드를 추가한다.
2. `Message.Item`을 부분 매칭으로 구독해 획득과 제거를 한 콜백에서 구분한다.
3. 페이로드의 `Sender`로 로컬 플레이어의 메시지만 UI에 표시한다.
4. 서버에서 발송한 메시지가 클라이언트에 자동 전달되지 않는 것을 확인한다.

## 정리

Gameplay Message Router의 핵심은 **태그 채널 + 구조체 페이로드**다.
태그로 메시지 종류를 구분하고, 페이로드에 처리에 필요한 데이터와 발신자 정보를 담는다.
리스너 수명과 네트워크 경계만 명확히 관리하면 시스템 간 직접 참조를 간단히 줄일 수 있다.
