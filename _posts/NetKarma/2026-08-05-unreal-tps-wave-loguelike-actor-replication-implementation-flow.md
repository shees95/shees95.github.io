---
title: "TPS 개발 : 액터 레플리케이션 구현 흐름 — 생성형 지뢰(LandMine) 예제"
date: 2026-08-05 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Networking, UnrealEngine-DedicatedServer]
description: "F키로 생성되는 지뢰 액터를 예제로 스폰 레플리케이션(Server RPC+WithValidation), 오너십, NetCullDistance/Property Replication, 컴포넌트 복제, 디버깅 로그까지 실제 구현 흐름 정리"
---

# TPS 개발 - 액터 레플리케이션 구현 흐름: 생성형 지뢰 예제

[전날]({% post_url UnrealEngine/2026-08-04-unreal-multiplayer-logging-replication-optimization %}) 정리한 개념들을 실제로 어떻게 엮어서 쓰는지, F키를 누르면 그 위치에 지뢰 액터가 생성되는 기능을 예제로 구현 흐름을 정리했다.

---

## 1. 통신할 액터 구현

`bReplicates = true;`로 지뢰(`ALandMine`) 액터 자체를 복제 대상으로 만든다.

## 2. 액터 스폰을 레플리케이트

```cpp
// Header
UFUNCTION(Server, Reliable, WithValidation)
void ServerRPCSpawnLandMine();
```

```cpp
// cpp
void AMyCharacter::ServerRPCSpawnLandMine_Implementation()
{
    // 스폰 위치 지정 후 SpawnActor로 실제 스폰
    GetWorld()->SpawnActor<ALandMine>(LandMineClass, SpawnTransform);
}

bool AMyCharacter::ServerRPCSpawnLandMine_Validate()
{
    // 유효성 검사 — false면 실행이 막히는 것에 그치지 않고, 해당 커넥션이 조작 의심으로 처리됨
    return true;
}
```

`_Implementation`은 **Reliable이라서 필요한 게 아니라**, `Server`/`Client`/`NetMulticast` 중 어떤 RPC 지정자든 붙으면 공통으로 필요한 구조다. UHT가 원래 이름의 스텁 함수를 자동 생성해 네트워크로 마샬링해주고, 실제 로직은 `_Implementation`에 담는다. Reliable/Unreliable은 그 위에 얹히는 "재전송 보장 여부"일 뿐, `_Implementation`이 생기는 이유와는 별개다.

`_Validate`는 `WithValidation`이 있을 때만 추가로 생성되고, `bool`을 리턴해 실행 여부를 결정한다. 여기서 `false`를 리턴하면 단순히 이번 호출만 무시되는 게 아니라 **해당 클라이언트 커넥션이 조작 의심으로 강제 종료(kick)될 수 있는 반치트 장치**에 가깝다 — 그래서 정상적인 게임플레이 상황에서도 나올 수 있는 값을 거르는 용도로 쓰면 안 되고, "이 값은 절대로 정상 클라에서 나올 수 없다"고 확신할 수 있는 조건에만 써야 한다.

## 3. 오너 설정

`SetOwner()`를 해줘야 한다는 게 "오너가 있어야 레플리케이션이 되니까"는 아니다 — **오너 없이도 `bReplicates = true`면 액터 자체는 relevant한 모든 클라에 복제된다.** SetOwner가 필요한 진짜 이유는:

- 이 지뢰가 나중에 스스로 `Server` RPC를 쏴야 한다면(`GetNetConnection()`이 액터 → 오너 → 폰 → 컨트롤러 순으로 커넥션을 탐색하므로) 그 체인이 필요
- `COND_OwnerOnly` 같은 조건부 레플리케이션을 쓰려면 오너 기준이 필요

`Controller()`는 서버와 오너 클라이언트에만 존재하고(다른 클라에선 null) 다른 클라에는 복제되지 않으므로, 오너 정보를 걸어둘 땐 모두에게 복제되는 Character나 PlayerState 쪽에 엮어두는 게 안전하다.

## 4. 기본 동기화

### 4-1. NetCullDistance와 재초기화 문제

멀어지면 `EndPlay`, 가까워지면 다시 `BeginPlay`가 도는 식이라 계속 초기화되는 문제가 있다. 정확히는 서버의 원본 액터가 파괴되는 게 아니라, **연관성을 잃은 그 클라이언트에서만** 프록시가 파괴되고, 다시 relevant해지면 그 클라에서 새 프록시로 재스폰되며 `BeginPlay`가 다시 도는 것이다.

- **(X) `bAlwaysRelevant = true`** — 컬링 자체를 안 하게 만들어서 성능 부하만 늘 뿐, 근본 해결책이 아니다.
- **(O) Property Replication** — `bIsExploded`가 `DOREPLIFETIME`으로 정식 등록돼 있으면, 어떤 클라가 새로 relevant해지든(재접근이든 늦게 접속한 유저든) **그 시점의 현재값이 초기 리플리케이션에 실려서 전달된다.** 그래서 이미 터진 지뢰가 새로 relevant해진 클라에서 "안 터진 상태"로 잘못 보이는 문제가 해결된다.

