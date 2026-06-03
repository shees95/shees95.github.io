---
title: "언리얼 프레임워크 구조 2"
date: 2026-06-03 14:15:00 +0900
categories: [UnrealEngine, Structure]
tags: [UnrealEngine-Structure]
description: "언리얼엔진의 엔진 시작부터 액터/컴포넌트 초기화까지"
---~~~~

# 언리얼엔진 프레임워크 구조 2

---

# 1. 엔진 시작

LaunchEngineLoop.cpp
```c++

  GEngine->Start();
  
```

GameEngine.cpp

```c++

  void UGameEngine::Init(IEngineLoop* InEngineLoop)
  {
    ...
    
    GameInstance = NewObject<UGameInstance>(this, GameInstanceClass);
		GameInstance->InitializeStandalone();
  }

```

---

# 2. 인스턴스 초기화

```c++

  void UGameInstance::InitializeStandalone(const FName InPackageName, UPackage* InWorldPackage)
  {
    // 월드 생성
    WorldContext = &GetEngine()->CreateNewWorldContext(EWorldType::Game);
    WorldContext->OwningGameInstance = this;
    
    // 맵 로드 중 맵 없음 이슈 해결을 위해 더미 월드 생성
    UWorld* DummyWorld = UWorld::CreateWorld(EWorldType::Game, false, InPackageName, InWorldPackage);
    DummyWorld->SetGameInstance(this);
    WorldContext->SetCurrentWorld(DummyWorld);
    
    // 여기서 인스턴스 초기화
    Init();
  }

```

GameInstance.cpp

```c++

  void UGameInstance::Init()
  {
    // 블루프린트 이벤트 Init 호출
    ReceiveInit();
    
    // 온라인 세션 객체 생성
    UClass* SpawnClass = GetOnlineSessionClass();
    OnlineSession = NewObject<UOnlineSession>(this, SpawnClass);
    OnlineSession->RegisterOnlineDelegates();
      
    // 전용 서버 아닐 때는 콘솔 입력 리스너 등록
    App->RegisterConsoleCommandListener(GenericApplication::FOnConsoleCommandListener::CreateUObject(this, &ThisClass::OnConsoleInput));
    
    ...
  }

```

GameInstance 에서는 게임 유지를 하는데 필요한 요소들을 Init 함.

---

# 3. GameInstance 생성 및 Start

```c++
 
  void UGameEngine::Start()
  {
    TRACE_CPUPROFILER_EVENT_SCOPE(UGameEngine::Start);
    UE_LOG(LogInit, Display, TEXT("Starting Game."));
  
    GameInstance->StartGameInstance();
  }
  
```





# 3. Browse

# 4. LoadMap

# 5. UWorld 생성

# 6 . ULevel과 Actor 초기화

# 7. Component 초기화

# 8. BeginPlay()

# 정리
