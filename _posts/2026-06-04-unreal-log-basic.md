---
title: "언리얼 로그 관련"
date: 2026-06-04 14:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Basic]
tags: [UnrealEngine-Log]
description: "로그에 대한 명령어 및 용법"
---

# 언리얼 로그 남기기

---

# 로그 카테고리 생성

```c++
  // .h 전처리기 부에 작성
  DECLARE_LOG_CATEGORY_EXTERN(LogName, Error/Warning/Display, All);
```

```c++
  // .cpp 실행부에 작성
  UE_LOG(LogName, Error, TEXT("카테고리 생성");
```

실행된 로그들은 필터링해서 볼 수 있음
만약 필터에 선언한 로그가 없다면, 실행되지 않았는지 의심이 필요

