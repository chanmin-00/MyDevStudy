# 문제 상황

댓글(comments 테이블)의 삭제를 시도했지만 해당 댓글을 참조하는 comment_likes 테이블의 외래 키 제약조건에 걸려 삭제가 실패했다.

```bash
key constraint fails (`ripple`.`comment_likes`, CONSTRAINT `FK3wa5u7bs1p1o9hmavtgdgk1go` FOREIGN KEY (`comment_id`) REFERENCES `comments` (`id`))
org.springframework.dao.DataIntegrityViolationException: could not execute statement [Cannot delete or update a parent row: a foreign key constraint fails (`ripple`.`comment_likes`, CONSTRAINT `FK3wa5u7bs1p1o9hmavtgdgk1go` FOREIGN KEY (`comment_id`) REFERENCES `comments` (`id`))] [delete from comments where id=?]; SQL [delete from comments where id=?]; constraint [null]
```

# 문제 원인

```java
@Entity
public class CommentJpaEntity extends BaseJpaEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Long id;
    
    ...

}
```

```java
@Entity
public class CommentLikeJpaEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id", nullable = false)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "comment_id")
    private CommentJpaEntity comment;

		...
}
```

## 외래키 제약 조건

데이터베이스 안의 CommetLikeJpaEntity 테이블에는 comment_id가 FK로 저장된다. 이러한 경우에 부모 행(comments)이 삭제되면 참조 무결성이 깨지기 때문에 삭제를 허용하지 않는다.

## Cascade 미설정

JPA에서는 연관된 엔터티를 함께 삭제하거나 갱신하려면 `cascade = CascadeType.REMOVE` 또는 `orphanRemoval = true` 같은 설정이 필요하다.

그러나 현재 코드에서는 Comment → CommentLike에 대해 양방향 연관관계가 설정되어 있지 않으며, Comment 객체가 List<CommentLike>를 포함하지 않고 있다.

따라서 JPA의 `cascade` 나 `orphanRemoval`이 작동할 수 없는 구조이다.

```java
@Entity
public class CommentLikeJpaEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "comment_id")
    private CommentJpaEntity comment;
}
```

- CommentLikeJpaEntity는 CommentJpaEntity를 참조하고 있지만, 반대로 CommentJpaEntity는 CommentLikeJpaEntity를 전혀 알지 못하는 단방향 관계이다.

하지만 내 코드는 양방향 매핑을 사용하지 않을 것이기 때문에 이 방법은 사용하지 않는다.

# 해결

외래 키 제약으로 인해 Comment 엔터티를 직접 삭제할 수 없는 상황에서는, 자식 엔터티인 CommentLike를 먼저 삭제한 후 에 부모인 댓글을 삭제해서 참조 무결성을 지킬 수 있도록 하였다.

```java
commentLikeRepository.deleteAllByCommentId(comment.getId());
commentRepository.delete(comment);
```

- comment_id가 주어진 댓글 ID인 모든 CommentLike 데이터를 먼저 삭제
- 외래 키 제약 조건을 우회하기 위해 자식 테이블에서 참조 제거
