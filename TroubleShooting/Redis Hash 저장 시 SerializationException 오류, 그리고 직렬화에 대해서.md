# 문제 상황

리플 프로젝트 레벨 테스트 기능을 리팩토링하면서 레벨 테스트용 퀴즈 정보를 Redis의 `Hash` 자료구조에 저장하려고 했으나 다음과 같은 예외가 발생했다.

```java
org.springframework.data.redis.serializer.SerializationException: Cannot serialize
    at org.springframework.data.redis.serializer.JdkSerializationRedisSerializer.serialize(JdkSerializationRedisSerializer.java:97)
    at org.springframework.data.redis.core.DefaultHashOperations.putAll(DefaultHashOperations.java:161)
    at com.ripple.BE.learning.application.quiz.QuizRedisService.saveMapToRedis(QuizRedisService.java:76)
    ...
Caused by: java.lang.IllegalArgumentException: DefaultSerializer requires a Serializable payload but received an object of type [com.ripple.BE.learning.dto.response.leveltest.LevelTestQuizAnswerDTO]
    at org.springframework.core.serializer.DefaultSerializer.serialize(DefaultSerializer.java:43)
    ...

```

오류가 난 코드는 QuizRedisService의 다음 부분이다.

```java
// Redis에 Map 저장
protected <K, V> void saveMapToRedis(final String key, final String type, final Map<K, V> data) {
    String newKey = getRedisKey(key, type);
    redisTemplate.opsForHash().putAll(newKey, data);
    redisTemplate.expire(key, Duration.ofMinutes(QUIZ_TIME));
}
```

오류 로그를 해석해 보면 핵심 메시지는 다음과 같다.

```java
IllegalArgumentException: DefaultSerializer requires a Serializable
```

즉, Redis의 DefaultSerializer가 데이터를 저장하기 위해 `Serializable`을 요구하지만 현재 저장하려는 Map<K, V>의 값(V)으로 전달된 LevelTestQuizAnswerDTO 객체가 Serializable을 구현하지 않았기 때문에 `IllegalArgumentException` 예외가 발생하였다.

그리고 이로 인해 Redis Serializer가 Redis에 저장하는 과정에서 직렬화할 수 없어 `SerializationException`이 연쇄적으로 발생한 것이다.

그러면 예외가 터진 원인을 정확히 파악하기 전에 먼저 **직렬화**란 무엇이고 Redis에 데이터를 저장하는 데 **왜 직렬화가 필요한지**부터 파악해 보자.

## 직렬화

자바에서의 직렬화에 대한 정의는 우아한형제들 기술 블로그를 참고하였다.

https://techblog.woowahan.com/2550/

> `자바 직렬화`란 자바 시스템 내부에서 사용되는 객체 또는 데이터를 외부의 자바 시스템에서도 사용할 수 있도록 `바이트(byte)` 형태로 데이터 변환하는 기술과 바이트로 변환된 데이터를 다시 객체로 변환하는 기술(역직렬화)을 아울러서 이야기합니다.
> 

> 시스템적으로 이야기하자면 JVM(Java Virtual Machine 이하 JVM)의 메모리에 상주(힙 또는 스택)되어 있는 객체 데이터를 바이트 형태로 변환하는 기술과 직렬화된 바이트 형태의 데이터를 객체로 변환해서 JVM으로 상주시키는 형태를 같이 이야기합니다.
> 

즉, 직렬화는 메모리에 저장된 객체를 파일이나 네트워크를 통해 전송하기 위해 바이트 형태로 변환하는 것을 의미한다.

이 기술 블로그에 따르면 자바 직렬화의 조건은 `자바 기본(primitive)` 타입과 `java.io.Serializable` 인터페이스를 구현한 객체만 직렬화할 수 있다고 한다. 이를 통해 내가 구현한 DTO가 Serializable 인터페이스를 구현하지 않았기 때문에 예외가 발생한 것임을 파악할 수 있었다.

## Redis에 데이터를 저장할 때 직렬화가 필요한 이유

Redis에 데이터를 저장할 때 직렬화가 필요한 이유에 대해서는 spring-data-redis 공식 문서를 참고하였다.

> From the framework perspective, **the data stored in Redis is only bytes**. While Redis itself supports various types, for the most part, these refer to the way the data is stored rather than what it represents. It is up to the user to decide whether the information gets translated into strings or any other objects.
> 

이 내용을 해석하면 다음과 같다.

