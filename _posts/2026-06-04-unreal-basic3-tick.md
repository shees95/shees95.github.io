---
title: "언리얼 Tick 사이클"
date: 2026-06-04 19:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Basic]
tags: [UnrealEngine, UnrealEngine/Tick]
description: ""
---

# 언리얼 클래스 제어

---

## 1. 클래스 생성

>Tool - New C++ Class - [부모 클래스 선택] 

public, private 설정하면 공개 여부를 설정할 수 있음

---

## 2. 전처리기 세팅
`#include "CoreMinimal.h"`
- 엔진 전역 타입, 매크로, 함수
- 맨 위에 포함

#include "GameFramework/Actor.h"`
- AActor 클래스 선언

`#include "Item.generated.h"`
- 언리얼 헤더툴이 자동 생성하는 코드를 포함하는 것
- 항상 마지막 줄에 위치해야함

---

## 3. `.h` 선언부 코드  

```c++
UCLASS()
class SPARTAPROJECT_API AItem : public AActor
{
		GENERATED_BODY()
	
public:
		// 생성자
		AItem();

protected:
	  // 액터가 월드에 스폰 (배치된) 직후 한 번만 호출
		virtual void BeginPlay() override;
		// 매 프레임마다 호출
		virtual void Tick(float DeltaTime) override;
};
```

`UCLASS`
- 리플렉션 시스템에 등록

`class SPARTAPROJECT_API AItem : public AActor`
- Actor를 상속

- 접두사 규칙 존재
  - `A` : Actor
  - `U` : Object
  - `F` : 구조체
  - `T` : 템플릿
  - `E` : 열거형
  - `I` : 인터페이스
  
- `ProjectName_API`
  - 클래스 모듈화  
  - 외부에서 사용 가능케 하는 매크로  

`GENERATED_BODY()`
- `UCLASS()`와 짝을 이루어, 언리얼 헤더툴 (UHT)이 자동 생성한 코드를 삽입해 주는 매크로

---

## 4. 컴포넌트 추가
`Component` : Actor의 역할, 속성 등을 갖도록 만들어주는 부픔 개념  

`Root Component`  
- 모든 Actor는 Root 컴포넌트를 갖아야함  
- Actor는 역할이 있는 대상이기 때문에 트랜스폼을 관리할 수 있는 Scene Component를 보통 루트로 설정함  

`Static Mesh Component` : 애니메이션이나 스켈레탈 본 없는 정적 3D 모델을 그리는 컴포넌트 

### 선언
```c++
  USceneComponent* Root;    // 씬 컴포넌트 선언
  UStaticMeshComponent* StaticMeshComp; // 스태틱 메쉬 컴포넌트 선언
```

### 생성
```c++
  CreateDefaultSubobject<T>(TEXT(""));  // 컴포넌트 생성 및 초기화
```

### 설정
```c++
  SetRootComponent(Root);  // 루트 컴포넌트를 생성한 씬 컴포넌트로 설정
  StaticMeshComp->SetupAttachment(Root);    // 스태틱 메쉬 컴포넌트를 Root에 부착 
```

이렇게 하면 구조적으로는 붙어있으나 Editor에서는 확인이 불가
리플렉션을 설정하지 않았기 때문

---

## 5. 메쉬, 머테리얼 경로로 설정

```c++
AMyActor::AMyActor()
{
  static ConstructorHelpers::FObjectFinder<UStaticMesh> MeshAsset(TEXT("/Game/Resources/Props/SM_Chair.SM_Chair"));
  if (MeshAsset.Succeeded())
  {
    StaticMeshComp->SetStaticMesh(MeshAsset.Object);
  }

  // Material을 코드에서 설정
  static ConstructorHelpers::FObjectFinder<UMaterial> MaterialAsset(TEXT("/Game/Resources/Materials/M_Metal_Gold.M_Metal_Gold"));
  if (MaterialAsset.Succeeded())
  {
    StaticMeshComp->SetMaterial(0, MaterialAsset.Object);
  }
}
```

`ConstructorHelpers::FObjectFinder<T>``
- 리소스를 경로 기반으로 로드
- 경로 작성은 /Game/~~~/ResourceType/ResourceCategory.ResourceName

---


## 6. 삭제
cpp 파일이 생성된 파일 경로에서 파일 삭제를 하면 클래스 삭제가 가능

> - 경로  
> C:\Users\shs\Documents\Unreal Projects\CH3_Personal_1\Source\CH3_Personal_1\Private  
> C:\Users\shs\Documents\Unreal Projects\CH3_Personal_1\Source\CH3_Personal_1\Public  
> - 파일명  
> MyActor.h, MyActor.cpp  

