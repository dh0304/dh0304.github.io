---
title: 단일 모듈에서 멀티 모듈 전환기 - (5) Bounded Context 설계
# description: >-
#   description
date: 2025-03-01 23:39:00 +0900
categories: [아키텍처, 프로젝트]
tags: [멀티모듈, ddd, 도메인주도개발, msa]     # TAG names should always be lowercase
pin: false
# image:
#   path: /commons/devices-mockup.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices.
---
## 들어가기 앞서

><span style='color:red'>단일 모듈에서 멀티 모듈 전환기에서 다루는 거의 모든 내용은 주관적인 관점을 가지고 있습니다.</span><br>다른 글의 내용을 인용할 경우 레퍼런스를 참조하거나 참고 문서에 존재합니다.
{: .prompt-info }

>**피드백은 언제나 환영합니다.** 글의 내용에 대한 의견이나 질문이 있으시면 댓글로 남겨주세요.
{: .prompt-tip }

<a target='_blank' href='/posts/single-to-multi-module-(4)'>'단일 모듈에서 멀티 모듈 전환기 - (4) JPA 엔티티 격리 및 DB 추상화'</a>에서 이어집니다.

---
## **현재 구조 분석**

### 도메인 간의 의존 관계

![image-20250211182404101](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/image-20250211182404101.png)

프로젝트에서 도메인 간의 의존 관계를 표현한 다이어그램입니다. **기대하는 의존 관계**와 <span style='color:red'>**존재 할 수도 있는 의존 관계**</span>를 나타냅니다.

> 도메인 간의 의존 관계는 도메인 모델뿐만아니라 서비스 계층, 구현 계층, 테스트 코드 등 모든 클래스와 객체 간의 관계를 포함합니다.<br>예) `MemberService` 에서 `StudyService` 를 의존하는 관계, `MemberService` 에서 `StudyRepository` 를 의존하는 관계
{: .prompt-info }

> 이후 섹션에서 도메인 간의 의존 관계는 도메인 모델 간의 의존 관계로 표현합니다.
{: .prompt-info }

### 문제점
- 단일 프로젝트 단일 모듈에서 도메인 간의 의존 관계는 복잡하게 얽혀 있다.
- 도메인 모델 간의 의존 관계는 **기대하는 의존 관계**이지만 서비스, 구현, 테스트 코드 등 도메인과 관련된 클래스의 의존 관계는 <span style='color:red'>**존재 할 수도 있는 의존 관계**</span>를 가질 수 있다.
- 복잡한 의존 관계는 순환 참조로 인해 문제가 발생할 수 있다.

### BC 도입의 필요성
- BC를 통해 도메인 간의 의존 관계를 관리하여 유지보수성을 높인다.
- 의존 관계를 관리하여 순환 참조를 방지한다. 그럼에도 순환 참조는 발생할 수 있는데 의존 관계가 명확하여 빠르게 문제를 해결할 수 있다.

> BC와 관련된 내용은 <a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(3)-도메인-모델#bounded-context란'>멀티 모듈 시리즈 - (3) 도메인 모델, Bounded Context란?</a>을 참고해 주세요.
{: .prompt-info }

---
## **도메인 모델 간의 관계 분석하기**
### 도메인 모델 간의 의존성 파악

우리가 기대하는 도메인 모델 간의 의존 관계는 아래와 같습니다.

![image-20250211185758362](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/image-20250211185758362.png)

### 도메인 모델의 책임과 역할 분석

`Member` 도메인 모델
- 회원은 개인정보를 가진다.
- 회원은 여러개의 스터디를 만들 수 있다.

`Cafe` 도메인 모델
- 스터디 모임의 공간을 제공하는 Cafe이다.
- 영업시간, 메뉴, 주소 등의 카페와 관련된 정보를 가진다. 

`Study` 도메인 모델
- Cafe가 제공하는 정보에 의존한다.
- 스터디는 카페가 제공하는 영업시간 내에서만 존재한다.

`StudyMember`
- 스터디에 참여한 모든 회원은 스터디원이 된다.
- 스터디장도 스터디원에 포함된다.

`Comment`
- 댓글은 스터디 QnA에 존재하며 스터디 QnA는 스터디에 대한 정보를 얻을 수 있는 공간이다.
- 회원은 스터디 QnA에 댓글을 적을 수 있다.

---
## **도메인 모델 관계를 통해 BC 설계**

### BC 설계의 접근 방식
- BC를 설계할 때 각각의 BC별로 어떠한 도메인 모델을 포함할 것인지에 대한 정답은 없다.
- 도메인마다 팀마다 상황마다 다르다.
- 우선, 관련된 도메인 모델끼리 묶어보고 비즈니스 규칙을 고민하여 BC를 구성한다.

### 프로젝트의 BC 구성

![image-20250212162152115](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/image-20250212162152115.png)
> 도메인 모델 다이어그램에서는 개별 모델 간의 의존 관계를 상세히 나타냈으나, BC 다이어그램에서는 BC 단위의 의존 관계만 표시했습니다.
{: .prompt-info }
### BC 간의 관계 정의

