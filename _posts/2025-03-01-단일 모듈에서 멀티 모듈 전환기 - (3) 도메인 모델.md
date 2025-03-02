---
title: 단일 모듈에서 멀티 모듈 전환기 - (3) 도메인 모델
# description: >-
#   description
date: 2025-03-01 17:13:00 +0900
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

<a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(2)-구현-계층'>'단일 모듈에서 멀티 모듈 전환기 - (2) 구현 계층'</a>에서 이어집니다.


---
## **협력 도구 클래스로 재사용 가능한 코드를 구성할 때의 문제점**

### 문제 상황
```java
public class CommentEditor {
    private final CommentRepository commentRepository;
    private final CommentValidator commentValidator;
    
    public void edit(String content, Comment comment) {
        commentValidator.validateContentNotBlank(content);
        commentValidator.validateContentLength(content);
        
        comment.changeContent(content);
    }
}
```

구현 계층에 존재하는 댓글을 수정하는 협력 도구 클래스입니다. 매개변수로 `String` 타입인 `content`를 받고 있습니다. 협력 도구 클래스는 재사용이 가능한 클래스입니다. 만약 댓글을 수정할 때 댓글 내용뿐만 아니라 다양한 요소들이 추가되어 같이 수정하게 되면 어떻게 될까요?

다음 코드와 같이 매개변수의 개수가 증가하게 됩니다.
```java
public void edit(String content, boolean isAnonymous, String category,
                boolean isSecret, List<String> hashtags, Comment comment) {
    ...
}
```

우리는 요구사항이 추가될 때마다 매개변수의 개수를 점점 늘려야만합니다. 우리는 매개변수의 개수가 점점 늘어나는 것이 맘에 들지 않습니다. 특히 `String` 타입의 매개변수가 많아진다면요?
```java
public void edit(String a, String b, String c ...) {
    ...
}
```

**휴면 에러**가 발생하기 딱 좋은 코드가 됩니다. 그럼 어떤 방법으로 문제를 해결할 수 있을까요?

### 해결 방안
- 컨트롤러에서 넘어오는 요청 DTO를 매개변수로 받는다.
- VO(Value Object)를 매개변수로 받는다.

#### 1. 컨트롤러에서 넘어오는 요청DTO 를 매개변수로 받는다.
```java
public void edit(EditCommentRequest request, Comment comment) {
    ...
    comment.change(request.getA(), request.getB(), ...);
}
```

댓글 수정에 필요한 요청 DTO를 매개변수로 받았습니다. 많은 매개변수를 하나의 DTO로 묶어서 해결했습니다. 코드가 깔끔해졌습니다. 하지만 이 코드에는 어떤 문제가 있을까요? 컨트롤러에서 넘어오는 **API 스펙**이 그대로 구현 계층인 **협력 도구 클래스**로 넘어오고 있습니다. <a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(2)-구현-계층#구현-계층의-장단점'>멀티 모듈 시리즈 - (2) 구현(Implementation) 계층</a>에서 다음과 같이 언급했습니다.

> 협력 도구 클래스를 통해 비즈니스 로직을 구현하고 **재사용성**을 높인다.

**재사용성**을 높인다고 언급했습니다. 요청 DTO를 매개변수로 받게되면 **클라이언트의 요구사항이 변경**될 때마다 **협력 도구 클래스의 메서드는 변경**됩니다. 협력 도구 클래스는 재사용이 가능한 클래스이므로 요청 DTO를 매개변수로 받을 수 없습니다.

#### 2. VO(Value Object)를 매개변수로 받는다.

> Value Object는 도메인에서 한 개 또는 그 이상의 속성들을 묶어서 특정 값을 나타내는 객체를 의미한다.

도메인과 관련있는 값들을 묶어서 VO를 구성하여 댓글 수정 메서드의 매개변수로 받았습니다.
```java
public class CommentContent {
    private String content;
    private boolean isAnonymous;
    private String category;
    private boolean isSecret;
    private List<String> hashtags;
}


public void edit(CommentContent content, Comment comment) {
    ...
    comment.change(content);
}
```

