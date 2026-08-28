---
title: "언리얼 DeepRaiders - GPU 제조사별 복셀 연산 오류"
date: 2026-08-28 22:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, DeepRaiders]
tags: [UnrealEngine, DeepRaiders, Voxel, GPU, AMD, NVIDIA, RDG, JumpFlood, TroubleShooting]
description: "AMD에서는 정상이고 NVIDIA에서는 반대로 동작한 Voxel JumpFlood GPU 동기화 오류 추적과 해결"
---

# GPU에 따라 달라진 복셀 편집 결과

복셀 최적화 이후 팀원이 눈총 전용 Frustum 흡수 Tool을 추가했다.

기능을 개발한 팀원에게서는 정상적으로 동작했지만 다른 팀원들의 환경에서는
눈이 쌓이지 않거나, 바닥을 제거했는데 오히려 위로 튀어나오는 문제가 발생했다.

같은 코드와 같은 데이터인데 개발자마다 결과가 달랐다.

## 처음 의심한 커밋

전체 브랜치와 커밋을 하나씩 확인한 결과,
눈총 전용 Frustum 흡수 Tool이 추가된 커밋부터 문제가 드러났다.

![처음 문제를 의심한 Frustum Tool 커밋](/assets/img/deepraiders-voxel-gpu-sync/suspected-tool-commit.png)

하지만 변경 내용을 보면 기존 복셀을 찾고 제거하는 기능이 대부분이었다.
복셀 추가 기능까지 망가뜨리거나 제거 방향을 반대로 만들 만한 원인을 찾기 어려웠다.

## 효과 없었던 시도

로컬 환경 또는 빌드 캐시 문제라고 생각해 다음 작업을 진행했다.

- 복셀 캐시 초기화와 새로고침
- Unreal 캐시 파일 삭제
- 솔루션 정리
- 전체 재빌드
- 브랜치와 커밋별 동작 비교

결과는 같았다.

기능 개발자의 PC에서는 정상이고 다른 팀원들의 PC에서는 계속 잘못 동작했다.

## GPU 제조사 차이 발견

팀원들의 하드웨어를 비교하던 중 차이점을 발견했다.

- 정상 동작한 개발자: AMD GPU
- 잘못 동작한 팀원들: NVIDIA GPU

이전에 복셀 최적화를 논의할 때도 AMD GPU 사용자는 렉이 적고,
다른 환경에서는 프레임 드랍이 크다는 차이가 있었다.

최근 JumpFlood 거리장 연산을 CPU에서 GPU로 옮긴 변경까지 연결해서 확인했다.

![JumpFlood GPU 연산이 추가된 커밋](/assets/img/deepraiders-voxel-gpu-sync/jump-flood-gpu-change.png)

문제가 발생한 Tool 커밋은 원인을 만든 것이 아니라,
기존 GPU 연산 오류가 눈에 띄는 형태로 드러나게 한 커밋에 가까웠다.

## 잘못된 복셀 결과

눈을 추가해도 표면에 제대로 쌓이지 않았다.

![눈 추가가 정상적으로 적용되지 않은 결과](/assets/img/deepraiders-voxel-gpu-sync/snow-add-failure.png)

제거 기능은 더 명확했다.
바닥을 깎아야 하는데 복셀이 반대 방향으로 튀어나왔다.

![제거했지만 복셀이 튀어나온 결과](/assets/img/deepraiders-voxel-gpu-sync/remove-inverted-result.png)

처음에는 GPU 제조사에 따라 배열 순서나 연산 방향이 뒤집히는 것을 의심했다.
하지만 실제 원인은 배열의 정방향과 역방향 차이가 아니었다.

## CPU 전환으로 원인 확정

같은 JumpFlood 연산을 GPU 대신 CPU 경로로 실행하자 모든 팀원의 PC에서 정상 동작했다.

![CPU 경로에서 정상 동작한 복셀](/assets/img/deepraiders-voxel-gpu-sync/cpu-fallback-result.png)

이 결과로 다음 범위를 제외할 수 있었다.

- Frustum Tool의 제거 수식
- 복셀 캐시
- Unreal 로컬 캐시
- 빌드 결과 차이
- 네트워크 복제

입력 데이터와 상위 로직은 같고 GPU 경로에서만 결과가 달랐으므로,
JumpFlood Compute Shader의 리소스 사용과 동기화를 확인했다.

## 실제 원인: GPU 리소스 동기화

JumpFlood는 여러 Compute Pass가 같은 두 버퍼를 번갈아 사용하는 Ping-Pong 방식이다.

