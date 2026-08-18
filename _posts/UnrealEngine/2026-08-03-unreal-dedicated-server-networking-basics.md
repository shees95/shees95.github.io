---
title: "TPS 개발 : TIL — 데디케이트 서버, NetRole, RPC 기초 정리"
date: 2026-08-03 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Networking, UnrealEngine-DedicatedServer]
description: "데디케이트 서버 실행 옵션과 NetDriver/NetConnection, LocalRole·RemoteRole의 Authority/Proxy 대칭 구조, RPC의 오너십별 도달 범위와 Reliable/Unreliable, 변수 레플리케이션 등록 흐름까지 언리얼 네트워킹 기초 정리"
---

# TPS 개발 - TIL: 데디케이트 서버, NetRole, RPC 기초 정리

언리얼 네트워킹 기초를 네 갈래로 정리했다 — 데디케이트 서버 실행/커넥션, 액터의 권한을 나타내는 NetRole, 원격으로 함수를 실행시키는 RPC, 변수를 복제시키는 레플리케이션 등록 흐름.

---

## 1. 데디케이트 서버 실행과 커넥션

### 에디터 멀티플레이어 옵션

- **Run Under One Process : false** — 서버와 클라이언트가 각각 별도 프로세스로 분리된다. 분리하지 않으면 프로세스 간에 에셋이 공유돼버릴 수 있다. 디버깅할 땐 `true`(단일 프로세스)가 브레이크포인트 등 훨씬 다루기 편하다.
- **Launch Separate Server : true** — 서버 전용 창이 따로 뜬다.
- **Allow Late Joining** — 체크하면 게임이 이미 시작된 뒤에도 Join이 가능한 세션으로 바뀐다.
- **Net Mode : Play as Client**로 설정해서 서버 창과 별개로 클라이언트를 실행한다.

`IsLocalPlayerController()`로 지금 이 폰/컨트롤러가 **로컬 플레이어 소유인지**를 체크해서, 그 로컬 플레이어에게만 필요한 데이터(HUD 갱신 등)를 전달하도록 분기한다.

### NetDriver / ServerConnection으로 판별

```
NetDriver == nullptr        → 스탠드얼론
NetDriver != nullptr        → NetDriver->GetNetMode()로 서버/클라 판별

ServerConnection == nullptr → 서버
ServerConnection != nullptr → 클라이언트
```

스탠드얼론에서는 `NetDriver` 객체 자체가 생성되지 않는다. 멀티플레이어(리슨 서버 포함)로 실행되면 `NetDriver`가 생성된다.

### ClientConnection vs ServerConnection

- **서버 입장에서 클라이언트가 접속하면** → `ClientConnection`이 생성된다.
- **클라이언트 입장에서 서버에 접속하면** → `ServerConnection`이 생성된다.

서로가 서로에게 `UNetConnection` 객체를 하나씩 만들고, 이 `UNetConnection`을 통해 실제 통신이 이루어진다. `UNetConnection`은 `UNetDriver`가 관리한다.

![NetDriver와 NetConnection 구조 및 네트워크 실행 흐름](/assets/img/unreal-networking/netdriver-connection-flow.png)

액터에서 `GetNetConnection()`을 호출하면 **액터 → 오너 → 폰 → 컨트롤러 → 커넥션** 순으로 타고 올라가며 커넥션 유무를 반환한다. 즉 **컨트롤러가 서버 커넥션의 근간**이다.

별도 설정 없이 `UE_LOG`를 그대로 쓰면 데디케이트 서버 cmd 창에 로그가 출력된다.

---

## 2. NetRole — Authority와 Proxy의 대칭 구조

### Authority와 Proxy

- **Authority** — 해당 액터에 대해 실제 권한을 갖고 있는 쪽의 속성값. 데미지 적용, 상태 변경 등 중대한 로직은 Authority를 가진 쪽에서 실행된다.
- **Proxy** — Authority를 가진 액터가 다른 쪽에 복제됐을 때 그쪽이 갖는 속성값. **Proxy는 허상, 즉 복제된 사본**이다.