```cpp
void ALandMine::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    DOREPLIFETIME(ThisClass, bIsExploded);
}
```

### 4-2. ReplicatedUsing

```cpp
UPROPERTY(ReplicatedUsing = OnRep_IsExploded)
bool bIsExploded;
```

## 5. 에임오프셋, 애니메이션 등 동기화

- 애니메이션 동기화는 캐릭터의 `Replicate Movement` 동기화 시 같이 이루어진다 — 위치/속도가 복제되면 AnimBP가 그 값을 읽어서 로코모션을 알아서 재현하기 때문에 별도 처리가 필요 없다.
- 에임 피치처럼 무브먼트 복제 범위 밖의 값은, 이전 값과 현재 값이 달라졌을 때만 갱신하도록 처리한다.

## 6. 공격 동기화

- 서버에서 공격/데미지 판정을 마친 뒤, 결과를 `OnRep`으로 명시해서 클라에 전달한다(RPC 남발 대신 상태값 복제 방식).
- 몽타주 노티파이가 여러 번 발동되는 문제는, 노티파이 타입을 **브랜칭 포인트(Branching Point)**로 바꾸면 몽타주 재생당 정확히 1번만 작동하도록 고칠 수 있다.

## 7. 최적화 및 UX 개선 — 지연 보상

클라이언트 입력 시점과 서버 처리 시점 사이의 네트워크 지연을 보정한다.

```
서버 딜레이 = 현재 서버 시간 - 인풋된 공격 시간(클라가 전달)
공격 재가능 시간 = 몽타주 재생 시간 - 서버 딜레이
```

이렇게 하면 핑이 있는 클라도 딜레이만큼 쿨다운을 앞당겨서 체감 반응성을 맞출 수 있다. 히트 판정도 **로컬에서 먼저 맞은 대상을 수집**하고 **서버에서 최종 검증**하는 방식으로 처리한다 — 단, 로컬 판정은 어디까지나 후보 수집용이고 실제 데미지 적용 여부는 반드시 서버가 재검증해야 한다(클라 값을 그대로 신뢰하면 안 된다는 원칙은 여기서도 동일).

## 8. 컴포넌트 동기화

```cpp
SetIsReplicatedByDefault(true);
```

이게 액터의 `bReplicates = true`와 완전히 같은 건 아니다. 이건 **컴포넌트 레벨 스위치**이고, 상위 **액터의 `bReplicates`가 꺼져 있으면 컴포넌트 쪽을 켜도 아무 것도 복제되지 않는다** — 액터 스위치가 선행 조건이다. 그 외 나머지 동작(`UPROPERTY(Replicated)`, `GetLifetimeReplicatedProps` 등록)은 액터와 동일한 레플리케이션 시스템을 그대로 탄다.

`SetIsReplicatedByDefault`는 생성자 전용이고, 런타임 중에 켜고 꺼야 한다면 `SetIsReplicated()`를 쓴다 — 이미 등록된 컴포넌트라면 네트워크 드라이버 쪽 재등록까지 처리해준다.

## 9. 네트워크 디버깅용 로그

`UActorComponent`에는 `GetLocalRole()`/`GetRemoteRole()`이 없다 — 컴포넌트는 자체 Role이 없고 `GetOwner()->GetLocalRole()`처럼 오너 액터의 Role을 빌려써야 한다. 그래서 컴포넌트 안에서 로그를 찍을 땐 `GetOwner()->GetLocalRole()` / `GetOwner()->GetRemoteRole()`을 문자열로 변환해서 로그 포맷에 끼워 넣는 매크로를 따로 만들어두면, `[Authority][SimulatedProxy] ...` 식으로 지금 이 로그가 어느 Role에서 찍힌 건지 한눈에 볼 수 있어 디버깅이 훨씬 수월해진다.

---

## 정리

- `_Implementation`은 Reliable 여부와 무관하게 모든 RPC 지정자에 공통이고, `_Validate`는 `WithValidation` 전용이며 실패 시 커넥션이 강제 종료될 수 있는 반치트 장치다
- `SetOwner`는 "레플리케이션 자체"가 아니라 액터 스스로 Server RPC를 쏘기 위한 커넥션 탐색과 `COND_OwnerOnly` 등 오너 기준 조건부 복제를 위해 필요하다
- NetCullDistance로 relevancy를 잃은 액터는 그 클라에서만 프록시가 재생성되는데, 이때 상태가 리셋되는 문제는 `bAlwaysRelevant`가 아니라 **Property Replication**(`DOREPLIFETIME`)으로 해결한다
- 공격/애니메이션 동기화는 RPC보다 상태값 복제 + `OnRep`을 기본으로 하고, 몽타주 노티파이 중복 발동은 브랜칭 포인트로 해결한다
- 지연 보상은 클라가 보낸 입력 시각과 서버 시각의 차이만큼 쿨다운을 보정하고, 히트 판정은 로컬 수집 + 서버 검증으로 처리한다
- 컴포넌트 복제는 액터 복제 위에 얹히는 하위 스위치라 액터의 `bReplicates`가 먼저 켜져 있어야 하고, 컴포넌트 로그는 오너 액터의 Role을 빌려서 찍어야 한다
