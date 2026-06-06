---
title: "코딩 테스트 오답노트"
date: 2026-06-01 11:40:00 +0900
categories: [CPP, CPP-CodingTest]
tags: [CPP, CPP-CodingTest, CPP-ErrorLog]
description: "코딩 테스트 오답노트"
---

# 코딩 테스트 오답 노트

---

## 1. Anagram

### 오답

* 문자열 정렬 후 erase로 하나씩 제거하며 매칭
* greedy 방식으로 같은 문자 “찾아서 지우기”

```cpp
sort(v1.begin(), v1.end());
sort(v2.begin(), v2.end());
```

### 문제점

* 순서는 중요하지 않음
* 빈도 체크만 하면 됨

### 정답

* 문자 개수 비교 문제로 변환

```cpp
count1[c], count2[c]
abs(count1 - count2) // 합산 절대값
```

---

## 2. map + struct Item 설계 문제

### 문제점 1: operator< 설계 오류

```cpp
attackPower → rarity → name (비일관 비교)
```

### 문제 분석

* strict weak ordering 위반 가능성
* map 내부 정렬 기준이 애매해짐
* key 비교 기준과 실제 사용 기준(name)이 다름

### 정답

```cpp
if (attackPower != other.attackPower)
    return attackPower < other.attackPower;

if (rarity != other.rarity)
    return rarity < other.rarity;

return name < other.name;
```

### 문제점 2: map을 선형 탐색처럼 사용

```cpp
for (auto& i : shop)
```

### 문제 분석

* map의 O(log N) 탐색 구조 무시
* key 기반 자료구조 장점 상실

### 정답

```cpp
auto it = shop.find(item);
```

---

## 3. MyVector 구현

### 문제점 : 메모리 할당 방식 혼용

```cpp
malloc(...) + delete[]
```

### 문제 분석

* malloc 은 free
* new 는 delete

### 정답

```cpp
Data = new T[Capacity];
Size = 0;
```

```cpp
T* New = new T[Capacity];
```
