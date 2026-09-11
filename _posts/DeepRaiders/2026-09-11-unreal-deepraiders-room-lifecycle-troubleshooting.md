---
title: "언리얼 DeepRaiders - 방 생명주기 정리와 서버 실행 트러블슈팅"
date: 2026-09-11 00:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, RoomService, Heartbeat, Packaging, TroubleShooting]
description: "heartbeat와 접속 이력으로 빈 방을 정리하고, 프로세스 종료 후 포트를 회수하며 서버 빌드·실행 문제와 남은 검증을 정리"
---

# 방 종료와 서버 실행 문제 정리

> 9월 7일~11일 주간 작업을 주제별로 나눈 회고다. 게시 날짜는 실제 구현일과 다를 수 있다. 일부 패키징과 실행 결과는 확인했지만 최근 변경까지 전체 시나리오 검증을 완료한 것은 아니다.

방을 생성하고 입장시키는 흐름에 이어, 사용이 끝났거나 응답이 끊긴 방을 정리하도록 구성했다.
개발 중 겪은 빌드·모듈 로딩·접속·월드 준비 오류도 단계별로 정리했다.

## 상태 보고와 heartbeat

GameServer는 방 상태와 실제 플레이어 수를 Master에 주기적으로 보고한다.
관리 과정에는 `Starting`이 있고, 실행 중 보고 상태는 `Waiting → Playing → Ending`으로 구분했다.

`Ending` 이후 늦게 도착한 Waiting 보고가 방을 다시 열지 않도록 종료 상태를 되돌리지 않게 처리했다.
GameServer에서도 Master 응답 중단을 감시해 관리 주체와 분리된 서버가 계속 남는 상황을 줄이도록 했다.

작업 기록 기준 시간 설정은 다음과 같다.

| 설정 | 값 | 목적 |
|---|---:|---|
| `StartupTimeout` | 90초 | 서버 준비 제한 |
| `HeartbeatInterval` | 2초 | 상태 보고 간격 |
| `HeartbeatTimeout` | 15초 | 보고 중단 판단 |
| `ReservationTimeout` | 30초 | 입장 예약 만료 |
| `EmptyRoomTimeout` | 60초 | 미사용 빈 방 정리 |
| `EndingTimeout` | 10초 | 종료 상태 유지 제한 |
| `RequestTimeout` | 110초 | Master 요청 처리 기한 |

## 첫 접속 대기와 사용 후 빈 방 구분

인원이 0명이라는 이유만으로 종료하면 생성 직후 첫 접속을 기다리는 방도 사라진다.
실제 플레이어가 들어온 이력을 `bHadPlayers`에 기록했다.

```cpp
// 플레이어 보고를 받을 때 접속 이력을 유지한다.
Room.Info.CurrentPlayers = Count;
Room.bHadPlayers |= Count > 0;
```

주기적인 정리 단계에서는 예약까지 포함한 점유 인원을 확인한다.

```cpp
const bool bRoomVacated = Room.bHadPlayers && Room.Occupied() == 0;
```

접속 이력이 없는 빈 방은 준비와 첫 입장을 기다리며 timeout을 적용한다.
접속 이력이 있고 실제 인원과 예약이 모두 0이면 사용이 끝난 방으로 판단해 종료 처리한다.

Master가 클라이언트 종료를 직접 즉시 감지하는 것은 아니다.
GameServer의 인원 보고를 받은 뒤 판단하므로, 여기서 즉시 정리는 기존 빈 방 timeout을 추가로 기다리지 않는다는 의미다.

## 방 제거와 포트 회수의 순서

종료 중복을 막고 새 입장 대상에서 제외하기 위해 먼저 `bStopping`을 설정한다.
다음은 종료 진입 부분의 발췌다.

```cpp
if (Room.bStopping)
{
    return;
}

Room.bStopping = true;
Backend->RemoveRoom(Room.Info.RoomId);
```

이후 대기 요청에 실패를 전달하고 서버 프로세스를 종료한다.
실제 종료를 확인한 정리 단계에서 핸들, 포트, 방 데이터를 회수한다.

```cpp
// 프로세스 종료가 확인된 이후의 정리 부분.
FPlatformProcess::CloseProc(Room.Process);
Ports.Remove(Room.Port);
Rooms.Remove(Id);
```

```text
종료 상태 전환 → 목록에서 제외 → 프로세스 종료
    → 종료 확인 → 핸들·포트·방 데이터 정리
```

종료 요청 순간과 포트 재사용 시점을 분리해 아직 포트를 사용하는 프로세스와 충돌하지 않도록 했다.

## 주소 설정과 URL 오류

| 설정 | 의미 |
|---|---|
| `MasterUrl` | 클라이언트가 요청할 Master 주소 |
| `BindAddress` | Master가 수신할 인터페이스 |
| `AdvertisedHost` | 접속 정보에 넣을 게임 서버 호스트 |

로컬 설정 예시는 다음과 같다.