### LocalRole과 RemoteRole — 한 액터가 두 롤을 동시에 가짐

액터는 **LocalRole**(이 컴퓨터에서의 롤)과 **RemoteRole**(연결된 상대 컴퓨터에서의 롤)을 동시에 갖는다. 데디케이트 서버가 어떤 액터에 대해 `LocalRole = Authority`, `RemoteRole = Proxy`를 부여받았다면, 그 액터가 복제된 클라이언트 쪽에서는 정확히 반대로 `LocalRole = Proxy`, `RemoteRole = Authority`를 부여받는다.

```
서버   : LocalRole = Authority,  RemoteRole = Proxy
클라   : LocalRole = Proxy,      RemoteRole = Authority
```

서로가 서로에게 대칭되도록 롤을 주고받는 구조 — "내 LocalRole"은 항상 "상대의 RemoteRole"과 같다.

### NetRole 종류

| Role | 설명 | 예시 |
|---|---|---|
| `ROLE_Authority` | 진짜 주인 | 서버의 모든 Actor |
| `ROLE_AutonomousProxy` | 로컬 플레이어가 조종 | 내 클라이언트의 내 캐릭터 |
| `ROLE_SimulatedProxy` | 남이 조종 | 내 클라이언트의 다른 플레이어 캐릭터 |
| `ROLE_None` | 복제 안 됨 | - |

### HasAuthority() vs IsLocalController()

- **`HasAuthority()`** — "지금 이 코드가 서버에서 실행되고 있는가"를 체크. 서버 체크용.
- **`IsLocalController()`** — "이 컨트롤러의 오너가 로컬 컴퓨터인가"를 체크. 로컬 오너 체크용.

용도가 다르다 — 같은 함수 안에서도 둘 다 필요한 경우가 흔하다.

---

## 3. RPC — 호출PC와 실행PC를 분리하는 통신

**RPC(Remote Procedure Call)** — 호출한 PC와 실제로 실행되는 PC가 달라도 되게 해주는 통신 기법. 데디케이트 서버에서 함수를 실행 요청하면, 실제 실행은 클라이언트에서 일어나는 식이다. 사운드/파티클 같은 **일시적 효과**에 주로 쓴다.

### Call vs Invoke vs Run

- **Call** — 정적. 호출과 실행이 컴파일 타임에 정해짐(Direct)
- **Invoke** — 동적. 런타임 중에 호출과 실행이 정해짐(함수 포인터, 동적 바인딩, RPC 같은 개념. Indirect)
- **Call/Invoke**는 "호출"을, **Run**은 "실행"을 가리키는 용어

### 액터 오너십

- `bReplicated = true` + `SetOwner(컨트롤러)` → **Client-Owned Actor**(클라이언트가 소유한 액터) 패밀리가 된다.
- 스폰은 됐지만 특정 클라이언트가 소유하지 않으면 **Server-Owned Actor**(아이템, 몬스터 등)다.

클라가 호출하고 서버가 실행하게 하려면 `Server` 키워드를 쓴다.

- **Client-owned actor**: 클라 로컬 컨트롤러에 패밀리로 오너 세팅이 되어 있음
- **Server-owned actor**: 클라 로컬 컨트롤러에 패밀리로 오너 세팅이 되어 있지 않음

### RPC 도달 범위 — 오너십에 따라 달라짐

**서버에서 호출한 RPC**

| 액터 종류 | NetMulticast | Server | Client |
|---|---|---|---|
| Client-owned | 서버 + 모든 클라 실행 | 서버 실행 | 오너 클라이언트 실행 |
| Server-owned | 서버 + 모든 클라 실행 | 서버 실행 | 서버 실행 |

**클라이언트에서 호출한 RPC**

| 액터 종류 | NetMulticast | Client | Server |
|---|---|---|---|
| 호출한 클라가 오너 | 요청된 클라만 | 클라 실행 | 서버 실행 |
| 다른 클라가 오너 | 요청된 클라만 | 클라 실행 | 드랍 |
| Server-owned | 요청된 클라만 | 클라 실행 | 드랍 |

