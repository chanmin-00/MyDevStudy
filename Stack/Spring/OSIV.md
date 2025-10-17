# 문제 상황

다음은 특정 유저의 위치 정보를 삭제하는 서비스 메서드이다.

```java
@Override
public void deleteLocation(Long locationId, Long userId) {
    Location location = locationPort.findByIdAndUserId(locationId, userId)
        .orElseThrow(() -> new CustomException(ErrorCode.LOCATION_NOT_FOUND));

    ...
}
```

LocationPort는 내부적으로 다음과 같이 동작한다.

```java
@Override
public Optional<Location> findByIdAndUserId(Long locationId, Long userId) {
    return repository.findByIdAndUserId(locationId, userId)
        .map(Location::from);
}
```

그리고 Location::from은 JPA 엔티티를 도메인 객체로 변환한다.

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

문제는 여기서 User.from이 호출되는 시점이다.

```java
public static User from(UserJpaEntity userJpaEntity) {
    return new User(userJpaEntity.getId(), userJpaEntity.getEmail(), userJpaEntity.getPassword());
}
```

`테스트 코드` 로 이 메서드를 실행했을 때, 다음과 같은 예외가 발생했다.

```bash
org.hibernate.LazyInitializationException: could not initialize proxy
[com.cotato.backend.domain.user.persistence.jpa.entity.UserJpaEntity#1] - no Session
```

이 예외는 지연 로딩(Lazy Loading) 된 UserJpaEntity 객체를 영속성 컨텍스트가 닫힌 이후에 접근하려고 할 때 발생한 것이다. 테스트 코드에서는 서비스 메서드에 `@Transactional`을 붙이지 않았기 때문에, 트랜잭션 범위가 존재하지 않고 EntityManager(Session)가 닫힌 상태에서 프록시 객체에 접근하게 된다.

즉, locationJpaEntity.getUser()는 프록시 객체를 반환하며, getEmail() 호출 시 실제 데이터를 불러오기 위해 초기화를 시도하지만 이 시점에는 이미 세션이 종료되어 있어서 예외가 발생하는 것이다.

그런데 이 코드를 `@Transactional` 없이 실행했음에도 불구하고, Swagger를 통해 HTTP 요청으로 호출할 때는 예외가 발생하지 않았다. 반면, 동일한 코드를 테스트 코드에서 실행하면 `LazyInitializationException`이 발생했다.

> **왜 테스트 코드에서는 예외가 발생하고, HTTP 요청을 통한 실행에서는 예외가 발생하지 않는 걸까?**
> 

# 문제 원인

HTTP 요청을 통한 실행에서 예외가 발생하지 않은 이유는 `OSIV(Open Session In View)` 설정 때문이다. 이 기회로 OSIV가 무엇인지 명확히 파악해보자.

## OSIV(Open Session In View)

OSIV는 HTTP 요청이 시작될 때 영속성 컨텍스트(EntityManager)를 열고, 요청이 완료될 때까지 이를 유지하는 기능이다. 이 기능이 활성화되어 있으면, 서비스 계층에서 트랜잭션이 종료된 이후에도 컨트롤러나 뷰에서 지연 로딩(Lazy Loading)이 가능해진다.

Spring Boot에서는 기본적으로 다음과 같이 OSIV가 활성화되어 있다

```
spring.jpa.open-in-view=true
```

따라서 아래와 같은 상황에서도 예외가 발생하지 않는다

- findByIdAndUserId 메서드 실행 → 트랜잭션 종료
- 이후 locationJpaEntity.getUser().getEmail() 호출 → 지연 로딩 발생
- 하지만 OSIV 덕분에 영속성 컨텍스트가 여전히 열려 있어 정상적으로 DB 접근 가능

즉, 트랜잭션이 종료되어도 영속성 컨텍스트가 열려 있기 때문에 지연 로딩이 가능하며, `LazyInitializationException`이 발생하지 않는 것이다.

> 영속성 컨텍스트는 트랜잭션 외부에서도 *조회* 목적이라면 사용할 수 있다. 이를 `Non-transactional read(트랜잭션 없이 읽기)` 라고 하며, 이 경우에도 OSIV가 활성화되어 있다면 Lazy 로딩이 정상적으로 작동할 수 있다.
> 

## OSIV 동작 원리

### JpaWebConfiguration

먼저 JpaWebConfiguration 설정을 보게 되면 다음과 같다.

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({WebMvcConfigurer.class})
@ConditionalOnMissingBean({OpenEntityManagerInViewInterceptor.class, OpenEntityManagerInViewFilter.class})
@ConditionalOnMissingFilterBean({OpenEntityManagerInViewFilter.class})
@ConditionalOnProperty(
     prefix = "spring.jpa",
     name = {"open-in-view"},
     havingValue = "true",
     matchIfMissing = true
)
protected static class JpaWebConfiguration {
				private static final Log logger = LogFactory.getLog(JpaWebConfiguration.class);
        private final JpaProperties jpaProperties;
        
