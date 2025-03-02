---
title: 단일 모듈에서 멀티 모듈 전환기 - (7) 멀티 모듈 전환 with Gradle
# description: >-
#   description
date: 2025-03-02 15:55:00 +0900
categories: [아키텍처, 프로젝트]
tags: [멀티모듈, ddd, 도메인주도개발, msa]     # TAG names should always be lowercase
pin: false
# image:
#   path: /commons/devices-mockup.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices.
---
## 들어가기 앞서

><span style='color:red'>단일 모듈에서 멀티 모듈 전환기에서 다루는 거의 모든 내용은 주관적인 관점을 가지고 있습니다.</span> 다른 글의 내용을 인용할 경우 레퍼런스를 참조하거나 참고 문서에 존재합니다.
>{: .prompt-info }

>**피드백은 언제나 환영합니다.** 글의 내용에 대한 의견이나 질문이 있으시면 댓글로 남겨주세요.
{: .prompt-tip }

<a target='_blank' href='/posts/single-to-multi-module-(6)'>'단일 모듈에서 멀티 모듈 전환기 - (6) 멀티 모듈 설계'</a>에서 이어집니다.

---
## **Gradle을 이용한 멀티 모듈 설계**

Gradle의 Java Plugin을 사용하여 모듈 간 의존성 관리를 합니다. 멀티 모듈 구성을 하기 위해선 Java Plugin(or Java-Library Plugin)과 Gradle의 Task와 관련된 내용을 알아야 합니다. 포스팅이 길어짐에 따라 Java Plugin과 Task에 관한 자세한 내용은 다루지 않습니다.

설정부터 테스트까지 많은 것을 다루려고 했으나, 설계에 초점이 맞춰진 포스팅인 만큼 설정과 테스트에 대한 지식이 많이 필요한 부분은 다루지 않기로 했습니다.
### 모듈 간 의존성 관리 설정
![멀티모듈-gradle](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/%E1%84%86%E1%85%A5%E1%86%AF%E1%84%90%E1%85%B5%E1%84%86%E1%85%A9%E1%84%83%E1%85%B2%E1%86%AF-gradle.svg)

- API 모듈 -> Domain 모듈은 `implemenation` 으로 의존 관계 설정
- DB 모듈 -> Domain 모듈은 `compileOnly` 로 의존 관계 설정
- API 모듈 -> DB 모듈은 `runtimeOnly` 로 의존 관계 설정

#### API 모듈 -> Domain 모듈 간 implementation 의존 관계 설정

API 모듈은 빌드 시 실행가능한 jar파일을 생성합니다. API 모듈은 Runtime 시점에 도메인 모듈의 코드를 동적으로 실행합니다. 따라서 API 모듈과 Domain 모듈 간 의존 관계 설정은 `implementation` 으로 지정합니다.

#### DB 모듈 -> Domain 모듈 간 compileOnly 의존 관계 설정

DB 모듈은 Compile 시점에 Domain 모듈에 존재하는 classpath를 알아야합니다. classpath를 알지 못하면 컴파일 오류가 발생하게 됩니다. `implementation` 대신 `compileOnly` 를 한 이유는 DB 모듈은 Domain 모듈의 코드를 동적으로 실행시킬 필요가 없기 때문입니다.

아래 코드는 DB 모듈이 `Study` 도메인 모델을 의존하지 않을 때 발생하는 컴파일 오류입니다.
![](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/f9o7Cno.png)

![](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/P7q3Nef.png)

#### API 모듈 -> DB 모듈 간 runtimeOnly 의존 관계 설정
각 모듈은 모듈만 따로 분리해도 많은 변경없이 사용 가능하도록 응집도를 최대한 높이도록 구성했습니다. 독립적으로 구성이 가능하려면 각 모듈의 설정파일 또한 각 모듈이 가지고 있어야합니다. 

API 모듈이 Runtime 시점에 DataSource를 알아야합니다. 따라서 Runtime 시점에 API 모듈이 DB 모듈의 설정파일을 알 수 있도록 `runtimeOnly` 로 구성했습니다.

