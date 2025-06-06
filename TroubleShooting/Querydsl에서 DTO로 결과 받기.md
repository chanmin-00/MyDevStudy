# 문제 상황

Querydsl을 통해서 사용자가 좋아요를 단 댓글 리스트를 반환하여 조회하는 기능을 개선하고 있다. 

문제는 반환해야 하는 정보에 댓글 고유의 정보(댓글 내용, 댓글 작성자 등)만 포함해야 하는 것이 아니다. 댓글이 달린 게시물의 제목, 게시물의 카테고리(PostType) 등 다른 책임을 가진 도메인의 정보도 함께 포함해야 한다.

<img width="302" alt="image" src="https://github.com/user-attachments/assets/71a3f061-9e25-444f-914c-3111ad152c5f" />


이 문제를 해결하기 위해 기존에는 아래 코드와 같이 구현했다.

```java
public List<LikeCommentResponseDTO> getMyLikeComments(final long userId) {
    List<Comment> commentsLikedByUser = commentLikeRepository.findCommentsLikedByUser(userId);

    return commentsLikedByUser.stream()
        .map(comment -> {
            Post post = postRepository.findById(comment.getPostId())
                .orElseThrow(() -> new PostException(POST_NOT_FOUND));
            return LikeCommentResponseDTO.of(comment, post);
        })
        .collect(Collectors.toList());
}	
```

```java
@Override
public List<CommentJpaEntity> findCommentsLikedByUser(long userId) {
    return queryFactory
        .select(commentJpaEntity)
        .from(commentLikeJpaEntity)
        .join(commentJpaEntity)
        .on(commentLikeJpaEntity.commentId.eq(commentJpaEntity.id)
            .and(commentJpaEntity.isDeleted.isFalse()))
        .where(commentLikeJpaEntity.userId.eq(userId))
        .orderBy(commentLikeJpaEntity.createdDate.desc())
        .fetch();
}
```

일단 Querydsl을 통해서 comment_like(사용자가 좋아요 누른 댓글 정보 중간 테이블)와 comment(댓글)를 join 하여 댓글 정보를 가져오고 있다.

그렇기 때문에 **댓글이 달린 게시글(Post)에 대한 정보는 포함할 수 없다.** 따라서 댓글 정보를 가져온 후, 각 댓글의 postId를 기준으로 다시 게시글을 개별적으로 조회하는 방식이 필요하다.

## 문제 발생

이 구조는 단순해 보이지만 다음과 같은 문제가 있다.

- 댓글을 먼저 조회한 뒤, 댓글마다 postId를 이용해 게시글을 별도로 조회한다.
- 결과적으로 댓글 개수만큼 게시글 조회 쿼리가 추가로 실행된다.

이로 인해 댓글 개수가 많을 경우 DB에 불필요한 부하가 발생하며, 성능 저하를 유발 할 수 있다.

예를 들어 3개의 게시글이 존재하고 각각의 게시글에 댓글이 1개씩 달려 있어 총 3개의 댓글이 있는 상황에서, 사용자가 이 3개의 댓글에 모두 좋아요를 누른 경우를 가정해보자.

이때 실제로 코드를 실행해보면 댓글은 한 번의 쿼리로 모두 가져오지만, 게시글 조회 쿼리는 댓글 수만큼 개별적으로 반복 실행되었다.

```bash
Hibernate: select cje1_0.id,cje1_0.user_id,cje1_0.content,cje1_0.created_date,cje1_0.is_deleted,cje1_0.like_count,cje1_0.modified_date,cje1_0.parent_id,cje1_0.post_id,cje1_0.reply_count from comment_likes clje1_0 join comments cje1_0 on clje1_0.comment_id=cje1_0.id where clje1_0.user_id=?
Hibernate: select pje1_0.id,pje1_0.user_id,pje1_0.comment_count,pje1_0.content,pje1_0.created_date,pje1_0.like_count,pje1_0.modified_date,pje1_0.scrap_count,pje1_0.title,pje1_0.post_type,pje1_0.used_date from posts pje1_0 where pje1_0.id=?
Hibernate: select pje1_0.id,pje1_0.user_id,pje1_0.comment_count,pje1_0.content,pje1_0.created_date,pje1_0.like_count,pje1_0.modified_date,pje1_0.scrap_count,pje1_0.title,pje1_0.post_type,pje1_0.used_date from posts pje1_0 where pje1_0.id=?
Hibernate: select pje1_0.id,pje1_0.user_id,pje1_0.comment_count,pje1_0.content,pje1_0.created_date,pje1_0.like_count,pje1_0.modified_date,pje1_0.scrap_count,pje1_0.title,pje1_0.post_type,pje1_0.used_date from posts pje1_0 where pje1_0.id=?
```

> 위 예시는 댓글이 3개일 때 발생한 **실제 쿼리 로그**이며 각 댓글이 달린 게시글을 별도로 조회하면서 총 4개의 쿼리가 실행되었다.
> 

응답값은 아래와 같이 온다.

```json
{
  "isSuccess": true,
  "code": "REQUEST_OK",
  "message": "request succeeded",
  "results": [
    {
      "id": 5,
      "content": "댓글1",
      "postName": "게시물1",
      "type": "FREE",
      "createdDate": "1시간 전"
    },
    {
      "id": 6,
      "content": "댓글2",
      "postName": "게시물2",
      "type": "FREE",
      "createdDate": "1시간 전"
    },
    {
      "id": 7,
      "content": "댓글3",
      "postName": "개시물3",
      "type": "FREE",
      "createdDate": "7분 전"
    }
  ]
}
```

