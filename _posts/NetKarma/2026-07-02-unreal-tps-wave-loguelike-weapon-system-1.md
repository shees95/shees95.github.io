---
title: "TPS 개발 : 무기 시스템 (1) — 상점에서 장착까지"
date: 2026-07-02 12:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project ,UnrealEngine-Project-TPS]
tags: [UnrealEngine, UnrealEngine-Project-TPS, UnrealEngine-Weapon, UnrealEngine-Equipment, UnrealEngine-Definition, UnrealEngine-Inventory]
description: "상점 구매 → 무기/노즐 분류 → 슬롯 보관 → 장착까지, 그리고 Item, Shop Definition의 한계"
---

# TPS 개발 - 무기 시스템 (1)

무기 시스템을 만들어야 하는데 이게 정말 빡셌다. 아이템이나 상점과 달리, 무기는 **실제로 손에 들리고 발사되는** 물건이라 챙길 게 훨씬 많았다.

---

## 상점에서 무기 사기

무기도 결국 상점에서 산다. 그래서 [상점 Definition]({% post_url NetKarma/2026-07-01-unreal-tps-wave-loguelike-shop %})을 통해 구매하는 흐름을 그대로 탄다.

구매 시점에 이게 **무기(Weapon)인지 노즐(Nozzle)인지** 구분해서 각각의 보유 리스트로 보낸다.

```
Shop 구매
  └─ 타입 분기
       ├─ Weapon → 보유 Weapon 리스트
       └─ Nozzle → 보유 Nozzle 리스트
```

각각은 **슬롯 형태**로 들어간다. 무기 슬롯, 노즐 슬롯을 나눠서 관리하는 구조다.

---

## 문제 — Shop Definition으로는 부족하다

여기서 벽에 부딪혔다.

상점에서 산 무기가 **Shop Definition** 형태로 넘어와 버리면, 정작 전투에 필요한 디테일한 값이 없다. 데미지, 연사, 반동, 탄 퍼짐 같은 **CombatAS(전투 어트리뷰트)에 들어갈 값**이 상점 Definition엔 담겨 있지 않은 것이다.

```
Shop Definition
  ├─ 가격 ✓
  ├─ 아이콘 ✓
  └─ 전투 스탯 ✗   ← 이게 없다!
```

상점은 "얼마에 파느냐"만 알지, "이 총이 어떻게 쏘느냐"는 모른다. 당연한 얘긴데 만들면서 부딪혔다.

---

## 해결 — Origin만 떼서 Weapon Definition 재빌드

[Definition 시스템]({% post_url NetKarma/2026-06-29-unreal-tps-wave-loguelike-item-definition-1 %})을 만들 때 잡아둔 **Origin** 구조가 여기서 빛을 발했다.

Shop Definition에서 **Origin만 떼내서**, 그걸로 다시 **Weapon Definition을 Build** 한다. 상점 껍데기를 벗기고 전투용 알맹이로 다시 태어나게 하는 것이다.

```
Shop Definition
  └─ Origin 추출
       └─ ItemManager::Build → Weapon Definition
            └─ CombatAS 값 채워짐 (데미지/연사/반동 ...)
```

Origin이 "모든 아이템의 공통 신원"이라, 상점용이든 전투용이든 같은 Origin에서 다른 Definition으로 재해석할 수 있었다. 3일 전 구조가 여기서 값을 한 것이다.

---

## 가방 분류와 장착 API

무기가 전투용으로 재빌드되면, 이제 **가방에 잘 분류**해서 넣는다. 무기는 무기끼리, 노즐은 노즐끼리.

그리고 UI가 이걸 다뤄야 하니, **선택·장착·조회용 함수**를 따로 떼줬다.

```cpp
// UI가 호출하는 장착 API (의사코드)

// 인덱스 또는 명칭으로 장착
void EquipWeapon(int32 SlotIndex);
void EquipWeaponByName(FName WeaponId);

// 현재 무엇이 장착됐는지
UWeaponInstance* GetEquippedWeapon() const;

// 가방에 뭐가 있는지
TArray<UWeaponInstance*> GetOwnedWeapons() const;
TArray<UNozzleInstance*> GetOwnedNozzles() const;
```

UI는 내부 구조를 몰라도 된다. **인덱스나 이름으로 장착**하고, "지금 뭐가 장착됐는지 / 가방에 뭐가 있는지"만 물어보면 되도록 인터페이스를 정리했다.

---

## 오늘 정리

- 무기도 **상점 Definition**을 통해 구매, 구매 시 Weapon / Nozzle 분류 후 슬롯 보관
- Shop Definition엔 **전투 스탯(CombatAS)이 없다** — 상점은 가격만 알기 때문
- **Origin만 떼서 Weapon Definition으로 재빌드**해 전투 값을 채움
- 가방 분류 + **인덱스/명칭 기반 장착·조회 함수**로 UI와 분리

무기를 장착하는 것까진 됐다. 하지만 아이콘·메쉬·몽타주가 여기저기 흩어져 있어서, 다음 날 이걸 하나로 통일한다.
