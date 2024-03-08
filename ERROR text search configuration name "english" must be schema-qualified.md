## 1\. 발생

해당 에러는 Postgresql에서 Full Text Search를 위해 tsvector 컬럼을 업데이트할 때 발생한다.

```
-- 특정 컬럼을 ts_vector로 변경하여 업데이트
UPDATE
	TABLE
SET
	tsvec_words = to_tsvector('english',COLUMN);
```

## 2\. 원인

해당 컬럼 (혹은 다른 컬럼) 에 테이블 row 업데이트/인서트 시 ts\_vector를 자동으로 업데이트하는 trigger가 걸려 있기 때문에 업데이트 간 충돌이 생겨 발생한다.

## 3\. 해결

트러거를 삭제 후 데이터 업데이트 후에 트리거를 재설정하면 해결된다.

####        3-1. 트리거 삭제

```
drop trigger TABLE_TRGGER on TABLE;
```

####        3-2. 트리거 생성

```
CREATE TRIGGER
  TABLE_TRIGGER
BEFORE INSERT OR UPDATE ON
  TABLE
FOR EACH ROW EXECUTE PROCEDURE
  tsvector_update_trigger(tsvec_words, 'english',COLUMN);
```