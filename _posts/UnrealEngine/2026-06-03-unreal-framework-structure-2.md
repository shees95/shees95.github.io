---
title: "언리얼 프레임워크 구조 2"
date: 2026-06-03 14:15:00 +0900
categories: [UnrealEngine, UnrealEngine-Base]
tags: [UnrealEngine, UnrealEngine-Structure, UnrealEngine-Framework]
description: "StartGameInstance 이후 흐름, Tick 구조, 월드 생성 과정"
---

# 언리얼엔진 프레임워크 구조 2

---

## 전체 흐름

```yaml
GEngine->Start()
└ GameInstance->StartGameInstance()
   └ Browse
      └ LoadMap
         ├ 기존 월드 정리
         ├ UWorld 생성 (InitWorld)
         ├ ULevel / Actor 로드
         ├ InitializeActorsForPlay
         └ UWorld::BeginPlay

EngineTick (매 프레임)
└ FEngineLoop::Tick
   └ GEngine->Tick
      └ UWorld::Tick
         ├ TickGroup(TG_PrePhysics) → Actor / Component Tick
         ├ 물리 시뮬레이션
         └ TickGroup(TG_PostPhysics) → Actor / Component Tick
```

---

## 1. 엔진 시작

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

## 2. 인스턴스 초기화

```c++
void UGameInstance::InitializeStandalone(const FName InPackageName, UPackage* InWorldPackage)
{
    WorldContext = &GetEngine()->CreateNewWorldContext(EWorldType::Game);
    WorldContext->OwningGameInstance = this;

    // 맵 로드 전 임시 더미 월드 생성
    UWorld* DummyWorld = UWorld::CreateWorld(EWorldType::Game, false, InPackageName, InWorldPackage);
    DummyWorld->SetGameInstance(this);
    WorldContext->SetCurrentWorld(DummyWorld);

    Init();
}
```

GameInstance.cpp

```c++
void UGameInstance::Init()
{
    ReceiveInit();

    UClass* SpawnClass = GetOnlineSessionClass();
    OnlineSession = NewObject<UOnlineSession>(this, SpawnClass);
    OnlineSession->RegisterOnlineDelegates();

    ...
}
```

GameInstance는 레벨이 바뀌어도 유지되는 객체다.  
Init에서는 온라인 세션, 콘솔 입력 리스너 등 게임 전체에서 공유되는 요소들을 준비한다.

---

## 3. StartGameInstance

```c++
void UGameEngine::Start()
{
    GameInstance->StartGameInstance();
}
```

GameInstance.cpp

```c++
void UGameInstance::StartGameInstance()
{
    FURL DefaultURL;
    FURL URL(&DefaultURL, *GetDefault<UGameMapsSettings>()->GetGameDefaultMap(), TRAVEL_Partial);

    FString Error;
    EBrowseReturnVal::Type BrowseRet = GetEngine()->Browse(*WorldContext, URL, Error);
    ...
}
```

여기서 설정 파일(GameMapsSettings)에 지정된 기본 맵 URL을 파싱하고, Browse를 호출한다.  
이 시점부터 실제 게임 맵 로드가 시작된다.

---

## 4. Browse

```c++
EBrowseReturnVal::Type UEngine::Browse(FWorldContext& WorldContext, FURL URL, FString& Error)
{
    if (URL.IsLocalInternal())
    {
        return LoadMap(WorldContext, URL, nullptr, Error)
            ? EBrowseReturnVal::Success
            : EBrowseReturnVal::Failure;
    }
    else if (URL.IsInternal())
    {
        // 네트워크 연결 (PendingNetGame 생성)
        ...
    }
    ...
}
```

Browse는 URL을 분석해 로컬 맵인지, 원격 서버 연결인지를 판단한다.  
일반 싱글플레이 게임은 항상 로컬 경로이므로 LoadMap으로 바로 이어진다.  
네트워크 게임에서는 PendingNetGame을 생성해 연결을 먼저 시도한 후 LoadMap을 호출한다.

---

## 5. LoadMap

LoadMap은 언리얼에서 가장 복잡한 함수 중 하나다.  
기존 월드를 정리하고, 새 UWorld를 만들고, 레벨을 로드하고, BeginPlay를 실행하는 전 과정이 이 함수 안에서 진행된다.

Engine.cpp

