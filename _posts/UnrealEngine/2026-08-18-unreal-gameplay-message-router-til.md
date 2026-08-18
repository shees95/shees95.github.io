---
title: "언리얼 TIL - Gameplay Message Router 사용법"
date: 2026-08-18 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-System]
tags: [UnrealEngine, GameplayMessageRouter, GameplayTag, TIL]
description: "Gameplay Tag를 채널로 사용해 구조체 페이로드를 방송하고 구독하는 방법 정리"
---

# Gameplay Message Router

`Gameplay Message Router`는 `GameplayTag`를 메시지 채널로 사용해 구조체 데이터를 전달하는 시스템이다.

발행자는 태그와 페이로드를 방송하고, 해당 태그를 구독 중인 리스너의 콜백이 호출된다.
발행자와 수신자가 서로를 직접 참조하지 않아도 된다는 점이 핵심이다.

```text
발행자 ── BroadcastMessage(태그, 페이로드) ──▶ Gameplay Message Subsystem
                                                      │
                                  태그와 페이로드 타입에 맞는 리스너 탐색
                                                      │
수신자 ◀──────────────────────────── 등록된 콜백 호출 ──┘
```

## 사용 흐름

1. 전달할 데이터를 `USTRUCT` 페이로드로 만든다.
2. 발행자가 서브시스템을 가져와 태그와 페이로드를 방송한다.
3. 수신자가 같은 태그와 페이로드 타입으로 콜백을 등록한다.
4. 수신이 필요 없어지면 리스너 등록을 해제한다.

## 1. 페이로드 선언

페이로드는 리플렉션이 가능한 구조체로 선언한다.
Blueprint에서도 같은 데이터를 문제없이 다루려면 `BlueprintType`과 `UPROPERTY`를 함께 사용한다.

```cpp
#pragma once

#include "CoreMinimal.h"
#include "NKMInventoryMergedMessage.generated.h"

USTRUCT(BlueprintType)
struct FNKMInventoryMergedMessage
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    TObjectPtr<AActor> Sender = nullptr;

    UPROPERTY(BlueprintReadOnly)
    int32 MergedCount = 0;
};
```

`GENERATED_BODY()`는 Unreal Header Tool이 구조체의 리플렉션 코드를 생성하는 데 필요하다.
`BlueprintType`은 C++ 메시지 전달 자체의 필수 조건은 아니지만, 페이로드를 Blueprint에서도 사용할 경우 필요하다.

하나의 메시지 태그에는 항상 같은 페이로드 타입을 사용하는 편이 안전하다.

## 2. 메시지 방송

먼저 현재 월드의 `UGameplayMessageSubsystem`을 가져온다.

```cpp
UGameplayMessageSubsystem& MessageSubsystem = UGameplayMessageSubsystem::Get(this);
```

전달할 페이로드를 만든 뒤 메시지 태그와 함께 방송한다.

```cpp
void UNKMInventoryComponent::BroadcastInventoryMerged(const int32 MergedCount)
{
    FNKMInventoryMergedMessage MergedMessage;
    MergedMessage.Sender = GetOwner();
    MergedMessage.MergedCount = MergedCount;

    UGameplayMessageSubsystem& MessageSubsystem = UGameplayMessageSubsystem::Get(this);
    MessageSubsystem.BroadcastMessage(
        NKMGameplayTags::Message_Inventory_Merged,
        MergedMessage);
}
```

태그는 특정 객체의 주소가 아니라 메시지의 종류를 나타내는 채널이다.
특정 발행자를 구분해야 한다면 예시처럼 `Sender` 또는 `Instigator`를 페이로드에 담는다.

## 3. 콜백 등록

수신자는 메시지를 받을 태그, 수신 객체, 콜백 함수를 등록한다.

```cpp
void UNKMInventoryWidget::NativeConstruct()
{
    Super::NativeConstruct();

    UGameplayMessageSubsystem& MessageSubsystem = UGameplayMessageSubsystem::Get(this);
    ListenerHandle = MessageSubsystem.RegisterListener(
        NKMGameplayTags::Message_Inventory_Merged,
        this,
        &ThisClass::OnInventoryMerged);
}
```