---
## **멀티 모듈에서 테스트 코드 관리**

테스트 코드는 단위 테스트, 통합 테스트, E2E 테스트가 존재합니다. 각 모듈은 응집도를 높이도록 구성했다고 언급 했는데요. 테스트 코드 또한 모듈에 맞게 존재하도록 구성하려 합니다. 그런데 다음과 같은 문제가 발생했습니다.

### 문제 발생
- Domain 모듈이 도메인과 관련된 통합 테스트를 실제 DB를 사용할 경우, Domain 모듈은 DB 모듈을 알아야한다.

<a target='_blank' href='/posts/single-to-multi-module-(6)#domain-모듈의-역할과-책임'>멀티 모듈 시리즈 - (6) 멀티 모듈 설계, Domain 모듈의 역할과 책임</a>에서 다음과 같이 정의했습니다.
> 핵심 비즈니스를 가지는 Domain 모듈은 외부(클라이언트, 기술 등)에 의존적이지 않아야 한다.  

프로덕션 레벨에서 Domain 모듈이 DB 모듈을 알지 못하게 하는 것은 당연합니다. 그런데 **테스트 레벨에서는 Domain 모듈이 DB 모듈을 알아도 괜찮을까요?**

아래 섹션에서 여러 해결 방안을 알아보고 어떤 것을 선택할 지 고민해 봅시다.

### 해결 방안
#### Domain 모듈이 DB 모듈을 알게한다.
Domain 모듈이 DB 모듈을 아는 것은 지금까지 지켜왔던 "Domain 모듈은 외부에 영향을 받지 않는다" 라는 내용을 위반하기 때문에 선택하지 않았습니다.

#### Domain 모듈의 통합 테스트에서 DB를 Mock으로 대체한다.
도메인과 관련된 통합 테스트를 Domain 모듈에 유지하기 위한 좋은 방법입니다. 하지만 저희는 통합 테스트에서 DB를 Mock으로 유지하는 방법은 선택할 수 없었습니다.

테스트 코드에서 DB를 Mock으로 대체할 경우, 데이터 정합성 문제를 테스트 코드에서 발견할 수 없는 문제를 만났습니다. 따라서 DB를 Mock으로 대체하는 방법 또한 선택하지 않았습니다.

#### 통합 테스트를 DB 모듈로 옮긴다.
마지막 선택지인 **통합 테스트를 DB 모듈로 옮기는 방법**을 선택했습니다. 사실 이 방법 또한 맘에 들지는 않았는데요. 각 모듈을 독립적으로 사용 가능하게 구성하지 못하기 때문입니다. 세가지 방법 중에 가장 **차선택**을 선택했습니다.

> 프로젝트 진행 당시에는 `통합 테스트를 DB 모듈로 옮긴다` 라는 방식을 선택했는데요. 글을 작성하고 있는 지금 다시 선택한다면 `Domain 모듈의 통합 테스트에서 DB를 Mock으로 대체한다` 방식도 고려해볼 것 같습니다.<br>
> 이 세가지 방법 중에 정답은 없습니다. 다만, 상황에 맞는 선택이 있을 뿐이죠.
{: .prompt-info }

### 해결
아래 그림과 같은 방식으로 DB 모듈에 도메인과 관련된 통합 테스트가 존재하게 되었습니다.
![](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/XPeXuUE.png)

---
## **멀티 모듈에서 Test Fixture 관리**

멀티 모듈에서 `Test Fixture`를 관리하는 내용은 하나의 포스팅으로 다뤄도 될만큼 내용이 많습니다. 따라서 간단히 언급만 하고 넘어갑니다.

### Java Test Fixtures 플러그인 
```
plugins {  
    id("java-test-fixtures")  
}
```

