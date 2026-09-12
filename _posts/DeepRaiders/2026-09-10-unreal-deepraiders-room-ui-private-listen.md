---
title: "언리얼 DeepRaiders - 방 목록 UI와 Private Listen 연결"
date: 2026-09-10 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, UMG, ListView, Blueprint, ListenServer]
description: "Title에 방 관리 위젯을 연결하고 Public Dedicated와 Private Listen 경로를 분리하며 표시와 입력 복귀 문제를 정리"
---

# 방 목록 UI와 Public·Private 생성 연결

RoomService의 방 요청을 타이틀 UI에 연결했다.
C++에서는 요청과 데이터, 접속 상태를 관리하고 WBP에서는 배치와 스타일, 클릭 이벤트를 연결했다.

## 타이틀에 배치한 위젯 재사용

RoomService와 CreateRoom은 매번 생성하는 대신 Title에 미리 배치한 인스턴스를 재사용하도록 구성했다.

```text
WBP_Title
└─ Root Overlay
   ├─ Overlay_Title
   ├─ WBP_Join
   ├─ WBP_Settings
   ├─ WBP_RoomService
   └─ WBP_CreateRoom
```

`Overlay_Title`에는 타이틀 메뉴만 넣는다.
방 목록까지 그 안에 넣으면 타이틀 메뉴를 숨길 때 함께 숨겨지므로, 같은 루트 아래에서 별도로 표시하도록 배치했다.

## ListView 데이터와 행 위젯 분리

방 구조체를 ListView에 직접 넣는 대신 `UDRRoomListItem` UObject에 담아 전달했다.

```text
방 목록 응답
    ↓
UDRRoomListItem 생성
    ↓
ListView.SetListItems
    ↓
Entry에 방 데이터 적용
```

ListView 행은 재사용될 수 있으므로 `NativeOnListItemObjectSet()`에서 현재 항목을 적용하고,
`NativeOnEntryReleased()`에서 참조를 정리하도록 했다.

행 선택만으로 접속하지 않으며, 입장 버튼이 `HandleJoinClicked()`를 호출하도록 연결해야 한다.

## Public과 Private 실행 경로

| 구분 | 서버 실행 | 입장 |
|---|---|---|
| Public | Master가 Dedicated 서버 실행 | 예약과 토큰 기반 |
| Private | 사용자 PC에서 Listen 서버 실행 | 기존 IP 직접 접속 |

Private 생성은 기존 부모 클래스의 Listen 생성 경로를 재사용한다.

```cpp
if (PrivateCheckBox->IsChecked())
{
    Super::HandleCreateMapClicked();
    return;
}
```

이 분기를 Master의 맵 허용 목록 검증보다 먼저 처리해 Master 없이도 선택한 맵을 Listen으로 열 수 있도록 했다.
IP 접속 역시 기존 `JoinListenServer()` 경로를 사용한다.

현재 Private는 실행 방식 구분이다. Master 목록 등록, 방 코드, 중계, 비밀번호나 초대 인증은 구현하지 않았다.

## 위젯이 Visible인데 화면에 보이지 않는 문제

기존 `Open()`에는 위젯의 `IsVisible()`이 참이면 반환하는 코드가 있었다.
위젯 자체가 Visible이어도 부모가 숨겨져 있으면 화면에는 보이지 않을 수 있어, 필요한 표시 처리가 생략될 여지가 있었다.

해당 조기 반환을 제거해 열기 과정의 표시 처리가 진행되도록 수정했다.

## 필수 바인딩과 BP 접근 오류

맵 선택 위젯에서 다음 바인딩 오류가 발생했다.

```text
A required widget binding "ComboBoxString_ChoiceMap" was not found
```

필수 기능인 맵 선택 ComboBox는 Optional로 바꾸지 않고 WBP의 이름과 타입을 C++ 선언에 맞췄다.
상태 문구처럼 없어도 기능이 유지되는 위젯은 `BindWidgetOptional`로 구분했다.

또한 BP에서 `RoomId`, `RoomState` 등을 읽는 데 필요한 접근 지정이 누락되어,
해당 바인딩 변수에 `BlueprintReadOnly`를 추가했다.

## 접속창을 닫은 뒤 클릭이 막히는 문제

Private 접속창의 내부 Overlay만 숨기고 위젯 자체를 Visible로 남겨 다른 UI 입력을 가로채는 문제가 있었다.
복귀 처리를 다음과 같이 정리했다.

- 접속 위젯 전체를 `Collapsed`로 변경
- 방 목록 다시 표시
- 입력 모드와 포커스 복구
- Master 요청 취소 후 버튼 활성화 상태 갱신

UI 복귀에는 보이는 화면뿐 아니라 입력과 요청 상태까지 함께 정리해야 했다.
Master가 없는 Private 생성·IP 접속과 접속창 종료 후 입력 복구는 후속 실행 검증에도 포함했다.