관련있는 값들을 묶은 VO를 매개변수로 받으니 요청 DTO가 변경될 때 댓글 수정 메서드는 영향을 받지 않게 됩니다. 물론 댓글 수정하는 요소가 추가되거나 변경되면 댓글 JPA 엔티티 내부는 변경이 일어납니다. 

### 해결 및 문제점

두가지의 해결 방안을 보았고 클라이언트와 연관되어있는 DTO를 통해 해결하는 것보다 도메인과 관련된 VO를 받아 해결하는 것이 클라이언트와 느슨한 연결을 가진다는 것을 알게되었습니다. 따라서 VO를 통해 문제를 해결했습니다.

---
## **현재 상황 및 문제점**

그럼 현재상황을 다시 볼까요?
- JPA 엔티티를 사용하고 있다.
- 협력 도구 클래스에서 VO를 사용하고 있다.
- JPA 엔티티 내부에서 VO를 의존하고 있다.
- JPA 엔티티와`@Embeddable`, `@Embedded`를 사용한 VO 객체, POJO의 VO 객체가 존재하고 있다.

이번엔 JPA 엔티티와 `@Embeddable`, `@Embedded`를 사용한 VO 객체, POJO의 VO객체가 존재하는 예시를 보겠습니다.
```java
@Entity
public class Cafe {
    @Embedded
    private AddressEmbeddable addressEmbeddable;
    
    public void changeAdderess(Address address) {
        this.addressEmbeddable.getRegion(address.getRegion());
    }
}


@Embeddable
public class AddressEmbeddable {
    private String region;
}


// POJO Address
public class Address {
    private String region;
}
```

`@Embeddable`이 붙은 `AddressEmbeddable` 객체와 POJO의 `Address` 객체가 존재합니다.

`@Embeddable` 과` @Embedded` 는 다음과 같은 특징을 가집니다.
- VO(값 타입)을 정의하고 사용할 때 JPA 엔티티에 사용됩니다.
- **테이블의 컬럼으로 포함**된다.

현재 상황에서 프로젝트에는 **@Embeddable이 붙은 VO와 POJO의 VO가 공존**하고 있습니다. 하나는 테이블의 컬럼과 연관된 VO이고 하나는 애플리케이션의 레벨에서 도메인과 관련된 VO입니다. 같은 역할을 하면서 하나는 기술에 의존하고 있고 하나는 순수한 자바 코드로 구성되어 있습니다. 즉, 관리 포인트가 두개로 늘어났습니다.

프로젝트에서 JPA 엔티티의 VO와 도메인과 관련된 VO가 공존함에 따라 저희는 **JPA라는 기술에 의존하지 않는 도메인 모델을 도입**하기로 결정했습니다.

### 해결: 도메인 모델 도입

도메인 모델을 도입하게 된 이유를 정리하자면 다음과 같습니다.
- JPA 기술에 의존한 VO와 POJO인 VO가 공존하고 있다.
- POJO VO 대신 JPA VO로 대체할 수도 있었으나 **DDD에서 말하는 도메인**이 무엇인지 궁금했다.
- **기술에 의존하지 않는 도메인 모델과 JPA 엔티티의 차이**는 무엇이고 어떤 **이점**이 존재하는지 궁금했다.

저희는 도메인 모델을 도입할 때 다음과 같은 규칙을 정했습니다.
- **어떠한 아키텍처에 대해서 공부하지 말것** (DDD, 헥사고날 등등)

이론을 공부하게 되면 상황에 맞는 코드를 구성하는 것이 아니라 이론 자체에 매몰되게 됩니다. 따라서 상황에 맞는 도메인 모델을 구성하여 어떤 장단점이 존재하는지, 어떻게 도메인 모델을 구성해야 좋은 도메인 모델이 나오는지에 초점을 맞췄습니다.

---
## **도메인 모델이란?**

### 도메인 모델 정의

