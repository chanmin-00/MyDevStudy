# 문제 상황

게시글(Post)을 저장한 직후 해당 게시글을 참조하는 이미지(Image)의 post 필드를 설정하고 저장하려는 시점에 다음과 같은 예외가 발생했다.

## 예외 메시지

```
org.hibernate.TransientPropertyValueException:
object references an unsaved transient instance
→ com.ripple.BE.image.domain.Image.post → com.ripple.BE.post.persistence.jpa.entity.PostJpaEntity
```

- Image 객체의 post 필드에 주입된 PostJpaEntity가 아직 저장되지 않은 비영속(transient) 상태로 간주되어 JPA가 예외를 발생시킨 것이다.

## cf) 영속성 컨텍스트

영속성 컨텍스트(Persistence Context)는 JPA가 엔티티 객체를 관리(추적)하는 메모리 상의 저장소이다. JPA가 관리 중인 엔티티를 담고 있는 1차 캐시 역할을 한다.

### 특징

- `동일성 보장` : 같은 트랜잭션 안에서 같은 ID를 가진 엔티티는 동일 객체로 유지된다.
- `변경 감지` : 엔티티의 값이 변경되면 flush 시 자동으로 SQL update를 수행한다.
- `flush` : 영속성 컨텍스트에 저장된 변경 내용을 실제 DB에 반영하는 시점이다.
- `영속 상태` : save 등으로 영속성 컨텍스트에 등록된 객체 상태
- `비영속 상태` : new 또는 변환으로 생성되었지만 아직 JPA가 관리하지 않는 상태

# 문제 원인

## 문제 발생 코드

```java
@Override
public void createPost(final CreatePostCommand createPostCommand) {
    User user = userService.findUserById(createPostCommand.authorId());
    Post post =
            Post.of(
                createPostCommand.title(),
                createPostCommand.content(),
                user,
                createPostCommand.type(),
                null);

    postRepository.save(post);

    if (createPostCommand.imageIds() != null) {
        for (long imageId : createPostCommand.imageIds()) {
            Image image =
                    imageRepository
                            .findById(imageId)
                            .orElseThrow(() -> new ImageException(IMAGE_NOT_FOUND));
            image.setPost(PostJpaEntity.from(post)); // 추후 Post로 변경
            imageRepository.save(image);
        }
    }
}
```

```java
Post post = Post.of(...); // new → 비영속 상태
postRepository.save(post); // 저장

Image image = imageRepository.findById(imageId).orElseThrow();
image.setPost(PostJpaEntity.from(post)); // 비영속 객체 주입
imageRepository.save(image); // 예외 발생
```

- PostJpaEntity.from(post)는 DB에 저장된 post와 같은 값을 가지더라도 JPA 입장에서는 새로 생성된 비영속 객체이다.
- image.setPost(...) 시, Hibernate는 비영속 객체를 참조하는 것은 불가능하다고 판단하게 되고 `TransientPropertyValueException`이 발생하게 된다.

# 해결

## 수정 코드

```java
Post saved = postRepository.save(post); // 영속 상태 객체

if (createPostCommand.imageIds() != null) {
    for (long imageId : createPostCommand.imageIds()) {
        Image image = imageRepository.findById(imageId)
                .orElseThrow(() -> new ImageException(IMAGE_NOT_FOUND));

        image.setPost(saved); // 영속 객체를 직접 주입
        imageRepository.save(image);
    }
}
```

`TransientPropertyValueException`은 비영속 객체를 연관관계에 주입했을 때 발생하는 Hibernate 예외이다.

이를 해결하기 위해 연관관계에 들어가는 객체를 영속 상태로 바꿔주어야 한다. 이를 위해 save()로 저장된 영속 객체를 그대로 사용하고, new, from() 등으로 새 객체를 생성해 참조하지 않도록 주의해야 한다.

JPA는 영속성 컨텍스트를 통해 객체 상태를 추적하고 flush 시점에 DB 반영, 동일 객체 유지, 자동 dirty checking 등의 기능을 제공한다. 따라서 JPA가 인식하지 못하는 객체(비영속)를 참조하게 되면 Hibernate는 무결성 및 동기화 위험을 감지하고 예외를 발생시키는 것이다.
