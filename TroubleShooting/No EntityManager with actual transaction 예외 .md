# 문제 상황

```java
@Override
public void createLocation(LocationCommand command) {
	User user = loadUserPort.findById(command.userId())
		.orElseThrow(() -> new CustomException(ErrorCode.USER_NOT_FOUND));

	Location location = Location.of(
		command.placeName(),
		command.latitude(),
		command.longitude(),
		command.pinned(),
		user
	);

	locationPort.save(location);
}
```

```java
@Override
public void deleteLocation(Long locationId, Long userId) {
	locationPort.deleteByIdAndUserId(locationId, userId);
}
```

서비스 클래스의 createLocation 메소드와 deleteLocation 메소드에 `@Transactional` 어노테이션이 붙지 않은 상황에서 두 메소드를 실행시켜 보았다.

createLocation의 save 메소드를 호출한 결과 정상 작동 하였지만, deleteByIdAnd 메소드를 호출한 결과 다음과 같은 오류가 발생했다.

### 에러 메시지

```bash
2025-05-20T15:30:39.173+09:00 ERROR 9017 --- [backend] [nio-8080-exec-2] c.c.b.c.e.GlobalExceptionHandler         : Unexpected error: No EntityManager with actual transaction available for current thread - cannot reliably process 'remove' call at /api/v1/locations/2
```

# 문제 원인

문제 원인을 파악해보자. 

<img width="823" alt="image" src="https://github.com/user-attachments/assets/7388ddb3-8be9-40a9-afcb-435c9ada442a" />


JpaRepository는 SimpleJpaRepository를 상속받는다. SimpleJpaRepository는 기본적으로 CRUD 연산을 위한 메소드들을 제공해준다.

- save, delete, deleteById, findById 등등

```java
@Transactional
public <S extends T> S save(S entity) {
    Assert.notNull(entity, "Entity must not be null");
    if (this.entityInformation.isNew(entity)) {
        this.entityManager.persist(entity);
        return entity;
    } else {
        return this.entityManager.merge(entity);
    }
}
```

```java
@Transactional
public void delete(T entity) {
    Assert.notNull(entity, "Entity must not be null");
    if (!this.entityInformation.isNew(entity)) {
        Class<?> type = ProxyUtils.getUserClass(entity);
        T existing = this.entityManager.find(type, this.entityInformation.getId(entity));
        if (existing != null) {
            this.entityManager.remove(this.entityManager.contains(entity) ? entity : this.entityManager.merge(entity));
        }
    }
}
```

기본 CRUD 메소드에는 기본적으로 `@Transactional` 어노테이션이 붙고 있다. 하지만 내가 사용하고자 하는 deleteByIdAndUserId 메소드는 내가 직접 작성한 커스텀 쿼리 메서드이다. 이 경우에는 `@Transactional` 어노테이션이 자동으로 붙지 않게 된다.

### ❓ 왜 기본적으로 붙지 않는가? (ChatGPT 참고)

> Spring Data JPA는 SimpleJpaRepository에서 정의된 기본 메소드들에 대해서만 명시적으로 @Transactional을 붙여 트랜잭션 처리를 보장해준다. 하지만 개발자가 정의한 커스텀 쿼리 메서드에는 자동으로 트랜잭션이 적용되지 않는다.
> 
> 
> 이는 스프링이 내부적으로 `@Transactional`을 메소드 단위로 AOP로 감싸는 방식이기 때문에, 개발자가 작성한 메소드에 명시하지 않는 이상 트랜잭션이 시작되지 않기 때문이다.
> 
> 즉, deleteById()처럼 프레임워크가 직접 구현한 메소드는 트랜잭션이 적용되어 있지만, deleteByIdAndUserId()처럼 개발자가 선언한 메소드는 개발자가 직접 트랜잭션을 지정해주지 않으면 아무런 트랜잭션 처리가 이루어지지 않는다.
> 

### ❓ 왜 JPA에서는 트랜잭션 어노테이션이 꼭 필요한가?

JPA는 내부적으로 EntityManager를 사용해 데이터를 처리한다. 그런데 EntityManager는 반드시 트랜잭션 안에서만 안전하게 동작한다. 그 이유는 다음과 같다

- `persist`, `merge`, `remove` 같은 영속성 작업은 실제로 DB에 반영되기 전까지는 영속성 컨텍스트에만 반영된다
- 이 컨텍스트는 트랜잭션이 시작되어야만 플러시(flush)되고, 커밋되거나 롤백되어 DB에 반영될 수 있다
- 트랜잭션이 없다면 JPA는 다음과 같은 동작을 할 수 없다
    - `remove()` 호출 시 영속성 컨텍스트에서 제거되었는지 추적할 수 없음
    - `flush()` 타이밍을 보장할 수 없음
    - DB와의 데이터 정합성을 보장할 수 없음

즉, JPA는 자동 커밋 기반이 아니라 트랜잭션 경계 안에서 데이터를 조작하도록 설계되어 있다.

서비스 계층에도 `@Transactional` 어노테이션이 붙어있지 않기 때문에 트랜잭션이 전파되지 않았다. 스프링에서는 트랜잭션은 호출자 → 피호출자로 전파되기 때문에 서비스 레이어에서 트랜잭션을 시작하지 않으면 내부에서 호출되는 Repository의 메소드 또한 트랜잭션 없이 실행된다.

결과적으로, `EntityManager.remove()`가 트랜잭션 없이 실행되었고, 이는 다음과 같은 예외로 이어졌다

```

No EntityManager with actual transaction available for current thread
```

# 문제 해결

이 문제는 트랜잭션이 명시되지 않은 상태에서 JPA의 삭제 연산이 수행되었기 때문에 발생했다. 해결 방법은 간단하다. 트랜잭션이 필요한 계층에 명시적으로 `@Transactional` 어노테이션을 추가해 트랜잭션을 시작해주면 된다.

서비스 계층에서 트랜잭션을 시작하도록 아래와 같이 `@Transactional`을 선언해준다:

```java
import org.springframework.transaction.annotation.Transactional;

@Transactional
@Override
public void deleteLocation(Long locationId, Long userId) {
	locationPort.deleteByIdAndUserId(locationId, userId);
}

```

이렇게 하면 `deleteByIdAndUserId()`에서 내부적으로 호출되는 `EntityManager.remove()`가 트랜잭션 안에서 수행되기 때문에 예외가 발생하지 않고 정상적으로 삭제가 이루어진다.

서비스 계층에서 트랜잭션을 선언하면 하위 레이어(예: Repository)로 트랜잭션이 전파되기 때문에, 일반적으로 트랜잭션 선언은 서비스 계층에서 관리하는 것이 권장된다고 한다.