        ...
        
```

```java
public class JpaProperties {
    private Map<String, String> properties = new HashMap();
    private final List<String> mappingResources = new ArrayList();
    private String databasePlatform;
    private Database database;
    private boolean generateDdl = false;
    private boolean showSql = false;
    private Boolean openInView;
    
    ...
```

- @ConditionalOnProperty(...) 어노테이션을 통해 `spring.jpa.open-in-view=true`인 경우에만 해당 설정이 작동된다. yml 파일에 생략된 경우에도 true로 간주한다.
- 조건이 맞을 경우 이 클래스는 OSIV 기능을 활성화하는 설정을 구성한다.

공식 문서의 openInView에 대한 설명은 다음과 같다. 이를 통해 `openInView` 설정을 통해 `OpenEntityMangerInViewInterceptor` 가 동작함을 확인할 수 있다.

<img width="735" alt="image" src="https://github.com/user-attachments/assets/953b68b1-195e-42b8-83d9-6ce791fad1a9" />


> OpenEntityManagerInViewInterceptor를 등록합니다. 요청을 처리하는 전체 과정 동안 JPA의 EntityManager를 스레드에 바인딩합니다.
> 

### OpenEntityManagerInViewInterceptor

opnInView가 true임을 통해 OpenEntityManagerInViewInterceptor에 등록한다고 했으니 OpenEntityManagerInViewInterceptor가 어떻게 동작하는지 알아보자.

OpenEntityManagerInViewInterceptor에 대해서 공식문서는 다음과 같이 명시하고 있다.

> Spring web request interceptor that binds a JPA EntityManager to the thread for the entire processing of the request. Intended for the "Open EntityManager in View" pattern, i. e. to allow for lazy loading in web views despite the original transactions already being completed.
This interceptor makes JPA EntityManagers available via the current thread, which will be autodetected by transaction managers. It is suitable for service layer transactions via org. springframework. orm. jpa. JpaTransactionManager or org. springframework. transaction. jta. JtaTransactionManager as well as for non-transactional read-only execution.
In contrast to OpenEntityManagerInViewFilter, this interceptor is set up in a Spring application context and can thus take advantage of bean wiring.
> 
- Spring 웹 요청 인터셉터로, 하나의 요청을 처리하는 전체 과정 동안 JPA EntityManager를 현재 스레드에 바인딩한다.
- 이는 `Open EntityManager in View` 패턴을 위한 것으로, 트랜잭션이 이미 종료되었더라도 웹 뷰에서 `지연 로딩(Lazy Loading)`이 가능하도록 하기 위해 사용된다.
- 이 인터셉터는 현재 스레드를 통해 `EntityManager`에 접근할 수 있도록 하며, 이는 트랜잭션 매니저에 의해 자동으로 감지된다.
- 이 인터셉터는 서비스 계층의 트랜잭션에 적합하며, JpaTransactionManager나 JtaTransactionManager를 사용하는 트랜잭션 환경에서도 사용할 수 있고, 트랜잭션이 없는 읽기 전용(read-only) 실행에도 적합하다.

`OpenEntityManagerInViewInterceptor` 의 구현 코드를 확인해보자. OpenEntityManagerInViewInterceptor는 웹 요청 전후에 EntityManager의 생명주기를 관리하고 있다.

```java
public class OpenEntityManagerInViewInterceptor extends EntityManagerFactoryAccessor implements AsyncWebRequestInterceptor {

	@Override
	public void preHandle(WebRequest request) throws DataAccessException {
		String key = getParticipateAttributeName();
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		if (asyncManager.hasConcurrentResult() && applyEntityManagerBindingInterceptor(asyncManager, key)) {
			return;
		}

		EntityManagerFactory emf = obtainEntityManagerFactory();
		if (TransactionSynchronizationManager.hasResource(emf)) {
			// 이미 존재하는 경우 단순히 참여 횟수만 증가
			Integer count = (Integer) request.getAttribute(key, WebRequest.SCOPE_REQUEST);
			int newCount = (count != null ? count + 1 : 1);
			request.setAttribute(key, newCount, WebRequest.SCOPE_REQUEST);
		} else {
			// EntityManager 새로 생성 및 바인딩
			try {
				EntityManager em = createEntityManager();
				EntityManagerHolder emHolder = new EntityManagerHolder(em);
				TransactionSynchronizationManager.bindResource(emf, emHolder);

				AsyncRequestInterceptor interceptor = new AsyncRequestInterceptor(emf, emHolder);
				asyncManager.registerCallableInterceptor(key, interceptor);
				asyncManager.registerDeferredResultInterceptor(key, interceptor);
			} catch (PersistenceException ex) {
				throw new DataAccessResourceFailureException("Could not create JPA EntityManager", ex);
			}
		}
	}