```text
Pass 1: Src 읽기 → Dst 쓰기
                    ↓ Swap
Pass 2: Src 읽기 → Dst 쓰기
                    ↓ Swap
Pass 3: Src 읽기 → Dst 쓰기
```

이전 구현은 직전 Pass가 쓴 버퍼를 다음 Pass에서 읽을 때
리소스의 읽기와 쓰기 상태 및 Pass 간 의존성을 충분히 명시하지 않았다.

구체적으로 다음 문제가 있었다.

- 읽기 전용 Source까지 UAV로 바인딩
- UAV Write 이후 SRV Read 전환이 불명확
- Compute Pass 사이의 Barrier와 Cache Visibility가 드라이버 동작에 의존
- GPU 결과를 CPU로 가져오는 Readback 동기화가 불안정
- CPU와 GPU의 Invalid Surface Position 기준 불일치

AMD 환경에서는 우연히 기대한 순서와 가시성이 유지됐지만,
NVIDIA 환경에서는 이전 Pass의 결과가 다음 Pass에 올바르게 보장되지 않았다.

그 결과 잘못된 Surface Position과 거리장이 만들어졌고,
추가와 제거가 반대로 보이는 복셀 편집 결과로 이어졌다.

## 1차 수정: SRV와 UAV 분리

Source는 읽기 전용 SRV, Destination은 쓰기용 UAV로 분리했다.

```cpp
SetSRVParameter(BatchedParameters, Src, SrcBuffer.SRV);
SetUAVParameter(BatchedParameters, Dst, DstBuffer.UAV);
```

직전 Pass에서 UAV로 기록된 Source Buffer는 다음 Dispatch 전에
`SRVCompute` 상태로 명시적으로 전환했다.

```cpp
RHICmdList.Transition(
    FRHITransitionInfo(
        SrcBuffer.UAV,
        ERHIAccess::UAVCompute,
        ERHIAccess::SRVCompute));
```

이 수정으로 AMD와 NVIDIA가 동일한 Resource Hazard와 Cache Visibility 규칙을 따르게 했다.

## 최종 수정: RDG 기반 JumpFlood

수동 RHI Buffer 관리만으로는 누락 가능성이 남아 있어 JumpFlood를 RDG 기반으로 변경했다.

각 Pass에 다음 의존성을 명시했다.

```text
Src Buffer → SRV Read
Dst Buffer → UAV Write
Pass 종료 → Src/Dst Swap
```

RDG가 Pass 사이의 Barrier와 리소스 상태 전환을 관리하므로
특정 GPU 드라이버의 암묵적인 처리에 의존하지 않게 됐다.

GPU 결과를 CPU로 가져올 때도 Source Buffer를 직접 Lock하지 않고
`FRHIGPUBufferReadback`의 Staging Buffer를 사용했다.

```cpp
FRHIGPUBufferReadback Readback(TEXT("Voxel.JumpFlood.Readback"));
AddEnqueueCopyPass(GraphBuilder, &Readback, SrcBuffer, NumBytes);
GraphBuilder.Execute();

RHICmdList.ImmediateFlush(EImmediateFlushType::FlushRHIThread);
Readback.Wait(RHICmdList, FRHIGPUMask::All());
```

마지막으로 CPU와 GPU가 동일한 Invalid Surface Position을 사용하도록 기준을 통일했다.

## 해결 결과

RDG 기반 연산과 Readback 동기화를 적용한 뒤 AMD와 NVIDIA에서 같은 결과가 나왔다.

- 눈 추가 정상 동작
- Frustum 흡수 정상 동작
- 제거 시 반대로 돌출되던 현상 해결
- GPU 최적화 경로 유지
- CPU Fallback 결과와 GPU 결과 일치

## 정리

이번 문제는 GPU 제조사별 배열 연산 방향 차이가 아니었다.

원인은 여러 Compute Pass가 공유하는 버퍼의 읽기와 쓰기 상태,
Barrier, GPU Readback을 명확하게 동기화하지 않은 것이었다.

CPU 전환은 성능을 포기한 임시 해결책이 아니라
상위 복셀 로직과 GPU 경로를 분리해서 원인을 확정하는 중요한 비교 실험이었다.

GPU Compute 결과가 제조사별로 다를 때는 수식만 보지 말고 다음 항목도 확인해야 한다.

- SRV와 UAV 사용 구분
- Pass 간 Resource Barrier
- Ping-Pong Buffer Swap
- GPU Cache Visibility
- CPU Readback Fence
- CPU와 GPU의 초기값 및 Invalid 값 일치

같은 셰이더 코드라도 동기화가 명시되지 않으면 특정 GPU에서만 정상으로 보일 수 있다.
