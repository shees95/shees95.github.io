---
title: "언리얼 DeepRaiders - TIL: RoomMaster 요청 처리 흐름 이해하기"
date: 2026-09-13 00:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, RoomService, HTTP, Callback, TIL]
description: "UI에서 Master로 요청을 보내고 결과를 받는 흐름과 주요 함수·구조체의 역할을 간략히 정리"
---

# RoomMaster의 요청 처리 흐름 이해하기

RoomService 코드를 읽으면서 전체 구조를 손으로 정리했다.
세부 구현보다는 UI의 요청이 어디로 전달되고, 각 함수가 무엇을 담당하는지 중심으로 살펴봤다.

## 1. 전체 구조

```text
UI → ClientSubsystem → RoomMaster
                          ├─ 포트·프로세스·입장 예약 관리
                          ├─ Backend
                          │   ├─ 인증
                          │   ├─ 방 정보 등록·삭제·검색
                          │   └─ 접속 정보 발급·검증
                          └─ 방별 GameServer 관리
```

UI에서 방 목록 조회나 입장을 요청하면 ClientSubsystem이 Master에 전달한다.
Master는 요청에 맞는 작업을 처리하고 결과를 돌려준다.

`RoomMasterCommandlet`은 Master를 실행하고 HTTP 요청을 받을 준비를 하는 진입점이다.
그 안의 `FMaster`가 실제 서버 프로세스, 포트, 예약과 방 상태를 관리한다.
Backend는 인증과 방 정보 검색, 접속 정보 발급 등을 담당한다.

## 2. Master가 사용하는 구조체

| 구조체 | 역할 | 주요 내용 |
|---|---|---|
| `FReply` | 요청에 대한 응답 관리 | 응답 콜백, 응답 기한, 완료 여부 |
| `FTicket` | 입장 예약 관리 | 토큰, 만료 시각, 사용·활성 여부 |
| `FRoom` | 방 하나의 정보와 운영 상태 관리 | 방 정보, 프로세스, 포트, 대기 요청, 예약 목록 |

`FReply`는 처리 결과를 JSON 형태의 HTTP 응답으로 보내는 데 사용한다.
처리가 바로 끝나지 않아도 콜백을 보관해 두었다가 나중에 응답할 수 있다.

`FTicket`은 접속 중인 사용자의 자리를 확보하는 용도다.
실제 입장까지 시간이 걸리므로 현재 인원뿐 아니라 예약도 정원에 포함한다.

`FRoom`은 방을 운영하는 데 필요한 정보를 묶는다.
방에서 처리를 기다리는 요청은 `Waiting`, 입장 예약은 `Tickets`에 보관한다.

## 3. FMaster의 주요 함수

### Handle(Request, Complete)

요청을 받아 검사하고, 어떤 작업인지에 따라 처리 함수를 연결하는 입구다.

- `Request`: 상대방이 보낸 요청
- `Complete`: 처리 결과를 응답할 때 사용하는 콜백

```text
Handle
    ↓ 요청 검사
    ├─ 클라이언트 요청 → 사용자 인증 → HandleClient
    └─ 게임 서버 요청 → HandleServer에서 서버 확인 후 처리
```

`HandleClient`는 `list`, `create`, `join`, `quick` 등 클라이언트의 작업을 처리한다.
`HandleServer`는 `describe`, `admit`, `report` 등 게임 서버의 정보 조회, 접속 검증, 상태 보고를 처리한다.

`Handle()`의 반환값은 요청을 맡아 처리한다는 의미이며, 실제 작업 결과는 응답으로 전달한다.

### Tick()

시간이 지나면서 확인해야 할 요청과 방 상태를 주기적으로 관리한다.

- 응답 기한과 입장 예약 만료 처리
- 서버의 상태 보고가 끊겼는지 확인
- 빈 방이나 종료할 방 정리
- 서버 종료 확인 후 포트 회수
- 방 목록 변경 확인과 구독 응답

## 4. ClientSubsystem의 요청과 콜백

방 목록을 요청할 때는 다음과 같이 호출한다.

```cpp
Send(TEXT("list"), Object());
```

POST·GET으로 요청 성격을 구분하는 것처럼,
이 코드에서는 `Operation`의 `list`, `join` 등으로 수행할 작업을 구분한다고 이해했다.
실제 HTTP 전송 방식은 모두 POST를 사용하고, 세부 작업 이름을 JSON의 `op`에 담는다.

| Operation | 요청할 작업 |
|---|---|
| `list` | 방 목록 조회 |
| `create` | 방 생성 |
| `join` | 특정 방 입장 |
| `quick` | 참가 가능한 방 찾기 |

`Send(Operation, Body)`가 작업 이름과 필요한 데이터를 보내면,
Master가 이를 처리하고 클라이언트는 완료 콜백에서 응답을 받는다.
받은 데이터는 이벤트를 통해 UI에 전달한다.

## 5. 방 목록 요청을 따라가 보기

```text
클라이언트 UI에서 목록 요청
    ↓
ClientSubsystem이 list 요청 전송
    ↓
Master의 Handle → HandleClient에서 목록 처리
    ↓
FReply로 HTTP 응답 전송
    ↓
클라이언트 완료 콜백에서 목록 수신
    ↓
OnRoomListReceived → UI에 표시
```

크게 보면 **요청을 보내고 → 서버가 처리하고 → 콜백에서 결과를 받아 화면에 반영하는 구조**다.
네트워크로는 응답 데이터가 전달되고, 클라이언트의 콜백이 그 결과를 처리한다.

이번에는 각 함수의 세부 코드보다 전체 연결을 먼저 정리했다.
이 흐름을 기준으로 보면 요청이 막혔을 때도 전송, Master 처리, 응답 수신, UI 반영 중 어디를 확인할지 찾기 쉬울 것 같다.
