리플 프로젝트에서는 댓글이 달리거나 좋아요가 눌리면 실시간 알림이 전달된다. 이를 위해 `SSE(Server-Sent Events)`와 `Redis Pub/Sub`을 활용해 스케일아웃 환경에서도 잘 작동하는 실시간 알림 시스템을 구축했다.

그런데 이 상황에서 예방하지 못한 문제가 하나 있었다. 댓글이나 좋아요에 대한 정보가 DB에 저장되지 않았음에도 알림이 사용자에게 전송되는 경우가 발생할 수 있다는 것이다. 

이번 글에서는 그 문제의 원인을 파악해보고, `@TransactionalEventListner` 를 통해 어떻게 해결했는지를 정리해보았다.

# 기존 방식

## 이벤트 기반 처리

기존 코드에서는 예를 들어 댓글이 작성되면 알림을 보내야 하는 것처럼, 어떤 행위에 반응해 다른 로직을 실행할 때 이벤트 기반 아키텍처(Event-Driven Architecture) 를 활용했다. 먼저 이벤트 기반 처리가 무엇인지 간단히 알아보고 왜 이걸 사용하게 되었는지 설명해보자.

### cf) Spring 공식 문서 설명

Spring에서는 **ApplicationEvent**와 **ApplicationListener**를 통해 이벤트 시스템을 제공한다. 공식 문서에서는 다음과 같이 설명하고 있다

> Event handling in the ApplicationContext is provided through the ApplicationEvent class and the ApplicationListener interface. If a bean that implements the ApplicationListener interface is deployed into the context, every time an ApplicationEvent gets published to the ApplicationContext, that bean is notified. Essentially, this is the standard Observer design pattern.
> 

> Spring에서의 이벤트 처리(Event handling)는 ApplicationEvent 클래스와 ApplicationListener 인터페이스를 통해 제공됩니다. 만약 ApplicationListener 인터페이스를 구현한 빈(Bean)이 애플리케이션 컨텍스트에 등록되어 있다면, ApplicationEvent가 ApplicationContext에 발행될 때마다 해당 빈이 알림을 받습니다. 본질적으로 이는 **표준 옵저버(Observer) 디자인 패턴** 을 따른 것입니다.
> 

즉, Spring 이벤트 시스템은 **표준 옵저버(Observer) 패턴**을 기반으로 하고 있으며, **이벤트가 발행되면(ApplicationEvent)** 이를 구독하고 있는 **리스너(ApplicationListener)** 가 반응하는 구조이다.

### 이벤트 기반 처리를 사용하지 않은 경우

```java
@Transactional
public void addCommentToPost(...) {
    Comment comment = commentRepository.save(...);
    ...
    notificationService.createNotification(post, comment); // 직접 호출
}
```

이런 방식은 간단해 보이지만, 다음과 같은 문제가 있을 수 있다.

- **단일 책임 원칙(SRP)** 위반 : 댓글 생성과 알림 전송이라는 두 개의 책임이 한 메서드에 공존한다.
- **트랜잭션 전파 문제** : 댓글은 정상적으로 저장되었음에도 알림 저장 중 예외가 발생하면 전체 트랜잭션이 롤백되어 댓글 저장까지 실패할 수 있다. 즉, 핵심 로직(댓글 작성)과 부가 로직(알림 전송) 을 분리하지 않으면 의도하지 않은 부작용이 생길 수 있다.

### 이벤트 기반 처리를 사용한 경우

```java
@Transactional
public void addCommentToPost(...) {
    ...
    eventPublisher.publishEvent(new CommentCreatedEvent(post, comment)); // 이벤트 발행
}
```

```java
@EventListener
@Transactional
public void createNotification(CommentCreatedEvent event) {
    notificationRepository.save(...);
    redisPublisher.publish(...); // Redis로 알림 발행
}
```

이 방식은 **알림 전송 책임을 분리**하고 **결합도**를 낮출 수 있는 장점이 있다. 또한 나중에 이메일, 푸시 등 **다양한 알림 채널을 확장하기도 훨씬 쉬운 구조**가 된다.

# 문제 발생

기존 코드에서 이벤트 기반 구조를 사용했음에도 불구하고 여전히 문제가 발생할 수 있는 상황이 존재했다. 예를 들어 댓글이나 좋아요가 **실제로 DB에 저장되지 않았음에도** **알림이 사용자에게 전송되는** 상황이 발생할 수 있는 것이다.

