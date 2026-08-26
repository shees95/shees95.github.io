---
title: "언리얼 DeepRaiders - GameplayTag 기반 UI Manager"
date: 2026-08-19 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, UMG, GameplayTag, LocalPlayerSubsystem, DataAsset]
description: "GameplayTag와 UI Config를 이용해 위젯을 조회하고 관리하는 DeepRaiders UI 구조 정리"
---

# GameplayTag 기반 UI Manager

DeepRaiders의 UI는 `PlayerController`, `UI Component`, `UI Manager Subsystem`,
`UI Config`, `Widget`이 하나의 흐름을 이루도록 구성했다.

기존에는 위젯 클래스를 직접 전달했지만, 현재는 사운드 매니저와 마찬가지로
호출부에서 `GameplayTag`만 전달하면 Config에서 태그에 맞는 대상을 선택하는 방식으로 변경했다.

![UI 시스템 시퀀스 흐름](/assets/img/unreal-ui-architecture/ui-sequence-flow.png)

## 전체 흐름

```text
PlayerController
    ↓ 입력 이벤트 전달
UI Component
    ↓ PushScreen(UI Screen Tag)
UI Manager Subsystem
    ↓ UI Config에서 태그 조회
UI Config DataAsset
    ↓ WidgetClass, Layer, Order 반환
UI Manager Subsystem
    ↓ 위젯 생성 및 Viewport 등록
Widget
```

`UI Component`는 필요한 UI의 클래스 대신 다음과 같은 화면 태그를 전달한다.

```cpp
DRGameplayTags::UI_Screen_HUD
DRGameplayTags::UI_Screen_QuickSlot
DRGameplayTags::UI_Screen_Inventory_Player
DRGameplayTags::UI_Screen_Inventory_Storage
DRGameplayTags::UI_Screen_Teleport
DRGameplayTags::UI_Screen_Shop
DRGameplayTags::UI_Screen_Scoreboard
```

UI Manager는 태그를 키로 `UI Config`를 조회한 뒤 위젯 클래스와 표시 설정을 가져온다.
따라서 호출부는 어떤 Widget Blueprint가 연결되어 있는지 알 필요가 없다.

## 태그 기반 UI Config

UI Config는 화면 태그와 위젯 정의를 `TMap`으로 연결한다.

```cpp
USTRUCT(BlueprintType)
struct FDRUIScreenDefinition
{
    GENERATED_BODY()

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    TSubclassOf<UUserWidget> WidgetClass;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    EDRUILayer Layer = EDRUILayer::Menu;

    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)
    int32 Order = 0;
};
```

```cpp
UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Screens")
TMap<FGameplayTag, FDRUIScreenDefinition> Screens;
```

하나의 화면 태그에는 다음 정보가 함께 등록된다.

| 값 | 역할 |
|---|---|
| `WidgetClass` | 생성할 Widget Blueprint 클래스 |
| `Layer` | HUD, VFX, Menu, Modal 중 표시 레이어 |
| `Order` | 해당 레이어의 기준 ZOrder에 더할 값 |

조회는 전달받은 태그를 키로 사용한다.

```cpp
const FDRUIScreenDefinition* FindScreen(FGameplayTag ScreenTag) const
{
    return Screens.Find(ScreenTag);
}
```

사운드 매니저의 태그 기반 조회와 목적은 같지만 현재 UI Config는
부모 태그를 따라가는 계층형 Resolve가 아니라 `TMap::Find()`를 이용한 정확한 키 조회다.

## PushScreen

화면을 열 때는 위젯 클래스 대신 화면 태그를 전달한다.

```cpp
PlayerInventoryWidget = Cast<UDRInventoryScreenWidget>(
    UIManager->PushScreen(DRGameplayTags::UI_Screen_Inventory_Player));
```

`PushScreen()`은 다음 순서로 동작한다.

1. 화면 태그와 UI Config가 유효한지 확인한다.
2. 같은 태그로 관리 중인 위젯이 있으면 다시 표시한다.
3. 없다면 UI Config에서 `FDRUIScreenDefinition`을 조회한다.
4. `WidgetClass`, `Layer`, `Order`로 위젯을 생성한다.
5. 생성된 위젯을 화면 태그와 함께 `ActiveScreens`에 등록한다.

```cpp
UUserWidget* UDRUIManagerSubsystem::PushScreen(FGameplayTag ScreenTag)
{
    if (!ScreenTag.IsValid() || !IsValid(UIConfig))
    {
        return nullptr;
    }

    if (UUserWidget* ExistingWidget = GetScreen(ScreenTag))
    {
        SetManagedWidgetVisible(ExistingWidget, true);
        return ExistingWidget;
    }

    const FDRUIScreenDefinition* Definition = UIConfig->FindScreen(ScreenTag);
    if (!Definition || !Definition->WidgetClass)
    {
        return nullptr;
    }

    UUserWidget* Widget = CreateManagedWidget(
        Definition->WidgetClass,
        Definition->Layer,
        Definition->Order);

    if (IsValid(Widget))
    {
        ActiveScreens.Add(ScreenTag, Widget);
    }

    return Widget;
}
```

`ActiveScreens` 역시 태그를 키로 사용한다.

```cpp
TMap<FGameplayTag, TObjectPtr<UUserWidget>> ActiveScreens;
```

이를 통해 위젯 조회, 열림 상태 확인, 제거 요청도 모두 같은 화면 태그를 기준으로 처리할 수 있다.

## 화면 조회와 제거

화면이 열려 있는지는 태그로 확인한다.