> 프레임워크 관점에서 보면, Redis에 저장되는 데이터는 단지 바이트(byte)일 뿐입니다. Redis 자체는 다양한 타입을 지원하지만, 이는 주로 데이터가 저장되는 방식일 뿐이며 그 데이터가 무엇을 의미하는지는 사용자가 결정해야 합니다. 즉, 정보를 문자열로 변환할지 또는 다른 객체로 변환할지는 사용자에게 달려 있습니다.
> 

따라서 Redis에 데이터를 저장하려면 메모리에 존재하는 객체를 바이트로 변환하는 직렬화 과정이 필수적으로 요구된다.

# 문제 원인

발생한 예외를 다시 살펴보자.

```java
DefaultSerializer requires a Serializable payload
```

이 메시지는 Redis의 기본 직렬화기(DefaultSerializer)가 직렬화할 수 있는 대상 객체를 저장할 때 반드시 객체가 Serializable 인터페이스를 구현해야 한다는 것을 명확히 나타낸다.

RedisCache와 RedisTemplate에서 기본적으로 사용하는 직렬화기는 `JdkSerializationRedisSerializer` 이다.

> JdkSerializationRedisSerializer, which is used by default for RedisCache and RedisTemplate.
> 
> 
> *(RedisCache와 RedisTemplate이 기본적으로 사용하는 JdkSerializationRedisSerializer이다.)*
> 

`JdkSerializationRedisSerializer`는 기본적으로 자바 직렬화(Java Serialization)를 사용하는 방식이다. 공식 문서를 통해 더 자세히 보면 다음과 같은 내용을 확인할 수 있다.

<img width="644" alt="image" src="https://github.com/user-attachments/assets/b4112150-92c0-4878-bca2-5506be833c18" />


앞서 살펴봤듯이, 자바의 기본 직렬화는 반드시 직렬화하려는 객체가 `java.io.Serializable` 인터페이스를 구현해야 한다는 조건이 존재한다. 따라서 JdkSerializationRedisSerializer를 사용하는 RedisTemplate을 통해 객체를 저장하려면 DTO와 같은 자바 객체가 해당 인터페이스를 반드시 구현하고 있어야만 한다.

그러나 현재 내 프로젝트에서 Redis에 저장하려고 했던 LevelTestQuizAnswerDTO 객체는 Serializable 인터페이스를 구현하지 않은 상태였고, 따라서 RedisTemplate이 이 객체를 직렬화할 수 없어 위와 같은 예외가 발생하게 된 것이다.

# 문제 해결

RedisTemplate의 기본 직렬화 방식인 `JdkSerializationRedisSerializer`는 저장하려는 객체가 반드시 `Serializable`을 구현해야 한다는 제약이 있다. 이 방식은 객체 구조가 변경되거나 Serializable 인터페이스를 구현하지 않은 객체를 저장할 때 제한적일 수 있다.

이러한 문제를 해결하기 위해 `GenericJackson2JsonRedisSerializer` 를 적용하였다. 이 Serializer는 Jackson 라이브러리를 활용하여 Java 객체를 JSON 형태로 직렬화하여 Redis에 저장한다.

기존 설정에서는 GenericJackson2JsonRedisSerializer가 RedisTemplate의 setValueSerializer에만 적용되어 있었다. 이번 수정에서는 추가적으로 setHashValueSerializer에도 이를 적용하여, Hash 자료구조의 값 또한 JSON 형태로 직렬화할 수 있도록 하였다.

### 적용 코드

```java
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(connectionFactory);

    GenericJackson2JsonRedisSerializer jsonSerializer = new GenericJackson2JsonRedisSerializer();

    template.setKeySerializer(new StringRedisSerializer());
    template.setValueSerializer(jsonSerializer);
    template.setHashValueSerializer(jsonSerializer);

    return template;
}

```

이 설정을 통해 Serializable 구현 여부와 상관없이 모든 객체를 Redis에 효율적이고 유연하게 저장할 수 있게 되었다.

# 참고

[Working with Objects through RedisTemplate :: Spring Data Redis](https://docs.spring.io/spring-data/redis/reference/redis/template.html?utm_source=chatgpt.com)

[[Spring] 스프링이 제공하는 레디스 직렬화/역직렬화(Redis Serializer/Deserializer)의 종류와 한계](https://mangkyu.tistory.com/402)

[[Cache] Redis 직렬화 방법에 대해서](https://velog.io/@choidongkuen/%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4-Redis-%EC%A7%81%EB%A0%AC%ED%99%94-%EB%B0%A9%EB%B2%95%EC%97%90-%EB%8C%80%ED%95%B4%EC%84%9C)