Spring 공식 문서를 보면 이 동작의 근거를 다음과 같이 확인할 수 있다

> But note that, by default, **event listeners receive events synchronously**. This means that the publishEvent() method blocks until all listeners have finished processing the event. One advantage of this synchronous and single-threaded approach is that, when a listener receives an event, it operates inside the transaction context of the publisher if a transaction context is available.
> 

> 하지만 기본적으로 이벤트 리스너는 **동기적으로 이벤트를 수신**한다는 점에 유의해야 합니다. 즉, `publishEvent()` 메서드는 **모든 리스너가 이벤트 처리를 마칠 때까지 블로킹(대기)** 합니다. 이러한 **동기적이고 단일 스레드 기반의 처리 방식**의 장점 중 하나는, 리스너가 이벤트를 수신할 때 **이벤트를 발행한 쪽의 트랜잭션 컨텍스트 내에서 동작한다는 것**입니다.
> 
> 
> (단, 발행자에게 트랜잭션 컨텍스트가 존재하는 경우에 한합니다.)
> 

즉, `@EventListener`는 기본적으로 동기적으로, 단일 스레드에서, 기존 트랜잭션 안에서 실행된다. 이벤트가 발행되는 순간 리스너가 같은 트랜잭션 안에서 바로 실행된다는 의미다.

테스트 코드를 통해서 어떻게 동작하는지 확인해보자.

### 테스트 코드 실행

```java
log.info("알림 전송 시작");
eventPublisher.publishEvent(new CommentCreatedEvent(post, comment));
log.info("알림 전송 완료");
```

```java
@Test
@DisplayName("댓글이 성공적으로 저장되지 않았음에도 알림이 발행되어버린다")
void commentCreatedEventTest() {
    // given
    String content = "댓글 내용";
    long postId = post1.getId();
    long userId = user1.getId();

    // when
    try {
        commentCommandService.addCommentToPost(userId, postId, content);
    } catch (Exception e) {
        log.error("예외 발생: {}", e.getMessage());
    }
}
```

이 테스트를 실행한 로그를 보면 다음과 같은 흐름을 확인할 수 있다

```bash
[    Test worker] o.s.orm.jpa.JpaTransactionManager        : Participating in existing transaction
[    Test worker] c.r.B.n.a.r.RedisNotificationPublisher   : Publishing notification to Redis topic: notification
[    Test worker] c.r.B.p.a.i.c.CommentCommandService      : 알림 전송 완료

....
[cTaskExecutor-3] o.s.orm.jpa.JpaTransactionManager        : Closing JPA EntityManager [SessionImpl(1201508091<open>)] after transaction
```

리스너는 요청 메서드와 **동일한 트랜잭션 안에서 실행**되었으며, 알림 발행도 **트랜잭션 커밋 전 이미 실행**되었음을 확인할 수 있다.

이렇나 경우에서 만약에서 롤백이 되어야 하는 상황이 생긴다면 어떻게 될까?

### 댓글 저장 시 예외가 발생하는 경우

```java
@Override
@Transactional
public void addCommentToPost(final long userId, final long postId, final String content) {
    Post post = findPostByIdForUpdate(postId);

    Comment comment = Comment.withoutId(content, userId, postId, null);
    commentRepository.save(comment);

    postRepository.save(post.increaseCommentCount());

    log.info("알림 전송 시작");
    eventPublisher.publishEvent(new CommentCreatedEvent(post, comment));
    log.info("알림 전송 완료");

    // 예외 발생
    throw new RuntimeException("예외 발생 테스트");
}
```

위 코드처럼 댓글 저장 후 알림까지 발행한 뒤 예외가 발생했다고 가정해보자. 테스트를 실행하면 트랜잭션이 롤백되고, 아래와 같은 로그를 확인할 수 있다

```bash
Test worker] c.r.B.n.a.r.RedisNotificationPublisher   : Publishing notification to Redis topic: notification
[    Test worker] c.r.B.p.a.i.c.CommentCommandService      : 알림 전송 완료
[    Test worker] o.s.orm.jpa.JpaTransactionManager        : Initiating transaction rollback
[    Test worker] o.s.orm.jpa.JpaTransactionManager        : Rolling back JPA transaction on EntityManager [SessionImpl(1903547260<open>)]
[    Test worker] o.s.orm.jpa.JpaTransactionManager        : Closing JPA 
```

