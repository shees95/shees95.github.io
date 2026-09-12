---
title: "언리얼 DeepRaiders - RoomMaster 기반 방 관리 구조"
date: 2026-09-07 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, RoomService, DedicatedServer, Plugin, Backend]
description: "직접 IP 접속에서 방 목록 기반 접속으로 확장하기 위해 RoomService 플러그인과 RoomMaster의 책임을 분리한 과정"
---

# RoomMaster 기반 방 관리 구조

기존 DeepRaiders는 클라이언트가 서버 IP로 직접 접속하는 구조였다.
타이틀에서 방을 조회하고 생성하거나 퀵매치로 참가하려면, 게임 서버 바깥에서 방을 관리할 주체가 필요했다.

이번 주에는 방 목록, 서버 프로세스 실행, 포트 할당, 입장 정원 예약을 담당하는 RoomMaster를 구현했다.
공통 기능은 `RoomService` 플러그인으로 분리하고 프로젝트의 UI와 게임 진행 상태를 연결했다.

## 서버 PC와 게임 서버 프로세스 구분

서버 PC 한 대에서도 여러 게임 서버 프로세스를 실행할 수 있다.
현재 구조는 방 하나당 Dedicated GameServer 프로세스 하나를 실행하는 방식이다.

```text
Client의 Title / RoomService WBP
    ↓
RoomServiceClientSubsystem
    ↓ HTTP 요청
RoomMaster
    ├─ 방 목록·검색·입장 예약·접속 정보 발급
    ├─ 프로세스·포트·방 생명주기 관리
    ├─ Room A → Dedicated GameServer :7100
    ├─ Room B → Dedicated GameServer :7101
    └─ Room C → Dedicated GameServer :7102
```

클라이언트는 Master에서 접속 정보를 받은 뒤 해당 GameServer로 이동한다.
실제 게임 통신은 클라이언트와 GameServer 사이에서 이루어지며, Master가 게임 패킷을 중계하지 않는다.

| 구성 요소 | 책임 |
|---|---|
| Client | 방 UI 표시, API 요청, 전달받은 서버로 접속 |
| RoomMaster | 방 검색·생성·예약, 서버 프로세스와 포트 관리 |
| Dedicated GameServer | 자기 방의 게임 실행과 상태 보고 |
| Backend | 인증·방 등록·검색·접속 정보 발급의 확장 지점 |

## 공통 기능은 플러그인으로 분리

방 정보 구조체, HTTP 통신, 프로세스 관리, 포트 풀, 예약과 토큰, 백엔드 추상화는 `Plugins/RoomService`에 모았다.
DeepRaiders 전용 WBP, 맵 이미지, 맵 선택 DataTable, 게임 규칙은 프로젝트에 남겼다.

| 주요 클래스 | 역할 |
|---|---|
| `URoomMasterCommandlet` | Master 실행과 관리 로직 |
| `URoomServiceClientSubsystem` | 클라이언트 API 요청과 접속 |
| `ARoomServiceGameModeBase` | 관리 방 초기화·상태 보고·접속 검증 |
| `URoomServiceSettings` | 실행 및 시간 설정 |
| `URoomServiceBackend` | 외부 백엔드 추상 계층 |
| `ULocalRoomServiceBackend` | 자체 Master용 기본 구현 |

다른 프로젝트에서도 공통 기능을 가져갈 수 있도록 구성했다.
다만 플러그인을 복사하는 것만으로 연결이 끝나지는 않는다. 해당 프로젝트의 GameMode에서 방 상태 보고와 접속 처리 경로를 연결해야 한다.

## 방 정보와 접속 권한 분리

`FRoomServiceInfo`는 생성 요청과 방 목록에 사용하는 공통 데이터다.

| 필드 | 의미 |
|---|---|
| `RoomId` | Master가 부여한 방 식별자 |
| `Title` | 방 이름 |
| `MapId` | 허용 맵 목록의 키 |
| `State` | 방 상태 |
| `CurrentPlayers`, `MaxPlayers` | 현재 인원과 최대 인원 |
| `bPrivate` | 비공개 여부 |
| `Attributes` | 게임 모드나 아이콘 식별자 등의 확장 데이터 |

생성 요청에서는 제목, 맵, 최대 인원을 전달하지만 방 식별자와 상태, 실제 인원은 Master가 결정한다.
클라이언트가 보낸 값을 실제 방 상태로 그대로 사용하지 않는다.

입장할 때는 별도의 `FRoomServiceConnection`을 전달한다.

```text
RoomId   → 입장할 방
Endpoint → 게임 서버 주소와 포트
Token    → 관리 방 접속용 일회용 토큰
```

방이 목록에 보인다는 사실만으로 입장 권한이 생기는 것은 아니다.
목록 표시 데이터와 실제 접속 정보를 분리해, 입장 조건 확인과 예약을 거친 뒤 접속 정보를 발급하도록 했다.

## 외부 백엔드를 연결할 지점

현재는 자체 Master의 로컬 백엔드를 사용한다.
EOS 같은 외부 서비스를 연결할 수 있도록 다음 기능을 `URoomServiceBackend`로 분리했다.

| 함수 | 역할 |
|---|---|
| `Authenticate` | 클라이언트 인증 |
| `PublishRoom`, `RemoveRoom` | 방 등록·갱신·제거 |
| `DiscoverRooms` | 방 검색 |
| `IssueConnection` | 접속 정보 발급 |
| `ValidateConnection` | 접속 토큰 검증 |

EOS SDK 연동 자체를 구현한 것은 아니다. 기본 인증도 로컬 개발용 익명 처리다.
외부 디렉터리를 연결하더라도 실제 프로세스와 포트, 정원 예약의 권한은 Master가 유지하도록 구성했다.
