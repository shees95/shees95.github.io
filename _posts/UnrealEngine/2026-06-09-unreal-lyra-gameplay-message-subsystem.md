---
title: "언리얼 라이라 - GameplayMessageSubsystem"
date: 2026-06-09 22:50:00 +0900
categories: [UnrealEngine, UnrealEngine-StarterProject]
tags: [UnrealEngine, UnrealEngine-Lyra, UnrealEngine-GameplayMessageSubsystem, UnrealEngine-MessageRouter]
description: "라이라의 GameplayMessageSubsystem으로 느슨한 결합 메시지 브로드캐스트/수신 구조 정리"
---

# GameplayMessageSubsystem

## 개념

`UGameplayMessageSubsystem`은 `GameplayMessageRouter` 플러그인에서 제공하는 `UGameInstanceSubsystem`이다.

송신자와 수신자가 서로를 직접 참조하지 않고, **GameplayTag 채널**을 통해 메시지를 주고받는 구조다.

```
송신자 → BroadcastMessage(Channel, Message)
                  ↓
       GameplayMessageSubsystem
                  ↓
수신자 ← RegisterListener(Channel, Callback)
```

---

## 주요 파일

```
Plugins/GameplayMessageRouter/Source/GameplayMessageRuntime/Public/GameFramework/
├── GameplayMessageSubsystem.h   ← 핵심 클래스
└── GameplayMessageTypes2.h      ← EGameplayMessageMatch, FGameplayMessageListenerParams
```

---

## 핵심 API

### 서브시스템 가져오기

```cpp
UGameplayMessageSubsystem& MessageSystem = UGameplayMessageSubsystem::Get(this);
```

### 메시지 브로드캐스트

```cpp
// 채널 태그 + USTRUCT 메시지 전송
MessageSystem.BroadcastMessage(Message.Verb, Message);
```

라이라 실사용 예시 (`LyraHealthSet.cpp`):

```cpp
// 메시지 생성
FLyraVerbMessage Message;
Message.Verb       = TAG_Lyra_Damage_Message;
Message.Instigator = Data.EffectSpec.GetEffectContext().GetEffectCauser();
Message.Target     = GetOwningActor();
Message.Magnitude  = Data.EvaluatedData.Magnitude;

// 월드에 알림
UGameplayMessageSubsystem& MessageSystem = UGameplayMessageSubsystem::Get(GetWorld());
MessageSystem.BroadcastMessage(Message.Verb, Message);
```

### 리스너 등록 (멤버 함수 바인딩)

```cpp
// Object의 멤버 함수로 직접 바인딩, WeakPtr로 안전하게 처리됨
ListenerHandle = MessageSubsystem.RegisterListener(
    TAG_Lyra_AddNotification_Message,
    this,
    &ThisClass::OnNotificationMessage
);
```

라이라 실사용 예시 (`LyraAccoladeHostWidget.cpp`):

```cpp
// 등록
ListenerHandle = MessageSubsystem.RegisterListener(
    TAG_Lyra_AddNotification_Message, this, &ThisClass::OnNotificationMessage);

// 해제
MessageSubsystem.UnregisterListener(ListenerHandle);
```

### 리스너 등록 (람다)

```cpp
FGameplayMessageListenerHandle Handle = MessageSystem.RegisterListener<FMyMessage>(
    MyChannel,
    [](FGameplayTag Channel, const FMyMessage& Msg)
    {
        // 처리
    }
);
```

### 리스너 해제

```cpp
MessageSubsystem.UnregisterListener(Handle);

// 또는 핸들에서 직접
Handle.Unregister();
```

---

## 채널 매칭 방식

```cpp
enum class EGameplayMessageMatch : uint8
{
    ExactMatch,   // "A.B" 등록 → "A.B" 만 수신 (기본값)
    PartialMatch, // "A.B" 등록 → "A.B", "A.B.C" 모두 수신
};
```

---

## GameplayMessageProcessor 패턴

라이라에서 리스너를 컴포넌트 단위로 묶어 관리하는 베이스 클래스 (`GameplayMessageProcessor.cpp`):

```cpp
void UGameplayMessageProcessor::BeginPlay()
{
    Super::BeginPlay();
    StartListening(); // 서브클래스에서 RegisterListener 호출
}

void UGameplayMessageProcessor::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    Super::EndPlay(EndPlayReason);
    StopListening();

    UGameplayMessageSubsystem& MessageSubsystem = UGameplayMessageSubsystem::Get(this);
    for (FGameplayMessageListenerHandle& Handle : ListenerHandles)
    {
        MessageSubsystem.UnregisterListener(Handle);
    }
    ListenerHandles.Empty();
}

void UGameplayMessageProcessor::AddListenerHandle(FGameplayMessageListenerHandle&& Handle)
{
    ListenerHandles.Add(MoveTemp(Handle));
}
```

서브클래스에서는 `StartListening()`에서 등록만 하면 됨:

```cpp
// AssistProcessor.cpp
void UAssistProcessor::StartListening()
{
    UGameplayMessageSubsystem& MessageSubsystem = UGameplayMessageSubsystem::Get(this);
    AddListenerHandle(MessageSubsystem.RegisterListener(TAG_Lyra_Elimination_Message, this, &ThisClass::OnEliminationMessage));
    AddListenerHandle(MessageSubsystem.RegisterListener(TAG_Lyra_Damage_Message,      this, &ThisClass::OnDamageMessage));
}
```

---

## 구조 요약

| 항목 | 설명 |
|---|---|
| 기반 클래스 | `UGameInstanceSubsystem` (게임 인스턴스 생존) |
| 채널 식별 | `FGameplayTag` |
| 메시지 타입 | 임의 `USTRUCT` |
| 송신 | `BroadcastMessage(Channel, Message)` |
| 수신 등록 | `RegisterListener(Channel, ...)` |
| 수신 해제 | `UnregisterListener(Handle)` or `Handle.Unregister()` |
| 안전성 | 멤버 함수 바인딩 시 내부적으로 `TWeakObjectPtr` 사용 |