`@EventListener`는 요청 메서드의 트랜잭션에 참여했기 때문에, Notification 엔티티 저장은 롤백된다. 하지만 **Redis 알림 발행은 외부 시스템(I/O)으로의 전송**이기 때문에 **롤백되지 않는다.**

결과적으로 댓글은 저장되지 않았지만 **사용자에게 알림이 전송되는 문제**가 발생해버리게 된다.

즉, `@EventListener`는 트랜잭션 안에서 실행되긴 하지만 그 안에서 외부 시스템과의 통신까지 처리하면 롤백되지 않는 문제가 생길 수 있는 것이다. 이 문제를 해결하기 위해서는 이벤트가 **트랜잭션 커밋 이후에 실행되도록 보장**해야 한다. 이를 보장하기 위해  `@TransactionalEventListener` 를 활용할 수 있다.

# 문제 해결

## @TransactionEventListener 적용

Spring 4.2부터는 이벤트 리스너를 트랜잭션의 특정 단계에 바인딩할 수 있도록 지원하고 있다. 공식 문서에서도 다음과 같이 설명하고 있다

> As of Spring 4.2, the listener of an event can be bound to a phase of the transaction. The typical example is to handle the event when the transaction has completed successfully. Doing so lets events be used with more flexibility when the outcome of the current transaction actually matters to the listener.
> 
> 
> You can register a regular event listener by using the `@EventListener` annotation. If you need to bind it to the transaction, use `@TransactionalEventListener`. When you do so, the listener is bound to the commit phase of the transaction by default.
> 
> The next example shows this concept. Assume that a component publishes an order-created event and that we want to define a listener that should only handle that event once the transaction in which it has been published has committed successfully.
> 

> Spring 4.2부터는 **이벤트 리스너를 트랜잭션의 특정 단계에 바인딩할 수 있게** 되었습니다. 가장 일반적인 예는 **트랜잭션이 성공적으로 완료된 이후에 이벤트를 처리하는 것**입니다.
> 
> 
> 이렇게 하면, **현재 트랜잭션의 결과가 리스너에게 실제로 중요할 때**, 이벤트를 보다 유연하게 사용할 수 있습니다. 일반적인 이벤트 리스너는 `@EventListener` 애너테이션을 사용해 등록할 수 있습니다. 하지만 트랜잭션에 바인딩하려면 `@TransactionalEventListener`를 사용해야 합니다.
> 
> 이렇게 설정하면 해당 리스너는 기본적으로 **트랜잭션 커밋 시점(AFTER_COMMIT)** 에 바인딩됩니다.
> 

즉, 이벤트 리스너는 `@TransactionalEventListener`를 통해 **트랜잭션 커밋 이후(AFTER_COMMIT)** 에 실행되도록 바인딩할 수 있다. 이는 트랜잭션이 실제로 성공했을 때만 알림을 보내고 싶다는 상황에 적합하다.

