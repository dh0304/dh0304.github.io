---
title: 단일 모듈에서 멀티 모듈 전환기 - (2) 구현(Implementation) 계층
# description: >-
#   description
date: 2025-02-28 20:50:00 +0900
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

<a target='_blank' href='/posts/single-to-multi-module-(1)'>'단일 모듈에서 멀티 모듈 전환기 - (1) 멀티 모듈이란?'</a>에서 이어집니다.

---
## **레이어드 아키텍처란? (Layered Architecture)**

![layered-architecture](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/sapr_0101.png)
<p align="center"><em>Layered Architecture</em> (출처: <a target='_blank' href='https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/ch01.html'>oreilly</a>)</p>
레이어드 아키텍처는 각 계층이 서로 독립적으로 구성되어 있어, 한 계층의 변경이 다른 계층에 영향을 주지 않게 설계할 수 있습니다. 

각 계층은 애플리케이션 내에서 특정 역할을 수행합니다.
- Presentation 계층: 사용자 인터페이스와 브라우저 통신 로직을 처리하는 책임을 가진다.
- Business 계층: 요청과 관련된 특정 비즈니스 로직을 실행하는 책임을 가진다.
- Persistence 계층: 데이터베이스와 상호작용하며 데이터를 저장 등의 관리의 책임을 가진다.
- Database 계층: 실제 데이터가 저장되는 물리적인 저장소이다.

