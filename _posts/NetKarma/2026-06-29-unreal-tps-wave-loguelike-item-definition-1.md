---
title: "TPS 개발 : 아이템 Definition 시스템 (1) — CSV에서 인스턴스까지"
date: 2026-06-29 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Inventory, UnrealEngine-Definition, UnrealEngine-DataTable, UnrealEngine-CSV]
description: "라이라 방식을 참고해 CSV → RawTableData → Definition → Instance 흐름으로 아이템 시스템 뼈대 잡기"
---

# TPS 개발 - 아이템 Definition 시스템 (1)

인벤토리 시스템을 만들기 시작했다. 아이템 하나를 화면에 띄우는 것보다, **아이템 데이터를 어떻게 정의하고 어디서 관리할 것인가**가 훨씬 큰 문제였다.

라이라(Lyra)가 이 문제를 푸는 방식이 깔끔해서 구조를 참고했다.

---

## 왜 Definition인가

처음엔 아이템마다 데이터 에셋을 하나씩 만드는 걸 생각했다. 하지만 아이템 종류가 늘어날수록 에셋 관리가 지옥이 된다.

- 밸런스 수정 한 번에 에셋을 수십 개 열어야 함
- 기획자가 수치를 바꾸려면 에디터를 켜야 함
- 아이템 추가 = 에셋 생성 노가다

그래서 **데이터 에셋 대신 CSV 테이블**을 데이터 소스로 삼기로 했다. 스프레드시트에서 수치를 관리하고, 그걸 언리얼로 끌어오는 구조다.

---

## 흐름 잡기

전체 파이프라인은 이렇게 정리됐다.

```
CSV (스프레드시트)
  └─ Parse → RawTableData (행 단위 원본 데이터)
       └─ Definition (필요에 맞게 쪼갠 정의)
            └─ ItemManager::Build → Item Instance (런타임 인스턴스)
```

핵심은 **각 단계가 역할이 다르다**는 것이다.

| 단계 | 역할 |
|---|---|
| CSV | 사람이 관리하는 원본 수치 |
| RawTableData | CSV를 파싱한 행 배열, 아직 의미 없음 |
| Definition | Raw를 "무기/탄/재료" 등 용도에 맞게 재해석 |
| Instance | Definition을 기반으로 실제 태어난 런타임 객체 |

---

## 1. CSV → RawTableData

CSV 문자열을 받아서 행 단위로 파싱하는 파서를 먼저 만들었다.

```cpp
// GoogleSheetParserBase.h
UCLASS(Abstract, BlueprintType, EditInlineNew)
class UGoogleSheetParserBase : public UObject
{
    // CSV 문자열 → 내부 행 배열로 변환
    bool Parse(const FString& RawCSV, FString& OutResult);

    // 특정 행 인덱스의 값 반환
    bool GetRowAt(int32 Index, TMap<FString, FString>& OutRow) const;

protected:
    // 자식이 행 처리 커스텀
    virtual void OnRowParsed(const TMap<FString, FString>& Row) {}

    TArray<TMap<FString, FString>> ParsedRows;  // 파싱된 원본 행들
    TArray<FString> Headers;                    // 컬럼명
};
```

`Parse`를 돌리면 CSV의 각 행이 `TMap<컬럼명, 값>` 형태의 **RawTableData**가 된다. 이 시점까지는 그냥 문자열 덩어리일 뿐, 이게 무기인지 탄인지는 아무도 모른다.

---

## 2. RawTableData → Definition

여기서부터가 Definition의 역할이다. Raw 데이터를 **용도에 맞게 쪼개서** 의미를 부여한다.

구조는 상속으로 잡았다.

```
UItemDefinition_Origin  (필수 공통 정의)
  ├─ ID, 이름, 아이콘, 설명 ...
  │
  ├─ UItemDefinition_Weapon   (무기용 재정의)
  ├─ UItemDefinition_Ammo     (탄용 재정의)
  └─ UItemDefinition_Material  (재료용 재정의)
```

- `Origin`은 **모든 아이템이 반드시 가지는 필수 데이터**를 담는다.
- 그 밑에서 무기인지, 탄인지, 재료인지에 따라 **필요한 필드를 다시 정의**한다.

무기 Definition은 데미지·연사속도 같은 걸 갖고, 탄 Definition은 탄 종류·효과를 갖고, 재료 Definition은 조합 특성을 갖는 식이다. 같은 CSV 행이라도 타입 컬럼에 따라 다른 Definition으로 해석된다.

> 이 "Origin + 타입별 재정의" 구조가 나중에 크래프팅·상점·무기로 확장할 때 계속 재활용된다.

---

## 3. Definition → Instance (ItemManager::Build)

재정의된 Definition은 아직 **설계도**일 뿐이다. 실제 게임에서 쓰려면 인스턴스로 태어나야 한다.

그 과정을 `ItemManager`의 **Build**가 담당한다.

```cpp
// 의사코드
UItemInstance* UItemManager::Build(const UItemDefinition_Origin* Def)
{
    // Definition 타입에 맞는 인스턴스 생성
    UItemInstance* Instance = NewObject<UItemInstance>(this);

    // Definition의 데이터를 인스턴스로 복사/초기화
    Instance->InitFromDefinition(Def);

    return Instance;
}
```

Build를 거치면서 Definition은 **하나의 인스턴스로 태어날 준비**를 마친다. 인벤토리에 실제로 들어가는 건 이 인스턴스다.

---

## 오늘 정리

- 데이터 에셋 대신 **CSV**를 데이터 소스로 채택
- `CSV → RawTableData → Definition → Instance` 4단계로 역할 분리
- Definition은 **Origin(필수) + 타입별 재정의(무기/탄/재료)** 상속 구조
- `ItemManager::Build`가 Definition을 인스턴스로 승격

아직은 구매(획득)만 고려한 구조다. 다음 날 크래프팅을 붙이면서 이 Definition 구조의 한계를 만나게 된다.
