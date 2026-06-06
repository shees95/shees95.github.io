---
title: "깃 페이지 Category, TOC 트리룰"
date: 2026-06-06 20:33:00 +0900
categories: [Git, Git-GitPage]
tags: [Git, GitPage, Category, TOC]
description: "깃 페이지 사용법"
---

# GitPage의 Category, TOC 용법

---

## Category
글들을 폴더처럼 분류해주는 기능

![category.png](../assets/img/gitpage-category%20toc/category.png)

### 사용법

헤더에 categories 를 대괄호 안에 컴마로 구분하여 작성한다.


```yaml
---
title: "깃 페이지 Category, TOC 트리룰"
date: 2026-06-06 20:33:00 +0900
categories: [Git, GitPages]
tags: [GitPages-Category, GitPages-TOC]
description: "깃 페이지 사용법"
---
```

### 문제점 1
Depth가 2단까지밖에 지원이 안되어, Tags로 세분류를 따로 나누는 것이 좋다.

```yaml
categories: [Git, GitPages,GitPages-Category]
```
이러면 GitPages 카테고리까지만 나타난다.

### 문제점 2
같은 카테고리면 하나로 묶인다.

예시로, 아래와 같이 카테고리가 분류가 되었다고 하자.

```yaml
categories: [Git, Push]
```

```yaml
categories: [Stack, Push]
```

이러면 Git - Push 카테고리에 Stack - Push 글이 들어오기도 하고  
동일하게 Stack - Psuh 카테고리에 Git - Push 글이 들어오기도 한다.  

카테고리의 의미가 무색해지는 오류가 발생하여  
가능한 선행 카테고리를 앞에 작성하는 것이 좋을듯 싶다.

```yaml
categories: [UnrealEngine, UnrealEngine-Basic]
```

```yaml
categories: [CPP, CPP-Basic]
```

## TOC(Table Of Contents)

글에 헤더 `#`를 작성하면 해당 행으로 갈 수 있는 바로가기 링크가 생긴다.

![toc.png](../assets/img/gitpage-category%20toc/toc.png)

해당 기능은 룰을 제대로 따르지 않으면 구조가 완전히 무너지는 현상을 볼 수 있다.

### 해결법 1
헤더 1개짜리 `#`는 가급적 쓰지 마라.  
`#`는 글의 제목을 작성할때 쓰이도록 만들어져, 매우 적게만 사용해야 안정적이다.  

```yaml
# 제목
## 소제목
### 대분류
#### 소분류
```

혹은

```yaml
# 제목
## 1단
### 2단
#### 3단 이하
```

이렇게 사용을 해야 문제없이 TOC 시스템이 수행되어 구조가 깨지지 않는다.

제일 위에 딱 1개만 `#1` 달고, 나머지는 무조건 `#2` 아래로 붙인다고 생각하고 작성하면 구조가 깨지지 않는다.

### 해결법 2
`#1` 에서 바로 `#3`으로 건너뛰기 금지.

헤더 단계를 건너뛰면 구조가 깨지는 이슈가 많이 생긴다.

### 해결법 3
줄나눔 `---` 은 `#2` 분류 정도로만.

과도한 줄나눔 처리는 TOC 구조를 쉽게 깨트릴 수 있다.

