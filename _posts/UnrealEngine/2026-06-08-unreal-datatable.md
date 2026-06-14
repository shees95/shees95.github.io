---
title: "언리얼 DataTable + GameState + GameInstance 싱글게임 구조"
date: 2026-06-08 22:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Framework]
tags: [UnrealEngine, UnrealEngine-DataTable, UnrealEngine-GameState, UnrealEngine-GameInstance]
description: "DataTable로 데이터 관리, GameState로 게임 상태 관리, GameInstance로 레벨 간 데이터 유지하는 싱글게임 구조"
---

# DataTable + GameState + GameInstance

싱글게임에서 세 클래스의 역할 분리

| 클래스 | 역할 | 생존 범위 |
|---|---|---|
| `DataTable` | 아이템/스탯 등 정적 데이터 정의 | 에셋 |
| `GameState` | 현재 레벨의 게임 상태 (점수, 코인 등) | 현재 레벨 |
| `GameInstance` | 레벨 이동해도 유지되는 데이터 | 게임 전체 |

---

## 1. DataTable

게임 내 정적 데이터(아이템 스탯, 스폰 테이블 등)를 테이블 형태로 관리

### 1-1. Row 구조체 선언

```c++
// SpawnLevelTable.h
#include "Engine/DataTable.h"

USTRUCT(BlueprintType)
struct FSpawnLevelTableRow : public FTableRowBase
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 Level = 1;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TSubclassOf<AActor> SpawnClass;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 SpawnCount = 1;
};
```

### 1-2. 에디터에서 DataTable 생성

`Content Browser` → 우클릭 → `Miscellaneous` → `Data Table` → Row 구조체 선택

### 1-3. 코드에서 읽기

```c++
// DataTable 참조 선언
UPROPERTY(EditAnywhere, Category="Data")
UDataTable* SpawnTable;

// FindRow로 데이터 읽기
FSpawnLevelTableRow* Row = SpawnTable->FindRow<FSpawnLevelTableRow>(FName("Level_1"), TEXT(""));
if (Row)
{
    // Row->SpawnClass, Row->SpawnCount 사용
}
```

---

## 2. GameState

현재 레벨에서의 게임 진행 상태를 관리  
점수, 코인 수집량, 남은 시간 같은 "지금 이 판의 상태"를 여기에 저장

### 2-1. 선언

```c++
// CH3_GameState.h
UCLASS()
class ACH3_GameState : public AGameStateBase
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintReadWrite, Category="Game")
    int32 Score = 0;

    UPROPERTY(BlueprintReadWrite, Category="Game")
    int32 CoinCount = 0;

    UFUNCTION(BlueprintCallable, Category="Game")
    void AddScore(int32 Amount) { Score += Amount; }

    UFUNCTION(BlueprintCallable, Category="Game")
    void AddCoin() { CoinCount++; }
};
```

### 2-2. 어디서든 접근

```c++
ACH3_GameState* GS = GetWorld()->GetGameState<ACH3_GameState>();
if (GS) GS->AddScore(100);
```

---

## 3. GameInstance

레벨이 바뀌어도 데이터가 유지됨  
누적 점수, 플레이어 설정, 최고 기록 등 "게임 전체에 걸친 데이터"를 여기에 저장

### 3-1. 선언

```c++
// CH3_GameInstance.h
UCLASS()
class UCH3_GameInstance : public UGameInstance
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintReadWrite, Category="Save")
    int32 TotalScore = 0;

    UPROPERTY(BlueprintReadWrite, Category="Save")
    int32 BestScore = 0;

    UFUNCTION(BlueprintCallable, Category="Save")
    void SaveScore(int32 Score)
    {
        TotalScore += Score;
        BestScore = FMath::Max(BestScore, Score);
    }
};
```

### 3-2. 어디서든 접근

```c++
UCH3_GameInstance* GI = Cast<UCH3_GameInstance>(GetGameInstance());
if (GI) GI->SaveScore(GS->Score);
```

---

## 4. 전체 흐름

```
게임 시작
  └─ DataTable에서 스폰 데이터 로드 → SpawnVolume이 읽어서 아이템 배치

플레이 중
  └─ 코인 먹음 → GameState.CoinCount++
  └─ 적 처치 → GameState.Score += 100

레벨 클리어
  └─ GameState.Score → GameInstance.SaveScore()
  └─ 다음 레벨로 이동해도 TotalScore, BestScore 유지
```

---

## 에디터 설정

`World Settings` → `Game Mode` → GameState, GameInstance 클래스 지정 필수  
GameInstance는 `Project Settings` → `Maps & Modes` → `Game Instance Class` 에서 지정