이처럼 댓글이 100개이고 댓글마다 서로 다른 게시글에 달려 있다면 게시글 조회 쿼리만 100번 추가로 실행된다. 이는 실제 운영 환경에서는 심각한 DB 부하로 이어질 수 있으며 데이터가 많아질수록 성능 저하가 커지게 된다.

# 문제 해결

이 문제를 해결하기 위해 QueryDSL에서 엔터티 정보만을 가져오는 대신 **결과 처리를 커스터마이징하여 필요한 필드만를 추가해서 조회하는** 방식인 **Projection** 기능을 활용했다

## Projection

> Querydsl을 이용해 엔터티 전체 정보를 가져오는 것이 아니라 조회 대상을 지정해 원하는 값만을 조회하는 것을 의미한다.
> 

이 기능을 활용하면 댓글과 게시글을 한 번에 조인하고 필요한 정보를 DTO로 직접 매핑할 수 있다. 즉, **단 하나의 쿼리로 모든 정보를 한 번에 가져올 수 있다.**

공식문서를 참고하면 다음과 같은 설명이 나온다.

[3.2. 결과 처리](http://querydsl.com/static/querydsl/4.0.0/reference/ko-KR/html/ch03s02.html)

> Querydsl은 **결과 처리를 커스터마이징 하기 위해 행 기반 변환을 위한 FactoryExpressions과 집합을 위한 ResultTransformer를 제공**하고 있다.
> 
> 
> com.querydsl.core.types.FactoryExpression 인터페이스는 빈 생성, 생성자 호출 그리고 더 복잡한 객체를 생성하기 위해 사용된다. **com.querydsl.core.types.Projections 클래스를 이용해서 FactoryExpression 구현체 기능에 접근**할 수 있다.
> 

즉, 필요한 데이터를 커스터마이징된 방식으로 직접 DTO에 매핑하기 위한 도구로서 `Projections` 클래스를 제공하고 있는 것이다.

## Projection 방법

Querydsl에서는 DTO 프로젝션을 4가지 방식으로 지원한다

1. `@QueryProjection`
2. `Projections.bean()`
3. `Projections.constructor()`
4. `Projections.fields()`

나는 이 중 `Projections.constructor()`를 사용하여 DTO의 생성자 방식으로 필요한 필드를 조회하고 댓글과 게시글을 **한 번에 Join**하여 매핑했다.

```java
@Override
public List<LikeCommentWithPostDTO> findLikedCommentsByUserIdWithPost(long userId) {
    return queryFactory
        .select(
            Projections.constructor(
                LikeCommentWithPostDTO.class,
                commentJpaEntity.id,
                commentJpaEntity.content,
                commentJpaEntity.createdDate,
                postJpaEntity.id,
                postJpaEntity.title,
                postJpaEntity.type
            )
        )
        .from(commentLikeJpaEntity)
        .join(commentJpaEntity).on(commentLikeJpaEntity.commentId.eq(commentJpaEntity.id))
        .join(postJpaEntity).on(commentJpaEntity.postId.eq(postJpaEntity.id))
        .where(commentLikeJpaEntity.userId.eq(userId), commentJpaEntity.isDeleted.isFalse())
        .orderBy(commentLikeJpaEntity.createdDate.desc())
        .fetch();
}
```

## 실행 결과

위에서 테스트 해본 경우와 동일한 상황에서 해당 기능을 호출해보았다.

```bash
Hibernate: select cje1_0.id,cje1_0.content,cje1_0.created_date,pje1_0.id,pje1_0.title,pje1_0.post_type from comment_likes clje1_0 join comments cje1_0 on clje1_0.comment_id=cje1_0.id join posts pje1_0 on cje1_0.post_id=pje1_0.id where clje1_0.user_id=? and cje1_0.is_deleted=? order by clje1_0.created_date desc
```

쿼리를 분석하면 다음과 같다.

- comment_likes 테이블에서 사용자의 좋아요 정보를 기준으로 comments와 posts를 각각 조인해 댓글 및 해당 게시글 정보를 한 번에 조회한다.
- select 절에서 DTO에 필요한 필드들만 명시적으로 선택해 조회하고 있다.

결과적으로 단 1회의 쿼리로 모든 필요한 정보를 조회하고 있기 때문에 성능 측면에서 매우 효율적임을 확인할 수 있다.

# 마무리

기존에는 도메인 엔터티 중심으로만 QueryDSL을 사용하다 보니 필요한 데이터가 여러 도메인에 걸쳐 있을 경우 조회 로직이 복잡해질 수밖에 없었다. 특히, 하나의 댓글 정보만으로는 부족하고 해당 댓글이 달린 게시글의 정보까지 함께 조회해야 하는 상황에서는 추가 쿼리를 매번 날리는 구조가 되어버려 성능 문제가 발생했다.

하지만 이번 개선에서는 QueryDSL의 Projection 기능을 적극 활용함으로써, 댓글과 게시글 정보를 한 번의 쿼리로 조회하고 바로 DTO로 매핑할 수 있었다. 단순히 쿼리 성능이 좋아졌을 뿐만 아니라 코드도 훨씬 간결하고 명확해졌다는 장점이 있었다.