	@Override
	public void afterCompletion(WebRequest request, @Nullable Exception ex) throws DataAccessException {
		// 참여 횟수를 감소시키고, 마지막이라면 EntityManager 종료
		if (!decrementParticipateCount(request)) {
			EntityManagerHolder emHolder = (EntityManagerHolder)
					TransactionSynchronizationManager.unbindResource(obtainEntityManagerFactory());
			EntityManagerFactoryUtils.closeEntityManager(emHolder.getEntityManager());
		}
	}
}

```

- `preHandle()`에서 EntityManager를 생성 및 바인딩하고, `afterCompletion()`에서 해제하고 있다.
- 이 구조는 하나의 HTTP 요청 동안 EntityManager가 유지됨을 의미한다.

그러면 실제로 EntityManager가 재사용되고 있는지 확인해보자.

```java
@Override
protected Object doGetTransaction() {
	JpaTransactionObject txObject = new JpaTransactionObject();
	txObject.setSavepointAllowed(isNestedTransactionAllowed());

	EntityManagerHolder emHolder = (EntityManagerHolder)
		TransactionSynchronizationManager.getResource(obtainEntityManagerFactory());

	if (emHolder != null) {
		txObject.setEntityManagerHolder(emHolder, false);
	}

	if (getDataSource() != null) {
		ConnectionHolder conHolder = (ConnectionHolder)
			TransactionSynchronizationManager.getResource(getDataSource());
		txObject.setConnectionHolder(conHolder);
	}

	return txObject;
}
```

- JPA에서는 트랜잭션 시작 시, `JpaTransactionManager`의 `doGetTransaction()` 메서드가 호출되어 EntityManagerHolder를 찾는다.
- 즉, `TransactionSynchronizationManager`에 저장된 `EntityManagerHolder`를 찾아 재사용한다. 이는 OSIV가 활성화되어 있을 경우, 동일한 요청 내의 여러 트랜잭션에서도 같은 EntityManager 가 사용됨을 의미한다.

아래 코드를 실행시켜 디버깅해보았다.

```java
User user1 = User.withoutId("string@naver.com", "password1!");
User user2 = User.withoutId("string1@naver.com", "password1!");
saveUserPort.save(user1);
saveUserPort.save(user2);
```

- spring.jpa.open-in-view=false 설정 시
    - 첫 번째 트랜잭션에서 EntityManagerHolder는 null이었고 새로운 EntityManager가 생성됨
    - 두 번째 트랜잭션에서도 EntityManagerHolder가 다시 null이었음
    - 즉, HTTP 요청 범위에서 EntityManager가 유지되지 않았다.
    
    <img width="536" alt="image" src="https://github.com/user-attachments/assets/35e3b161-41c9-4b15-b72f-20b7e0f87aa3" />

    
- spring.jpa.open-in-view=true 설정 시
    - 첫 번째 트랜잭션에서 생성된 EntityManager의 주소가 두 번째 트랜잭션에서 EntityManagerHolder를 통해 그대로 재사용됨을 확인할 수 있었다.
    
    <img width="529" alt="image" src="https://github.com/user-attachments/assets/35f85c31-abd1-4953-97c7-6dd0c2a2a761" />

    

# 문제 해결

OSIV(Open Session In View)의 동작 원리를 이해함으로써, 테스트 코드에서는 지연 로딩이 실패하고 Swagger나 HTTP 요청에서는 성공하는 이유를 명확히 확인할 수 있었다.

나의 테스트 코드는 HTTP 요청을 거치지 않고 서비스 메서드를 직접 호출하기 때문에, `OpenEntityManagerInViewInterceptor`가 작동하지 않는다. 그 결과 요청 범위의 영속성 컨텍스트가 생성되지 않고, 트랜잭션이 끝난 시점에서 지연 로딩을 시도하면 `LazyInitializationException`이 발생한다.

반면 Swagger나 웹 요청의 경우, DispatcherServlet을 통해 요청이 들어오면서 인터셉터가 실행되고, EntityManager가 요청이 끝날 때까지 유지된다. 이로 인해 트랜잭션이 없어도 지연 로딩이 가능하다.

실제로 OSIV를 `false`로 설정했을 때는 두 번의 트랜잭션 동안 `EntityManagerHolder`가 null로 나타나 매번 새로 생성되었고, `true`로 설정했을 때는 같은 주소의 `EntityManager`가 재활용되는 것이 디버깅을 통해 확인되었다.

OSIV는 트랜잭션이 없어도 지연 로딩을 가능하게 해주는 장점이 있지만, 요청이 끝날 때까지 DB 커넥션을 점유하기 때문에 자원 낭비의 문제가 발생할 수 있다. 따라서 서비스의 성격과 상황에 따라 OSIV 설정 여부를 신중하게 선택하는 것이 중요하다고 생각된다.

# 참고

https://jaehoney.tistory.com/390

[https://ykh6242.tistory.com/entry/JPA-OSIVOpen-Session-In-View와-성능-최적화](https://ykh6242.tistory.com/entry/JPA-OSIVOpen-Session-In-View%EC%99%80-%EC%84%B1%EB%8A%A5-%EC%B5%9C%EC%A0%81%ED%99%94)