### 적용 예시

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void createCommentNotification(final CommentCreatedEvent event) {
    Post post = event.post();
    Comment comment = event.comment();

    Notification notification = Notification.withoutId(
        post.getAuthorId(),
        post.getId(),
        post.getTitle(),
        comment.getContent(),
        NotificationType.COMMENT
    );

    saveAndPublish(notification);
}
```

위와 같이 @TransactionalEventListener를 사용하고, phase를 `AFTER_COMMIT`으로 설정함으로써 **댓글이 실제로 저장된 이후**에만 알림을 발행하도록 보장할 수 있다. 참고로 phase 속성을 생략하면 기본값으로 AFTER_COMMIT이 적용된다.

### 적용 확인

```bash
actionalApplicationListenerMethodAdapter : Registered transaction synchronization for org.springframework.context.PayloadApplicationEvent[source=org.springframework.web.context.support
```

일단 정상적으로 리스너가 등록되었음을 확인할 수 있다.

예외가 발생하지 않은 경우 다음과 같이 트랜잭션 종료 후 알림 수신 로그가 출력된다.

```java
... Closing JPA EntityManager ...
... Received notification from Redis: NotificationSseDTO[...]
```

예외가 발생한 경우에는, **트랜잭션이 롤백되면서 알림 수신 로그는 발생하지 않는다**

```java
... Closing JPA EntityManager ...
... 예외 발생: 예외 발생 테스트
```

## PROPAGATION_REQUIRES_NEW 적용

하지만 `@TransactionalEventListener`를 적용한 뒤, 다음과 같은 테스트를 수행하면 실패하게 된다

```java
@Test
@DisplayName("댓글이 성공적으로 저장되고, 알림이 성공적으로 생성되어야 한다")
void commentCreatedEventTest() {
    // given
    String content = "댓글 내용";
    long postId = post1.getId();
    long userId = user1.getId();

    // when
    try {
        commentCommandService.addCommentToPost(userId, postId, content);
    } catch (Exception e) {
        log.error("예외 발생: {}", e.getMessage());
    }

    // then
    Notification notification =
        notificationRepository.findByUserIdOrderByCreatedDateDesc(userId).stream()
            .filter(n -> n.getPostId() == postId)
            .findFirst()
            .orElse(null);

    assertThat(notification).isNotNull();
}
```

실패 로그:

```bash
java.lang.AssertionError: 
Expecting actual not to be null
```

즉 알림이 정상적으로 저장되지 않았음을 뜻한다.

### Notification이 저장되지 않는 이유

이 원인은 다음 글을 통해 확인할 수 있다. 

> **WARNING:** if the `TransactionPhase` is set to `AFTER_COMMIT` (the default), `AFTER_ROLLBACK`, or `AFTER_COMPLETION`, the transaction will have been committed or rolled back already, but the transactional resources might still be active and accessible. As a consequence, any data access code triggered at this point will still "participate" in the original transaction, but changes will not be committed to the transactional resource.
> 

> `TransactionPhase`가 기본값인 `AFTER_COMMIT`, 혹은 `AFTER_ROLLBACK`, `AFTER_COMPLETION`으로 설정되어 있을 경우, **트랜잭션은 이미 커밋되었거나 롤백된 상태**이지만, **트랜잭션 자원(예: DB 커넥션 등)은 여전히 활성화되어 접근이 가능할 수 있습니다.**
> 
> 
> 그 결과, 이 시점에 실행되는 **데이터 접근 코드**는 **여전히 원래 트랜잭션에 참여하는 것처럼 보일 수 있지만**, **그 이후의 변경 사항은 실제 트랜잭션 자원에 커밋되지 않습니다.**
> 

즉, 트랜잭션이 커밋되거나 롤백된 상태이지만 트랜잭션의 수명을 늘려서 참여만 할 수 있게 된 상태가 된것이다. 그러니까 나는 참여만 할 수 있는 상태에서 Notifiaciion을 DB에 저장하려고 했기 때문에 저장이 되지 않았던 것이다.

해결 방법은 `@Transactional`에 `propagation = Propagation.REQUIRES_NEW`를 명시하여 **완전히 독립된 트랜잭션에서 알림 저장을 수행하는 것**이다.

즉, 트랜잭션 전파 설정을 새롭게 정의하여 **기존 트랜잭션에 참여하지 않고 별도의 트랜잭션으로 알림 로직을 수행하도록** 해야 한다.

### cf) Propagation.REQUIRES_NEW

<img width="517" alt="image" src="https://github.com/user-attachments/assets/56bcf200-14cc-427f-987a-7cd7241f307c" />


> `PROPAGATION_REQUIRES_NEW`, in contrast to `PROPAGATION_REQUIRED`, always uses an independent physical transaction for each affected transaction scope, never participating in an existing transaction for an outer scope. In such an arrangement, the underlying resource transactions are different and, hence, can commit or roll back independently, with an outer transaction not affected by an inner transaction’s rollback status and with an inner transaction’s locks released immediately after its completion. Such an independent inner transaction can also declare its own isolation level, timeout, and read-only settings and not inherit an outer transaction’s characteristics.
> 

> `PROPAGATION_REQUIRES_NEW`는 `PROPAGATION_REQUIRED`와 달리, 항상 각 메서드 호출 시 **독립된 물리적 트랜잭션**을 생성합니다. 즉, 이미 외부 트랜잭션이 진행 중이더라도 거기에 참여하지 않고, **자체적으로 새로운 트랜잭션을 시작**합니다. 이 구조에서는 내부 트랜잭션과 외부 트랜잭션이 서로 다른 자원을 사용하므로, 내부 트랜잭션에서 커밋이나 롤백이 발생해도 외부 트랜잭션에는 영향을 주지 않습니다. 또한 내부 트랜잭션이 종료되면 그에 사용된 데이터베이스 락 등도 즉시 해제되며, 내부 트랜잭션은 외부 트랜잭션의 설정(격리 수준, 타임아웃, 읽기 전용 등)을 상속받지 않고 **자신만의 설정을 가질 수 있습니다.** 이 방식은 트랜잭션을 완전히 분리하여 처리하고 싶을 때 유용합니다.
> 

### 적용

Notification이 저장되지 않던 문제를 해결하기 위해, 새로운 트랜잭션을 생성하여 그 안에서 알림 저장 및 발행을 수행하도록 코드를 수정했다.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void createCommentNotification(final CommentCreatedEvent event) {
    Post post = event.post();
    Comment comment = event.comment();

    Notification notification =
        Notification.withoutId(
            post.getAuthorId(),
            post.getId(),
            post.getTitle(),
            comment.getContent(),
            NotificationType.COMMENT
        );

    saveAndPublish(notification);
}
```

> **주의:** 테스트 코드에는 @Transactional 을 붙이지 않아야 한다. 테스트 메서드 전체에 트랜잭션이 걸리게 되면 내부에서 발행한 이벤트 역시 테스트 메서드가 커밋되기 전까지는 AFTER_COMMIT 조건을 만족하지 못하므로 리스너가 동작하지 않게 된다.
> 

아래 로그를 보면, 댓글 작성 트랜잭션이 커밋된 이후 기존 트랜잭션이 **일시 중단(Suspend)** 되고, `@Transactional(propagation = REQUIRES_NEW)` 설정에 따라 **새로운 트랜잭션이 별도의 EntityManager로 생성 및 실행**되는 과정을 확인할 수 있다:

```bash

[Test worker] o.s.orm.jpa.JpaTransactionManager : Committing JPA transaction on EntityManager [SessionImpl(530891662<open>)]
[Test worker] o.s.orm.jpa.JpaTransactionManager : Found thread-bound EntityManager [SessionImpl(530891662<open>)] for JPA transaction
[Test worker] o.s.orm.jpa.JpaTransactionManager : Suspending current transaction, creating new transaction with name [com.ripple.BE.notification.application.NotificationPublisher.createCommentNotification]
[Test worker] o.s.orm.jpa.JpaTransactionManager : Opened new EntityManager [SessionImpl(721234952<open>)] for JPA transaction
[Test worker] o.s.orm.jpa.JpaTransactionManager : Exposing JPA transaction as JDBC [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@44953958]
[Test worker] o.s.orm.jpa.JpaTransactionManager : Found thread-bound EntityManager [SessionImpl(721234952<open>)] for JPA transaction
```

- 즉, 원래 트랜잭션은 잠시 중단된 상태로 남겨지고, 알림 저장 및 발행을 위한 완전히 새로운 트랜잭션이 별도로 시작된다.

이후 아래 로그를 통해, 알림이 정상적으로 저장된 뒤 Redis에 발행되고, 다른 서버 또는 리스너가 해당 알림을 **정상적으로 수신**했음을 확인할 수 있다:

```bash
[ssageListener-1] RedisNotificationSubscriber : Received notification from Redis: NotificationSseDTO[receiverId=1, title=제목1, content=댓글 내용, type=COMMENT]
[Test worker] EventListenerTest : 생성된 알림: com.ripple.BE.notification.domain.Notification@36c9698e

```

- 이렇게 `REQUIRES_NEW` 전파 속성과 `AFTER_COMMIT` 이벤트 시점을 함께 활용하면, 댓글이 DB에 성공적으로 저장된 이후에만 알림을 보장할 수 있다.

# 결론

이번 글에서는 댓글 정보가 반드시 DB에 정상적으로 저장된 이후에만 알림이 발행되도록 만들기 위해 `@TransactionalEventListener`를 활용해 보았다.

그리고 그 과정에서 발생한 **AFTER_COMMIT 시점의 트랜잭션 자원 접근 문제**를 PROPAGATION.REQUIRES_NEW 전파 설정을 통해 해결할 수 있었다.

트랜잭션 기반의 이벤트 설계에서는 단순히 `@EventListener`를 사용하는 것에 그치지 않고, 언제 실행되는지(phase) 와 어떤 트랜잭션 위에서 실행되는지(propagation) 를 반드시 함께 고려해야 한다는 점을 배웠다.

# 참고

[EventListener & TransactinoalEventListener를 통해 문제 해결해보기 (예제편)](https://c-king.tistory.com/609)
