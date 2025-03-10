---
title: 단일 모듈에서 멀티 모듈 전환기 - (6) 멀티 모듈 설계
# description: >-
#   description
date: 2025-03-02 00:57:00 +0900
categories: [아키텍처, 프로젝트]
tags: [멀티모듈, ddd, 도메인주도개발, msa]     # TAG names should always be lowercase
pin: false
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

<a target='_blank' href='/posts/single-to-multi-module-(5)'>'단일 모듈에서 멀티 모듈 전환기 - (5) Bounded Context 설계'</a>에서 이어집니다.

---
## **설계한 Bounded Context(BC) 다시보기**

<a target='_blank' href='/posts/single-to-multi-module-(5)'>멀티 모듈 시리즈 - (5) Bounded Context 설계</a>에서 BC를 설계하고 BC 간의 의존 관계를 맺었습니다. 설계한 BC들을 가지고 멀티 모듈을 구성하게 됩니다.
![image-20250212185600950](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/image-20250212185600950.png)

---
## **모듈 분리 전략**
### 모듈 분리 기준
- API 모듈, Domain 모듈, DB 모듈로 분리한다.
- 각 BC는 모듈로 구성된다.
- 모듈 자체를 그대로 분리하더라도 독립적으로 사용할 수 있도록 모듈을 구성한다.
- 모듈 분리는 정답이 없다. 비즈니스에 따라 적절히 분리한다.

### API-Domain-DB 모듈로 분리한 이유
- Domain 모듈은 비즈니스를 담당하는 모듈로 비즈니스와 관련 없는 외부의 변경에 영향을 받지 않게 하기 위함이다.
- Domain 모듈은 API 스펙이 변경되어도 비즈니스 로직은 변하지 않게한다.
- Domain 모듈은 DB 기술을 모르도록 하여 나중에 기술이 변해도 비즈니스 로직은 변하지 않게한다.

즉, 모듈로 분리하여 의존성을 관리하고 변경에 영향을 최소화한다.
### 모듈 간 의존 관계 설정
- BC 간의 의존 관계가 모듈 간 의존 관계이다.

---
## **모듈 관계 구조**
### 모듈 관계 다이어그램
![Api-Domain-Db 관계 2](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/Api-Domain-Db%20%E1%84%80%E1%85%AA%E1%86%AB%E1%84%80%E1%85%A8%202.svg)

API 모듈과 DB 모듈이 Domain 모듈을 의존한다. **Domain 모듈은 API 모듈과 DB 모듈을 의존하지 않는다.**

![멀티모듈 api-여러Domain-db](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/%E1%84%86%E1%85%A5%E1%86%AF%E1%84%90%E1%85%B5%E1%84%86%E1%85%A9%E1%84%83%E1%85%B2%E1%86%AF%20api-%E1%84%8B%E1%85%A7%E1%84%85%E1%85%A5Domain-db.svg)

위 다이어그램은 **API-Domain-DB 모듈 간의 의존 관계**를 나타냅니다. Domain 모듈 간의 의존 관계는 표현하지 않았습니다. (Domain 모듈 간의 의존 관계 ==  BC 간의 의존 관계)

### 도식화로 보는 멀티 모듈의 계층 구조 오해
![Api-Domain-Db 관계 1](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/Api-Domain-Db%20%E1%84%80%E1%85%AA%E1%86%AB%E1%84%80%E1%85%A8%201.svg)

모듈 관계 도식화를 가로에서 세로로 바꿔 보았습니다. 가로로 볼 때는 잘 느껴지지 않았던 구조가 보이지 않나요? `Controller -> Service -> Repository` 계층처럼 보입니다. 그럼 "Domain 모듈이 DB 모듈을 의존해야하는 것 아니야?" 라고 생각하실 수 있을 것 같아요. 처음에 이 구조를 봤을 때 저 또한 그렇게 생각했어요.

실제로 Domain 모듈에는 `Service` 클래스와 `Implement` 클래스가 존재합니다. 그리고 `Repository` 인터페이스도 존재하는데요. DB 모듈에 존재하는 `Repository` 구현체가 `Repository` 인터페이스를 구현합니다.  

즉, 이 구조는 Domain 모듈이 `Repository` 인터페이스를 정의함으로써 DB 모듈이 `Repository` 구현체를 구현한 DI(Dependency Injection, 의존성 주입)를 적용한 IOC(Inversion Of Control, 제어의 역전)가 적용된 것인데요. IOC가 적용됨으로써 레이어드 아키텍처인 `Controller -> Service -> (Implement) -> Repository` 계층은 유효합니다.

