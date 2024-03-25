## 1\. 날짜 형태로 형변환

데이터 베이스에서 날짜형태로 형변환을 하는 것은 다음과 같은 방법으로 쉽게 가능하다.

```
-- Unix타임(int)형 변환
SELECT to_timestamp(1658792421)

-- varchar 타입 변환
SELECT to_timestamp('20231026','yyyymmdd')

-- 날짜형을 char로 변환
SELECT to_char(to_timestamp(1658792421), 'DD-MM-YYYY')
```

2\. 유효한 날짜형태 검증

데이터 정제가 완료되지 않아 조회하려는 데이터에 날짜유형에서 벗어난 데이터 ('20231301',202301', '20231232' 등)가 하나라도 있을 경우 조회 자체가 안된다. 그럴 경우 날짜 규격에 맞지 않는 데이터를 보정 후 연산해야 하는 경우가 있는데 단순 월별 케이스문으로 분리하여 날짜 유형에 어긋나는 경우를 찾을 수도 있지만 row마다 날짜 유형이 다르거나 윤달을 체크할 수 없다.

그래서 날짜 형태자체를 변환시도하고 성공 여부에 따라 결과값을 추출하는 함수를 생성해야 한다.

```
CREATE OR REPLACE FUNCTION VALIDATE_DATE(S VARCHAR) RETURNS INT AS
$$
BEGIN
    IF COALESCE(S, '-') = '-' THEN
        RETURN -1;
    END IF;
    PERFORM S::DATE;
    RETURN 0;
EXCEPTION
    WHEN OTHERS THEN
        RETURN 1;
END;
$$ LANGUAGE PLPGSQL;
```

이제 VALIDATE\_DATE() 함수를 실행시키면, 날짜유형, 윤달에 상관없이 해당 데이터가 유효한 날짜라면 0, 유효하지 않은 데이터라면 1, null이라면 -1을 리턴하게 된다.

다음과 같이 유효한 날짜 유형의 데이터만 조회하거나

```
SELECT date_column from table
WHERE VALIDATE_DATE(date_column) = 0;
```

날짜 유형에 어긋나는 데이터들을 일괄 null 업데이트하여 처리할 수 있다.

```
UPDATE table SET date_column = NULL
WHERE VALIDATE_DATE(date_column) = 1
```