> 도메인 모델(domain model)은 행위와 데이터를 둘 다 아우르는 도메인의 개념 모델(도메인에서 중요한 개념과 관계를 추상화)이다.
> 
> <a target='_blank' href='https://ko.wikipedia.org/wiki/%EB%8F%84%EB%A9%94%EC%9D%B8_%EB%AA%A8%EB%8D%B8#cite_ref-PoEAABook_1-0'>출처: 위키디피아 - 도메인 모델</a>
>
> 도메인 모델은 단순 클래스 다이어그램이 아니고, **도메인의 핵심을 간략히 단순화해서 표현할수 있는 모든 것**이 도메인 모델이다. 도메인 모델을 봤을 때 **도메인의 개념** 뿐 아니라, **코드도 함께 이해될 수 있는 구조**를 찾는 것.
> 
> <cite>— 조영호, 『오브젝트』</cite>

조영호님의 저서 `오브젝트`는 이 글에서 다루고자 하는 도메인 모델의 개념을 잘 설명하고 있습니다. 이 프로젝트에서는 애플리케이션에서 도메인 모델을 봤을 때 도메인의 개념과 코드로 함께 이해할 수 있는 코드를 만들고자 했습니다.

---
## **JPA 엔티티와 도메인 모델의 차이점**

### JPA 엔티티
> JPA 엔티티는 데이터베이스 테이블과 매핑될 수 있는 자바 클래스이며, 이러한 클래스들은 **데이터의 구조를 정의**하고 자바의 **객체지향 세계와 데이터베이스의 관계형 세계 사이의 다리 역할**을 합니다.

JPA 엔티티는 다음과 같은 특징을 가집니다.
- JPA 엔티티에 정의된 필드는 DB 테이블의 컬럼으로 매핑된다.
- 애플리케이션에서 JPA라는 기술에 의존한 도메인 모델이다.

아래 코드는 JPA 엔티티의 예시입니다.
```java
@Entity
public class StudyEntity {
    @Id
    private Long id;
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cafe_id")
    private CafeEntity cafe;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private MemberEntity member;
    
    @Embedded
    private StudyPeriod studyPeriod;
    
    @Enumerated(EnumType.STRING)
    private MemberComms memberComms;
    
    private int maxParticipants;
    private String introduction;
    private int views;
    
    @Enumerated(EnumType.STRING)
    private RecruitmentStatus recruitmentStatus = RecruitmentStatus.OPEN;
    ...
}
```

### 도메인 모델