클라이언트가 소유하지 않은 액터(또는 다른 클라가 소유한 액터)에 대해 `Server` RPC를 호출하면 서버에서 **드랍**된다 — 오너십이 없는 액터를 통해 서버 로직을 실행시키는 걸 막기 위한 구조다.

**서버의 `NetMulticast`는 부하가 심하다** — 서버가 연결된 모든 클라이언트에 전송해야 하므로 남발하면 안 된다.

### WithValidation — 위변조 체크

`WithValidation`을 붙이면 `_Validate()`에서 먼저 위변조 여부를 체크하고, 통과해야 `_Implementation()`이 실행된다. 클라이언트가 보낸 값을 서버가 그대로 신뢰하지 않고 검증할 때 쓴다.

### Reliable vs Unreliable

- **Reliable** — 서버에서 꼭 실행돼야 하는 함수. 충돌, 데미지, 스폰 등 게임 상태에 직접 영향을 주는 로직.
- **Unreliable** — 유실돼도 괜찮은 함수. 사운드, 이펙트 등 연출성 처리.

---

## 4. 변수 레플리케이션 등록 흐름

RPC가 "함수 실행"을 복제하는 거라면, 변수 값 자체를 복제하려면 별도로 등록해줘야 한다. 순서는 다음과 같다.

1. **생성자에서 `bReplicates = true;`** — 이 액터 자체가 네트워크로 복제되는 액터임을 선언
2. **헤더에서 `UPROPERTY`에 `Replicated` 지정** — 복제 대상 변수를 표시

   ```cpp
   UPROPERTY(Replicated)
   int32 Health;
   ```

   함수를 복제하려면 `UFUNCTION`에 RPC 지정자(`Server`/`Client`/`NetMulticast`)를 붙이는 것과는 별개로, 변수는 `UPROPERTY(Replicated)`로 표시해야 한다.

3. **`GetLifetimeReplicatedProps`에서 `DOREPLIFETIME(ThisClass, 변수명);`로 등록**

   ```cpp
   void AMyActor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
   {
       Super::GetLifetimeReplicatedProps(OutLifetimeProps);

       DOREPLIFETIME(ThisClass, Health);
   }
   ```

`UPROPERTY(Replicated)`만 붙이고 `GetLifetimeReplicatedProps`에 등록을 안 하면 실제로는 복제되지 않는다 — `UPROPERTY`가 "복제 가능한 변수"라는 메타데이터를 선언하는 자리라면, `DOREPLIFETIME`은 "이 변수를 실제로 복제 목록에 넣는" 등록 절차라 두 단계 다 필요하다.

---

## 정리

- `NetDriver`/`ServerConnection` 유무로 스탠드얼론·서버·클라를 판별하고, 통신은 양쪽의 `UNetConnection`(서버는 ClientConnection, 클라는 ServerConnection)을 `UNetDriver`가 관리하며 이루어진다
- 액터는 LocalRole/RemoteRole 두 롤을 동시에 가지며, 서버-클라 양쪽에서 대칭(Authority↔Proxy)으로 부여된다. 서버 체크는 `HasAuthority()`, 로컬 오너 체크는 `IsLocalController()`
- RPC는 호출PC ≠ 실행PC를 가능하게 하는 기법이고, 액터가 Client-owned냐 Server-owned냐에 따라 `Server`/`Client`/`NetMulticast`의 실제 도달 범위가 달라진다 — 오너십 없는 액터의 `Server` RPC는 드랍된다
- 게임 상태에 영향 주는 로직은 Reliable, 연출성 로직은 Unreliable로 나눈다
- 변수 복제는 `bReplicates = true`(액터 단위) → `UPROPERTY(Replicated)`(변수 표시) → `GetLifetimeReplicatedProps`의 `DOREPLIFETIME` 등록(실제 복제 목록 추가), 세 단계가 모두 있어야 완성된다