```ini
MasterUrl="http://127.0.0.1:7000"
BindAddress=127.0.0.1
AdvertisedHost=127.0.0.1
```

외부 접속을 받는 구성에서는 수신 인터페이스와 실제로 도달 가능한 주소를 구분해야 한다.
이번 구현의 `BindAddress`는 도메인을 허용하지 않으며, 모든 인터페이스 수신에는 `0.0.0.0`을 사용한다.
클라이언트용 Master 주소와 광고할 게임 서버 주소에는 접속 가능한 호스트를 설정한다.

작업 중 따옴표 없는 URL의 `//`가 주석으로 처리되어 `http:/rooms`로 요청하는 오류가 있었다.
URL을 따옴표로 감싸도록 설정을 정리했다.

## 빌드와 접속 문제를 단계별로 구분

### DLL 잠금

에디터를 닫아도 RoomMaster가 프로젝트와 플러그인 DLL을 사용하고 있으면 빌드가 파일을 덮어쓰지 못했다.
빌드 전에는 Master와 관련 서버 프로세스까지 종료해야 했다.

### BuildId와 NetworkVersion 불일치

소스 엔진과 런처 엔진을 번갈아 사용하면서 플러그인 DLL의 BuildId가 맞지 않는 모듈 로딩 문제가 발생했다.
서버가 실행된 뒤에는 서로 다른 NetworkVersion으로 접속이 거절되는 경우도 있었다.

```text
RemoteNetworkVersion=241800109
LocalNetworkVersion=3938116733
OutdatedClient
```

당시에는 소스 엔진 서버와 런처 엔진 클라이언트를 혼용하고 있었다.
모듈 로딩 실패와 클라이언트 접속 거절을 구분하고 양쪽 엔진 구성을 맞춰 확인할 필요가 있었다.

### 긴 경로와 응답 파일 오류

링커 응답 파일인 `.rsp`가 존재하는데도 열지 못하는 문제가 있었다.
작업 디렉터리와 상대 경로를 합치면 260자에 도달하는 상황이었고, UBA를 꺼도 재현됐다.

해당 경우 엔진 빌드 도구가 `.rsp`를 절대 경로로 전달하도록 수정한 뒤 후속 로그에서 빌드 성공을 확인했다.
엔진 전체 빌드에서는 소스 파일 경로 자체도 제한에 걸려, 엔진 루트를 짧게 두는 별도 대응이 필요했다.

### Cook 설정과 데이터 누락

서버 기본 GameMode가 이동 전 에셋을 가리켜 `GlobalDefaultServerGameMode`를 최종 에셋 위치로 수정했다.
`Shop_Buy`의 하드코딩 경로에는 빠진 `Item` 폴더를 반영했고,
`FDRItemDataTableRow::Category`에는 `EDRItemCategory::Ore` 기본값을 추가했다.

패키징된 맵에서는 복셀 가공에 필요한 Static Mesh CPU 데이터가 없어 다음 오류도 발생했다.

```text
Enable Allow CPU Access
Mesh voxel carve failed
```

프로세스 실행 성공만으로 게임 월드 준비까지 정상이라고 판단할 수 없었다.
CPU Access 오류의 해소 여부와 실제 복셀 가공 결과는 추가 확인 대상으로 남겼다.

## 이번 주 구현 상태와 남은 검증

방 목록·생성·입장 API, 방별 프로세스 실행, 준비 보고, 포트 회수, 예약과 토큰 검증,
heartbeat, Private Listen 분기, 에디터 서버 모드와 실행기를 소스에 구현했다.
퀵매치의 방 없음 처리와 사용 후 빈 방 정리도 변경했다.

패키징과 일부 실행 결과는 확인했지만, 최근 수정까지 자동화 테스트와 다중 클라이언트 검증을 완료한 것은 아니다.

| 검증할 상황 | 확인할 결과 |
|---|---|
| 빈 방 목록에서 퀵매치 | 생성창 표시, 자동 서버 생성 없음 |
| 참가 가능한 방의 퀵매치 | 예약과 자동 접속 완료 |
| 마지막 인원 퇴장 | 목록 제외, 프로세스 종료, 포트 회수 |
| 마지막 자리에 동시 입장 | 예약 포함 정원 초과 방지 |
| Server·Master 비정상 종료 | 남은 방·예약·자식 프로세스 정리 |
| Master 없는 Private | Listen 생성·IP 접속과 UI 입력 복구 |
| 패키징된 맵 실행 | 복셀 가공과 실제 월드 준비 완료 |

이번 작업에서는 버튼 입력부터 접속 완료까지를 하나의 성공 여부로 보지 않는 것이 필요했다.
Client 요청, Master 처리, 서버 준비, 접속 승인, 맵 준비를 나누어 로그를 연결해야 실제로 막힌 단계를 찾을 수 있었다.

<!-- 캡처: 마지막 인원 퇴장 후 목록·프로세스·포트 회수 순서. 검증 전 항목을 성공 화면처럼 제시하지 않고 실제 확보한 결과만 첨부. -->