`Member` BC
- 회원과 관련된 BC이다.

`Cafe` BC
- 카페와 관련된 BC이다.

`Study` BC
- 스터디와 관련된 BC이다.
- 스터디와 관련된 도메인 모델은 `Stduy`, `StudyMember`, `Coordinator`가 있다.
- `StudyMember`, `Coordinator` 는 `Study` 와 관련된 도메인 모델이므로 BC의 이름은 대표가 되는 도메인인 `Study` BC이다.

> <a target='_blank' href='/posts/single-to-multi-module-(3)#동일-엔티티의-context별-도메인-모델-예시'>멀티 모듈 시리즈 - (3) 도메인 모델, 동일 엔티티의 Context별 도메인 모델 예시</a>에서 다뤘듯이 `Coordinator` 도메인 모델의 실제 JPA 엔티티는 `Member` JPA 엔티티이다.
{: .prompt-info }

`QnA` BC
- 댓글과 관련된 BC이다.
- BC의 이름은 QnA이다. 스터디와 관련된 댓글은 스터디 QnA에 존재하기 때문에 `Comment` BC가 아니라 QnA BC이다.
- QnA와 관련된 도메인 모델은 `Comment` 가 있다.

---
## **BC 설계 시 고려사항**

### 프로젝트에서 고려한 도메인 모델의 BC 분리 예시

#### QnA BC 설계 시 고려했던 점

다음 ERD은 이전에 보았던 <a target='_blank' href='/posts/single-to-multi-module-(1)#프로젝트-이해를-위한-erd'>프로젝트 ERD</a>입니다.
![erd](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202025-02-02%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%206.15.05.png)

`Study` 엔티티는 `Comment` 엔티티와 일대다 관계이다.
- 하나의 스터디는 여러개의 댓글을 가질 수 있다.

엔티티 간의 관계에서는 DB의 테이블 설계의 룰을 따릅니다. 하지만 도메인 모델의 경우에는 어떨까요? 다음 이미지를 보고 다시 질문을 드릴게요.

![image-20250212175111851](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/image-20250212175111851.png)


스터디 모집 페이지에서는 다음과 같은 특징을 가집니다.
- 스터디 페이지의 QnA에서는 회원이 스터디에 궁금한 점을 물어볼 수 있다.
- 스터디장을 포함한 회원들은 답변을 할 수 있다.

여기서 다시 질문을 드려볼게요. 스터디 모집 페이지의 QnA에 댓글을 다는 행위는 스터디와 **직접적으로 관련이 있는 행위**인가요? 저는 직접적으로 관련이 있는 행위가 아니라고 생각했습니다. 이유는 다음과 같습니다.

- 스터디 페이지의 QnA에 댓글을 다는 행위는 스터디와 **간접적으로 관련이 있는 행위**이다.
- `Study` 도메인 모델이 직접적으로 알아야할 정보는 아니다.

그럼 `Study` BC에서 `Comment` 도메인 모델을 포함시켜야 할까요? 다음 다이어그램은 `Study` BC에 `Comment` 도메인 모델을 포함시켰습니다.

![image-20250212181029912](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/image-20250212181029912.png)

  `Study`  BC와 QnA BC`Study` BC에서 `Comment` 도메인 모델을 포함시키게 된다면 다음과 같은 문제가 발생하게됩니다.
- `Comment` 도메인 모델이 `Study` 도메인 모델뿐만 아니라 `StudyMember` 도메인 모델과 `Coordinator` 도메인 모델까지 직접적으로 참조할 수 있다.

하지만 이런 의견도 존재할 수 있는데요.
- `Comment` 도메인 모델이 `Study` BC 안에 존재하더라도 코드상에서 직접 참조만 안하면 그만 아닐까요?

저도 이 의견에 동의합니다. 하지만 저희는 직접 사용하지 않을 것이라는 불확실성보다는 BC간의 의존관계를 통해 확실한 방법을 선택하기로 했습니다. 그리고 지금 선택한 BC 간의 의존 관계, 도메인 모델의 위치는 미래에 요구사항이 변경되어 바뀔 수도 있습니다.

이러한 의견들을 종합하여 저희는 `Comment` 도메인 모델을 `QnA` BC에 포함하여 관리하도록 했습니다.
![단일모듈 BC 설계2](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/%E1%84%83%E1%85%A1%E1%86%AB%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%86%E1%85%A9%E1%84%83%E1%85%B2%E1%86%AF%20BC%20%E1%84%89%E1%85%A5%E1%86%AF%E1%84%80%E1%85%A82.svg)

### 새로운 요구사항에 따른 Review 도메인 모델 추가

다음과 같은 새로운 요구사항이 생겼습니다.
- 스터디원들은 다른 스터디원들에 대한 평가를 남길 수 있다.

새로운 요구사항이 발생하여 `Review` 도메인 모델을 만들었습니다. 스터디원들은 다른 스터디원에 대한 평가를 남겨야 하므로 `Review` 도메인 모델은 `Study` BC에 존재하는 것이 맞아보입니다.

