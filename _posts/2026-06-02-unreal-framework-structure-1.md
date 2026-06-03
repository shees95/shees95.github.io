---
title: "언리얼 프레임워크 구조 1"
date: 2026-06-02 17:15:00 +0900
categories: [UnrealEngine, Structure]
tags: [UnrealEngine-Structure]
---

# 언리얼엔진 프레임워크 구조 1

---

## 전체 흐름

```yaml
WinMain
↓
GuardedMain
↓
EnginePreInit
↓
CoreUObject 로드
↓
UClass 등록
↓
CDO 생성
↓
생성자 호출
↓
EngineInit
↓
GEngine 생성
↓
EngineTick
↓
UWorld Tick
↓
Actor Tick
```

---

# 1. WinMain

언리얼 엔진의 최초 진입점

> Engine\Source\Runtime\Launch\Private\Windows\LaunchWindows.cpp

## 호출 구조

```yaml
WinMain
└ LaunchWindowsStartup
   └ GuardedMain
```

LaunchWindows.cpp

```c++
int32 WINAPI WinMain(_In_ HINSTANCE hInInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ char* pCmdLine, _In_ int32 nCmdShow)
{
    int32 Result = LaunchWindowsStartup(hInInstance, hPrevInstance, pCmdLine, nCmdShow, nullptr);
    LaunchWindowsShutdown();

    return Result;
}
```

---

# 2. GuardedMain

실질적인 엔진 실행 흐름

Launch.cpp

```c++
int32 GuardedMain(const TCHAR* CmdLine)
{
    int32 ErrorLevel = EnginePreInit(CmdLine);

#if WITH_EDITOR
    if (GIsEditor)
    {
        ErrorLevel = EditorInit(GEngineLoop);
    }
#endif

    ErrorLevel = EngineInit();

    while (!IsEngineExitRequested())
    {
        EngineTick();
    }

    EditorExit();

    return ErrorLevel;
}
```

## 호출 흐름

```yaml
GuardedMain
├ EnginePreInit
├ EditorInit
├ EngineInit
└ EngineTick
```

---

# 3. EnginePreInit

엔진 실행 준비 단계

LaunchEngineLoop.cpp

```c++
int32 FEngineLoop::PreInit(const TCHAR* CmdLine)
{
    // Config 초기화
    // CommandLine 파싱
    // Platform 초기화
    // ModuleManager 초기화
    // PluginManager 초기화
    // Trace 초기화
    // CrashReporter 준비
    // Core Module 로딩
    // Startup Module 로딩
}
```

모듈을 로딩 및 실행할 곳은 LoadingPhase 형태로 관리
```c++

namespace ELoadingPhase
{
  enum Type
  {
  /** As soon as possible - in other words, uplugin files are loadable from a pak file (as well as right after PlatformFile is set up in case pak files aren't used) Used for plugins needed to read files (compression formats, etc) */
  EarliestPossible,
  
      /** Loaded before the engine is fully initialized, immediately after the config system has been initialized.  Necessary only for very low-level hooks */
      PostConfigInit,
  
      /** The first screen to be rendered after system splash screen */
      PostSplashScreen,
  
      /** Loaded before coreUObject for setting up manual loading screens, used for our chunk patching system */
      PreEarlyLoadingScreen,
  
      /** Loaded before the engine is fully initialized for modules that need to hook into the loading screen before it triggers */
      PreLoadingScreen,
  
      /** Right before the default phase */
      PreDefault,
  
      /** Loaded at the default loading point during startup (during engine init, after game modules are loaded.) */
      Default,
  
      /** Right after the default phase */
      PostDefault,
  
      /** After the engine has been initialized */
      PostEngineInit,
  
      /** Do not automatically load this module */
      None,
  
      // NOTE: If you add a new value, make sure to update the ToString() method below!
      Max
    };
  
    /**
     * Converts a string to a ELoadingPhase::Type value
     *
     * @param	The string to convert to a value
     * @return	The corresponding value, or 'Max' if the string is not valid.
     */
    PROJECTS_API ELoadingPhase::Type FromString( const TCHAR *Text );
  
    /**
     * Returns the name of a module load phase.
     *
     * @param	The value to convert to a string
     * @return	The string representation of this enum value
     */
    PROJECTS_API const TCHAR* ToString( const ELoadingPhase::Type Value );
  }
}
```

