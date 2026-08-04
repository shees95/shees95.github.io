---
title: "TPS 개발 : TIL — 동적 컴포넌트 추가·제거 패턴"
date: 2026-08-04 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Component]
description: "CreateDefaultSubobject 없이 런타임에 NewObject로 컴포넌트를 새로 만들고 붙였다 떼는 패턴 — 추가는 Attach 후 Register, 제거는 Unregister 후 DestroyComponent"
---

# TPS 개발 - TIL: 동적 컴포넌트 추가·제거 패턴

지금까지는 컴포넌트를 생성자에서 미리 다 붙여놓고, 필요할 때 메시만 `SetStaticMesh(nullptr)`로 껐다 켰다 하는 식으로 처리했다. 근데 컴포넌트 자체를 런타임에 새로 만들었다 없앴다 하는 것도 가능하다는 걸 정리했다.

---

## 1. 동적 컴포넌트 추가

```cpp
void AMyActor::AddDynamicMeshComponent()
{
    // 1. NewObject로 생성 (CreateDefaultSubobject 아님!)
    UStaticMeshComponent* NewMesh = NewObject<UStaticMeshComponent>(this, TEXT("DynamicMesh"));

    // 2. 설정
    NewMesh->SetStaticMesh(SomeMesh);
    NewMesh->SetRelativeLocation(FVector(100, 0, 0));

    // 3. Attach (Register 전에!)
    NewMesh->AttachToComponent(RootComponent, FAttachmentTransformRules::KeepRelativeTransform);

    // 4. Register 필수!
    NewMesh->RegisterComponent();
}
```

### 순서가 중요한 이유

- **`CreateDefaultSubobject`는 생성자 전용** — 런타임에 새 컴포넌트를 만들 땐 `NewObject`를 쓴다.
- **`AttachToComponent`는 반드시 `RegisterComponent()` 전에** — `RegisterComponent()` 시점에 컴포넌트의 Transform이 확정되기 때문에, Attach를 먼저 해서 부모-자식 관계와 상대 Transform을 잡아준 다음 Register해야 한다. 순서가 바뀌면 위치가 틀어지거나 씬에 제대로 반영되지 않을 수 있다.
- **`RegisterComponent()`를 빼먹으면** 컴포넌트가 생성만 되고 실제로 씬에 등록되지 않아서 렌더링/충돌 등이 전혀 동작하지 않는다.

동적으로 만든 컴포넌트를 나중에 참조해야 한다면(제거하거나 값을 계속 바꿔야 한다면) `UPROPERTY`로 포인터를 들고 있어야 한다 — 로컬 변수로만 두면 함수 끝나고 GC 대상이 될 수 있다.

## 2. 동적 컴포넌트 제거

```cpp
void AMyActor::RemoveDynamicMeshComponent()
{
    if (!DynamicMesh)
    {
        return;
    }

    // 1. Unregister — Register의 반대, 씬에서 등록 해제
    DynamicMesh->UnregisterComponent();

    // 2. Destroy — Detach + GC 대상 등록
    DynamicMesh->DestroyComponent();
    DynamicMesh = nullptr;
}
```

- 추가가 `Attach → Register` 순이었다면, 제거는 그 대칭으로 **`Unregister`를 먼저** 해줘야 한다. 등록 해제 없이 바로 파괴하면 씬 그래프/렌더 상태에 등록된 채로 파괴 절차를 타게 되는 셈이라, Register와 짝을 맞춰 Unregister를 명시적으로 호출한다.
- 그다음 `DestroyComponent()`로 부모로부터 Detach하고 가비지 컬렉션 대상으로 넘긴다.
- 포인터를 들고 있었다면 **`nullptr`로 정리**해서 이후 코드가 죽은 컴포넌트를 참조하지 않도록 한다.
- 여러 개를 배열로 관리 중이라면 배열에서도 같이 제거해야 한다(안 그러면 댕글링 포인터가 배열에 남는다).

### CreateDefaultSubobject로 만든 컴포넌트는?

생성자에서 `CreateDefaultSubobject`로 만든 컴포넌트는 클래스 구조(CDO)의 일부라 원칙적으로 "항상 존재"하는 걸 전제로 설계된다. 이런 컴포넌트를 `DestroyComponent()`로 없애는 것도 기술적으로는 가능하지만, 보통은 `SetVisibility`/`SetActive`/메시 비우기 등으로 **끄고 켜는 쪽**을 쓰고, 진짜 개수가 가변적인 경우(장착 무기 개수가 매번 다르다 등)에 한해 `NewObject` + 동적 추가/제거 패턴을 쓰는 게 맞다.

---

## 정리

- 런타임에 컴포넌트를 새로 만들 땐 `CreateDefaultSubobject`가 아니라 `NewObject`
- 추가 순서: `NewObject` → 설정 → `AttachToComponent`(Register 전!) → `RegisterComponent()`
- 제거 순서: `UnregisterComponent()`(Register의 대칭) → `DestroyComponent()`(Detach + GC 등록) → 포인터 `nullptr` 정리
- 개수가 고정이면 생성자에서 미리 붙여두고 껐다 켰다, 개수가 가변적이면 동적 추가/제거 패턴 — 상황에 맞게 선택
