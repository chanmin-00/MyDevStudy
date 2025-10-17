# 문제 상황

리플 프로젝트에서 특정 키워드를 **포함**하는 용어(title)를 검색하는 기능을 구현하는  과정에서, **LIKE %키워드%** 조건으로 조회할 때 인덱스를 타지 않는 현상을 확인했다.

## = 비교 연산

3000건 정도의 데이터가 데이터베이스에 있을 때, 우선 **LIKE 조건**을 사용하지 않고 **= 비교 연산**을 적용했을 때 인덱스가 작용되는지 여부를 확인해보자.

### 인덱스 미사용

```sql
DROP INDEX idx_title ON terms;

SET profiling = 1;

SELECT * FROM terms WHERE title = '유니콘기업';

SHOW PROFILES;
```

<img width="654" alt="image" src="https://github.com/user-attachments/assets/b75b3f34-01de-4195-80c1-b5336521c061" />


- `쿼리 실행 시간` : 0.004482

### 인덱스 사용

```sql
CREATE INDEX idx_title ON terms(title);

SET profiling = 1;

SELECT * FROM terms WHERE title = '유니콘기업';

SHOW PROFILES;
```

<img width="650" alt="image" src="https://github.com/user-attachments/assets/a16c6cc5-0676-4be1-b847-e8720c25769b" />


- `쿼리 실행 시간` : 0.000868

쿼리 실행 시간이 인덱스 적용으로 인해 **`0.004482초` → `0.000868초`** 로 대폭 감소했음을 확인할 수 있다. 이를 통해 인덱스가 적용되어 조회 성능이 향상되었음을 알 수 있다.

## LIKE 비교

이번에는 ‘유니콘’이라는 단어를 **포함**하고 있는지 여부를 LIKE 조건을 사용하여 쿼리를 실행해보았다.

### 인덱스 미사용

```sql
DROP INDEX idx_title ON terms;

SET profiling = 1;

SELECT * FROM terms WHERE title LIKE '%유니콘%';

SHOW PROFILES;
```

<img width="650" alt="image" src="https://github.com/user-attachments/assets/4c33d44f-8bd4-4e42-bfce-a2aef41cbe42" />


- `쿼리 실행 시간` : 0.00768

### 인덱스 사용

```sql
CREATE INDEX idx_title ON terms(title);

SET profiling = 1;

SELECT * FROM terms WHERE title LIKE '%유니콘%';

SHOW PROFILES;
```

<img width="648" alt="image" src="https://github.com/user-attachments/assets/b7adb36b-e34c-40ff-9b6a-d75b2ac3ddb3" />


- `쿼리 실행 시간` : 0.00649

쿼리 실행 시간에 큰 차이가 없음을 확인할 수 있다. 실제로 인덱스가 적용되지 않았음을 의심할 수 있으며 `EXPLAIN` 키워드를 사용하여 인덱스 사용 여부를 확인해볼 수 있다.

## EXPLAIN 키워드로 인덱스 사용 여부 확인

```sql
EXPLAIN SELECT * FROM terms WHERE title = '유니콘기업';
```

<img width="272" alt="image" src="https://github.com/user-attachments/assets/5f075ac9-6c1a-4127-b82a-fe517a1b77f2" />


```sql
EXPLAIN SELECT * FROM terms WHERE title LIKE '%유니콘%';
```

<img width="267" alt="image" src="https://github.com/user-attachments/assets/db13a5fc-5d2c-491e-9c45-443f2b5a7765" />

`possible_keys`는 **사용 가능한 인덱스 목록**을 의미한다. `=` 비교 연산에는 인덱스가 존재하지만, `LIKE` 비교 연산에는 인덱스가 존재하지 않음을 확인할 수 있다.

## 문제 발생

내가 스프링에서 용어 검색을 위해 작성한 실제 코드를 살펴보면 `LIKE` 쿼리가 발생하고 있음을 확인할 수 있다.

```java
@Override
    public Page<TermWithScrapDTO> findByKeyword(String keyword, Pageable pageable, long userId) {
        BooleanExpression predicate = null;

        if (keyword != null && !keyword.trim().isEmpty()) {
            predicate = termJpaEntity.title.containsIgnoreCase(keyword);
        }

        List<TermWithScrapDTO> termList = getTermsWithScrapByPageable(pageable, predicate, userId);

        JPAQuery<Long> countQuery =
                queryFactory.select(termJpaEntity.count()).from(termJpaEntity).where(predicate);

        return PageableExecutionUtils.getPage(termList, pageable, countQuery::fetchOne);
    }
```

이 코드를 실행했을 때 실제 발생하는 쿼리는 다음과 같다.

