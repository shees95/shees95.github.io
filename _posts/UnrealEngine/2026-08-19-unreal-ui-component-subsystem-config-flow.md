---
title: "언리얼 TIL - UI Component, Subsystem, Config를 이용한 UI 흐름"
date: 2026-08-19 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-UI]
tags: [UnrealEngine, UMG, UIComponent, Subsystem, DataAsset, TIL]
description: "PlayerController의 입력부터 Widget 생성과 표시까지 UI 계층별 역할과 처리 흐름 정리"
---

# UI 시스템 구조

UI 시스템은 `Widget`, `UI Component`, `UI Manager Subsystem`, `UI Config`를 이용해 하나의 흐름으로 구성했다.

각 객체가 입력 처리, 데이터 가공, 위젯 관리, 클래스 설정이라는 하나의 역할만 담당하도록 분리하는 것이 핵심이다.

![UI 시스템 시퀀스 흐름](/assets/img/unreal-ui-architecture/ui-sequence-flow.png)

## 전체 흐름

```text
PlayerController
    ↓ 입력 및 Pawn 변경 이벤트 전달
UI Component
    ↓ 데이터 가공 후 표시·숨김·생성·제거 요청
UI Manager Subsystem
    ↓ UI Config에서 WidgetClass 조회
UI Config DataAsset
    ↓ WidgetClass 반환
UI Manager Subsystem
    ↓ 위젯 생성 또는 캐시 재사용 후 화면에 표시
Widget
```

`PlayerController`는 입력을 감지해 `UI Component`로 전달한다.
`UI Component`는 입력에 필요한 데이터를 가공하고, 어떤 위젯을 어떤 상태로 변경할지 결정한다.

그 후 `UI Manager Subsystem`에 다음과 같은 처리를 요청한다.

- 위젯 생성 또는 제거
- 위젯 표시 또는 숨김
- `Visible` 상태 변경
- 필요한 데이터 전달

`UI Manager Subsystem`은 `UI Config`에서 요청에 맞는 위젯 클래스를 조회한다.
반환된 `WidgetClass`를 인스턴스화하거나 기존 위젯을 재사용하고 Viewport에 표시한다.

## 구성 요소별 역할

### PlayerController

플레이어 입력과 Pawn 변경처럼 UI 갱신의 시작점이 되는 이벤트를 감지한다.

컨트롤러가 위젯을 직접 생성하거나 제거하지 않고 이벤트를 `UI Component`에 전달한다.
따라서 입력 처리와 UI 구현이 직접 결합되지 않는다.

### UI Component

UI와 관련된 데이터와 상태를 관리하는 중간 계층이다.

- 컨트롤러에서 입력 이벤트 수신
- UI에 필요한 데이터 가공
- ViewModel 또는 델리게이트 바인딩
- 표시할 위젯과 상태 결정
- `UI Manager Subsystem`에 위젯 처리 요청
- 위젯에서 발생한 클릭 및 닫기 이벤트 처리

`UI Component`는 어떤 UI가 필요한지는 판단하지만, 위젯의 생성과 Viewport 관리는 직접 수행하지 않는다.

### UI Manager Subsystem

프로젝트의 위젯 인스턴스와 생명주기를 관리한다.

- `UI Config`에서 `WidgetClass` 조회
- 위젯 생성
- 생성된 위젯 캐시 및 재사용
- Viewport 추가
- 표시 및 숨김 처리
- 더 이상 필요하지 않은 위젯 제거

여러 객체가 각자 위젯을 생성하지 않고 서브시스템을 통해 요청하므로 위젯 관리 지점을 하나로 모을 수 있다.

### UI Config DataAsset

UI 식별자와 실제 위젯 클래스의 연결 정보를 저장한다.

```text
UI 식별자 → WidgetClass
```

위젯 클래스를 코드에 직접 지정하지 않고 Config에서 조회하므로 위젯 교체와 설정 변경이 쉬워진다.
새로운 위젯을 추가할 때도 관리 코드보다 Config 데이터를 중심으로 확장할 수 있다.

### Widget

최종적으로 화면을 표시하고 사용자의 UI 입력을 받는다.

표시할 데이터는 `UI Component`의 ViewModel이나 델리게이트를 통해 전달받는다.
버튼 클릭이나 닫기 같은 이벤트는 다시 `UI Component`로 전달한다.

Widget은 게임플레이 로직이나 위젯 생성 정책을 직접 처리하지 않고 화면 표현에 집중한다.

## 객체 관계

![UI 시스템 객체 관계](/assets/img/unreal-ui-architecture/ui-structure-flow.png)

객체 간 요청 방향은 다음과 같다.

1. `PlayerController`가 입력 이벤트를 `UI Component`에 전달한다.
2. `UI Component`가 `UI Manager Subsystem`에 위젯 표시 또는 제거를 요청한다.
3. `UI Manager Subsystem`이 `UI Config`에서 위젯 클래스를 조회한다.
4. `UI Manager Subsystem`이 위젯을 생성하거나 재사용해 화면에 표시한다.
5. `UI Component`가 ViewModel 또는 델리게이트를 통해 위젯에 데이터를 전달한다.
6. Widget의 클릭이나 닫기 이벤트가 `UI Component`로 돌아온다.

## 이 구조의 장점

- 입력 처리와 위젯 생성 코드가 분리된다.
- 위젯 인스턴스와 생명주기를 한곳에서 관리할 수 있다.
- 위젯 클래스가 Config에 모여 있어 교체와 확장이 쉽다.
- Widget이 화면 표현에만 집중하므로 게임플레이 로직과의 결합도가 낮아진다.
- 위젯 캐시를 이용하면 반복적인 생성과 제거 비용을 줄일 수 있다.

## 정리

이 UI 구조는 각 계층의 책임을 다음과 같이 나눈다.

| 구성 요소 | 책임 |
|---|---|
| `PlayerController` | 입력과 Pawn 변경 이벤트 전달 |
| `UI Component` | 데이터 가공, UI 상태 판단, 처리 요청 |
| `UI Manager Subsystem` | 위젯 생성, 캐시, 표시, 숨김, 제거 |
| `UI Config DataAsset` | UI 식별자와 WidgetClass 연결 |
| `Widget` | 데이터 표시와 사용자 UI 이벤트 전달 |

컨트롤러의 입력이 UI 컴포넌트로 전달되면 컴포넌트가 데이터를 처리하고,
UI 매니저 서브시스템이 Config에서 위젯 클래스를 찾아 화면에 표시하는 흐름이다.

이렇게 역할을 분리하면 UI가 늘어나도 입력, 데이터, 위젯 관리 코드가 한 객체에 몰리지 않는다.
