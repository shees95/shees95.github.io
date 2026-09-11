---
title: "언리얼 DeepRaiders - 입장 예약과 퀵매치 접속 처리"
date: 2026-09-09 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, RoomService, QuickMatch, Networking, Async]
description: "동시 입장에 대비한 정원 예약과 토큰 발급 순서, 방 없음과 통신 실패를 구분하는 퀵매치 흐름"
---

# 입장 예약과 퀵매치

> 9월 7일~11일 주간 작업을 주제별로 나눈 회고다. 게시 날짜는 실제 구현일과 다를 수 있다. 최근 퀵매치 변경과 동시 입장 시나리오는 추가 실행 검증이 필요하다.

방 목록에서 인원수만 확인하고 접속시키면 동시에 들어오는 요청을 처리하기 어렵다.
4인 방에 3명이 있을 때 두 요청이 같은 빈자리 하나를 보고 통과할 수 있기 때문이다.

Master에서 실제 인원에 입장 처리 중인 예약을 더해 정원을 계산하도록 구성했다.

## 참가 조건을 한곳에 모으기

`RoomMasterCommandlet.cpp`의 `CanJoin()`에서 공통 참가 조건을 확인한다.

```cpp
bool CanJoin(const FRoom& Room) const
{
    return Room.bRegistered
        && !Room.bStopping
        && Room.Info.State == TEXT("Waiting")
        && FPlatformTime::Seconds() - Room.LastReport <= Settings->HeartbeatTimeout
        && Room.Occupied() < Room.Info.MaxPlayers;
}
```

등록 완료, 종료 여부, 대기 상태, heartbeat 유효성, 예약을 포함한 정원을 함께 검사한다.
Private 제외와 맵 필터는 퀵매치 검색 단계에서 추가한다.

## 비동기 발급 전에 자리 예약

접속 토큰 발급을 기다리는 동안 다른 입장 요청이 들어올 수 있다.
따라서 발급 완료 후가 아니라 발급 요청 전에 자리를 예약한다.

```cpp
// Reserve의 예약 생성 부분 발췌. 이후 비동기 접속 정보 발급을 요청한다.
const FString Slot = NewSecret();
FTicket& Ticket = Room.Tickets.Add(Slot);
Ticket.Deadline = FPlatformTime::Seconds() + Settings->ReservationTimeout;
```

```text
참가 조건 확인
    ↓
예약 생성
    ↓
비동기 접속 정보 발급
    ↓
Master·방·예약·요청 상태 재검증
    ↓
접속 정보 응답
```

콜백이 돌아올 때는 방이 종료됐거나 요청이 이미 끝났을 수 있다.
Master와 방의 존재, 예약 유지 여부, Waiting 상태, 요청의 유효성을 다시 확인하고 발급 실패나 상태 변경 시 예약을 제거한다.

접속하지 않은 예약은 `ReservationTimeout` 이후 회수한다.
토큰 검증 후에도 실제 플레이어 보고가 오기 전까지 예약을 유지해 그 사이에 자리가 다시 열리지 않도록 했다.

## 퀵매치에서 방이 없을 때

처음에는 참가 가능한 방이 없으면 Master가 새 서버를 자동 생성했다.
UI 요구사항에 맞춰 방이 없을 때 사용자가 생성 화면에서 선택하도록 변경했다.

| 결과 | 처리 |
|---|---|
| 참가 가능한 방 있음 | 예약 후 접속 정보 발급 |
| 참가 가능한 방 없음 | `no_joinable_room` 응답 후 생성 화면 표시 |
| Master 통신 실패 | 오류 처리 |

클라이언트의 응답 처리에서는 방 없음만 별도로 분기한다.

```cpp
if (Error == TEXT("no_joinable_room"))
{
    HandleCreateRoomClicked();
    return;
}
```

Master는 WBP를 알 필요 없이 검색 결과를 반환하고, 화면 전환은 클라이언트가 결정한다.
통신 실패를 빈 방 목록으로 취급하지 않도록 구분했다.

검색 모드는 맵에 관계없이 참가하는 `AnyMap`과 선택한 맵만 검색하는 `SelectedMap`으로 나눴다.
퀵매치는 Public이면서 등록과 heartbeat가 유효하고, 종료 중이 아닌 Waiting 방의 빈자리를 찾는다.

## 접속 정보 수신 후 이동

UI는 연결 중 상태와 버튼 활성화를 관리하고, Subsystem은 연결 정보 검증과 이동을 담당한다.
다음은 `ConnectToRoom()`의 검증 이후 이동 부분이다.

```cpp
// 연결 정보 검증 이후의 이동 부분만 발췌.
const FString Url = Connection.Endpoint + TEXT("?RoomToken=") + Connection.Token;
Controller->ClientTravel(Url, TRAVEL_Absolute);
```

GameServer는 전달된 토큰을 검증한다.
토큰을 포함한 전체 URL은 UI나 로그에 출력하지 않도록 한다.

`ClientTravel()` 호출 자체가 접속 성공은 아니다.
이후 버전 불일치, timeout, 토큰 거절 등이 발생할 수 있어 접속 시도와 완료를 나누어 확인해야 한다.

## 남은 검증

방이 없는 퀵매치에서 생성창만 열리고 서버가 자동 실행되지 않는지, 방이 있을 때 예약과 실제 접속이 완료되는지 확인해야 한다.
마지막 한 자리에 여러 클라이언트가 동시에 요청하는 경우도 실행 검증이 남아 있다.

<!-- 캡처: 방 없음→생성창, 방 있음→접속 흐름. 동시 입장 검증 시 실제 인원과 예약 수를 함께 기록하되 토큰은 제외. -->
