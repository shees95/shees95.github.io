---
title: "디자인패턴 GOF 2"
date: 2026-06-02 08:30:00 +0900
categories: [Dev, DesignPattern]
tags: [DesignPattern-GOF]
description: "GOF 패턴의 각 사용방법"
---

---

# GOF 패턴 - 심화

--- 

## 생성
어떻게 생성할 것인가

### Singleton
> 게임에서 객체를 하나만 만들겠다.
- GameManager
- AudioManager
- LogManager

### Factory Method
> Base 부모를 기반으로하고, 자식을 객체로 만들겠다.
- Enemy_Spider
- Enemy_Ant
- Enemy_Boss

### Abstract Factory
> 비슷한 객체끼린 한 번 더 묶어 추상화
* TrapBase
  * SpinTrapBase
    * SpinPlatform
    * SpinBar
  * ProjectileTrapBase
    * ArrowTrap
    * ThrowingBallTrap
  * MoveTrapBase
    * MovePlatform
    * PushingBarTrap
  * DropTrapBase
    * DropPlatform
    * DropBombTrap

### Builder
> 복잡한 객체를 단계적으로 생성
- 생성자에서 한 번에 입력 (X)
- .SetHP(), .SetAttack(), .Build() (O)

### Prototype
> 복사해서 생성

```cpp
  Monster* Clone();
  
  DuplicateObject();
```

---

## 구조
어떻게 연결할 것인가

### Adapter
> 호환되지 않는 인터페이스 연결

### Bridge
> 기능과 구현 분리
* Weapon
  * Sword
  * Gun
* Renderer
  * DX12
  * Vulkan

### Composite
> 트리 구조 표현
```c++
Actor
├─ Mesh
├─ Weapon
└─ Camera
```

### Decorator
> 기능을 동적으로 추가

```c++
  Character
    +
    PosionBuff
    +
    SpeedBuff
    +
    ...
```

### Facade
> 복잡한 시스템을 단순 인터페이스로 제공
외부
```c++
  GameSystem->StartGame();
```
내부
```c++
  LoadMap();
  CreatePlayer();
  InitUI();
  ...
```

### Flyweight
> 공통 데이터 공유
* 메쉬와 머테리얼은 공유
* 위치는 개별 보관  
  \-> 메모리 절약

## Proxy (※ 이 부분은 좀 더 공부가 필요 ※)
> 실제 객체에 바로 접근하지 말고, 대리자 이용


---

## 행위
상호작용 정의

### Observer
> 상태 변경 통지
```c++
  DECLARE_DYNAMIC_MULTICAST_DELEGATE
```

### Strategy
> 상태에 따라 동작 방식 변경
```c++
  MovementComponent
  -> Walk, Fly, Swim
```

```c++
  CharacterChange
  -> Controller Mapping Context
```

### Command (※ 이 부분은 좀 더 공부가 필요 ※)
> 명령을 객체화


### State
> 상태에 따라 동작 변경

```c++
  StateMachine
  -> ChangeAnimation
```

### Template Method
> 동작 흐름 고정, 세부 구현만 변경

```c++
  Attack()
  {
    BeforeAttack();
    DoAttack();
    AfterAttack();
  }
```

### Visitor (※ 지금은 잘 안쓰는 패턴 ※)
> 객체 구조 수정 없이 기능 추가  
> 모든 객체를 수용할 수 있게 오버라이딩된 공통 메소드를 갖는 클래스 생성


### Mediator
> 객체 간 직접 통신 제거  
> 객체의 상태를 UI, State 등 다른 서브 클래스에게 전달해줄 수 있는 부모 클래스를 갖게 함   
> Delegate와의 차이점으로는 어디로 데이터를 보내야할지 알고 있다


### Chain of Responsibility
> 요청을 순서대로 처리
```c++
  Input > UI > Character > World
```

### Iterator
> 컬렉션 순회
```c++
  for(auto& Item : Items)
```

### Memento
> 상태 저장/복원
```c++
  SaveGame();
  Undo();
  CheckPoint();
```

### Iterpreter (※ BP나 Behavior Tree 같은걸로 이미 구현이 되어있음 ※)
> 문법 해석



## 자주 봤던 문법 패턴
* Observer (Delegate) : 이벤트 기반
* Strategy            : 이동 방식 알고리즘 교체
* State               : Idle -> Active -> Cooldown
* Factory Metho       : 객체는 자식이 생성
* Singleton           : GameManager
* Command             : 
* Composite           : 컴포넌트 붙이기
* Facade              : 외부 호출은 단순하게





