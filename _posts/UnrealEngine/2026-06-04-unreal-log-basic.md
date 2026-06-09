---
title: "언리얼 로그 기초"
date: 2026-06-04 14:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Basic]
tags: [UnrealEngine-Log]
description: "UE_LOG 사용법부터 커스텀 카테고리, 화면 출력까지"
---

## 로그가 필요한 이유

브레이크포인트를 걸기 어려운 멀티플레이, 틱 단위 연산, 비동기 처리 같은 상황에서 로그는 사실상 유일한 디버깅 수단이다. 언리얼은 `UE_LOG` 매크로와 출력 레벨(Verbosity) 시스템을 통해 로그를 체계적으로 관리할 수 있다.

---

## UE_LOG 기본 문법

```cpp
UE_LOG(카테고리, 레벨, TEXT("메시지 %s %d"), *SomeString, SomeInt);
```

| 인자 | 설명 |
|------|------|
| 카테고리 | 로그를 묶는 그룹 이름 |
| 레벨 | 출력 조건을 결정하는 심각도 |
| TEXT() | 유니코드 문자열 리터럴 래퍼 |

`%s`에 `FString`을 넘길 때는 반드시 `*`로 역참조해야 한다.

```cpp
FString Name = TEXT("Player");
UE_LOG(LogTemp, Warning, TEXT("이름: %s"), *Name);
```

---

## Verbosity 레벨

레벨은 심각도 순서로 아래로 갈수록 낮아진다. 카테고리에 설정된 레벨보다 낮은 메시지는 출력되지 않는다.

| 레벨 | 용도 |
|------|------|
| `Fatal` | 강제 크래시. 복구 불가능한 오류 |
| `Error` | 기능 고장. 반드시 수정 필요 |
| `Warning` | 잠재적 문제. 동작은 계속됨 |
| `Display` | 일반 정보. 빌드에서도 출력됨 |
| `Log` | 개발용 정보. 기본 레벨 |
| `Verbose` | 상세 추적용 |
| `VeryVerbose` | 매우 상세. 성능 영향 주의 |

실무에서 자주 쓰는 조합은 개발 중엔 `Log`, 릴리즈 직전 검토용엔 `Warning`, 치명적 오류엔 `Error`다.

---

## 커스텀 로그 카테고리 만들기

`LogTemp`는 편하지만 출력 로그가 많아지면 필터링이 어렵다. 기능 단위로 카테고리를 분리하면 관련 로그만 빠르게 볼 수 있다.

**헤더 파일 (.h)**

```cpp
// 외부에서 참조 가능하도록 선언
DECLARE_LOG_CATEGORY_EXTERN(LogMyGame, Log, All);
```

**소스 파일 (.cpp)**

```cpp
// 실제 정의 (한 번만)
DEFINE_LOG_CATEGORY(LogMyGame);
```

**사용**

```cpp
UE_LOG(LogMyGame, Warning, TEXT("커스텀 카테고리 로그"));
```

`DECLARE_LOG_CATEGORY_EXTERN`의 세 번째 인자(`All`)는 컴파일 타임에 허용할 최대 레벨이다. `All`로 두면 모든 레벨이 컴파일되고, `Warning`으로 설정하면 `Log` 이하는 빌드 자체에서 제거된다.

---

## 화면에 직접 출력하기

에디터 뷰포트나 게임 화면에 메시지를 띄울 때는 `GEngine->AddOnScreenDebugMessage`를 쓴다.

```cpp
if (GEngine)
{
    GEngine->AddOnScreenDebugMessage(
        -1,          // Key: -1이면 매 호출마다 새 줄, 양수면 같은 Key는 덮어씀
        5.f,         // 표시 지속 시간 (초)
        FColor::Red, // 색상
        TEXT("화면 출력 테스트")
    );
}
```

Key를 고정값으로 주면 틱마다 호출해도 같은 자리에서 값이 갱신되어 디버그 오버레이처럼 쓸 수 있다.

```cpp
// 매 틱마다 속도를 화면에 표시
GEngine->AddOnScreenDebugMessage(1, 0.f, FColor::Green,
    FString::Printf(TEXT("Speed: %.1f"), Velocity.Size()));
```

---

## 출력 로그 창 활용

에디터 상단 메뉴 **Window → Output Log** 에서 확인할 수 있다.

- **Filters** 버튼으로 카테고리 단위 필터링 가능
- 검색창에 카테고리명 입력하면 해당 로그만 표시
- 로그가 아예 안 보인다면 카테고리가 필터에서 꺼져 있거나, 해당 코드가 실행되지 않은 것

---

## 자주 하는 실수

**FString에 * 빠뜨리기**

```cpp
// 컴파일 오류
UE_LOG(LogTemp, Log, TEXT("%s"), MyString);

// 정상
UE_LOG(LogTemp, Log, TEXT("%s"), *MyString);
```

**GEngine nullptr 확인 생략**

에디터 밖(서버 빌드 등)에서는 `GEngine`이 null일 수 있으므로 항상 null 체크 후 호출한다.

**로그 레벨 착각**

`Display`는 `Log`보다 **높은** 심각도다. 로그가 안 보일 때 레벨을 올려서 확인하자.

---

## 정리

| 목적 | 방법 |
|------|------|
| 간단한 디버그 | `UE_LOG(LogTemp, Log, ...)` |
| 기능별 분류 | 커스텀 카테고리 선언 |
| 화면 표시 | `GEngine->AddOnScreenDebugMessage` |
| 로그 확인 | Output Log + 카테고리 필터 |

로그는 코드베이스 곳곳에 남기되, 릴리즈 전에 `Warning` 이상만 남기도록 정리하는 습관을 들이면 좋다.