콜백은 방송된 채널과 페이로드를 받는다.

```cpp
void UNKMInventoryWidget::OnInventoryMerged(
    FGameplayTag Channel,
    const FNKMInventoryMergedMessage& Message)
{
    // 병합된 인벤토리 정보를 UI에 반영한다.
    UpdateMergedItemCount(Message.MergedCount);
}
```

등록 결과로 받은 핸들은 멤버 변수로 보관한다.

```cpp
FGameplayMessageListenerHandle ListenerHandle;
```

## 4. 리스너 해제

객체가 더 이상 메시지를 받을 필요가 없다면 등록을 해제한다.

```cpp
void UNKMInventoryWidget::NativeDestruct()
{
    ListenerHandle.Unregister();
    Super::NativeDestruct();
}
```

리스너 핸들의 생명주기를 명시적으로 관리하면 파괴된 UI에 불필요한 메시지가 전달되는 것을 막을 수 있다.

## 태그 라우팅

리스너는 태그 매칭 방식에 따라 수신 범위를 정할 수 있다.

| 매칭 방식 | 동작 |
|---|---|
| `ExactMatch` | 등록한 태그와 완전히 같은 채널만 수신 |
| `PartialMatch` | 등록한 태그와 그 하위 채널까지 수신 |

예를 들어 `Message.Inventory`를 부분 매칭으로 구독하면
`Message.Inventory.Merged`, `Message.Inventory.Added` 같은 하위 채널을 함께 받을 수 있다.

## 장점

- 전역 서브시스템을 통해 어디서든 접근하기 쉽다.
- 발행자와 수신자가 서로의 존재를 몰라도 된다.
- Gameplay Tag와 매칭 방식으로 필요한 메시지만 필터링할 수 있다.
- 여러 시스템이 같은 메시지를 구독하기 쉬워 시스템 간 결합도를 낮출 수 있다.

델리게이트는 일반적으로 발행자 인스턴스를 알아낸 뒤 다음과 같이 직접 바인딩해야 한다.

```cpp
Publisher->OnEvent.AddDynamic(WorldObject, &ThisClass::CallbackFunction);
```

Message Router에서는 발행자 인스턴스 대신 공유된 메시지 태그만 알면 된다.

## 단점과 주의점

- 여러 리스너의 실행 순서에 의존하면 안 된다.
- 호출 관계가 코드에 직접 드러나지 않아 발행 위치와 수신 위치를 추적하기 어렵다.
- 직접 호출이나 델리게이트보다 태그 탐색과 타입 확인에 따른 비용이 추가된다.
- 매 프레임처럼 매우 자주 갱신되는 데이터에는 직접 호출이나 델리게이트가 더 적합하다.
- 구독 전에 방송된 메시지는 나중에 받을 수 없으므로 상태 저장 용도로 사용할 수 없다.
- 서버에서 방송한 메시지가 클라이언트로 자동 복제되지는 않는다.

따라서 Message Router는 실행 순서가 중요한 핵심 로직보다
UI 알림, 인벤토리 변경 알림처럼 시스템 사이의 결합도를 낮추고 싶은 이벤트에 적합하다.

## 정리

Gameplay Message Router의 핵심은 **Gameplay Tag 채널 + USTRUCT 페이로드**다.

```cpp
UGameplayMessageSubsystem& MessageSubsystem = UGameplayMessageSubsystem::Get(this);

MessageSubsystem.BroadcastMessage(
    NKMGameplayTags::Message_Inventory_Merged,
    MergedMessage);

ListenerHandle = MessageSubsystem.RegisterListener(
    Tag,
    this,
    &ThisClass::CallbackFunction);
```

발행자는 태그와 페이로드만 방송하고, 수신자는 필요한 태그만 구독한다.
이 방식으로 서로를 직접 참조하지 않으면서 필요한 데이터를 전달할 수 있다.
