---
title: 단일 모듈에서 멀티 모듈 전환기 - (1) 멀티 모듈이란?
# description: >-
#   description
date: 2025-02-21 22:49:00 +0900
categories: [아키텍처, 프로젝트]
tags: [멀티모듈, ddd, 도메인주도개발, msa]     # TAG names should always be lowercase
pin: true
# image:
#   path: /commons/devices-mockup.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices.
---
## **들어가기 앞서**

><span style='color:red'>단일 모듈에서 멀티 모듈 전환기에서 다루는 거의 모든 내용은 주관적인 관점을 가지고 있습니다.</span><br>다른 글의 내용을 인용할 경우 레퍼런스를 참조하거나 참고 문서에 존재합니다.
{: .prompt-info }

>**피드백은 언제나 환영합니다.** 글의 내용에 대한 의견이나 질문이 있으시면 댓글로 남겨주세요.
{: .prompt-tip }

단일 모듈에서 멀티 모듈 전환기(이하 '멀티 모듈 시리즈')에서는 단일 모듈에서 아키텍처가 변경되고 설계가 지속적으로 바뀌면서 최종적으로 멀티 모듈로 전환하는 내용을 다룹니다. 점진적으로 아키텍처가 변경하는 과정을 보여주며 실제 프로젝트의 리팩터링은 테스트 코드가 존재한다는 전제하에 진행됩니다.

저의 온라인 개발스승인 재민님이 [작성한 글](https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63){:target="_blank"}에 나온 아키텍처를 최종적으로 따라갑니다. 멀티모듈 시리즈를 읽다보면 DDD(도메인 주도 개발), 클린 아키텍처과 관련된 내용이 언급되나, 저는 DDD 및 클린 아키텍처에 대해 제대로 공부해본적이 없습니다. DDD와 클린 아키텍처에서 언급되는 개념들과 무관하게, 유지보수하기 좋은 코드와 설계를 다룹니다.

기술 스택: Java, Spring, Jpa, Gradle

---
## **용어 정의**

이 프로젝트에서는 JPA 기술을 사용하고 있습니다. 도메인은 JPA 엔티티와 POJO(기술에 의존하지 않은 자바 클래스)로 이루어진 도메인 모델로 구성됩니다.

멀티모듈 시리즈에서는 두 용어와 관련된 내용이 나올 때 다음과 같이 구별해서 작성합니다.
- **JPA 엔티티**: **JPA 엔티티** 또는 **JPA 엔티티 객체**
- **도메인 모델**: **도메인 모델** 또는 **도메인 모델 객체**

객체가 붙은 용어는 메모리가 할당된 것을 의미합니다.

**도메인**이라는 용어도 사용하는데, IT분야에서 많이 사용하는 **문제를 해결해야하는 영역**으로 이해해주시기 바랍니다.

___
## **프로젝트 이해를 위한 ERD**

![image](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202025-02-02%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%206.15.05.png)

핵심 엔티티는 `Member`, `Cafe`, `Study`, `StudyMember`, `Comment`입니다(id를 제외한 컬럼은 생략).

**비즈니스 규칙**
- 1개의 카페는 여러개의 스터디를 가질 수 있다.
- 회원은 여러개의 스터디를 생성 할 수 있다.
  - `Study` 엔티티의 `member_id`는 스터디를 생성한 스터디장을 의미한다.
- 1개의 스터디는 여러개의 스터디원을 가질 수 있다.
  - 스터디에 참여한 모든 회원은 스터디원이 된다.
  - 스터디장도 스터디원에 포함된다.
- 회원은 스터디 페이지에 댓글을 적을 수 있다.

---
## **단일 모듈에서 멀티 모듈로의 전환 이유**
처음부터 단일 모듈에서 멀티 모듈로의 전환을 고려하고 시작된 프로젝트는 아니었습니다. 유지보수하기 좋은 코드를 고민하는 과정에서 설계가 지속적으로 변했습니다. 필요에 의해 도메인 모델을 도입하고 프로젝트에서 JPA 엔티티와 도메인 모델이 공존하게 되었습니다.

이 문제를 해결하고자 JPA 엔티티를 격리하기 위해 멀티 모듈을 도입하였습니다. 순차적으로 무엇이 필요하게 되었고 어떻게 개선해나갔는지 알아보겠습니다.

