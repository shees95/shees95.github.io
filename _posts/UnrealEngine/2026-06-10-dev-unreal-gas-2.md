---
title: "언리얼 GAS 사용법 2"
date: 2026-06-10 23:24:00 +0900
categories: [UnrealEngine, UnrealEngine-GAS]
tags: [UnrealEngine, UnrealEngine-GAS, UnrealEngine-ASC, UnrealEngine-AS, UnrealEngine-GA, UnrealEngine-GE]
description: "어떤걸 써야하나?"
---

## 용도 정리

---

### ASC

#### 언제 사용 해야하나?
GAS 기능 하나라도 쓰는 액터
```yaml
+ 플레이어, 적, 보스 : 항상
+ 플랫폼, 트랩 : GE를 주고 받을 때

- 순수 환경 오브젝트 : 불필요
- 
```

### AttributeSet

#### 언제 사용 해야하나?
수치로 관리 되는 상태가 있을 때
```yaml
+ HP, 데미지, 스테미너

- 쿨타임, 지속시간 상태 플래그 : 태그로 처리, AS 불필요
- 플랫폼, 발사체 : 불필요 
```

### GameplayEffect

#### 언제 사용 해야하나?
상태 변경이 필요할 때
```yaml
+ 힐/데미지 적용
+ 버프/디버프
+ 스턴
+ 쿨타임

- 즉각적인 로직 처리는 함수로 호출하는게 나을 수 있음
```

### GameplayAbility
플레이어/AI가 발동시키는 행동
```yaml
+ 공격, 스킬, 점프, 대시 (입력에 바인딩)
+ 마나(비용) 관리

- 일방적으로 가해지는 효과 : GE에서 직접 Apply
- 단순 환경 트리거 : 함수 호출하는게 나을 수 있음
```

