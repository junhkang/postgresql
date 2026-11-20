# Index-Only 스캔과 Covering 인덱스, Index-only스캔의 효율적인 사용

> 전체 글은 블로그에서 계속 업데이트한다 → **[https://junhkang.com/posts/70/](https://junhkang.com/posts/70/)**

Index-Only 스캔은 heap 대신 인덱스만으로 결과를 반환하지만, visibility map에서 확인되지 않은 페이지는 heap을 읽어야 한다. INCLUDE로 만드는 커버링 인덱스와 일반 복합 인덱스의 차이, 성능 테스트 결과를 정리한다.

## 다루는 내용
- 1. Index-Only Scans
- 2. Visiblity
- 3. Covering index
- 4. Covering Index VS 일반 Index
- 5. 표현식(expresions)과 Index-only 스캔
- 6. 부분 인덱스 (Partial Index)와 Index-only 스캔
- 7. 결론 및 성능 테스트

전체 본문, 예제 SQL, 그림은 [블로그 원문](https://junhkang.com/posts/70/)에 있다. 이 저장소의 이전 버전 본문은 git 히스토리에 남아 있다.
