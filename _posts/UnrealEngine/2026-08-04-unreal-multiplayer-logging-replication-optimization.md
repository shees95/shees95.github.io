---
title: "TPS 개발 : TIL — 멀티플레이 디버깅, 액터 초기화 순서와 레플리케이션 최적화"
date: 2026-08-04 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Networking, UnrealEngine-DedicatedServer]
description: "멀티플레이 디버깅용 에디터 옵션, GameMode→GameState→BeginPlay 호출 흐름과 PostNetInit/PostInitializeComponents 순서, ReplicateUsing·OnRep, NetUpdateFrequency·NetCullDistance·NetPriority·NetDormancy 등 레플리케이션 최적화 옵션, TearOff·Replication Graph/Iris·서브시스템까지 정리"
---

# TPS 개발 - TIL: 멀티플레이 디버깅, 초기화 순서, 레플리케이션 최적화

[전날]({% post_url UnrealEngine/2026-08-03-unreal-dedicated-server-networking-basics %}) RPC와 변수 레플리케이션 등록 흐름을 봤다면, 오늘은 멀티플레이 디버깅 환경, 액터가 초기화되는 순서, 그리고 레플리케이션 빈도/범위를 조절하는 최적화 옵션들을 정리했다.

---

## 1. 멀티플레이 디버깅용 에디터 옵션

**Launch Separate Server : false, Run Under One Process : true** — 이 조합이면 중단점과 로깅이 정상적으로 먹힌다. 서버 창을 따로 안 띄우고 단일 프로세스로 묶어야 디버거가 서버/클라 양쪽 코드에 다 걸린다.

## 2. 액터 초기화 순서

`GameMode`의 `StartPlay`가 `GameState`를 경유해서 모든 액터에게 `BeginPlay`를 호출하도록 되어 있다.

- **`PostNetInit()`** — 액터 속성 세팅이 완료되고 클라이언트에 복제되었을 때 호출. `PostNetInit` → `BeginPlay` 순서.
- **`PostInitializeComponents()`** — 액터에 부착된 컴포넌트가 모두 준비된 상태에서 호출.
- **`StartPlay()`** — 게임이 시작될 때 호출(로그인 절차와는 별개).
- **`BeginPlay()`** — 게임모드의 `StartGame` 함수를 통해 실제로 게임이 시작될 때 호출.
- **`PossessedBy`** — 빙의(소유) 시점, 컨트롤러-폰 패밀리가 구성되는 지점.

## 3. NetLoadOnClient

레벨에 고정적으로 배치되는 액터는 `NetLoadOnClient : true`로 설정해서, 서버가 복제해주는 게 아니라 클라이언트가 레벨 로드 시 **스스로 스폰**하도록 만들 수 있다. 정적 배치 액터까지 서버가 매번 복제해줄 필요가 없기 때문.

## 4. ReplicateUsing과 OnRep

```cpp
UPROPERTY(ReplicateUsing=OnRep_Health)
int32 Health;
```

`ReplicateUsing`을 붙이면 변수가 변경될 때마다 델리게이트처럼 지정한 함수가 호출된다.

### C++ OnRep vs 블루프린트 RepNotify 차이

| | C++ (`OnRep_`) | 블루프린트 (RepNotify) |
|---|---|---|
| 호출 대상 | 클라에서만 호출 | 서버·클라 모두 호출 |
| 명시적 호출 | 가능 | 불가능 |
| 호출 시점 | 값이 실제로 변경될 때만 | 항시 호출 |

C++ 쪽이 더 세밀하게 통제 가능하다 — 값이 바뀔 때만, 클라 쪽에서만, 필요하면 코드에서 직접도 호출할 수 있다.

## 5. NetUpdateFrequency

레플리케이션을 시도하는 **최대 횟수**(초당)를 정하는 값. 실제 갱신은 변경이 있을 때만 일어나지만, 이 값이 그 상한을 정한다.

## 6. 네트워크 보간

원격 액터의 움직임을 매끄럽게 보이려고 서버 갱신 사이 구간을 클라에서 예측 보간한다.

- `HasAuthority()` 체크로 서버 자신은 보간하지 않도록 분기
- 갱신 주기 세팅 — 기본적으로 0.01초마다 1회 하던 걸, 예를 들어 1초에 1회로 주기를 늘려서 테스트/최적화
- `LerpRatio`(갱신 진행도)를 세팅
- `LerpRatio`만큼 이동방향/이동속도/회전속도 등을 예측해서 보간