```java
Hibernate: select tje1_0.id,tje1_0.title,tje1_0.description,tje1_0.initial,tsje1_0.id is not null from terms tje1_0 left join term_scraps tsje1_0 on tje1_0.id=tsje1_0.term_id and tsje1_0.user_id=? where lower(tje1_0.title) like ? escape '!' limit ?,?
```

- 쿼리에서 `LIKE` 연산이 발생하고 있는 것을 확인할 수 있다.

즉, 나는 현재 Term 도메인에서 `title` 컬럼에 대해 인덱스를 생성하고 있지만, 실제 키워드 검색은 **포함 조건**을 기반으로 수행되고 있기 때문에 인덱스가 무용지물인 상황이다.

# 문제 원인

우선 내가 사용하는 MySQL에서 어떤 인덱스 종류를 사용하고 있는지 확인해보자.

```sql
SHOW INDEX FROM movies;
```

<img width="400" alt="image" src="https://github.com/user-attachments/assets/8470f078-c021-4655-ba12-5df0d774a616" />


- 인덱스의 종류는 **B-TREE**임을 확인할 수 있다.

> 그렇다면 B-TREE 인덱스는 왜 `LIKE '%키워드%'` 조건에서 인덱스를 사용할 수 없는 걸까?
> 

## B-Tree 탐색 방식

