---
title: "언리얼 DeepRaiders - 방별 Dedicated Server 실행과 개발용 런처"
date: 2026-09-08 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, RoomService, DedicatedServer, Process, Packaging]
description: "포트 확보부터 서버 준비 보고까지 방 생성 흐름을 구성하고 에디터 서버 모드와 Master 실행기를 추가한 기록"
---

# 방별 서버 실행과 준비 완료 처리

> 9월 7일~11일 주간 작업을 주제별로 나눈 회고다. 게시 날짜는 실제 구현일과 다를 수 있다. 코드 발췌는 설명에 필요한 부분만 담았다.

RoomMaster가 방마다 독립적인 게임 서버를 실행하도록 구성했다.
핵심은 프로세스 시작과 클라이언트가 접속할 수 있는 시점을 구분하는 것이었다.

## 방 생성 순서

```text
CreateRoom 요청
    ↓ 요청 데이터와 MapId 검증
사용 가능한 포트 확보
    ↓
게임 서버 프로세스 실행
    ↓
서버 초기화·준비 보고·방 등록
    ↓
입장 정원 예약과 접속 정보 발급
    ↓
클라이언트 접속 시도
```

클라이언트는 임의의 맵 경로나 실행 인자 대신 `MapId`를 전달한다.
Master가 허용 목록에서 해당 키의 실제 맵 경로를 선택한다.

각 방은 월드와 메모리가 분리된 프로세스에서 실행된다.
한 방의 프로세스가 종료돼도 다른 방 프로세스는 독립적으로 유지되지만, 방마다 엔진 실행 비용이 들어가므로 `MaxRooms`로 개수를 제한했다.

## 프로세스 시작은 준비 완료가 아니다

`CreateProc()`에 성공해도 모듈 로딩, 맵 로딩, GameMode 생성, 네트워크 리스닝 등이 끝나지 않았을 수 있다.
따라서 서버의 준비 보고와 방 등록 이후 접속 정보를 전달하도록 했다.

또한 서버 준비 보고와 프로젝트의 지형 생성·경기 카운트다운 완료는 별개다.
실제 경기 참가 가능 시점은 프로젝트의 게임 상태와 연결해야 한다.

프로세스 실행 자체가 실패하면 미리 확보한 포트를 반환한다.
다음은 `RoomMasterCommandlet.cpp`의 실행 실패 처리 발췌다.

```cpp
if (!Room->Process.IsValid())
{
    Ports.Remove(Port);
    Reply->Fail(TEXT("server_launch_failed"), 503);
    return;
}
```

이미 실행된 방의 포트는 종료 요청만으로 반환하지 않는다.
프로세스 종료를 확인한 뒤 반환해 종료 중인 방과 새 방에 같은 포트를 배정하지 않도록 했다.

## 개발용 에디터 서버 모드

처음에는 패키징된 Server exe만 실행했지만, 개발 중에는 Server Target 빌드와 Cook·패키징을 반복하는 비용이 컸다.
`bUseEditorServer` 설정으로 에디터 서버 실행 경로를 추가했다.

```cpp
FString GetServerExecutable(const URoomServiceSettings* Settings)
{
    return Settings->bUseEditorServer
        ? FString(FPlatformProcess::ExecutablePath())
        : Settings->ServerExecutable;
}
```

| 설정 | 실행 파일 |
|---|---|
| `bUseEditorServer=True` | 현재 Master를 실행한 엔진 실행 파일 |
| `bUseEditorServer=False` | 설정한 패키징 서버 실행 파일 |

에디터 모드에서는 프로젝트 경로와 선택한 맵, `-server`, 방별 포트 및 관리 인자를 별도로 구성한다.
부모 Master의 `-run=RoomMaster`를 그대로 넘기지 않는다.

개념적인 실행 형태는 다음과 같다. 비밀값은 예시에서 생략했다.

```text
UnrealEditor-Cmd.exe "Project.uproject" /Game/Maps/SelectedMap -server -port=7100
```

공통 방 생명주기를 유지하면서 실행 방식만 선택할 수 있게 했다.
에디터 모드에서도 플러그인 코드 수정 후 Editor 모듈 빌드는 필요하며, 이 설정이 Master 자체를 자동 실행하지는 않는다.

## 더블클릭 실행기

긴 실행 명령을 매번 입력하지 않도록 `RoomMasterLauncher.cs`와 `BuildDedicatedServer.ps1`을 추가했다.
결과는 프로젝트 아래 `DedicatedServer` 폴더에 모았다.

```text
DedicatedServer/
├─ StartRoomMaster.exe
├─ EnginePath.txt
└─ WindowsServer/
   └─ 패키징된 서버와 콘텐츠
```

`StartRoomMaster.exe`는 `EnginePath.txt`의 엔진으로 Master를 시작하는 런처다.
현재 프로젝트와 엔진이 필요하며, 게임과 Master를 모두 포함한 독립 배포 파일은 아니다.

스크립트의 `-LauncherOnly` 옵션은 실행기만 생성하고, 옵션을 빼면 서버 패키징까지 수행하도록 구성했다.
패키징 서버를 옮길 때는 exe만 복사하지 않고 Cook된 콘텐츠를 포함한 패키지 폴더를 유지해야 한다.

<!-- 캡처: Master 실행 후 방별 포트로 서버가 생성되는 화면. 실행 성공과 준비 보고 시점을 구분하고 명령행의 비밀값은 가림. -->