어떤 모듈은 언제, 어떤 플랫폼에서 로딩이 되는지, 시작이 되는지를 위 enum type으로 구분하여 동작시킨다
PreInit 에서는 GEngine에 필요한 대부분의 필수 모듈들이 탑재가 된다

Ex)
```c++
  IProjectManager::Get().LoadModulesForProject(ELoadingPhase::PreLoadingScreen)
```

---

# 4. CoreUObject 로드

PreInit 과정에서 CoreUObject 모듈이 로드된다.

LaunchEngineLoop.cpp

```c++
int32 FEngineLoop::PreInitPreStartupScreen(const TCHAR* CmdLine)
{
    LoadCoreModules();
    LoadPreInitModules();

    ProcessNewlyLoadedUObjects();

    LoadStartupCoreModules();
    LoadStartupModules();
}
```

---

## 모듈 로딩

LaunchEngineLoop.cpp

```c++
bool FEngineLoop::LoadCoreModules()
{
    return FModuleManager::Get().LoadModule(TEXT("CoreUObject")) != nullptr;
}
```

ModuleManager.cpp

```c++
IModuleInterface* FModuleManager::LoadModuleWithFailureReason(...)
{
    ...

    ModuleInfo->Module->StartupModule();

    ...
}
```

## 모듈 로드 과정

```yaml
LoadModule
↓
DLL 로드
↓
모듈 객체 생성
↓
StartupModule()
↓
내부 자료구조 초기화
```

---

# 4-1. Loading Phase

언리얼은 모듈을 단계별로 로드한다.

주요 단계

```text
PreEarlyLoadingScreen
PreLoadingScreen
Default
PostDefault
PostEngineInit
```

---

## 부팅 시간 측정

```c++
SCOPED_BOOT_TIMING("IProjectManager::Get().LoadModulesForProject(ELoadingPhase::PreLoadingScreen)");

IProjectManager::Get().LoadModulesForProject(ELoadingPhase::PreLoadingScreen);
```

언리얼은 모듈 로딩 과정에서

* 현재 로딩 단계
* 모듈 로딩 시간

을 함께 추적한다.

---

# 정리

```yaml
WinMain
↓
GuardedMain
↓
EnginePreInit
↓
CoreUObject 로드
↓
UClass 등록
↓
CDO 생성
↓
생성자 호출
↓
EngineInit
↓
GEngine 생성
↓
EngineTick
↓
UWorld Tick
↓
Actor Tick
```

생성자가 BeginPlay보다 먼저 호출되는 이유와, 생성자에서 GetWorld()를 사용하면 안 되는 이유는 CoreUObject의 CDO 생성 과정에서 찾을 수 있다.


---

# 5. CoreUObject

CoreUObject는 언리얼 객체 시스템의 핵심 모듈이다.

```yaml
Core
└ CoreUObject
   ├ UObject
   ├ UClass
   ├ Reflection
   ├ CDO
   ├ Package
   ├ Serialization
   ├ GC
   ├ Asset Reference
   └ Object Loading
```

포함 핵심 기능

* UObject
* UClass
* Reflection
* Garbage Collection
* Package
* Serialization

---

# 6. UClass 등록과 CDO 생성

모듈 로드 과정 중

```c++
ProcessNewlyLoadedUObjects();
```

가 호출된다.

이 과정에서

```yaml
새로운 UClass 등록
↓
Reflection 정보 구축
↓
CDO 생성
↓
생성자 호출
```

이 수행된다.

---

# 7. 생성자에서 주의할 점

생성자는 실제 게임 시작 시에만 호출되는 것이 아니다.

CDO 생성 과정에서도 호출된다.

```yaml
CoreUObject 로드
↓
UClass 등록
↓
CDO 생성
↓
생성자 호출
```