[MySQL :: MySQL 8.0 Reference Manual :: 10.3.9 Comparison of B-Tree and Hash Indexes](https://dev.mysql.com/doc/refman/8.0/en/index-btree-hash.html)

공식 문서에 따르면 다음과 같은 내용을 확인할 수 있다.

> A B-tree index can be used for column comparisons in expressions that use the =, >, >=, <, <=, or BETWEEN operators.
> 
> 
> The index also can be used for LIKE comparisons if the argument to LIKE is a constant string that does not start with a wildcard character.
> 
> The following SELECT statements do not use indexes:
> 
> - SELECT * FROM tbl_name WHERE key_col LIKE '%Patrick%';
> - SELECT * FROM tbl_name WHERE key_col LIKE other_col;

> The following SELECT statements do not use indexes:
> 
> - SELECT * FROM *tbl_name* WHERE *key_col* LIKE '%Patrick%';
> - SELECT * FROM *tbl_name* WHERE *key_col* LIKE *other_col*;
> 
> In the first statement, the LIKE value begins with a wildcard character. In the second statement, the LIKE value is not a constant.
> 

요약하자면:

- `LIKE '유니콘%'`처럼 **접두사가 고정된 패턴**은 B-TREE 인덱스를 사용할 수 있다.
- 반면 `LIKE '%유니콘%'`처럼 **앞부분이 와일드카드(%)로 시작하는 경우** 문자열의 시작이 불확실하므로 인덱스를 사용할 수 없다.
- 또한 `LIKE other_col`처럼 비교 대상이 상수가 아닌 경우에도 인덱스를 사용할 수 없다.

이러한 제한은 B-TREE 인덱스의 **구조적 특성** 때문이다. 이 특성을 이해하기 위해 B-Tree의 탐색 구조를 좀 더 자세히 살펴보자.

## B-Tree 탐색 구조와 범위 검색

<img width="644" alt="image" src="https://github.com/user-attachments/assets/d777a298-d95e-47ad-bfff-ebde45cf49a2" />

B-Tree는 기본적으로 **정렬된 키 순서로 저장**하는 균형 트리구조이다. 각 노드는 여러 키 값을 포함하고 있으며 이 키 값들은 **사전 순서**에 따라 정렬되어 있다. 

이 구조 덕분에 B-Tree는 `WHERE column >= ? AND column < ?` 같은 **범위 검색**에 매우 효과적이다. 

MySQL 공식 문서에서도 다음과 같은 범위 탐색 방식의 예시를 제시하고 있다:

> In the first statement (LIKE 'Patrick%'), only rows with 'Patrick' <= key_col < 'Patricl' are considered.
> 

즉, `'유니콘%'`이라는 조건을 검색할 경우, `'유니콘' <= title < '유니콘힣'` 범위를 가진 **정렬된 키 집합에서 빠르게 탐색**할 수 있다.

- 예시로 '유니콘가', 유니콘나', '유니콘다' 와 같은 값들이 연속적으로 정렬되어 있다고 가정하면, 트리 구조를 따라 **해당 범위 내에서만** 빠르게 값을 찾을 수 있다.

## 와일드카드(%)가 앞에 붙는 경우

반면, `LIKE '%유니콘%'`처럼 **앞부분에 와일드카드가 포함된 조건**에서는 상황이 다르다.

검색 대상 문자열이 '가유니콘', '나유니콘', '다유니콘'처럼 문자열의 중간 또는 끝에 키워드를 포함하고 있다면, B-Tree 인덱스는 접두사가 주어지지 않았기 때문에, 탐색의 시작점을 결정할 수 없고, 인덱스 탐색이 아닌 전체 테이블 스캔(Full Table Scan) 이 발생하게 된다.

이는 B-Tree 인덱스의 탐색 방식이 문자열의 왼쪽부터 일치하는 값으로 분기해나가는 구조이기 때문이다. 즉, 인덱스를 활용하려면 앞부분부터 고정된 조건이 필요하다.

# 문제 해결

기존 인덱스 방식이 적용되지 않는 문제를 해결하기 위해 다른 검색 성능 향상 방식을 적용시켜 보자.

검색 성능을 향상시키기 위해서는 `Full Text Search`, 혹은 `검색엔진(예: Elasticsearch)` 을 활용하는 방법이 있다. 

Elasticsearch는 다양한 형태소 분석과 고급 검색 기능을 제공하지만, 추가적인 인프라 구축 및 운영 비용이 발생하므로 현재 데이터 규모(수천 건 수준) 를 고려할 때는 MySQL에서 기본으로 제공하는 Full Text Search 기능을 우선 적용해보기로 한다.

## Full Text Search

Full Text Search(전문 검색)는 MySQL이 제공하는 텍스트 검색 기능이다. 단어 단위로 텍스트를 인덱싱해 훨씬 빠르게 자연어 기반 검색이 가능하도록 도와준다.

[MySQL :: MySQL 8.0 Reference Manual :: 14.9 Full-Text Search Functions](https://dev.mysql.com/doc/refman/8.0/en/fulltext-search.html)

> A full-text index in MySQL is an index of type FULLTEXT.
> 
> 
> Full-text indexes can be used only with InnoDB or MyISAM tables, and can be created only for CHAR, VARCHAR, or TEXT columns.
> 
> A FULLTEXT index definition can be given in the CREATE TABLE statement when a table is created, or added later using ALTER TABLE or CREATE INDEX.
> 

> Full-text searching is performed using MATCH() and AGAINST() syntax. MATCH() takes a comma-separated list of columns to be searched. AGAINST() takes the search string and an optional search modifier.
> 

요약하면 

- FULLTEXT 인덱스는 CHAR, VARCHAR, TEXT 컬럼에서만 사용 가능하다.
- InnoDB 또는 MyISAM 스토리지 엔진에서만 지원된다.
- 전문 검색은 MATCH(컬럼명)과 AGAINST('검색어' [검색 모드]) 구문으로 수행된다.

## Full Text Search 적용 방법

1. **Full Text Index 생성**
    
    ```sql
    ALTER TABLE terms ADD FULLTEXT INDEX idx_title (title);
    ```
    
2. **검색 쿼리 실행**
    
    ```sql
    SELECT * FROM terms
    WHERE MATCH(title) AGAINST('유니콘' IN NATURAL LANGUAGE MODE);
    ```
    

## Full Text Search 성능 확인

```sql
ALTER TABLE terms ADD FULLTEXT INDEX idx_title (title);

SET profiling = 1;

SELECT * FROM terms WHERE MATCH(title) AGAINST('유니콘' IN NATURAL LANGUAGE MODE);

SHOW PROFILES;
```

<img width="648" alt="image" src="https://github.com/user-attachments/assets/27e2c9cb-9544-4049-bc6d-afbd2efb6d62" />


- `쿼리 실행 시간` : 0.000439

기존의 LIKE 방식보다 쿼리 실행 시간이 대폭 감소했음을 확인할 수 있고, idx_title 인덱스를 활용하여 쿼리를 실행했음을 확인할 수 있다.

### cf) 2글자 검색 오류

문제는 2글자가 검색이 안된다는 것이다. 예를 들어 환율 같이 2글자인 단어는 검색되지 않는다. 이 문제는 MySQL에서 인덱스를 생성할 때 최소 토큰 길이(min token size)에 제한이 있기 때문이다.

MySQL의 기본 설정은 다음과 같다

- InnoDB: innodb_ft_min_token_size = 3 (3글자 이상만 인덱싱됨)
- MyISAM: ft_min_word_len = 4

이를 해결하기 위해서 로컬 개발 환경과 RDS 환경에 다음과 같은 설정을 해주어야 한다.

1. **로컬 개발 환경**
    - MySQL 설정 파일인 my.conf 파일에 접속해 최소 토큰 크기를 수정한다.
    - 설정 예시(/opt/homebrew/etc/my.cnf)
        
        ```bash
        ➜  etc git:(stable) vi my.cnf 
        
        # Default Homebrew MySQL server config
        [mysqld]
        # Only allow connections from localhost
        bind-address = 127.0.0.1
        mysqlx-bind-address = 127.0.0.1
        innodb_ft_min_token_size = 2 # 위 내용 추가을 추가한다.
        ```
        
    - 설정 후 MySQL 서버를 재시작 한 후, `SHOW VARIABLES LIKE 'innodb_ft_min_token_size';` 명령을 통해 최소 토큰 크기 적용을 확인할 수 있다.
        
        <img width="597" alt="image" src="https://github.com/user-attachments/assets/e5aa3f80-792d-4099-9bee-931488149c37" />

        
2. **RDS**
    - RDS에서는 직접 my.cnf 파일을 수정할 수 없기 때문에 파라미터 그룹을 통한 변경이 필요하다.
        - RDS 콘솔 → 파라미터 그룹
        - `default.mysql8.0` 그룹은 수정 불가 → 새로운 파라미터 그룹 생성
        - 해당 그룹에서 `innodb_ft_min_token_size`와 `ft_min_word_len` 값을 2로 변경
        - DB 인스턴스 → 수정 → 파라미터 그룹 교체
        - 변경 후에는 인스턴스를 재시작

이렇게 설정하고 나면 2글자 검색도 잘 진행되는 것을 확인할 수 있다.

## 스프링 적용 방식

MySQL에서 `FULLTEXT` 인덱스를 생성했다면 이제 이를 Spring 애플리케이션에서 활용해보자. JDBC를 사용하여 `MATCH ... AGAINST` 구문을 직접 실행함으로써 전문 검색 기능을 사용할 수 있다.

아래는 title, description 컬럼을 대상으로 전문 검색을 수행하고, 해당 용어를 사용자가 스크랩했는지 여부까지 함께 조회하는 코드이다

```java
@Transactional(readOnly = true)
public Page<TermWithScrapDTO> searchTerms(String keyword, Pageable pageable, long userId) {
    if (keyword == null || keyword.trim().isEmpty()) {
        return Page.empty(pageable);
    }

    String searchSql = """
        SELECT t.id, t.title, t.description, t.initial,
               IF(ts.id IS NOT NULL, true, false) AS is_scrapped
        FROM terms t
        LEFT JOIN term_scraps ts ON t.id = ts.term_id AND ts.user_id = ?
        WHERE MATCH(t.title, t.description) AGAINST (? IN NATURAL LANGUAGE MODE)
        ORDER BY t.id DESC
        LIMIT ? OFFSET ?
        """;

    String countSql = """
        SELECT COUNT(*)
        FROM terms t
        WHERE MATCH(t.title, t.description) AGAINST (? IN NATURAL LANGUAGE MODE)
        """;

    return getTermWithScrapDTOS(keyword, pageable, userId, searchSql, countSql);
}

private Page<TermWithScrapDTO> getTermWithScrapDTOS(String keyword, Pageable pageable, long userId,
                                                    String searchSql, String countSql) {
    List<TermWithScrapDTO> content = jdbcTemplate.query(
        searchSql,
        (rs, rowNum) -> new TermWithScrapDTO(
            rs.getLong("id"),
            rs.getString("title"),
            rs.getString("description"),
            rs.getString("initial"),
            rs.getBoolean("is_scrapped")
        ),
        userId, keyword, pageable.getPageSize(), pageable.getOffset()
    );

    Integer total = jdbcTemplate.queryForObject(countSql, Integer.class, keyword);
    return PageableExecutionUtils.getPage(content, pageable, () -> total != null ? total : 0);
}

```

jdbc를 통해서 Full Text Search Query를 수행할 수 있도록 하였다.

### 인덱스 전제 조건

위 코드에서 `MATCH(t.title, t.description)` 구문이 사용되므로, MySQL에서는 다음과 같은 복합 `FULLTEXT` 인덱스를 사전에 생성해두어야 한다

```sql
ALTER TABLE terms ADD FULLTEXT INDEX idx_title_description (title, description);
```

또한 단일 컬럼 검색도 고려할 경우에는 아래처럼 단일 컬럼 인덱스도 함께 생성해두는 것이 좋다

```sql
ALTER TABLE terms ADD FULLTEXT INDEX idx_title (title);
```

이러한 인덱스는 스키마 초기화 시점(`schema.sql`)에 추가하거나 JDBC를 활용하여 애플리케이션 시작 시 동적으로 존재 여부를 확인한 후 생성할 수도 있다.

# 결론

MySQL의 Full Text Search는 B-Tree 인덱스로는 불가능한 유연한 자연어 검색을 가능하게 해준다. 특히 `LIKE '%검색어%'` 구문으로 인해 성능이 저하되던 문제를 해결할 수 있으며 검색어 길이에 따른 인덱싱 제약만 해결하면 2글자 단어 검색도 문제 없이 수행할 수 있다.

# 참고

[인덱스 - B-Tree](https://velog.io/@rg970604/%EC%9D%B8%EB%8D%B1%EC%8A%A4-B-Tree)

[👉 [DB] B+Tree 인덱스는 왜 LIKE 연산시 Full Scan을 할까?](https://bitcodic.tistory.com/261)