```c++
bool UEngine::LoadMap(FWorldContext& WorldContext, FURL URL, UPendingNetGame* Pending, FString& Error)
{
    // 1. 기존 월드 정리
    CleanupWorld(WorldContext);

    // 2. 패키지에서 맵 로드 or 새 UWorld 생성
    UPackage* WorldPackage = LoadPackage(nullptr, *URL.Map, LOAD_None);
    UWorld* NewWorld = UWorld::FindWorldInPackage(WorldPackage);

    // 3. 월드 초기화
    NewWorld->InitWorld();

    // 4. WorldContext에 등록
    WorldContext->SetCurrentWorld(NewWorld);
    NewWorld->SetGameInstance(WorldContext.OwningGameInstance);

    // 5. 레벨과 액터 준비
    NewWorld->InitializeActorsForPlay(URL);

    // 6. 게임 시작
    NewWorld->BeginPlay();

    return true;
}
```

---

## 6. UWorld 생성 과정

월드는 두 가지 방식으로 만들어진다.  
맵 파일이 있는 경우에는 패키지에서 역직렬화로 복원되고, 런타임에 빈 월드가 필요한 경우에는 CreateWorld로 직접 생성된다.

```c++
UWorld* UWorld::CreateWorld(const EWorldType::Type InWorldType, bool bInformEngineOfWorld, ...)
{
    UPackage* WorldPackage = InWorldPackage ? InWorldPackage : CreatePackage(nullptr);
    UWorld* NewWorld = NewObject<UWorld>(WorldPackage, ...);

    NewWorld->WorldType = InWorldType;
    NewWorld->InitializeNewWorld(...);

    return NewWorld;
}
```

월드가 생성되면 InitWorld가 호출되어 월드 단위의 시스템들이 초기화된다.

```c++
void UWorld::InitWorld(const InitializationValues IVS)
{
    // 물리 씬 생성
    CreatePhysicsScene();

    // 네비게이션 시스템
    FNavigationSystem::AddNavigationSystemToWorld(*this, ...);

    // AI 시스템
    CreateAISystem();

    // FX 시스템
    Scene = GetRendererModule().AllocateScene(...);

    // World Subsystem 초기화
    SubsystemCollection.Initialize(this);
}
```

World Subsystem은 UWorldSubsystem을 상속받으면 자동으로 등록된다.  
InitWorld 시점에 SubsystemCollection이 해당 서브클래스들을 찾아 인스턴스를 생성한다.  
월드가 소멸할 때 함께 소멸한다.

---

## 7. ULevel과 Actor 초기화

UWorld는 하나 이상의 ULevel로 구성된다.  
항상 존재하는 PersistentLevel이 있고, 스트리밍으로 추가되는 레벨들이 있다.

맵 파일(.umap)은 UPackage 형태로 직렬화되어 있다.  
LoadMap 과정에서 이 패키지를 역직렬화해 ULevel과 그 안의 Actor 인스턴스들을 복원한다.

```yaml
UPackage 로드
└ UWorld 역직렬화
   └ ULevel 역직렬화
      └ Actor 인스턴스 복원 (CDO 복사 기반)
```

Actor가 패키지에서 복원될 때, 생성자는 이미 CDO 생성 시점에 실행된 상태다.  
실제 인스턴스는 CDO를 복사한 형태로 만들어지고, 저장된 프로퍼티 값들로 덮어쓴다.

레벨이 월드에 추가될 때 AddToWorld가 호출된다.

```c++
void UWorld::AddToWorld(ULevel* Level, ...)
{
    // 레벨의 Actor들을 월드에 등록
    // 컴포넌트 등록
    // InitializeActorsForPlay
}
```

---

## 8. Component 초기화

InitializeActorsForPlay에서 각 Actor에 대해 컴포넌트 초기화 흐름이 진행된다.

```yaml
InitializeActorsForPlay
└ 각 Actor
   ├ PreInitializeComponents
   ├ InitializeComponents
   │   └ 각 Component
   │       └ RegisterComponent
   │           ├ CreateRenderState   (렌더링 등록)
   │           └ CreatePhysicsState  (물리 등록)
   └ PostInitializeComponents
```

RegisterComponent 시점에 컴포넌트가 렌더링 시스템과 물리 씬에 등록된다.  
이 시점에서는 아직 게임 로직이 시작되지 않은 상태다.

PostInitializeComponents는 모든 컴포넌트가 초기화된 이후에 호출된다.  
액터 단위에서 추가 설정이 필요한 경우 이 시점에 처리한다.  
네트워크 게임에서는 레플리케이션 설정도 이 시점에 완료된다.

---

## 9. BeginPlay

모든 Actor와 Component가 초기화되면 월드는 BeginPlay를 실행한다.

```c++
void UWorld::BeginPlay()
{
    AGameModeBase* const GameMode = GetAuthGameMode();
    if (GameMode)
    {
        GameMode->StartPlay();
    }
}
```

```yaml
UWorld::BeginPlay
└ AGameModeBase::StartPlay
   └ AGameStateBase::HandleBeginPlay
      └ 각 Actor::DispatchBeginPlay
         └ Actor::BeginPlay
            └ 각 Component::BeginPlay
```