중요한 것은 레이어드 아키텍처는 계층이 정해져있는 것이 아니라 조금씩 다른 구성을 가질 수 있습니다.
![layered-architecture](https://raw.githubusercontent.com/donghyun0304/ImageRepo/master/uPic/img.png)

---
## **전통적인 레이어드 아키텍처**

### 전통적인 레이어드 아키텍처의 구조

레이어드 아키텍처에서 각 계층을 부르는 용어는 사람마다 다를 수 있습니다. 하지만 보통 의미는 일맥상통합니다. 많이 언급되는 `Controller`, `Service`, `Repository` 계층을 보겠습니다.

![image-20250226193139289](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/image-20250226193139289.png)
<p align="center"><em>전통적인 레이어드 아키텍처</em> (출처: <a target='_blank' href="https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63">김재민님의 지속 성장 가능한 소프트웨어</a>)</p>

전통적인 레이어드 아키텍처는 다음과 같은 특징을 가집니다.
- `Controller` -> `Service` -> `Repository`순으로 단방향 의존을 가집니다.
- 비즈니스 로직은 `Service` 에 구현해야합니다.

### 전통적인 레이어드 아키텍처의 Service 계층의 문제점

아래 코드는 댓글 수정을 하는 비즈니스 로직입니다.
```java
public void editComment(String content, Long commentId, Long memberId) {
    if(!StringUtils.hasText(content)) {  // 댓글의 내용이 공백인지 검증한다.
        throw new RuntimeException(COMMENT_CONTENT_NOT_BLANK); 
    }
    if(content.length() > 100) {  // 댓글의 길이가 100자 초과인지 검증한다.
        throw new RuntimeException(COMMENT_CONTENT_TOO_LONG);
    }
    
    Comment comment = commentQueryRepository.find(commentId)  // 댓글을 찾아온다.
            .orElseThrow(() -> new RuntimeException(COMMENT_NOT_FOUND));

    if (!comment.isAuthor(memberId.getId())) {  // 댓글 수정을 요청한 회원이 댓글 작성자인지 확인한다.
        throw new RuntimeException(COMMENT_PERMISSION_DENIED);
    }
    if (commentRepository.existsReplies(commentId)) {  // 댓글에 답변이 달렸는지 확인한다.
        throw new RuntimeException(COMMENT_HAS_NOT_REPLY);
    }
    
    comment.changeContent(content);
}
```

댓글을 수정하기 위해 다음 로직을 수행합니다.
1. 댓글의 내용이 `null`인지 검증한다.
2. 댓글의 길이가 100자 초과인지 검증한다.
3. 댓글을 찾아온다.
4. 댓글 수정을 요청한 회원이 댓글 작성자인지 확인한다.
5. 댓글에 답변이 달렸는지 확인한다.
6. 댓글을 수정한다.

댓글 수정이라는 간단한 기능도 6단계의 로직을 수행합니다.

서비스 계층에 존재하는 댓글 수정 메서드는 다음 책임을 지고있습니다.
- 요청 데이터 검증
- 엔티티 찾기
- 비즈니스 규칙 검증
- 데이터 수정

현재 서비스 계층이 가지고 있는 여러 책임 중에서 **어느 부분이 비즈니스 로직**일까요? 코드만 보고는 어느 부분이 비즈니스 로직인지 한눈에 확인하기 어렵습니다. 물론 예제 코드이기때문에 어렵지 않게 확인할 수도 있지만 현실 세계의 비즈니스는 더 복잡할 것입니다.

김재민님의 [지속 성장 가능한 소프트웨어를 만들어가는 방법](https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63){:target="_blank"}에서는 다음과 같은 의견을 남겨주셨습니다.
> 제가 느끼기엔 **비즈니스 로직**보다는 **상세한 구현 로직**에 가깝다고 느껴집니다. 신규 입사자가 왔을 때 이 코드를 기준으로 비즈니스 로직을 설명하여 이해시키고 업무에 쉽게 적응하도록 도울 수 있을까요? 저는 아쉬운 부분이 있다고 생각합니다.
>
> **비즈니스 로직은 상세 구현 로직은 잘 모르더라도 비즈니스의 흐름은 이해 가능한 로직이어야 한다.**

---
## **구현(Implemenation) 계층이란?**

### 구현 계층 정의
![implement-layer](https://raw.githubusercontent.com/dh0304/ImageRepo/master/uPic/image-20250204202615541.png)
<p align="center"><em>구현 계층이 포함된 레이어드 아키텍처</em> (출처: <a target='_blank' href="https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63">김재민님의 지속 성장 가능한 소프트웨어</a>)</p>
서비스 계층(Business Layer), 구현 계층(Implement Layer)에 대해서만 살펴보겠습니다.
>**Business Layer**는 **비즈니스 로직을 투영하는 레이어**입니다. 만약 코드가 계속 성장하여 비즈니스 로직이 너무 많아지거나 결합이 되어야 하는 경우 당연히 상위 레이어를 더 쌓아올립니다.
>
>**Implement Layer**는 [위에 코드](#전통적인-레이어드-아키텍처의-service-계층의-문제점)에서 봤던 **비즈니스 로직을 이루기 위해 도구**로서 상세 구현 로직을 갖고 있는 클래스들이 있습니다. 이곳은 가장 많은 클래스들이 존재하고 있으면서 구현 로직을 담당하기 때문에 **재사용성도 높은 핵심 레이어**입니다.

구현 계층이 포함된 레이어드 아키텍처는 여러가지 제약이 존재합니다.
> 1. 레이어는 위에서 아래로 순방향으로만 참조 되어야한다.
> 2. 레이어의 참조 방향이 역류 되지 않아야한다.
> 3. 레이어의 참조가 하위 레이어를 건너 뛰지 않아야한다.
> 4. 동일 레이어 간에는 서로 참조하지 않아야한다. (Implemen Layer는 동일 레이어 안에서 서로 참조 가능하다.)

### 구현 계층의 책임과 역할

구현 계층은 다음 책임과 역할을 가지고 있습니다.
> 네번째 규칙에서 Implement Layer은 서로 참조가 가능하게 한 이유는 Implement Layer 클래스들의 **재사용성**을 늘리고 **협력이 가능한 높은 완결성의 도구 클래스**들을 더 많이 만들게 합니다. 또한 비즈니스 로직의 오염을 막기 위한 규칙이기도 하고 잘 만들어진 구현체의 재사용을 유도하기 위해 이런 규칙을 가지고 있습니다.

**재사용성**을 늘리고 **협력**이 가능한 높은 완결성의 **도구 클래스**이며 비즈니스 로직의 오염을 막는 것이 핵심입니다.

---
## **구현 계층을 통한 서비스 코드 개선**

### 개선 전
```java
public void editComment(String content, Long commentId, Long memberId) {
    if(!StringUtils.hasText(content)) {  // 댓글의 내용이 공백인지 검증한다.
        throw new RuntimeException(COMMENT_CONTENT_NOT_BLANK); 
    }
    if(content.length() > 100) {  // 댓글의 길이가 100자 초과인지 검증한다.
        throw new RuntimeException(COMMENT_CONTENT_TOO_LONG);
    }

    Comment comment = commentQueryRepository.find(commentId)  // 댓글을 찾아온다.
            .orElseThrow(() -> new RuntimeException(COMMENT_NOT_FOUND));

    if (!comment.isAuthor(memberId.getId())) {  // 댓글 수정을 요청한 회원이 댓글 작성자인지 확인한다.
        throw new RuntimeException(COMMENT_PERMISSION_DENIED);
    }
    if (commentRepository.existsReplies(commentId)) {  // 댓글에 답변이 달렸는지 확인한다.
        throw new RuntimeException(COMMENT_HAS_NOT_REPLY);
    }
	
    comment.changeContent(content);
}
```

### 개선 후
```java
public class QnaService {
    private final CommentEditor commentEditor;
    private final CommentReader commentReader;
    private final CommentValidator commentValidator;
    
    public void editComment(String content, Long commentId, Long memberId) {
        Comment comment = commentReader.read(commentId); // 댓글을 찾아온다.
        commentValidator.validateCommentAuthor(comment, memberId); // 댓글 수정을 요청한 회원이 댓글 작성자인지 확인한다.
        commentValidator.validateNoReplies(
            commentReader.existsReplies(commentId)
        );  // 댓글에 답변이 달렸는지 확인한다.

        commentEditor.edit(content, comment);
    }
}
```

개선 한 코드의 비즈니스 로직은 다음과 같습니다.
1. 댓글을 찾아온다.
2. 댓글 수정을 요청한 회원이 댓글 작성자인지 확인한다.
3. 댓글에 답변이 달렸는지 확인한다.

요청 데이터의 검증은 어디로 갔을까요? 코드를 왜 이렇게 구성 했는 지 알아보겠습니다.

### 코드 분석

아래 클래스들은 구현 계층에 존재하며 클래스들의 역할은 다음과 같습니다.
- `CommentEditor`: create, update, delete의 역할
- `CommentReader`: read의 역할
- `CommentValidator`: 요청 데이터, 객체의 행동에 대한 검증의 역할

[개선 전](#개선-전) 서비스 계층의 코드에서 공백, 길이 검증은 `CommentEditor`에게 위임했습니다.

다음 두 로직을 볼까요?
- 댓글 수정을 요청한 회원이 댓글 작성자인지 확인한다.
- 댓글에 답변이 달렸는지 확인한다.

여러분이 보기엔 두가지의 검증은 비즈니스 로직이라고 생각하시나요? 저는 댓글을 수정할 때 댓글 작성자인지 확인하고 댓글에 답변이 달렸는지 확인하는 것은 **핵심 비즈니스 로직**이라고 생각했습니다. 

그럼 댓글이 공백인지, 길이는 100자인지 확인하는 코드는 비즈니스 로직이라고 생각하시나요? 저는 댓글 수정에 있어서 **공백과 길이 검증**은 **비즈니스 로직**이라고 생각하지만, **핵심 비즈니스 로직이라고 생각하진 않았습니다**.

따라서 **핵심 비즈니스 로직은 서비스 계층에 존재**하도록 하고 핵심이 아닌 **단순 비즈니스 로직은 구현 계층에 존재**하도록 구성했습니다.

```java
// 개선전
public class QnaService {
    public void editComment(String content, Long commentId, Long memberId) {
        if(!StringUtils.hasText(content)) {  // 댓글의 내용이 공백인지 검증한다.
            throw new RuntimeException(COMMENT_CONTENT_NOT_BLANK); 
        }
        if(content.length() > 100) {  // 댓글의 길이가 100자 초과인지 검증한다.
            throw new RuntimeException(COMMENT_CONTENT_TOO_LONG);
        }
        ...   
	}
}

// 개선후
public class CommentEditor {
    public void edit(String content, Comment comment) {
        commentValidator.validateContentNotBlank(content); // 댓글의 내용이 공백인지 검증한다.
        commentValidator.validateContentLength(content);  // 댓글의 길이가 100자 초과인지 검증한다.
        comment.changeContent(content);
    }
}
```

### 전체 코드
```java
public class CommentReader {
    private final CommentQueryRepository commentQueryRepository;

    public Comment read(Long commentId) {
        return commentQueryRepository.findWithMember(commentId)
            .orElseThrow(() -> new RuntimeException(COMMENT_NOT_FOUND));
    }

    public boolean existsReplies(Long commentId) {
        return commentQueryRepository.hasReplies(commentId);
    }
}


public class CommentEditor {
    private final CommentRepository commentRepository;
    private final CommentValidator commentValidator;

    public void edit(String content, Comment comment) {
        commentValidator.validateContentNotBlank(content);
        commentValidator.validateContentLength(content);
        
        comment.changeContent(content);
    }
}


public class CommentValidator {
    public void validateContentNotBlank(String content) {
        if (!StringUtils.hasText(content)) {
            throw new RuntimeException(COMMENT_CONTENT_NOT_BLANK);
        }
    }
    
    public void validateContentLength(String content) {
        if(content.length() > 100) {
            throw new RuntimeException(COMMENT_CONTENT_TOO_LONG);
        }
    }

    public void validateCommentAuthor(Comment comment, Long memberId) {
        if (!comment.isAuthor(memberId)) {
            throw new RuntimeException(COMMENT_PERMISSION_DENIED);
        }
    }

    public void validateNoReplies(boolean hasReplies) {
        if (hasReplies) {
            throw new RuntimeException(COMMENT_HAS_REPLY);
        }
    }
}
```

## **구현 계층의 장단점**

**장점**
- 서비스 계층의 많은 책임을 구현 계층에게 위임하여 서비스 계층에서 비즈니스의 흐름을 한눈에 파악할 수 있다.
- 협력 도구 클래스를 통해 비즈니스 로직을 구현하고 재사용성을 높인다.
- 중요한 비즈니스 규칙과 덜 중요한 비즈니스 규칙을 구별할 수 있다.
- 구현 계층이 서비스 계층을 떠받치고 있기 때문에 협력 도구 클래스는 요청 DTO와 응답 DTO에 오염되지 않는다.

**단점**
- 간단한 비즈니스 로직이라면 서비스 계층과 구현 계층은 아무런 역할없이 거쳐가는 계층이 될 수 있다.
- 관리포인트가 증가하여 생산성이 하락할 수 있다.

---
## **구현 계층 설계 시 고려사항**

### 적절한 분리 기준
- 핵심 비즈니스 로직은 서비스 계층에 유지하고 부가적인 비즈니스 로직은 구현 계층으로 위임할 수 있다.
- 재사용이 가능한 기능이라면 구현 계층으로 내린다.

### 재사용성을 높이기 위한 방법
- 요청 DTO와 응답 DTO로부터 격리한다.
- Query(읽기)와 Command(쓰기)를 분리한다.

---
## **단일 모듈에서 멀티 모듈 전환기 목차**

- <a target='_blank' href='/posts/single-to-multi-module-(1)'>단일 모듈에서 멀티 모듈 전환기 - (1) 멀티 모듈이란?</a>
- <a target='_blank' href='/posts/single-to-multi-module-(2)'><span style='color: #ef5369'>단일 모듈에서 멀티 모듈 전환기 - (2) 구현 계층</span></a>
- <a target='_blank' href='/posts/single-to-multi-module-(3)'>단일 모듈에서 멀티 모듈 전환기 - (3) 도메인 모델</a>
- <a target='_blank' href='/posts/single-to-multi-module-(4)'>단일 모듈에서 멀티 모듈 전환기 - (4) JPA 엔티티 격리 및 DB 추상화</a>
- <a target='_blank' href='/posts/single-to-multi-module-(5)'>단일 모듈에서 멀티 모듈 전환기 - (5) Bounded Context 설계</a>
- <a target='_blank' href='/posts/single-to-multi-module-(6)'>단일 모듈에서 멀티 모듈 전환기 - (6) 멀티 모듈 설계</a>
- <a target='_blank' href='/posts/single-to-multi-module-(7)'>단일 모듈에서 멀티 모듈 전환기 - (7) 멀티 모듈 전환 with Gradle</a>
- <a target='_blank' href='/posts/single-to-multi-module-(8)'>단일 모듈에서 멀티 모듈 전환기 - (8) 회고 및 마무리</a>

---
## **참고 문서**
- [https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63](https://geminikims.medium.com/%EC%A7%80%EC%86%8D-%EC%84%B1%EC%9E%A5-%EA%B0%80%EB%8A%A5%ED%95%9C-%EC%86%8C%ED%94%84%ED%8A%B8%EC%9B%A8%EC%96%B4%EB%A5%BC-%EB%A7%8C%EB%93%A4%EC%96%B4%EA%B0%80%EB%8A%94-%EB%B0%A9%EB%B2%95-97844c5dab63){:target="_blank"}
- [https://engineerinsight.tistory.com/63](https://engineerinsight.tistory.com/63){:target="_blank"}
- [https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/ch01.html](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/ch01.html){:target="_blank"}