## 7. 거리 기반 연관성(Relevancy) 최적화

`NetCullDistance`, `NetRelevantFor`로 거리별 연관성을 체크해서 레플리케이션 여부를 최적화한다. 플레이어에게서 멀어지면 그 액터는 **아예 클라에서 사라진다** — 레플리케이션 자체를 하지 않기 때문.

## 8. NetPriority

레플리케이션 **우선순위**. 우선순위가 높다고 갱신 **빈도 자체**가 높아지는 건 아니다. 다만 패킷이 무거울수록(대역폭이 제한적일수록) 우선순위 높은 액터가 갱신될 확률이 상대적으로 높아진다.

## 9. NetDormancy — 휴면

`Never`, `Awake`, `Initial`, `DormantAll` 네 상태가 있다.

```cpp
SetNetDormancy(DORM_Initial); // 시작할 때 1회 레플리케이션되고 휴면됨
FlushNetDormancy();           // 휴면 해제(다시 레플리케이션 대상으로)
```

정적이거나 거의 안 바뀌는 액터를 계속 레플리케이션 대상으로 잡아둘 필요가 없을 때, 초기 1회만 복제하고 휴면시켜서 트래픽을 아낄 수 있다.

## 10. 라이프타임 컨디션

`GetLifetimeReplicatedProps`에 등록할 때 컨디션 옵션값(`COND_OwnerOnly` 등)으로도 "누구에게 복제할지"를 제어할 수 있다.

## 11. RPC와 액터 네트워크 라이프사이클

- 액터의 레플리케이션은 **레플리케이션 그래프의 생명주기**를 따라 관리된다 — 위치 기반 연관성, 항상 복제(Always Relevant) 등 조건에 따라 그래프 상에서 언제 복제 대상에 들어가고 빠지는지가 정해진다.
- **TearOff** — 서버가 해당 액터와의 네트워크 연결을 끊는 처리. 서버 쪽에서 더 이상 갱신을 보내지 않지만, **클라이언트에는 마지막으로 받은 상태가 그대로 남아있다.** 그 시점 이후로는 서버가 property를 바꾸거나 RPC를 보내도 클라에 반영되지 않으므로, 죽는 연출처럼 "이후엔 서버 개입 없이 클라가 알아서 마무리해도 되는" 상황에 쓴다.

## 12. NetDriver — Replication Graph vs Iris

NetDriver 레벨에서 레플리케이션 방식으로 **Replication Graph**를 쓸지 **Iris**를 쓸지 선택할 수 있다.

- **Iris** — 대규모 인원 처리, 캐싱, 안티 ESP(다른 클라이언트에 불필요한 정보가 새어나가는 걸 막는 것) 등에서 이점이 있다.

## 13. 서브시스템

- 언리얼에 이미 만들어져 있는 서브시스템들이 있고(`GameInstanceSubsystem`, `WorldSubsystem`, `LocalPlayerSubsystem` 등), 각각 생명주기가 정해져 있어서 용도에 맞는 걸 골라 쓴다.
- 액터로는 다루기 애매한 것(액터 생명주기를 넘어서는 데이터 저장 등)을 보통 서브시스템이 담당한다.
- 서브시스템에서 **Tick은 부하가 크므로 지양**하고, 이벤트/델리게이트 형식으로 처리하는 게 낫다.

---

## 정리

- 디버깅은 `Launch Separate Server : false` + `Run Under One Process : true` 조합으로 중단점/로깅을 확보
- 액터 초기화는 `PostNetInit` → `PostInitializeComponents` → `BeginPlay` 순으로, `GameMode.StartPlay`가 `GameState`를 경유해 전체 액터에 전파한다
- `ReplicateUsing`(C++)은 클라 전용·명시 호출 가능·변경 시에만 호출, 블루프린트 RepNotify는 서버/클라 모두·항시 호출로 성격이 다르다
- `NetUpdateFrequency`(빈도 상한), `NetCullDistance`/`NetRelevantFor`(거리 연관성), `NetPriority`(우선순위), `NetDormancy`(휴면), 라이프타임 컨디션까지 조합해서 레플리케이션 트래픽을 조절한다
- 액터는 레플리케이션 그래프의 생명주기를 따르며, TearOff로 서버 연결을 끊어도 클라에는 마지막 상태가 남는다 — NetDriver는 Replication Graph/Iris 중 선택 가능하고 Iris는 대규모 인원에 유리
- 서브시스템은 생명주기별로 이미 정해져 있고, Tick 대신 이벤트 기반으로 처리하는 게 낫다
