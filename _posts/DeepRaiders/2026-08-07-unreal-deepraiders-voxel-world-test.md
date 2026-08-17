---
title: "DeepRaiders 개발 : 복셀 월드와 구 형태 굴착 테스트"
date: 2026-08-07 18:00:00 +0900
categories: [UnrealEngine, UnrealEngine-Project, UnrealEngine-Project-DeepRaiders]
tags: [UnrealEngine, DeepRaiders, Voxel, ProceduralGeneration]
description: "복셀 전용 월드를 구성하고 Remove Sphere를 이용해 절차적 굴착을 시험한 기록"
---

# DeepRaiders 개발 - 복셀 월드와 구 형태 굴착 테스트

[전날]({% post_url DeepRaiders/2026-08-06-unreal-deepraiders-voxel-plugin-setup %}) 복셀 플러그인을 프로젝트에 적용한 뒤, 실제 게임에서 사용할 수 있는 형태로 월드를 구성했다.

---

## 복셀 월드 구성

일반 월드에 기능을 덧붙이는 대신 복셀 전용 월드를 부모로 두고 새로운 월드를 생성했다. 복셀 지형의 생성과 수정 기능을 한곳에서 관리할 수 있어, 굴착이 핵심인 DeepRaiders의 월드 구조에도 잘 맞았다.

## Remove Sphere 테스트

임의의 위치와 반경을 지정해 `Voxel Tool::Remove Sphere`를 호출했다. 결과는 다음과 같았다.

- 지정한 위치를 중심으로 구 형태의 지형이 제거된다.
- 제거된 공간의 표면이 복셀 데이터에 맞춰 다시 생성된다.
- 위치와 반경을 바꾸면 서로 다른 크기의 빈 공간을 만들 수 있다.

단순한 구 하나만으로도 플레이어의 굴착, 폭발물, 자동 드릴처럼 여러 기능의 기반을 만들 수 있었다. 여러 지점을 이어서 제거하면 터널이나 동굴도 표현할 수 있을 것으로 보였다.

다만 아래쪽으로 계속 테스트하던 중 특정 깊이부터 복셀이 더 이상 생성되지 않았다. 굴착 로직보다는 복셀 월드 자체의 생성 범위가 원인일 가능성이 있어 월드 바운드 설정을 확인하기로 했다.

---

## 정리

- 복셀 전용 월드를 기반으로 DeepRaiders의 굴착 월드를 구성했다.
- 위치와 반경을 이용해 구 형태의 빈 공간을 절차적으로 만들 수 있었다.
- 일정 깊이 이후 복셀이 생성되지 않아 월드 바운드 설정을 추가로 확인해야 했다.
