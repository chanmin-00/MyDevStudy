# 문제 상황

[MySQL Full Text Search를 활용한 검색 성능 개선](https://chanmin-tstory.tistory.com/entry/MySQL-Full-Text-Search%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%9C-%EA%B2%80%EC%83%89-%EC%84%B1%EB%8A%A5-%EA%B0%9C%EC%84%A0)

지난번에 Full Text Index를 활용하여 검색 성능을 향상시켰다. 그런데 문제 상황을 맞이했다. 내가 원했던 결과들이 반환되지 않는 것이다.

예를 들어 다음과 같은 Term 테이블의 `description` 필드에 아래와 같은 텍스트가 있다고 하자.

```
싱글족 가운데 두 곳 이상에 거처를 두거나 잦은 여행과 출장 등으로 오랫동안 집을 비우는 사람들을 일컫는 말. 0.5인 가구는 1인 가구보다 집에 머무는 시간이 훨씬 더 짧다. 평소에는 직장 근처에 방을 얻어 혼자 살지만 주말에는 가족들의 거처로 찾아가 함께 시간을 보내는 경우도 여기에 속한다.
```

여기서 설명에 ‘싱글’이 포함된 단어를 조회하고자 쿼리를 날렸지만, 예상과 달리 결과값이 제대로 반환되지 않았다. 실제로 이러한 문제가 발생하고 있는지 정량적인 수치로 확인해보자.

### LIKE 비교 연산을 활용하여 검색하기

```sql
SELECT COUNT(*) FROM terms WHERE description LIKE '%싱글%';
```

- `반환 결과 행 수` : 4

![image](https://github.com/user-attachments/assets/1a3dd961-ec6c-44bd-a1e8-d520431e8a98)


### Full Text Index를 활용하여 검색하기

```sql
-- description 전용 FULLTEXT 인덱스 생성
ALTER TABLE terms ADD FULLTEXT INDEX idx_description (description);

SELECT COUNT(*) FROM terms WHERE MATCH(description) AGAINST('싱글' IN NATURAL LANGUAGE MODE);
```

- `반환 결과 행 수` : 2

![image](https://github.com/user-attachments/assets/683d7493-b7c7-4d60-91a3-54340ea40e3b)


LIKE 연산자는 조건에 부합하는 모든 결과를 반환하므로, 이를 기준으로 반환된 행의 수를 확인하면, Full Text Index를 활용한 검색의 결과 행 수가 더 적다는 점을 통해 원하는 결과값이 다 반환되지 않았음을 확인할 수 있다.

# 문제 원인

결과가 적게 반환이된 원인을 파악해보자.

이는 검색하고자 했던 단어가 **Full Text Index에 인덱싱되지 않았기 때문**인걸로 파악할 수 있다. 그렇다면 **어떤 기준으로 인덱스가 생성되며**, 왜 ‘싱글’이라는 단어는 인덱스에 포함되지 않았을까?

## Full Text Index 생성 방식

Full Text Index는 주로 두 가지 방식으로 생성된다. **`Stop-Word Parser` 방식**과 **`N-gram Parser` 방식**이다.

### Stop-Word Parser

공백, 탭, 문장 부호 또는 사용자가 정의한 문자열을 기준으로 텍스트를 분리한 후, 설정된 `innodb_ft_min_token_size` 값 이상인 토큰만 인덱싱하는 방식이다.

예를 들어 다음과 같은 문장을 Stop-Word Parser 방식으로 처리한다고 가정하자.

1. **공백과 구두점을 기준으로 문장 분리**

```bash
["싱글족", "가운데", "두", "곳", "이상에", "거처를", "두거나", "잦은", "여행과", "출장", "등으로", "오랫동안", "집을", "비우는", "사람들을", "일컫는", "말"]
```

1. **`innodb_ft_min_token_size = 2` 에 따라 2글자 이상 토큰만 인덱싱**

```bash
["싱글족", "가운데", "이상에", "거처를", "두거나", ...]
```

이처럼 인덱싱된 결과에는 '싱글'이 포함되지 않고 '싱글족'만 포함되어 있기 때문에, **'싱글'이라는 단어로 검색했을 때 결과가 적게 나오는 현상**이 발생한다.

### N-gram Parser

이 문제를 해결하기 위해 **N-gram Parser**를 사용할 수 있다. 이 방식은 문자열을 문자 단위로 1자씩 밀면서 n글자 단위로 토큰을 생성하여 인덱싱한다.

1. **문자 단위로 2글자씩 토큰화 (n=2)**

```bash
['싱글', '글족', '족 ', ' 가', '가운', '운데', '데 ', ' 두', ...]
```

1. **길이 2 이상인 유의미한 토큰만 추출**

```bash
['싱글', '글족', '가운', '운데', '이상', '상에', '거처', '처를', '두거', '거나', '잦은', ...]
```

이처럼 **‘싱글’이라는 단어 자체가 토큰으로 생성되어 인덱싱**되기 때문에, 해당 단어로 검색 시에도 정확히 매칭된 결과를 얻을 수 있다.

---

# 문제 해결

이 문제를 해결하기 위해 기존의 Stop-word 기반 Full Text Index를 제거하고, `ngram parser`를 사용하는 방식으로 테이블 정의를 변경했다:

```sql
CREATE TABLE terms (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    created_date DATETIME(6) NOT NULL,
    modified_date DATETIME(6) NOT NULL,
    description TEXT NOT NULL,
    initial VARCHAR(255) NOT NULL,
    title VARCHAR(255) NOT NULL,

    -- ngram 기반 FULLTEXT 인덱스
    FULLTEXT INDEX idx_description (description) WITH PARSER ngram,
    FULLTEXT INDEX idx_title (title) WITH PARSER ngram,
    FULLTEXT INDEX idx_title_description (title, description) WITH PARSER ngram,

    -- 일반 B-Tree 인덱스
    INDEX idx_initial (initial)
) ENGINE=InnoDB;
```

> 주의할 점:
> 
> 1. **`WITH PARSER ngram`** 옵션을 반드시 명시해야 한다.
> 2. 기존 인덱스가 있다면, DROP 후 ADD로 재생성해야 적용된다.
> 3. `ngram_token_size` 기본값은 2이며, 2글자 단위로 토큰이 생성된다.

해당 설정값은 다음 명령어로 확인할 수 있다:

```sql
SHOW VARIABLES LIKE 'ngram_token_size';
```

필요 시 `my.cnf`에서 값을 수정하고, 서버를 재시작해야 변경이 적용된다.

## 성능 비교

실제 성능 차이를 확인하기 위해 `profiling` 기능을 사용하여 두 방식의 쿼리 실행 시간을 비교했다.

### ngram 기반 Full Text Search

```sql
SET profiling = 1;

SELECT * FROM terms WHERE MATCH(description) AGAINST('싱글' IN NATURAL LANGUAGE MODE);

SHOW PROFILES;
```

![image](https://github.com/user-attachments/assets/30871e52-3103-4b1f-a695-7d1e0313d607)


### LIKE 연산자 사용

```sql
SET profiling = 1;

SELECT * FROM terms WHERE description LIKE '%싱글%';

SHOW PROFILES;

```

![image](https://github.com/user-attachments/assets/7f503116-1a05-4068-978d-46f0653f6db8)


`COUNT(*)`로 확인한 결과 두 방식 모두 동일한 행 수를 반환했지만, **ngram 기반 Full Text Search는 훨씬 더 빠르게 처리**되었다.

> ✅ 주의
> 
> 
> ngram 방식은 문자열 단위로 인덱싱되기 때문에, 의미 단위 매칭이 아닌 **단순 문자 일치로 더 많은 결과가 나올 수 있다.** 검색 정밀도가 필요한 경우 필터링 로직을 추가해야 한다.
> 

# 참고

https://blog.harampark.com/blog/mysql-full-text-index/
