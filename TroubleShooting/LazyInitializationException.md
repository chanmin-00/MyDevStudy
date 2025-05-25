# 문제 상황

```java
@SpringBootTest
@ActiveProfiles("test")
class LazyLoadingTest {

	@Autowired
	private LocationJpaRepository locationJpaRepository;

	@Autowired
	private UserJpaRepository userJpaRepository;

	@Autowired
	private LocationService locationService;

	private Long savedLocationId;
	private Long savedUserId;

	@BeforeEach
	void setUp() {
		// 테스트용 유저와 위치 저장
		UserJpaEntity user = userJpaRepository.save(
			UserJpaEntity.builder()
				.id(1L) // ID를 명시적으로 설정
				.email("test@naver.com")
				.password("1234")
				.build()
		);

		LocationJpaEntity location = locationJpaRepository.save(
			LocationJpaEntity.builder()
				.id(1L) // ID를 명시적으로 설정
				.placeName("Test Place")
				.latitude(37.0)
				.longitude(127.0)
				.user(user)
				.build()
		);

		savedLocationId = location.getId();
		savedUserId = user.getId();
	}

	@Test
	void lazyLoading_shouldFailWithoutTransaction() {
		locationService.deleteLocation(savedLocationId, savedUserId);
	}

}
```

이 테스트 코드 실행 중 다음과 같은 예외 상황이 발생하였다.

```bash
could not initialize proxy [com.cotato.backend.domain.user.persistence.jpa.entity.UserJpaEntity#1] - no Session
org.hibernate.LazyInitializationException: could not initialize proxy [com.cotato.backend.domain.user.persistence.jpa.entity.UserJpaEntity#1] - no Session
	at org.hibernate.proxy.AbstractLazyInitializer.initialize(AbstractLazyInitializer.java:165)
	at org.hibernate.proxy.AbstractLazyInitializer.getImplementation(AbstractLazyInitializer.java:314)
	at org.hibernate.proxy.pojo.bytebuddy.ByteBuddyInterceptor.intercept(ByteBuddyInterceptor.java:44)
	at org.hibernate.proxy.ProxyConfiguration$InterceptorDispatcher.intercept(ProxyConfiguration.java:102)
	at com.cotato.backend.domain.user.persistence.jpa.entity.UserJpaEntity$HibernateProxy$nIz66TSA.getEmail(Unknown Source)
	at com.cotato.backend.domain.user.domain.User.from(User.java:26)
```

## 발생 위치

해당 예외는 아래와 같은 코드 흐름 속에서 발생했다.

```java
@Override
public void deleteLocation(Long locationId, Long userId) {
	Location location = locationPort.findByIdAndUserId(locationId, userId)
		.orElseThrow(() -> new CustomException(ErrorCode.LOCATION_NOT_FOUND));

	locationPort.delete(location);
}
```

```java
@Override
public Optional<Location> findByIdAndUserId(Long locationId, Long userId) {
	return repository.findByIdAndUserId(locationId, userId)
		.map(Location::from);
}
```

```java
public static Location from(LocationJpaEntity locationJpaEntity) {
	return new Location(
			locationJpaEntity.getId(),
			locationJpaEntity.getPlaceName(),
			locationJpaEntity.getLatitude(),
			locationJpaEntity.getLongitude(),
			locationJpaEntity.isPinned(),
			User.from(locationJpaEntity.getUser())
	);
}
```

```java
public static User from(UserJpaEntity userJpaEntity) {
	return new User(userJpaEntity.getId(), userJpaEntity.getEmail(), userJpaEntity.getPassword());
}
```

즉, LocationJpaEntity를 도메인 객체로 변환하는 과정에서 내부에 연관된 UserJpaEntity의 getEmail()을 호출하는 시점에 `Lazy 로딩`이 시도되었고 이때 Hibernate 세션이 닫혀 있어서 예외가 발생한 것이다.

# 문제 원인

```java
@Entity
public class LocationJpaEntity extends BaseEntity {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "user_id")
	private UserJpaEntity user;

}
```

LocationJpaEntity는 UserJpaEntity와 `@ManyToOne` 관계를 맺고 있으며, `fetch = FetchType.LAZY`로 설정되어 있다. 이 설정은 연관 객체를 실제로 접근할 때까지 데이터베이스에서 로딩을 지연시키는 전략이다. 따라서 해당 연관 객체는 초기에는 실제 엔터티가 아닌 `프록시(proxy)` 객체로 주입된다.

실제 쿼리 로그를 확인해보면 다음과 같이 User에 대한 조회 쿼리는 발생하지 않은 것을 알 수 있다

