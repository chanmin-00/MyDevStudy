# 문제 상황

현재 시스템에서 사용하는 `PostDTO` 는 게시글 작성, 수정, 조회, 요약 등 다양한 상황에서 공통으로 사용되고 있다.

이 DTO는 본래 엔터티로 변환되기 전 계층 간 데이터 전달을 위한 객체이다. 그런데 너무 많은 역할을 하도록 설계된 결과, 다음과 같은 문제들이 발생한다.

- 생성 시 다수의 필드를 `null`로 채워야 한다.
- 목적에 따라 의미 없는 필드들이 함께 존재한다.
- DTO의 역할, 책임, 사용처가 불분명해진다.
- 유지보수, 테스트, 문서화, API 응답 처리 등 모든 영역에서 복잡도가 증가한다.

현재는 작성, 수정, 응답, 요약까지 모든 책임이 혼합된 상태이다.

### 1. 게시글 작성 요청 시

```java
public static PostDTO toPostDTO(final PostRequest postRequest) {
    return new PostDTO(
        null,
        postRequest.title(),
        null,
        postRequest.content(),
        postRequest.type(),
        null, null, null,
        null, null, null,
        null, null, postRequest.usedDate(),
        null
    );
}
```

- 작성 시 필요한 필드: `title`, `content`, `type`, `usedDate`
- 그 외 필드는 모두 `null`로 입력

### 2. 게시글 상세 조회 시

```java
public static PostDTO toPostDTO(Post post, CommentListDTO comments, ImageListDTO images) {
    return new PostDTO(
        post.getId(),
        post.getTitle(),
        CommunityUserDTO.toCommunityUserDTO(post.getAuthor()),
        post.getContent(),
        post.getType(),
        post.getLikeCount(),
        post.getCommentCount(),
        post.getScrapCount(),
        images,
        post.getIsScrapped(),
        post.getIsLiked(),
        post.getIsAuthor(),
        post.getCreatedDate(),
        post.getModifiedDate(),
        post.getUsedDate(),
        comments
    );
}
```

- 조회 시 필요한 필드가 대부분 사용되지만, 작성용과 구조가 완전히 다르다.

DTO 하나로 모든 작업을 처리하고 있다. 작성, 수정, 응답, 요약을 모두 PostDTO 하나로 해결하려고 하고 있다.

# 문제점

내 코드는 다음과 같은 문제점들을 지니고 있다.

## 객체 불변성 약화

PostDTO는 record로 선언되어 있다. record는 불변(immutable) 데이터 객체를 간결하게 표현하기 위해 설계된 문법이다. 내부적으로는 다음과 같은 특징을 가진다.

- 모든 필드는 자동으로 private final로 선언된다.
- getter는 필드명() 형태로 자동 생성되며 setter는 존재하지 않는다.
- equals(), hashCode(), toString()도 자동 구현된다.
- 따라서 record는 완전하고 유효한 값 객체를 표현하는 데 최적화된 구조이다.

하지만 현재 PostDTO는 다양한 컨텍스트를 하나의 record로 처리하려다 보니 대부분의 생성자 인자에 null을 넣어 생성하는 경우가 많다. 이는 record의 불변성과 생성자 완전성(complete initialization)을 무력화시켜 의미 없는 `부분 유효 객체(partially valid object)`가 되어버린다. 결과적으로 record가 제공하려던 타입 안전성, 불변성, 단순함이라는 모든 장점이 사라지고, 오히려 코드의 안정성을 해치는 요소가 된다.

## **null-safe 코드 강제**

모든 필드를 무조건 초기화해야 하기 때문에, 실제로 사용하지 않는 필드조차 null로 채워 넣게 된다. 이는 서비스나 컨트롤러에서 매번 `.field() != null` 조건문을 사용해야 하는 코드 부담으로 이어질 수 있다.

## **테스트 코드 작성 비효율**

테스트 코드에서도 title, content만 테스트하고 싶은 상황에서도 PostDTO 생성자에 불필요한 null들을 일일이 입력해야 합니다. 이는 테스트 생산성을 저하시킵니다.

# 리팩토링하기

### DTO를 목적별로 분리하자

```java
public record CreatePostCommand(
        long authorId, String title, String content, PostType type, List<Long> imageIds) {

    public static CreatePostCommand of(
            final long authorId,
            final String title,
            final String content,
            final PostType type,
            final List<Long> imageIds) {
        return new CreatePostCommand(authorId, title, content, type, imageIds);
    }
}
```

```java
public record UpdatePostCommand(
        long postId,
        long authorId,
        String newTitle,
        String newContent,
        PostType newType,
        List<Long> newImageIds) {

    public static UpdatePostCommand of(
            final long postId,
            final long authorId,
            final String newTitle,
            final String newContent,
            final PostType newType,
            final List<Long> newImageIds) {
        return new UpdatePostCommand(postId, authorId, newTitle, newContent, newType, newImageIds);
    }
}
```

- 각 상황에 맞는 DTO를 정의하면 null 없이 필수 필드만 정의할 수 있고 의미가 명확해진다.

# 결론

`DTO`는 계층 간 데이터를 전달하기 위한 객체이다. 그 목적에 따라 불필요한 필드를 제거하고, 컨텍스트에 따라 분리된 구조로 사용하는 것이 바람직할 것 같다.

- 단일 DTO로 모든 작업을 처리하려는 시도는 결국 유지보수성과 설계 일관성을 무너뜨린다.