따라서 생성자 실행 시점에는

```text
GEngine
UWorld
GameMode
Actor
PlayerController
```

등이 존재하지 않을 수 있다.  
UCLASS() 를 사용해서 정의된 클래스의 생성자에 게임 플레이 관련 코드가 포함되어서는 안되는 이유중 하나이다.

## 7-1. 생성자에서 수행하는 작업

* 기본값 초기화
* Tick 설정
* Replication 설정
* CreateDefaultSubobject
* 컴포넌트 구조 구성

---

# 7-2. 생성자에서 CreateDefaultSubobject를 사용하는 이유

```c++
CreateDefaultSubobject()
```

는 단순히 컴포넌트를 생성하는 함수가 아니다.

실제로는 CDO에 액터 기본 구조 정의를 기록하는 것에 가깝다.

```yaml
액터 기본 구조 정의
↓
CDO에 기록
```

예시)


생성자에서 호출
```c++
Mesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Mesh"));
```

액터 구조 CDO에 기록
```yaml
AMyActor_CDO
└ Mesh_CDO
```

SpawnActor 시 복사, 생성이 실행
```yaml
CDO
↓
복사
↓
실제 Actor 생성
```

---

# 8. Engine Init

PreInit이 끝나면 Init()을 통해 실제 엔진 객체 생성 후 실행

LaunchEngineLoop.cpp

```c++
int32 FEngineLoop::Init()
{
    // GEngine 객체 생성
    GEngine = NewObject<UEngine>(GetTransientPackage(), EngineClass);
    
    //GEngine
    // Editor = UUnrealEdEngine : public UEditorEngine, public FNotifyHook
    // UEditorEngine : UEngine
    #IF WITH_EDIT : GEngine = NewObject<UEngine>(GetTransientPackage(), EngineClass);
    // Game = UEngine
    #IF !WITH_EDIT : GEngine = GEditor = GUnrealEd = NewObject<UUnrealEdEngine>(GetTransientPackage(), EngineClass);
    
    // 객체 초기화
    // 게임 인스턴스 초기화
    GEngine->Init(this);
    
    SetEngineStartupModuleLoadingComplete();
    
    // 엔진 시작
    // 게임 인스턴스 실행
    GEngine->Start();
}
```

# 9. 엔진 시작 전, 게임 인스턴스 생성 및 맵 초기화

엔진 초기화에서 게임 인스턴스 생성
게임 시작으로 StartGameInstace() 호출

GameEngine.cpp

```c++

  void UGameEngine::Init(IEngineLoop* InEngineLoop)
  {
    ...
    
    GameInstance = NewObject<UGameInstance>(this, GameInstanceClass);
		GameInstance->InitializeStandalone();
  }

```


# 10. Engine Start

이 시점부터 실제 엔진이 동작하기 시작한다.

GameEngine.cpp

```c++

  void UGameEngine::Start()
  {
    TRACE_CPUPROFILER_EVENT_SCOPE(UGameEngine::Start);
    UE_LOG(LogInit, Display, TEXT("Starting Game."));
  
    GameInstance->StartGameInstance();
  }

```

```yaml
EngineInit
├ Create GEngine
├ GEngine->Init()
└ GEngine->Start()
```

---

# 11. EngineTick

Launch.cpp

```c++
LAUNCH_API void EngineTick()
{
    GEngineLoop.Tick();
}
```

---

## Tick 구조

LaunchEngineLoop.cpp

```c++
void FEngineLoop::Tick()
{
    BeginExitIfRequested();

    GEngine->Tick(FApp::GetDeltaTime(), bIdleMode);

    FFrameEndSync::Sync(FFrameEndSync::EFlushMode::EndFrame);
}
```

---

## 호출 흐름

```yaml
EngineTick
└ FEngineLoop::Tick
   └ UEngine::Tick
      └ UWorld::Tick
         ├ Actor Tick
         └ Component Tick
```

---

## 스레드 동기화

프레임 종료 시점에 동기화 수행

* Game Thread
* Render Thread

---