```bash
Hibernate: select lje1_0.id,lje1_0.created_at,lje1_0.latitude,lje1_0.longitude,lje1_0.pinned,lje1_0.place_name,lje1_0.updated_at,lje1_0.user_id from location_jpa_entity lje1_0 where lje1_0.id=? and lje1_0.user_id=?
```

문제는 location.getUser() 이후 user.getEmail()을 호출하는 시점이다. 이 시점에 프록시 객체는 Hibernate 세션을 통해 실제 데이터를 로딩하려고 시도한다. 하지만 테스트 코드의 서비스 메서드에는 `@Transactional`이 붙어 있지 않기 때문에 트랜잭션이 종료되고 그에 따라 `영속성 컨텍스트(Session)`도 함께 종료되어 있는 상태이다.(영속성 컨텍스트는 보통 트랜잭션과 생명주기를 같이한다)

```bash
at com.cotato.backend.domain.user.persistence.jpa.entity.UserJpaEntity$HibernateProxy$nIz66TSA.getEmail(Unknown Source)
```

즉 내부적으로 @Transactional이 적용된 findByIdAndUserId() 메서드의 실행이 끝나면서 세션도 함께 닫히고, 그 이후 프록시 객체 초기화 시도가 발생하면서 `LazyInitializationException`이 발생한 것이다.

실제로 Hibernate 내부 예외 발생 코드를 보면 다음과 같다:

```java
protected void permissiveInitialization() {
    if (this.session == null) {
        if (this.sessionFactoryUuid == null) {
            throw new LazyInitializationException("could not initialize proxy [" + this.entityName + "#" + this.id + "] - no Session");
        }
    ...
}
```

- `session`과 `sessionFactoryUuid`가 null인 경우 위와 같은 예외가 발생한다는 것을 확인할 수 있다.

Hibernate의 Lazy Loading은 세션이 살아 있는 동안에만 정상 작동한다. 이 세션은 보통 트랜잭션과 생명주기를 함께하기 때문에 트랜잭션이 없거나 일찍 종료되면 Lazy 로딩이 실패하게 된다.

# 문제 해결

문제의 핵심은 `LazyLoading` 이 필요한 시점에 Hibernate 세션이 닫혀 있었기 때문에 발생한 예외였다. 따라서 해결책은 영속성 컨텍스트를 유지하여 세션이 닫히지 않도록 보장하는 것이다. 이를 위해 다음과 같은 방법을 고려하였다.

- 서비스 메서드에 `@Transactional`을 붙여 세션 생명주기를 보장한다.

## @Transactional 추가

가장 직관적인 해결 방법은 테스트나 서비스 메서드에 `@Transactional`을 추가하는 것이다. 트랜잭션이 유지되는 동안 Hibernate 세션도 유지되므로 Lazy 필드에 안전하게 접근할 수 있게 된다.

테스트 실행 결과 `User` 조회를 위한 쿼리가 정상적으로 실행되었음을 로그에서 확인할 수 있다:

```bash
Hibernate: select lje1_0.id, ... from location_jpa_entity lje1_0 where lje1_0.id=? and lje1_0.user_id=?
Hibernate: select uje1_0.id, uje1_0.email, uje1_0.password from users uje1_0 where uje1_0.id=?
```

## LazyLoading 자체를 발생시키지 않기

해결 과정을 거치며 다음과 같은 의문이 생겼다

> Location 도메인을 만들기 위해 꼭 User 객체를 데이터베이스에서 조회해야 할까?
> 

사실 삭제 기능을 수행하기 위해서는 `Location`의 `id`만으로 충분하다.

즉, `User` 정보 없이도 삭제 로직은 아무 문제 없이 작동할 수 있다

```java
@Override
public void delete(Location location) {
	repository.deleteById(location.getId());
}
```

이러한 상황에서는 굳이 LazyLoading을 유발할 필요가 없다. 따라서 LocationJpaEntity를 Location 도메인 객체로 변환할 때 User 정보를 null로 설정하여 Lazy 필드 접근 자체를 막았다

```java
public static Location from(LocationJpaEntity locationJpaEntity) {
	return new Location(
		locationJpaEntity.getId(),
		locationJpaEntity.getPlaceName(),
		locationJpaEntity.getLatitude(),
		locationJpaEntity.getLongitude(),
		locationJpaEntity.isPinned(),
		null // User 정보는 필요하지 않으므로 조회하지 않음
	);
}
```

이 방식으로 코드를 수정한 후 실행한 테스트에서도 User에 대한 조회 쿼리가 발생하지 않음을 확인할 수 있었다

```bash
Hibernate: select lje1_0.id, ... from location_jpa_entity lje1_0 where lje1_0.id=? and lje1_0.user_id=?
Hibernate: delete from location_jpa_entity where id=?
```

결과적으로, User에 대한 LazyLoading 없이도 Location 삭제가 성공적으로 수행되었다.
