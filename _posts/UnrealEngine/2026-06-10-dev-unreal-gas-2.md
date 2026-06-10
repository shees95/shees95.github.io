---
title: "언리얼 GAS 사용법 2"
date: 2026-06-06 17:24:00 +0900
categories: [UnrealEngine, UnrealEngine-GAS]
tags: [UnrealEngine, UnrealEngine-GAS, UnrealEngine-ASC, UnrealEngine-AS, UnrealEngine-GA, UnrealEngine-GE]
description: "GAS를 쓰면서 알게된 추가적인 내용"
---

## AttributeSet

---

### AttributeSet 내에서 초기값 수정

```c++
#pragma once

#include "CoreMinimal.h"
#include "CH3_ASBase.h"
#include "CH3_ASHealth.generated.h"

UCLASS()
class CH3_PERSONAL_1_API UCH3_ASHealth : public UCH3_ASBase
{
	GENERATED_BODY()
	
public:
	UCH3_ASHealth();
	
	UPROPERTY(BlueprintReadOnly, Category="Attributes")
	FGameplayAttributeData HP;
	ATTRIBUTE_ACCESSORS(UCH3_ASHealth, HP);
	
	UPROPERTY(BlueprintReadOnly, Category="Attributes")
	FGameplayAttributeData MaxHP;
	ATTRIBUTE_ACCESSORS(UCH3_ASHealth, MaxHP);
	
};
```

이렇게 작성해보니 초기값을 어떻게 받아야할지 모르겠더라.

그래서 

```c++
#pragma once

#include "CoreMinimal.h"
#include "CH3_ASBase.h"
#include "CH3_ASHealth.generated.h"

UCLASS()
class CH3_PERSONAL_1_API UCH3_ASHealth : public UCH3_ASBase
{
	GENERATED_BODY()
	
public:
	UCH3_ASHealth();
	
	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Attributes")
	float DefaultMaxHP;
	
	UPROPERTY(BlueprintReadOnly, Category="Attributes")
	FGameplayAttributeData HP;
	ATTRIBUTE_ACCESSORS(UCH3_ASHealth, HP);
	
	UPROPERTY(BlueprintReadOnly, Category="Attributes")
	FGameplayAttributeData MaxHP;
	ATTRIBUTE_ACCESSORS(UCH3_ASHealth, MaxHP);
	
};
```

이렇게 수정해서 Default값은 에디터에서 설정하고, Attribute들의 Init을 따로 해주는 방식으로 진행함.

### 