위에서 정의한 [도메인 모델](#도메인-모델-정의)의 의미를 포함하면서 '<a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(1)-멀티-모듈이란?#용어-정의'>멀티 모듈 시리즈 - (1) 멀티 모듈이란?</a>'에서 다음과 같이 정의했습니다. 

> 기술에 의존하지 않은 자바 클래스(POJO)로 구성된 도메인 모델: **도메인 모델**

멀티모듈 시리즈의 도메인 모델은 다음과 같은 특징을 가집니다.
- JPA라는 기술에 의존하지 않고 순수한 자바 코드(POJO)로 작성된다.
- JPA 엔티티와 POJO 도메인 모델은 일대일 대응이 아니다.
- JPA 엔티티와 비교해서 POJO 도메인 모델은 유연하다.
- 관련있는 데이터끼리 VO를 구성하고 VO는 도메일 모델에 속한다.

아래 코드는 POJO 도메인 모델을 코드로 나타낸 것입니다.
```java
public class Study {
    private Long id;
    private StudyContent content;
    private Long cafeId;
    private Long memberId;
    private RecruitmentStatus recruitmentStatus;
    ...
}
```

---
## **도메인 모델 구성하기**

[이전 섹션](#해결-도메인-모델-도입)에서 다음과 같은 이유로 도메인 모델을 도입하기로 했습니다.

>  - JPA 기술에 의존한 VO와 POJO인 VO가 공존하고 있다.
>  - POJO VO대신 JPA VO로 대체할 수도 있었으나 **DDD에서 말하는 도메인**이 무엇인지 궁금했다.
>  - **기술에 의존하지 않는 도메인 모델과 JPA 엔티티의 차이**는 무엇이고 어떤 **이점**이 존재하는지 궁금했다.

### 도메인 모델의 구조

JPA 엔티티와 다르게 도메인 모델은 정해진 구조가 없습니다. 어떤 기술에도 의존하지 않은 POJO로 구성되기때문에 팀내에서 협의를 통해 도메인 모델을 구성하면 됩니다. 관련있는 데이터끼리 VO로 묶고 도메인 모델에서 VO를 다루면 됩니다. 단, **도메인 모델은 JPA 엔티티와 일대일 대응이 아니라는 것**을 인지하고 구성하면 더 좋습니다.

아래 코드는 `Study` 도메인 모델의 예시입니다.
```java
public class Study {
    private Long id;
    private StudyContent content;
    private Long cafeId;
    private Long memberId;
    private RecruitmentStatus recruitmentStatus;
    ...
}


public class StudyContent {
    private String name;
    private Schedule schedule;
    private MemberComms memberComms;
    private int maxParticipantCount;
    private String introduction;
    private List<StudyTagType> tags;
    ...
}
```

저희는 도메인 모델을 설계할 때 관련있는 데이터를 VO로 묶었습니다. 

`도메인이름Content`라고 정의한 VO도 볼 수 있는데요. 이 VO는 다음과 같은 특징을 가집니다.
- 가변적인 데이터를 묶은 VO이다.

클라이언트의 요청에 따라 데이터가 변할 수 있는 데이터를 VO를 통해 묶었습니다. 코드 레벨에서 가변적인 데이터는 같이 다니는 경우가 많았기때문입니다.

---
## **도메인 모델의 장단점**

**장점**
- 도메인 모델은 유연하다. 어떠한 기술에 의존하지 않기 때문에 손쉽게 변경할 수 있다.
- DB 테이블에 의존적이지 않기때문에 표현하고 싶은 형태로 구성할 수 있다.
- 유연한만큼 장기적으로 유지보수하기 좋다.

**단점**
- 도메인 모델은 유연한 만큼 관리해야 할 클래스가 늘어난다.
- 도메인 모델을 도입하고 ORM 기술로서 JPA 엔티티를 사용한다면 도메인 모델과 JPA 엔티티 모두 관리해야한다.
- 생산성이 떨어진다.

---
## **도메인 모델 설계 시 고려사항**

### JPA 엔티티와 도메인 모델은 일대일 대응이 아니다

도메인 모델을 설계할 때 가장 많이 하는 실수가 있습니다. 도메인 모델과 JPA 엔티티는 일대일 대응이 아니라는 점입니다. 일대일 대응으로 구성하게 되면 도메인 모델의 이점을 살리지 못합니다.

> 도메인 모델을 봤을 때 **도메인의 개념** 뿐 아니라, **코드도 함께 이해될 수 있는 구조**를 찾는 것.

도메인 모델이 유연한만큼 코드도 유연하게 작성하고 개념을 코드로 이해할 수 있도록 구성해보세요. 이것이 도메인 모델의 장점입니다.

### Study JPA 엔티티의 Study 도메인 모델 분리 사례

다음 그림은 이전에 보았던 프로젝트의 ERD 입니다.
![erd](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202025-02-02%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%206.15.05.png)

'<a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(1)-멀티-모듈이란?#프로젝트-이해를-위한-erd'>멀티 모듈 시리즈 - (1) 멀티 모듈이란?</a>'에서 다음과 같은 요구사항이 존재했습니다.

> 회원은 여러개의 스터디를 생성 할 수 있다.
> - **Study 엔티티의 member_id** 는 스터디를 생성한 **스터디장**을 의미한다.

아래 코드는 `Study` 도메인의 JPA 엔티티와 도메인 모델을 나타냅니다.
```java
@Entity
public class StudyEntity {
    @Id
    private Long id;
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cafe_id")
    private CafeEntity cafe;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id")
    private MemberEntity member;  // Study의 스터디장을 의미한다.
    
    @Embedded
    private StudyPeriod studyPeriod;
    
    @Enumerated(EnumType.STRING)
    private MemberComms memberComms;
    
    private int maxParticipants;
    private String introduction;
    private int views;
    
    @Enumerated(EnumType.STRING)
    private RecruitmentStatus recruitmentStatus = RecruitmentStatus.OPEN;
    ...
}


public class Study {
    private Long id;
    private StudyContent content;
    private Long cafeId;
    private Long memberId;  // Study의 스터디장을 의미한다.
    private RecruitmentStatus recruitmentStatus;
    ...
}
```

#### 문제 상황

프로젝트를 진행하면서 저와 팀원은 `Study` 도메인에 대해서 의견을 계속 나누었는데요.  `Study`를 만든 회원을 의미하는 **스터디장**이라는 용어가 계속 쓰이기 시작했습니다.

> "스터디에는 스터디원이 존재하고 스터디장이 존재해요. 스터디장은 회원을 관리할 수 있죠."

저희는 다음과 같은 문제를 맞닥뜨리게 되었습니다.
- `memberId`는 `Study` 도메인 모델 외부에서도 돌아다니게 되는데, 외부에서 돌아다니는  `Long`타입의 `memberId`가 의미하는 것이 **서비스를 이용하는 회원의** `memberId`를 의미하는 것인지 **스터디장**으로서의 `memberId`를 의미하는 것인지 모르겠다.
- 코드에서 돌아다니는 `memberId`의 변수명을 `memberId`라고 지을 경우 `memberId`가 **서비스를 이용하는 회원**인지 **스터디장으로서의 회원**인지 모르겠다.

```java
public void validateCoordinatorIsInStudy(Study study, List<Long> memberIds) {  
    boolean isCoordinator = memberIds.stream()	// List<Long> memberIds 가 스터디장으로서의 회원이라고 보장할 수 있을까?
		.anyMatch(study::isManagedBy);// 누군가 다른 memberId를 넣어준다면 어떡하지?

    if (!isCoordinator) {
        throw new RuntimeException(STUDY_INVALID_LEADER);
    }
}


public class Study {
    private Long id;
    private StudyContent content;
    private Long cafeId;
    private Long memberId;  // Study의 스터디장을 의미한다.
    private RecruitmentStatus recruitmentStatus;
  
    public boolean isManagedBy(Long memberId) {  // memberId가 스터디장인지 확인한다.
        return this.memberId.equals(memberId);
    } 
}
```

#### 해결 방안
- `Coordinator`(스터디장) 라는 도메인 모델을 만들어 `memberId`를 감싼다. 

저희는 다음과 같은 해결 방안을 고안하여 `Coordinator` 도메인 모델을 만들었습니다.

#### 해결: Coordinator 도메인 모델 생성

따라서 `Study` Jpa 엔티티와 `Study` 도메인 모델을 일대일 대응으로 구성하지 않고 유연하게 구성함으로써 문제를 해결했습니다.
```java
public void validateCoordinatorIsInStudy(Study study, List<Coordinator> coordinators) {
    boolean isCoordinator = coordinators.stream()  // List<Coordinator> 로 받기 때문에 안전하다.
        .anyMatch(study::isManagedBy);
        
    if (!isCoordinator) {
        throw new RuntimeException(STUDY_INVALID_LEADER);
    }
}


public class Study {
    private Long id;
    private StudyContent content;
    private Long cafeId;
    private Coordinator coordinator;  // memberId를 감싼 Coordinator
    private RecruitmentStatus recruitmentStatus;
    
    public boolean isManagedBy(Coordinator coordinator) {
      return this.coordinator.isSameCoordinator(coordinator);
    }
}


public class Coordinator {
    private Long memberId;
    private String nickname;
    
    public boolean isSameCoordinator(Coordinator coordinator) {
        return this.memberId.equals(coordinator.getId());
    }
}
```

### Bounded Context에 따른 도메인 용어의 분리

[Study JPA 엔티티의 Study 도메인 모델 분리 사례](#study-jpa-엔티티의-study-도메인-모델-분리-사례)와 이어지는 내용으로 DDD에서 언급하는 Bounded Context에 대해서 간략하게 다룹니다. 

#### Bounded Context란?
> Bounded Context는 Domain-Driven Design(DDD)의 핵심 패턴입니다. 이는 DDD의 전략적 설계 섹션의 중심이며, 대규모 모델과 팀을 다루는 것에 관한 것입니다. DDD는 대규모 모델을 여러 Bounded Context로 나누고 그들 간의 상호 관계를 명시적으로 다룸으로써 큰 모델을 다룹니다.
> 
> <a target='_blank' href='https://martinfowler.com/bliki/BoundedContext.html'>출처: Martin Fowler - Bounded Context</a>

글로 나타낸 정의는 이해하기 어려울 수 있으므로 다이어그램을 통해 살펴보겠습니다.

![image-20250208182707644](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/image-20250208182707644.png)

위 다이어그램은 두개의 Bounded Context(BC) 가 존재합니다. `Study` BC와 `Member` BC 인데요. `Study` 라는 경계를 가진 Context와 `Member`라는 경계를 가지는 Context 가 있습니다.

Martin Fowler는 BC에 대해서 다음과 같이 언급했습니다.

> 더 큰 도메인을 모델링하려고 할수록, 단일 통합 모델을 구축하기가 점점 더 어려워집니다. 큰 조직의 **다른 부분에서 다른 그룹의 사람들은 미묘하게 다른 어휘를 사용**합니다. 모델링의 정밀성은 이런 상황에 빠르게 부딪히며, 종종 많은 혼란을 초래합니다. 일반적으로 이 혼란은 도메인의 핵심 개념에 집중됩니다. 
> 
><a target='_blank' href='https://martinfowler.com/bliki/BoundedContext.html'>출처: Martin Fowler - Bounded Context</a>

"사람들은 미묘하게 다른 어휘를 사용한다." 라고 언급하고 있네요. 우리는 [Study JPA 엔티티의 Study 도메인 모델 분리 사례](#study-jpa-엔티티의-study-도메인-모델-분리-사례)에서 보았듯이 `Study` 도메인에 관해 이야기할 때 **스터디장**이라는 어휘를 사용했죠. 실제로 스터디장은 `Member`인데 말이죠.

#### 동일 엔티티의 Context별 도메인 모델 예시

![image-20250208184002007](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/image-20250208184002007.png)

실제로는 `Study` BC의 `Coordinator`와 `Member` BC의 `Member`는 같음을 알 수 있습니다.

---
## **단일 모듈에서 멀티 모듈 전환기 목차**

- <a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(1)-멀티-모듈이란?'>단일 모듈에서 멀티 모듈 전환기 - (1) 멀티 모듈이란?</a>
- <a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(2)-구현-계층'>단일 모듈에서 멀티 모듈 전환기 - (2) 구현 계층</a>
- <a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(3)-도메인-모델'><span style='color: #ef5369'>단일 모듈에서 멀티 모듈 전환기 - (3) 도메인 모델</span></a>
- <a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(4)-JPA-엔티티-격리-및-DB-추상화'>단일 모듈에서 멀티 모듈 전환기 - (4) JPA 엔티티 격리 및 DB 추상화</a>
- <a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(5)-Bounded-Context-설계'>단일 모듈에서 멀티 모듈 전환기 - (5) Bounded Context 설계</a>
- <a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(6)-멀티-모듈-설계'>단일 모듈에서 멀티 모듈 전환기 - (6) 멀티 모듈 설계</a>
- <a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(7)-멀티-모듈-전환-with-Gradle'>단일 모듈에서 멀티 모듈 전환기 - (7) 멀티 모듈 전환 with Gradle</a>
- <a target='_blank' href='/posts/단일-모듈에서-멀티-모듈-전환기-(8)-회고-및-마무리'>단일 모듈에서 멀티 모듈 전환기 - (8) 회고 및 마무리</a>

---
## **참고 문서**
- <a target='_blank' href='https://ksh-coding.tistory.com/83'>https://ksh-coding.tistory.com/83</a>
- <a target='_blank' href='https://ko.wikipedia.org/wiki/%EB%8F%84%EB%A9%94%EC%9D%B8_%EB%AA%A8%EB%8D%B8#cite_ref-PoEAABook_1-0'>https://ko.wikipedia.org/wiki/%EB%8F%84%EB%A9%94%EC%9D%B8_%EB%AA%A8%EB%8D%B8#cite_ref-PoEAABook_1-0</a>
- <a target='_blank' href='https://martinfowler.com/eaaCatalog/domainModel.html'>https://martinfowler.com/eaaCatalog/domainModel.html</a>
- <a target='_blank' href='https://www.geeksforgeeks.org/jpa-entity-introduction/'>https://www.geeksforgeeks.org/jpa-entity-introduction/</a>