`Test Fixture` 관리는 `java-test-fixtures` 플러그인을 통해 관리합니다. 이 플러그인에 대한 자세한 내용은 [이 블로그](https://toss.tech/article/how-to-manage-test-dependency-in-gradle)를 참고해주세요.

프로젝트에는 두가지의 `Test Fixture`가 존재합니다.
- JPA 엔티티 객체의 영속화를 도와주는 <span style='color:red'>Persister</span> 클래스들
- 도메인 모델 객체를 만들어주는 <span style='color:green'>Helper</span> 클래스들

![테스트 코드와 픽스처](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/%E1%84%90%E1%85%A6%E1%84%89%E1%85%B3%E1%84%90%E1%85%B3%20%E1%84%8F%E1%85%A9%E1%84%83%E1%85%B3%E1%84%8B%E1%85%AA%20%E1%84%91%E1%85%B5%E1%86%A8%E1%84%89%E1%85%B3%E1%84%8E%E1%85%A5.svg)

기존에는 JPA 엔티티 객체만을 위한 `Test Fixture`가 존재했으나, JPA 엔티티와 도메인 모델로 분리가 됨에 따라 JPA 엔티티 `Test Fixture`와 도메인 모델 `Test Fixture` 두가지가 존재하게 되었습니다. 따라서 테스트 코드를 작성할 때 두가지를 활용해서 작성하였습니다.

단일 모듈의 경우 하나의 test 디렉토리에서 `Test Fixture`를 관리했지만 멀티 모듈로 분리하고 나서는 `Test Fixture`들은 각각의 역할에 맞는 위치로 이동해야합니다.

도메인 모델 `Test Fixture`는 Domain 모듈에 존재하고 JPA 엔티티 `Test Fixture`은 DB 모듈에 존재하게 됩니다.
![](https://i.imgur.com/KbjR2Ys.png)
![](https://i.imgur.com/liUuQs2.png)

하지만 이렇게 구성할 경우 DB 모듈의 테스트 관련 의존성이 복잡해지게 됩니다. 
> DB 모듈은 Domain 모듈의 main Jar파일, test Jar파일, testFixtures Jar파일을 알아야만합니다. 이와 관련된 내용은 java-test-fixtures 플러그인 관련 문서를 찾아보면 됩니다.
{: .prompt-info }

![db테스트 관련 의존성](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/db%E1%84%90%E1%85%A6%E1%84%89%E1%85%B3%E1%84%90%E1%85%B3%20%E1%84%80%E1%85%AA%E1%86%AB%E1%84%85%E1%85%A7%E1%86%AB%20%E1%84%8B%E1%85%B4%E1%84%8C%E1%85%A9%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC.svg)

DB 모듈에서 많은 Domain 모듈의 테스트 관련 의존성이 필요합니다.

---
## **단일 모듈에서 멀티 모듈 전환기 목차**

- <a target='_blank' href='/posts/single-to-multi-module-(1)'>단일 모듈에서 멀티 모듈 전환기 - (1) 멀티 모듈이란?</a>
- <a target='_blank' href='/posts/single-to-multi-module-(2)'>단일 모듈에서 멀티 모듈 전환기 - (2) 구현 계층</a>
- <a target='_blank' href='/posts/single-to-multi-module-(3)'>단일 모듈에서 멀티 모듈 전환기 - (3) 도메인 모델</a>
- <a target='_blank' href='/posts/single-to-multi-module-(4)'>단일 모듈에서 멀티 모듈 전환기 - (4) JPA 엔티티 격리 및 DB 추상화</a>
- <a target='_blank' href='/posts/single-to-multi-module-(5)'>단일 모듈에서 멀티 모듈 전환기 - (5) Bounded Context 설계</a>
- <a target='_blank' href='/posts/single-to-multi-module-(6)'>단일 모듈에서 멀티 모듈 전환기 - (6) 멀티 모듈 설계</a>
- <a target='_blank' href='/posts/single-to-multi-module-(7)'><span style='color: #ef5369'>단일 모듈에서 멀티 모듈 전환기 - (7) 멀티 모듈 전환 with Gradle</span></a>
- <a target='_blank' href='/posts/single-to-multi-module-(8)'>단일 모듈에서 멀티 모듈 전환기 - (8) 회고 및 마무리</a>

---
## **참고 문서**