GameMode가 먼저 StartPlay를 통해 게임 진행을 시작한다.  
이후 레벨에 존재하는 Actor들의 BeginPlay가 순서대로 호출된다.  
Component의 BeginPlay는 Actor의 BeginPlay 내부에서 호출된다.

---

## 10. Tick 흐름

Tick은 매 프레임 실행되는 사이클이다.  
언리얼의 틱 시스템은 단순히 오브젝트를 순서대로 업데이트하는 방식이 아니다.  
TickGroup과 의존성 그래프를 통해 실행 순서를 제어한다.

### TickGroup

```c++
namespace ETickingGroup
{
    enum Type
    {
        TG_PrePhysics,       // 물리 전
        TG_StartPhysics,
        TG_DuringPhysics,
        TG_EndPhysics,
        TG_PostPhysics,      // 물리 후
        TG_PostUpdateWork,
        TG_NewlySpawned,
        TG_MAX,
    };
}
```

Actor와 Component는 자신의 Tick이 어느 그룹에서 실행될지 지정할 수 있다.  
기본값은 TG_PrePhysics다.

### FTickFunction

```c++
PrimaryActorTick.bCanEverTick = true;
PrimaryActorTick.TickGroup = TG_PrePhysics;
RegisterActorTickFunction(Level);
```

Actor와 Component는 생성 시점에 FTickFunction을 통해 FTickTaskManager에 등록된다.  
FTickFunction은 실제 Tick 콜백, 그룹, 의존성 정보를 포함한다.

같은 그룹 내에서도 실행 순서를 강제하려면 AddPrerequisite를 사용한다.

```c++
// CharacterMovement의 Tick이 Actor Tick 이후에 실행되도록 강제
CharacterMovement->PrimaryComponentTick.AddPrerequisite(this, PrimaryActorTick);
```

### UWorld::Tick

```c++
void UWorld::Tick(ELevelTick TickType, float DeltaSeconds)
{
    // TG_PrePhysics 그룹 실행
    TickTaskManager.RunTickGroup(TG_PrePhysics);

    // 물리 시뮬레이션 시작
    StartPhysicsSim();

    // TG_DuringPhysics 그룹 실행 (물리와 병렬)
    TickTaskManager.RunTickGroup(TG_DuringPhysics, false);

    // 물리 결과 동기화
    FetchResultsPhysics();

    // TG_PostPhysics 그룹 실행
    TickTaskManager.RunTickGroup(TG_PostPhysics);

    // TG_PostUpdateWork 그룹 실행
    TickTaskManager.RunTickGroup(TG_PostUpdateWork);
}
```

물리 시뮬레이션을 기준으로 Pre/Post가 나뉜다.  
TG_PrePhysics 그룹의 Tick이 모두 끝난 후 물리 연산이 시작되고,  
물리 결과가 나온 후 TG_PostPhysics 그룹이 실행된다.

### 전체 Tick 흐름

```yaml
EngineTick
└ FEngineLoop::Tick
   └ GEngine->Tick
      └ 각 UWorld->Tick
         ├ RunTickGroup(TG_PrePhysics)
         │   └ FTickFunction::ExecuteTick
         │       ├ AActor::TickActor → Tick()
         │       └ UActorComponent::TickComponent
         ├ StartPhysicsSim
         ├ RunTickGroup(TG_PostPhysics)
         │   └ FTickFunction::ExecuteTick
         │       └ Actor / Component Tick
         └ FetchResultsPhysics
```

UEngine::Tick에서는 등록된 UWorld들을 순서대로 Tick한다.  
에디터에서는 에디터 월드와 PIE 월드가 별도로 존재하기 때문에 Tick도 각각 실행된다.

---

## 정리

StartGameInstance 이후의 흐름을 크게 세 단계로 볼 수 있다.

첫 번째는 월드 생성 및 시스템 초기화다.  
UWorld 오브젝트가 생성되고, 물리·네비게이션·AI·서브시스템 등 월드 단위의 인프라가 구축된다.

두 번째는 레벨과 Actor 로드다.  
패키지에서 직렬화된 레벨 데이터를 역직렬화해 Actor 인스턴스들을 복원한다.  
컴포넌트가 등록되고, 렌더링과 물리 상태가 만들어지는 시점이다.

세 번째는 BeginPlay다.  
GameMode부터 시작해 Actor, Component 순서로 게임 로직이 시작된다.

이 과정이 끝나면 매 프레임 EngineTick이 실행되며,  
TickGroup 기반의 스케줄링에 따라 Actor와 Component의 Tick이 순서대로 처리된다.
