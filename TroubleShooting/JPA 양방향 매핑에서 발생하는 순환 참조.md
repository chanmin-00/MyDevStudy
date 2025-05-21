# 순환 참조

순환 참조란 두 객체가 서로를 직접 참조하는 구조이다.

```java
class A {
    B b;
}

class B {
    A a;
}
```

이 구조는 다음과 같은 계층에서 흔히 발생한다.

- JPA Entity (양방향 매핑)
- Service 계층 (서로 호출)
- DTO ↔ Domain
- DI (Spring Bean)
- JSON 직렬화

# JPA 양방향 매핑에서의 순환 참조

예를 들어, 다음과 같이 `User`와 `Location` 간 양방향 연관관계를 설정할 수 있다.

```java
@Entity
public class UserJpaEntity {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	...
	
	@OneToMany(mappedBy = "user")
	private List<LocationJpaEntity> location;

	...

}
```

```java
@Entity
public class LocationJpaEntity {

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private Long id;

	@Setter
	@ManyToOne(fetch = FetchType.LAZY)
	@JoinColumn(name = "user_id")
	private UserJpaEntity user;

	...

}
```

- 이 구조는 다음과 같은 무한 루프를 유발할 수 있다.
- UserJpaEntity → List<LocationJpaEntity> → UserJpaEntity → List<LocationJpaEntity> 식으로 무한 루프가 발생할 수 있다.

순환 참조를 테스트하고자 작성한 테스트 코드는 다음과 같다.

```java
@Test
@Transactional
@DisplayName("순환 참조 발생 여부 테스트 - JSON 직렬화")
void testCircularReferenceOnJsonSerialization() throws Exception {
    // given
    UserJpaEntity userJpaEntity = new UserJpaEntity(null, "string@naver.com", "string1!", new ArrayList<>());
    UserJpaEntity savedUser = userJpaRepository.save(userJpaEntity);

    LocationJpaEntity location = new LocationJpaEntity(null, savedUser, "서울", 0, 0, true);
    LocationJpaEntity savedLocation = locationJpaRepository.save(location);

    savedUser.getLocation().add(savedLocation);
    savedLocation.setUser(savedUser);

    // when + then
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.enable(SerializationFeature.FAIL_ON_SELF_REFERENCES); // 순환 감지 활성화

    try {
        String json = objectMapper.writeValueAsString(savedUser);
        log.info("JSON: {}", json);
    } catch (JsonProcessingException e) {
        log.error("JSON 직렬화 중 오류 발생: {}", e.getMessage());
        throw e;
    }
}
```

이 테스트 코드를 실행하면 다음과 같은 오류가 발생한다. JSON 직렬화 과정에서 객체가 반복적으로 서로를 참조하면서 무한 루프에 빠지는 오류가 발생한다.

```java
Document nesting depth (1001) exceeds the maximum allowed (1000, from `StreamWriteConstraints.getMaxNestingDepth()`) (through reference chain: com.cotato.backend.domain.user.persistence.jpa.entity.UserJpaEntity["location"]->org.hibernate.collection.spi.PersistentBag[0]->com.cotato.backend.domain.location.persistence.jpa.entity.LocationJpaEntity["user"]->com.cotato.backend.domain.user.persistence.jpa.entity.UserJpaEntity["location"]->org.hibern
```

# 양방향 연관관계의 문제점

양방향 연관관계는 Jackson 직렬화 경우 뿐만 아니라 다음과 같은 문제를 유발할 가능성이 있다곤 한다.

1. `책임 분산` : 양방향 관계는 객체 간 역할과 책임을 모호하게 만든다. 예를 들어, Team ↔ Member 양방향 관계에서 다음과 같은 문제가 발생할 수 있다.
    
    ```java
    @Data
    class Member {
        private Team team;
    
        public int getTotalTeamSalary() {
            return team.getMembers().stream()
                .mapToInt(Member::getSalary)
                .sum();  // 팀원이 팀 전체 월급을 계산함
        }
    }
    ```
    
    Team의 책임이 Member에게 위임되어 있다.  책임은 팀 단위에서 관리되어야 하며, TeamService로 분리하는 것이 적절하다.
    
2. `복잡도 증가` : 어디서 객체가 갱신되는지 추적이 어려워진다.
3. `Cascade 전이 오류` : 양방향 관계에서 `CascadeType.ALL` 혹은 `orphanRemoval = true` 등의 설정이 양쪽에 걸쳐 있을 경우 다음과 같은 문제가 발생할 수 있다.
    - 한쪽만 삭제했는데 양쪽 다 삭제되는 상황
    - 수정, 삭제 시 영속성 전이 범위를 예측하기 어려움
    
    ```java
    @OneToMany(mappedBy = "team", cascade = CascadeType.ALL)
    private List<Member> members;
    
    @ManyToOne
    @JoinColumn(name = "team_id")
    private Team team;
    ```
    
    이 상태에서 팀을 삭제할 경우 멤버도 함께 삭제된다. 그런데 멤버가 또 다른 연관 엔티티를 가진다면, Cascade 전이가 꼬일 수 있다.
    

…

# 해결 방안

### 1. **단방향 연관관계만 사용**

```java
@Entity
public class LocationJpaEntity {
    @ManyToOne(fetch = LAZY)
    @JoinColumn(name = "user_id")
    private UserJpaEntity user;
}
```

- User는 Location 참조를 제거한다.
- 필요 시 Repository 기반 조회로 대체할 수 있다.

```java
locationRepository.findByUserId(id);
```

### **2. 직렬화 방지 어노테이션 활용**

```java
@OneToMany(mappedBy = "user")
@JsonIgnore // Jackson 직렬화 제외
private List<LocationJpaEntity> location;
```

### 3. **식별자(ID)만 저장 (간접 참조)**

```java
@Entity
public class LocationJpaEntity {
    private Long userId;
}
```

- 도메인 간 결합도 완화
- 쿼리로 사용자 조회

# 양방향 매핑을 지양하자

JPA의 양방향 매핑은 순환 참조의 주요 원인이다. 가능하면 단방향 설계를 유지하고, 정말 필요한 경우에만 양방향 매핑을 사용한다.

- JPA에 도메인을 맞추지 말고, 도메인에 JPA를 덧입히는 방식으로 설계해야 한다.
- 양방향 매핑은 “할 수 있으니 하는 것”이 아니라, “정말 필요한 경우에만” 선택한다.

> Hibernate 문서도 양방향 매핑을 쿼리 생성을 쉽게 하기 위함으로 소개하고 있지만, 순환 참조의 우려가 존재한다고  말하고 있다.
>
