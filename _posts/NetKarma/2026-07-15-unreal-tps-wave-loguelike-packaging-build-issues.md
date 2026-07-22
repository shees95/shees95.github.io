---
title: "TPS 개발 : 패키징 빌드 이슈 모음"
date: 2026-07-15 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Packaging, UnrealEngine-Build, TroubleShooting]
description: "모듈 의존성, MSVC 버전, 리다이렉트 중복, 데이터 로드 실패까지 — 첫 패키징 시도에서 만난 빌드 이슈 모음"
---

# TPS 개발 - 패키징 빌드 이슈 모음

에디터에서는 멀쩡하던 프로젝트를 실제로 **패키징**하려니 문제가 줄줄이 튀어나왔다. 겪은 순서대로 정리한다.

---

## 1. 의존성 순서 문제로 모듈 삭제

빌드 중 모듈 의존성 순서 문제가 발생해서, 아래 모듈들을 정리(삭제)해야 했다.

```
UnrealEd
PropertyEditor
ToolMenus
LevelEditor
WorkspaceMenuStructure
```

전부 **에디터 전용 모듈**이다. 패키징(Shipping/Runtime) 빌드에는 필요 없는데 의존성에 걸려 있었던 것들이다.

---

## 2. GoogleSheetLoader의 에디터 의존도

CSV 파서로 쓰던 `GoogleSheetLoader` 플러그인이 **에디터 전용으로 만들어져 있어서** 다른 모듈들의 의존도가 높았다.

빌드 버전에서 이용 불가능한 헤더들을 `#if WITH_EDITOR`로 감싸서 빼내면, **에디터 의존도를 런타임 코드에서 분리**할 수 있다.

```cpp
#if WITH_EDITOR
    // 에디터 전용 CSV 로드 로직
#endif
```

에셋 매니저 쪽에서도 에디터 전용으로 뺀 구글 시트 데이터를 참조하고 있어서, 이 부분도 함께 정리가 필요했다.

---

## 3. MSVC 컴파일러 버전 문제

언리얼 5.7 버전부터는 **MSVC v143, 14.44** 버전이 필요하다.

`%APPDATA%\Unreal Engine\UnrealBuildTool\BuildConfiguration.xml`에서 빌드 툴 버전을 지정한다.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<Configuration xmlns="https://www.unrealengine.com/BuildConfiguration">
  <WindowsPlatform>
    <Compiler>VisualStudio2022</Compiler>
    <CompilerVersion>14.44.35207</CompilerVersion>
  </WindowsPlatform>
</Configuration>
```

`%USERPROFILE%\Documents\Unreal Engine\UnrealBuildTool\BuildConfiguration.xml`에도 동일한 설정이 필요하다.

### 함정 — 설정해도 다른 버전을 잡는다

config를 위처럼 설정해도, UBT가 14.44를 지정해주지 않고 **14.38을 계속 잡아버렸다.** 결국 **14.38 버전을 아예 삭제**해서 강제로 14.44만 남게 만들어 해결했다.

---

## 4. 패키징 시도 — 리다이렉트 중복 데이터

빌드가 완료되고 언리얼에서 패키징을 시도하니, 이번엔 **리다이렉트에서 중복 데이터**가 발견됐다.

```ini
+PrimaryAssetTypesToScan=(PrimaryAssetType="NKMInitPrimaryDataAsset", AssetBaseClass=".../NKMInitPrimaryDataAsset", Directories=(Path="/Game/NetKarma/Data"), ...)
+PrimaryAssetTypesToScan=(PrimaryAssetType="GoogleSheetConfig",       AssetBaseClass=".../GoogleSheetConfig",       Directories=(Path="/Game/NetKarma/Data"), ...)
```

→ 프라이머리 데이터 에셋은 **테스트용으로 만든 개별 경로**였다.

```ini
+ClassRedirects=(OldName="/Script/NetKarmaGame.MyClass", NewName="/Script/NetKarmaGame.NKMInitPrimaryDataAsset")
+ClassRedirects=(OldName="/Script/NetKarmaGame.MyClass", NewName="/Script/NetKarmaGame.UNKMLoadingViewModel")
```

→ 이 역시 테스트 단계에서 만든 개별 경로가 남아있던 것. 정리 대상이었다.

---

## 5. 데이터가 제대로 로드되지 않는 이슈

마지막으로, 패키징된 빌드에서 **데이터가 제대로 로드되지 않는** 문제가 남았다. 세 가지를 함께 적용해서 해결했다.

- 데이터 에셋에 **명시적으로 경로 작성**
- 패키징 설정을 **Always Cook**으로 변경
- 모듈 타입을 **Editor에서 Runtime**으로 변경

에디터에서만 로드되던 경로 해석이나 모듈 로딩 방식이, 패키징된 빌드에서는 다르게 동작한다는 걸 이번에 제대로 겪었다.

---

## 오늘 정리

- 에디터 전용 모듈(UnrealEd, PropertyEditor 등)은 의존성에서 제거
- `GoogleSheetLoader`처럼 에디터 전용 플러그인은 `#if WITH_EDITOR`로 런타임과 분리
- UE 5.7+는 **MSVC v143 14.44** 필요, BuildConfiguration.xml 지정해도 안 먹히면 구버전 컴파일러 자체를 삭제
- 리다이렉트/PrimaryAssetTypesToScan에 남은 **테스트용 항목**은 반드시 정리
- 데이터 미로딩 이슈는 **명시적 경로 + Always Cook + Runtime 모듈 전환** 세 가지로 해결

에디터 빌드는 관대하지만, 패키징 빌드는 모든 걸 명시적으로 요구한다는 걸 이번에 확실히 배웠다.