```cpp
if (UIManager->IsScreenOpen(DRGameplayTags::UI_Screen_Inventory_Player))
{
    CloseInventoryScreen();
}
```

위젯을 제거할 때도 같은 태그를 사용한다.

```cpp
UIManager->PopScreen(DRGameplayTags::UI_Screen_Inventory_Storage);
```

`PopScreen()`은 `ActiveScreens`에서 태그에 대응하는 위젯을 찾고,
Viewport와 관리 컨테이너에서 제거한 뒤 입력 모드를 갱신한다.

플레이어 인벤토리처럼 자주 다시 여는 화면은 제거하지 않고 숨겨서 재사용할 수도 있다.

```cpp
UIManager->SetManagedWidgetVisible(PlayerInventoryWidget, false);
```

즉, 모든 화면을 자동으로 캐싱하는 구조는 아니며 각 UI의 생명주기에 따라
숨겨서 재사용할지 `PopScreen()`으로 제거할지 UI Component가 결정한다.

## 레이어와 입력 모드

UI Manager는 위젯 정의의 `EDRUILayer`에 따라 기준 ZOrder를 결정한다.

| 레이어 | 기준 ZOrder | 입력 처리 |
|---|---:|---|
| `HUD` | 0 | 게임 입력 유지 |
| `VFX` | 50 | 게임 입력 유지 |
| `Menu` | 100 | Game and UI |
| `Modal` | 200 | UI Only |

활성화된 Modal이 있으면 `FInputModeUIOnly`, Menu가 있으면
`FInputModeGameAndUI`, 둘 다 없으면 `FInputModeGameOnly`로 전환한다.

따라서 각 UI Component가 마우스 커서와 입력 모드를 개별적으로 설정하지 않아도
UI Manager가 현재 표시 중인 최상위 UI 레이어를 기준으로 일관되게 처리할 수 있다.

## 구성 요소별 역할

### PlayerController

플레이어 입력과 Pawn 변경처럼 UI 갱신의 시작점이 되는 이벤트를 감지하고
해당 기능의 UI Component에 전달한다.

### UI Component

- 입력 이벤트 수신
- UI에 필요한 데이터 가공
- 열거나 닫을 화면 태그 결정
- UI Manager에 `PushScreen()` 또는 `PopScreen()` 요청
- ViewModel 및 델리게이트 연결
- 위젯에서 발생한 클릭과 닫기 이벤트 처리
- 화면을 숨겨 재사용할지 제거할지 결정

UI Component는 어떤 UI가 필요한지는 판단하지만 실제 위젯 클래스와 생성 방법은 알지 못한다.

### UI Manager Subsystem

- 화면 태그로 UI Config 조회
- 위젯 생성과 Viewport 등록
- 태그별 활성 위젯 관리
- 위젯 표시, 숨김, 제거
- 레이어별 ZOrder 적용
- 활성 레이어에 맞는 입력 모드와 마우스 커서 갱신

`UDRUIManagerSubsystem`은 `ULocalPlayerSubsystem`이므로 로컬 플레이어별 UI 상태를 독립적으로 관리한다.

### UI Config DataAsset

화면 태그와 `WidgetClass`, `Layer`, `Order`의 연결을 보관한다.
위젯 Blueprint나 표시 순서를 바꿀 때 호출 코드를 수정하지 않고 DataAsset 설정을 변경할 수 있다.

### Widget

최종 화면 표시와 사용자의 UI 입력을 담당한다.
데이터는 UI Component의 ViewModel이나 델리게이트를 통해 받고,
클릭이나 닫기 이벤트는 다시 UI Component로 전달한다.

## 객체 관계

![UI 시스템 객체 관계](/assets/img/unreal-ui-architecture/ui-structure-flow.png)

## 사운드 매니저와의 공통점

두 시스템 모두 기능 코드와 실제 애셋 사이에 GameplayTag와 Config를 둔다.

```text
UI
화면 태그 → UI Manager → UI Config → WidgetClass

Sound
사운드 태그 → GC_Sound → Sound Library → Sound
```

호출부는 구체적인 Widget 또는 Sound 애셋을 직접 참조하지 않는다.
태그와 DataAsset의 연결만 변경해 실제 애셋을 교체할 수 있으므로 기능 코드와 콘텐츠의 결합도가 낮아진다.

## 장점

- 위젯 클래스를 기능 코드에서 직접 참조하지 않는다.
- 화면 조회, 생성, 상태 확인, 제거가 하나의 태그를 기준으로 통일된다.
- 위젯 클래스, 레이어, 표시 순서를 DataAsset에서 관리할 수 있다.
- UI Component와 UI Manager의 역할이 분리된다.
- 로컬 플레이어별 위젯과 입력 모드를 한곳에서 관리한다.
- 새로운 화면은 태그와 Config 정의를 추가하는 방식으로 확장할 수 있다.

## 정리

DeepRaiders UI Manager의 핵심은 **UI Screen GameplayTag + UI Config**다.

```cpp
UIManager->PushScreen(DRGameplayTags::UI_Screen_Inventory_Player);
UIManager->IsScreenOpen(DRGameplayTags::UI_Screen_Inventory_Player);
UIManager->PopScreen(DRGameplayTags::UI_Screen_Inventory_Player);
```

UI Component는 화면 태그만 전달하고, UI Manager가 Config에서 위젯 클래스와
레이어 정보를 찾아 생성부터 입력 모드 갱신까지 처리한다.

사운드 매니저와 동일하게 태그를 콘텐츠 선택의 기준으로 사용해
기능 코드와 실제 애셋의 직접 참조를 줄인 구조다.