---
## **모듈(Module)이란?**
[Oracle 공식문서](https://www.oracle.com/corporate/features/understanding-java-9-modules.html){:target="_blank"}에서 모듈을 다음과 같이 정의합니다.
> - 모듈은 패키지보다 상위 수준의 집합 단위로, 재사용이 가능한 관련 패키지의 모음이다.

[Gradle 공식문서](https://docs.gradle.org/current/samples/sample_java_modules_multi_project.html){:target="_blank"}에서는 모듈에 대해서 다음과 같이 언급합니다.
> - Java 모듈은 Java9부터 사용 가능한 Java 자체의 기능으로, 더 나은 캡슐화를 가능하게 한다.
> - Gradle에서는 Java 소스를 포함하는 각 source set을 모듈로 전환 할 수 있다. 일반적으로 이와 같은 Java 모듈이 있는 프로젝트에서는 서브프로젝트의 main source set이 하나의 모듈을 나타낸다.


![image-20250202214358824](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/image-20250202214358824.png)

스프링 프로젝트를 생성하면 `main` 디렉토리를 볼 수 있는데, 이 `main` 디렉토리는 Gradle 빌드 도구를 통해 모듈로 전환할 수 있습니다. 프로젝트를 생성하고 모듈과 관련된 어떠한 설정도 하지 않았다면 단일 프로젝트면서 단일 모듈로 구성됩니다.

아래 이미지는 단일 모듈에서 여러가지의 도메인을 다루고 있습니다.

![image](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/image-20250202210924360.png)

___
## **멀티 모듈이란?**
- 단일 프로젝트에서 여러개의 모듈로 나눈 것.

### Why 멀티모듈?
- 모듈 간 격리를 통하여 소프트웨어를 강력하게 통제가 가능하다.
- 각 모듈의 책임과 경계가 명확해진다.
- 모듈 별 관심사 분리로 응집도가 향상된다.
- 불필요한 의존성 제거로 결합도가 감소한다.

멀티모듈은 많은 장점이 존재합니다. 반면에 단점도 존재합니다. 단점에 관해서는 이후 글에서 다룹니다.

아래 이미지는 단일 프로젝트의 멀티 모듈을 나타냅니다.

![image](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/image-20250202222217340.png)

___
## **단일 모듈에서 멀티 모듈 전환기 목차**

- <a target='_blank' href='/posts/single-to-multi-module-(1)'><span style='color: #ef5369'>단일 모듈에서 멀티 모듈 전환기 - (1) 멀티 모듈이란?</span></a>
- <a target='_blank' href='/posts/single-to-multi-module-(2)'>단일 모듈에서 멀티 모듈 전환기 - (2) 구현 계층</a>
- <a target='_blank' href='/posts/single-to-multi-module-(3)'>단일 모듈에서 멀티 모듈 전환기 - (3) 도메인 모델</a>
- <a target='_blank' href='/posts/single-to-multi-module-(4)'>단일 모듈에서 멀티 모듈 전환기 - (4) JPA 엔티티 격리 및 DB 추상화</a>
- <a target='_blank' href='/posts/single-to-multi-module-(5)'>단일 모듈에서 멀티 모듈 전환기 - (5) Bounded Context 설계</a>
- <a target='_blank' href='/posts/single-to-multi-module-(6)'>단일 모듈에서 멀티 모듈 전환기 - (6) 멀티 모듈 설계</a>
- <a target='_blank' href='/posts/single-to-multi-module-(7)'>단일 모듈에서 멀티 모듈 전환기 - (7) 멀티 모듈 전환 with Gradle</a>
- <a target='_blank' href='/posts/single-to-multi-module-(8)'>단일 모듈에서 멀티 모듈 전환기 - (8) 회고 및 마무리</a>

---
## **참고 문서**
- [https://www.oracle.com/corporate/features/understanding-java-9-modules.html](https://www.oracle.com/corporate/features/understanding-java-9-modules.html){:target="_blank"}
- [https://docs.gradle.org/current/samples/sample_java_modules_multi_project.html](https://docs.gradle.org/current/samples/sample_java_modules_multi_project.html){:target="_blank"}
- [https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63](https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63){:target="_blank"}