아래 그림은 실제 프로젝트의 `Cafe` 도메인 모듈의 `Repository` 인터페이스를 정의한 모습입니다.
![](https://i.imgur.com/9ARCzvr.png)

---
## **Domain 모듈 설계**
### Domain 모듈의 역할과 책임
- Domain 모듈은 핵심 비즈니스를 가진다.
- 핵심 비즈니스를 가지는 Domain 모듈은 외부(클라이언트, 기술 등)에 의존적이지 않아야 한다.

### 의존성 관리
- Domain 모듈에서 외부를 의존해야한다면 정말로 필요한지 고민하고 다른 모듈로 분리할 수는 없는지 확인해본다.

---
## **DB 모듈 설계**

### DB 모듈의 역할과 책임
- DB 모듈은 Domain 모듈의 추상화된 `Repository` 인터페이스를 구현한다. 인터페이스를 구현함으로써 Domain 모듈이 구현 기술을 모르게 한다.

### 의존성 관리
- DB 구현 기술을 의존한다.
- 다른 모듈이 DB 구현 기술을 모르게 한다.

---
## **API 모듈 설계**
### API 모듈의 역할과 책임
- API 모듈은 클라이언트와 맞닿아있다. HTTP 요청과 응답에 대한 처리를 담당한다.

### 의존성 관리
- REST API 문서화를 위한 라이브러리
- HTTP 요청/응답 처리 관련 컴포넌트(Spring Web, Jackson 등)
- 요청 유효성 검증을 위한 라이브러리(Validation 등)
- 보안 관련 컴포넌트(Spring Security 등)

---
## **그 이외의 모듈 설계**

### Auth 모듈 구성
#### Auth 모듈의 역할과 책임
- Auth 모듈은 인증(Authentication)을 담당하는 모듈이다.
- Auth 모듈은 Domain 모듈이 아니다.

#### Auth 모듈 도식화
![멀티모듈 api-여러Domain-db-auth](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/%E1%84%86%E1%85%A5%E1%86%AF%E1%84%90%E1%85%B5%E1%84%86%E1%85%A9%E1%84%83%E1%85%B2%E1%86%AF%20api-%E1%84%8B%E1%85%A7%E1%84%85%E1%85%A5Domain-db-auth.svg)

- API 모듈이 Auth 모듈을 의존한다.
- Auth 모듈은 `Member` 모듈을 의존한다.

### Common 모듈 구성

#### Common 모듈의 역할과 책임
- Common 모듈은 여러 모듈의 공통적인 부분을 담당한다.

#### Common 의 저주
Common이라는 함정에 빠져 공통적으로 사용한다고 공통적으로 사용하는 모든 것들을 Common 모듈에 넣게 된다면 모든 의존성은 Common 모듈로 향하게 됩니다. 멀티 모듈을 구성한 주요 목적은 모듈을 통한 관심사의 응집과 격리입니다. 하지만 모든 모듈은 Common 모듈을 바라보게 되면서 제대로 된 응집과 격리를 이뤄낼 수 없습니다.

#### Common에 대한 개인적인 생각
어디서 읽었는지 기억은 안나지만 어느 댓글의 내용이 기억에 남습니다.
> Common은 비즈니스 요소가 없는 경우에만 Common에 넣습니다.

이 댓글을 보고 Common 모듈에 어떤 요소들을 넣어야 할 지 대해서 고민을 많이 해봤는데요. 다음과 같은 결론을 내렸습니다.
- 비즈니스가 담겨있는 요소는 넣지 말 것
- 모든 모듈이 사용하는 Common을 만들지 말 것
두가지 원칙을 가지고 우선적으로 Domain 모듈에서 공통적으로 사용하면서 비즈니스 요소를 가지지 않는 Common-Domain 모듈을 만들었습니다.

아래 이미지는 실제 프로젝트에서 사용하는 Common 모듈입니다.
![](https://i.imgur.com/2kJ1a6d.png)

---
## **부록**
### 부록. Domain 모듈로 전환 시 컨트롤러 계층의 API 스펙으로 사용하는 DTO 분리하기

```java
public class StudyService {
    ....

    public CreateStudyResponse createStudy(CreateStudyRequest request) {
        ....
        return createStudyResponse;
    }
}
```
Domain 모듈이 따로 분리된 상태가 아니라면 대부분의 `Service` 클래스의 코드 스타일은 `RequestDTO` 를 매개 변수로 받고 `ResponseDTO` 를 반환값으로 보내주는 스타일입니다. 하지만 이 스타일의 문제점은 무엇일까요? 서비스 계층이 요청과 응답 스펙인 API 스펙에 영향을 미치고 있다는 점입니다.

우리는 지금까지 도메인 모델을 만들고 BC를 구성하고 각 BC를 모듈로 구성했습니다. 이 글에서 Domain 모듈은 다음과 같은 특성을 가진다고 했습니다.
> 핵심 비즈니스를 가지는 Domain 모듈은 외부(클라이언트, 기술 등)에 의존적이지 않아야한다. 

위에 작성된 `Service` 코드는 `RequestDTO` 와 `ResponseDTO`를 의존하고 있습니다. 도메인 모듈은 외부의 변화에 영향을 미치지 말아야 하는데 API 스펙이 변경될 때마다 영향을 받고 있습니다. 그럼 어떻게 구성하면 될까요?
```java
public class StudyService {
    ....

    public Long createStudy(Long memberId, LocalDateTime now, StudyContent content, Long cafeId) {
        ....
    }
}
```
도메인 모델과 관련된 값들을 매개변수와 반환값으로 넘겨주면 됩니다. 이렇게 구성하면 `Service`클래스(Domain 모듈에 포함되어 있는)는 외부에 영향을 받지 않게 됩니다.

그런데 이렇게 구성하게 되면 다음과 같은 의견이 나올 수 있습니다.
> 이 구성은 컨트롤러에서 도메인 모델에 대해서 아는 것 아닌가요?

네 맞습니다. 이렇게 구성하게 되면 `Controller` 클래스에서 도메인 모델을 알게됩니다. 그럼 저는 이렇게 다시 물어보고 싶어요.
> `Controller` 클래스에서 도메인 모델을 알면 안되는 이유가 뭔가요?

그럼 이렇게 답변하지 않을까요?
> `Controller` 는 Presentation Layer이기 때문에 도메인 모델을 알면 외부로 노출될 가능성이 있어요.

네 맞습니다. 외부로 노출될 가능성이 있죠. 하지만 외부로 노출된 것은 아니지 않을까요? 
```java
public class StudyController {
    @GetMapping
    public ResponseEntity<Study> getStudy(Long studyId) {
        ...
    }
}
```
위 코드와 같이 반환값으로 도메인 모델을 전달해주었을 경우엔 외부로 노출이 되겠지만 우리는 코드를 이렇게 작성하지 않습니다.
```java
public class StudyController {
    @GetMapping
    public ResponseEntity<StudyResponse> getStudy(Long studyId) {
        ...
    }
}
```
이런식으로 반환하게 되죠.

또 이런 질문도 있을 것 같아요.
> 휴먼에러가 발생하면요?

휴먼에러는 발생할 수 있습니다. 이 질문에 대한 대답도 어디서 보았던 답변으로 대체하겠습니다.
> Controller에서 도메인 로직을 실행시키는 개발자와 일을 하지 않으면 된다.<br>현업에서는 많은 코드리뷰를 통해 Approve된 코드들이 main 브랜치에 머지되므로 오염 가능성이 적다.

또 다른 해결 방안도 있습니다.
- `Controller` -> `Service` 로 넘겨주기 전에 `DTO`를 변환해서 넘겨준다.
이 해결방안은 <a target='_blank' href='https://techblog.woowahan.com/2711/'>잊을만 하면 돌아오는 정산 신병들</a>  라는 포스팅에서 언급되었습니다.

마지막으로 제 의견을 정리해보겠습니다.
- `Service` 클래스에서 `DTO`를 의존하는 방법이 틀렸다라는 것이 아니다.
- 어떻게 구성할 지는 아키텍처와 팀 스타일에 따라 달라진다. 특히, **팀 스타일에 맞추는 것은 굉장히 중요**하다.
- 단일 모듈이라면 `Service` 클래스에서 `DTO`를 의존하는 것은 생산성에 도움이 된다.
- 단일 모듈에서 멀티 모듈로 분리가 되었다. 여러 모듈 중 도메인 모듈이 존재하고 도메인 모듈은 외부에 영향을 받지 않도록 설계하기 위함이다.
- 외부에 영향을 받지 않게 하기 위해선 도메인과 관련된 값들을 매개변수와 반환값으로 사용해야한다.

---
### 부록. 화면에 맞는 데이터를 한방에 가져올 경우 DTO는 어디에 존재해야 하나요?

[부록. Domain 모듈로 전환 시 컨트롤러 계층의 API 스펙으로 사용하는 DTO 분리하기](#부록-domain-모듈로-전환-시-컨트롤러-계층의-api-스펙으로-사용하는-dto-분리하기)에서는 `DTO`를 도메인 모듈이 의존하지 않는 것을 보았는데요. 사실 이 방식은 이상에 가깝습니다. 도메인 모듈이 API 응답값을 넘겨주지 않고 도메인 모델 또는 관련된 값을 반환하게 된다면 `Controller` 에서 응답에 맞는 스펙으로 바꿔주어야 합니다.
```java
@GetMapping  
public ResponseEntity<StudyDetailResponse> getStudyDetail(@PathVariable Long studyId) {  
    Study study = studyQueryService.getStudy(studyId);  
    int viewCount = studyQueryService.getViewCount(studyId);  
    int participantCount = studyMemberQueryService.getParticipantCount(studyId); 
    Cafe cafe = cafeQueryService.getCafe(study.getCafeId());  
  
    StudyDetailResponse response = StudyDetailResponse.of(cafe, study, viewCount, participantCount);  // 도메인 모델 또는 관련된 값을 Response로 매핑해야한다.
    return ResponseEntity.ok(response);  
}
```

도메인 모델 또는 관련된 값을 반환해서 `Controller` 에서 매핑하는 방식을 선택할 수도 있지만 매번 이 방식만 고집할 수는 없습니다. 이 방식은 필요한 도메인 모델들을 반환함으로써 여러번의 쿼리가 나가게 됩니다. 하지만 성능이 중요한 API는요? 성능이 중요한 API의 경우 한방 쿼리를 만들어 화면에 맞는 데이터를 수집해야합니다. 아래와 같은 방식의 코드를 작성해서말이죠.

```java
@GetMapping  
public ResponseEntity<StudyDetailResponse> getStudyDetail(@PathVariable Long studyId) {  
    StudyDetailResponse response = studyQueryService.getStudyDetail(studyId);
    return ResponseEntity.ok(response);  
}


public class StudyQueryRepositoryImpl implements StudyQueryRepository {
    public StudyDetailResponse getCafeStudy(Long studyId) {
       ... // Repository 구현체에서 API 응답에 맞는 데이터를 반환한다. 
    }
}
```

그럼 `DTO`는 어느 모듈에 존재해야할까요? 다음 이미지는 이전에 보았던 모듈 관계 도식화입니다.
![Api-Domain-Db 관계 2](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/Api-Domain-Db%20%E1%84%80%E1%85%AA%E1%86%AB%E1%84%80%E1%85%A8%202.svg)

모듈 간의 관계는 API -> Domain <- DB 을 나타내고 있습니다. (사실 Gradle 빌드 툴에서 API 모듈은 DB 모듈을 `runtimeOnly` 로 의존하고 있으나 이 부분은 관련 없으니 생략합니다.)

도식화를 보면 `DTO`는 Domain 모듈에 있어야한다는 것을 알 수 있습니다. Domain 모듈에 있어야 API 응답으로 나갈 수 있습니다.

그럼 이런 의견이 나올 수 있겠죠?
> 지금까지 Domain 모듈이 외부에 의존하지 않게 하려고 모듈로 분리한 거 아니예요?

네 맞아요. 하지만 이렇게 분리 하더라도 한방 쿼리의 경우 `DTO`를 `Domain` 모듈에서 의존할 수 밖에 없습니다. 저는 아직까지 다른 해결방안을 찾지 못했습니다. 혹시나 다른 해결방안이 떠오른다면 댓글로 남겨주세요.

마지막으로 제 의견을 정리해보겠습니다.
- 한방 쿼리의 경우 `DTO`는 Domain 모듈에서 가지고 있는다.
- 은탄환은 없다. 어느 정도 유연함은 가져가자.

---
## **단일 모듈에서 멀티 모듈 전환기 목차**

- <a target='_blank' href='/posts/single-to-multi-module-(1)'>단일 모듈에서 멀티 모듈 전환기 - (1) 멀티 모듈이란?</a>
- <a target='_blank' href='/posts/single-to-multi-module-(2)'>단일 모듈에서 멀티 모듈 전환기 - (2) 구현 계층</a>
- <a target='_blank' href='/posts/single-to-multi-module-(3)'>단일 모듈에서 멀티 모듈 전환기 - (3) 도메인 모델</a>
- <a target='_blank' href='/posts/single-to-multi-module-(4)'>단일 모듈에서 멀티 모듈 전환기 - (4) JPA 엔티티 격리 및 DB 추상화</a>
- <a target='_blank' href='/posts/single-to-multi-module-(5)'>단일 모듈에서 멀티 모듈 전환기 - (5) Bounded Context 설계</a>
- <a target='_blank' href='/posts/single-to-multi-module-(6)'><span style='color: #ef5369'>단일 모듈에서 멀티 모듈 전환기 - (6) 멀티 모듈 설계</span></a>
- <a target='_blank' href='/posts/single-to-multi-module-(7)'>단일 모듈에서 멀티 모듈 전환기 - (7) 멀티 모듈 전환 with Gradle</a>
- <a target='_blank' href='/posts/single-to-multi-module-(8)'>단일 모듈에서 멀티 모듈 전환기 - (8) 회고 및 마무리</a>

---
## 참고 문서
- <a target='_blank' href='https://techblog.woowahan.com/2711/'>https://techblog.woowahan.com/2711/</a>