다음 다이어그램은 `Study` BC에 `Review` 도메인 모델이 포함되는 것을 보여줍니다.
![image-20250212185600950](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/image-20250212185600950.png)

그런데 머지않은 미래에 다음과 같은 요구사항이 생길 수도 있다라는 말이 들려왔는데요.
- 스터디원들은 스터디를 진행한 카페에 대해 평가를 남길 수 있다.

이러한 정보를 듣고 `Review` BC를 생성하고 `Review` 도메인 모델을 포함시켜야하나 고민이 듭니다. 따라서 `Review` BC를 분리해야 할지 판단하기 위해 몇가지 기준을 고려해보겠습니다.

`Review` 도메인 모델을 `Review` BC로 분리해야할 경우
- 리뷰의 비즈니스 로직이 복잡한가?
  - 리뷰에 대한 기능이 단순히 CRUD를 넘어서는 복잡한 비즈니스 규칙이 존재한다면 , `Review` BC로 관리하는 것이 낫다.
- `Review` 도메인 모델이 여러 BC와 관련이 깊은가?
  - `Review` 도메인 모델 `Study` BC뿐만 아니라 `Cafe` BC와 강하게 연관되어 있다면, `Review` BC으로 분리하여 다른 도메인 모델과의 의존성을 최소화 할 수 있다.
- `Review` 도메인 모델이 `Study` 도메인 모델과 `Cafe` 도메인 모델과의 생명주기가 일치하는가?
  - `Study` 도메인 모델이나  `Cafe` 도메인 모델이 삭제되더라도 `Review` 도메인 모델은 여전히 별도로 관리되어야 한다면, `Review` 를 독립된 BC로 다루는 것이 적합하다.
- `Review` 도메인 모델 자체가 독립적인 비즈니스 가치를 가지는가?
  - 리뷰 데이터가 다른 비즈니스의 데이터로 사용된다면, `Review` BC로 분리하는 것이 낫다.

`Review` 도메인 모델을 `Study` BC에 남겨야할 경우
- `Review` 도메인 모델이 `Study` BC와 강하게 결합되어 있는가?
  - `Review` 도메인 모델이 가 `Study` 도메인 모델의 하위 도메인으로 간주될 정도로 강한 결합 관계를 가지고 있다면, `Study` BC 내부에서 관리하는 것이 낫다.
- `Review` 도메인 모델의 복잡성이 낮은 경우
  - `Review` 도메인 모델이 단순히 `Study` 도메인 모델과 `Cafe` 도메인 모델에 대한 간단한 CRUD로 구성되고, 별도의 도메인 규칙이 없는 경우 굳이 분리할 필요가 없다.

아래 다이어그램은 `Review` BC를 만들고 `Review` 도메인 모델은 `CafeReview` 도메인 모델과 `StudyReview` 도메인 모델로 분리되어 포함합니다.
![image-20250212194308143](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/image-20250212194308143.png)

저희는 `Review` BC를 만들지 않고 `Review` 도메인 모델을 `Study` BC에 유지하는 방식을 선택했습니다. 이유는 다음과 같습니다.
- 머지않은 미래에 요구사항이 바뀔 가능성이 있지만 바뀌지 않을 가능성도 있다.
- 확정된 요구사항이 아니기 때문에 요구사항은 언제든지 바뀔 수 있다. 요구사항이 바뀌면 설계도 바뀐다.
![image-20250212185600950](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/image-20250212185600950.png)

---
## **단일 모듈에서 멀티 모듈 전환기 목차**

- <a target='_blank' href='/posts/single-to-multi-module-(1)'>단일 모듈에서 멀티 모듈 전환기 - (1) 멀티 모듈이란?</a>
- <a target='_blank' href='/posts/single-to-multi-module-(2)'>단일 모듈에서 멀티 모듈 전환기 - (2) 구현 계층</a>
- <a target='_blank' href='/posts/single-to-multi-module-(3)'>단일 모듈에서 멀티 모듈 전환기 - (3) 도메인 모델</a>
- <a target='_blank' href='/posts/single-to-multi-module-(4)'>단일 모듈에서 멀티 모듈 전환기 - (4) JPA 엔티티 격리 및 DB 추상화</a>
- <a target='_blank' href='/posts/single-to-multi-module-(5)'><span style='color: #ef5369'>단일 모듈에서 멀티 모듈 전환기 - (5) Bounded Context 설계</span></a>
- <a target='_blank' href='/posts/single-to-multi-module-(6)'>단일 모듈에서 멀티 모듈 전환기 - (6) 멀티 모듈 설계</a>
- <a target='_blank' href='/posts/single-to-multi-module-(7)'>단일 모듈에서 멀티 모듈 전환기 - (7) 멀티 모듈 전환 with Gradle</a>
- <a target='_blank' href='/posts/single-to-multi-module-(8)'>단일 모듈에서 멀티 모듈 전환기 - (8) 회고 및 마무리</a>

---
## **참고 문서**
