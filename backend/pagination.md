# Singit 스와이프 페이징 전략

## 문제 정의

- Singit 숏폼은 스와이프 기반 영상 소비 UX를 제공하며, 사용자가 중간 콘텐츠를 클릭한 뒤 이어서 `next`/`prev` 스와이프 시 영상 순서 일관성이 필수.
- 쇼트폼 메인 피드에서 사용자 행동(클릭+스와이프)을 고려하면, 단순 `OFFSET` 쿼리로는 비일관성이 발생.
- 정렬 기준에 따라 데이터 특성이 다름
  - `latest`(최신) : `si_sing`에 실시간으로 데이터가 추가됨. 순서 기반 탐색 시 포인트 값이 계속 증가.
  - `views`, `likes`, `recommend`(인기/추천) : 상대적으로 안정적이며 빠르게 변하지 않음.

## 페이징 방식 비교 (Offset vs Cursor)

| 방식 | 장점 | 단점 | Singit 적용 가능성 |
|---|---|---|---|
| `OFFSET` | 구현 단순, 기존 요청자 위치정보(`page`) 없음 | 최신 추가시 커서 이슈(새 데이터 삽입으로 동일 OFFSET 다른 결과), 큰 OFFSET 성능저하 | `recommend` 피드의 고정급 인기 순에서 제한적 사용
| `CURSOR` | 안정적 순서 유지(주어진 시작점 기준), 중복/누락 방지, 대규모 스케일 | 복잡한 비교조건(복합정렬시), start cursor 정보 유지 필요 | `latest` + 변화 많고 순차적으로 소비되는 경우 우선 적용

### 실제 문서화된 코드 흐름
`model/singit/swipe.py`:
- `feed_type=="recommend" && sort=="recommend"` → `_get_swipe_feed_offset()` (OFFSET 기반)
- `feed_type=="recommend" && sort!="recommend"` → `_get_swipe_feed_cursor()` (커서 기반)
- `feed_type=="following"`, `"my"`, `"bookmark"`, `"same_song"` 등은 커서 기반

```python
if feed_type == "recommend" and sort == "recommend":
    return await _get_swipe_feed_offset(...)
else:
    return await _get_swipe_feed_cursor(...)
```

## 해결 방법 (정렬별 전략 분리)

1. 추천(추천순) / 인기정렬: `OFFSET`
   - `feed_type="recommend"` + `sort="recommend"` 고정된 리스트에서 `offset`으로 `start_sing_id`를 기준으로 위치 계산.
   - 이미 존재하는 순서를 최대한 유지하게 함.
   - 트래픽/배치 주기에서 재정렬 빈도 낮음을 활용.

2. 최신 / 변화 가능한 정렬: `CURSOR`
   - `sort=latest`, `views`, `likes`, `date` 등에서 `start_sing_id` 기준 커서 조건으로 `<=`, `>=`, `timestamp` 비교.
   - `direction`(next/prev) 쌍방향 지원.
   - `order by` 복합키로 순서 확정.

3. 특수 피드별 규칙 (스와이프 중복/누락 축소)
   - `following`, `my`, `profile`, `bookmark`, `same_song` 모두 cursor 방식.
   - Cursor 조건은 `score`, `best_sing`, `sing_id` 같이 복합키로 구현.

## 설계 판단 근거

- **중요한 문제**: 사용자 클릭 후 스와이프 경험에서 리스트의 일관성(flicking gap, 중복)이 직접 UX 품질에 영향을 줌.
- **데이터 형태**
  - `latest`: 증분 데이터, 시퀀스가 계속 바뀜 -> offset으로 재계산 시 위치 왜곡 발생
  - `recommend` 인기 데이터: 전체순위가 비교적 고정, 상위집합 선호 -> Offset으로도 실무 가능
- **성능**
  - offset 큰값 시 DB 부하. 추천순 정렬에서 `limit <= 20`로 제한하며 의미있는 값만 추출.
  - cursor는 각 스타트행의 복합 비교로 index 활용 가능.
- **개발 복잡도**
  - 코드에서는 페이드 타입/정렬별 분기 처리로 유연성 확보. 
  - 커서 논리 복잡 하지만 일관성 보장에 가치 존재.

## 결과

- `swipe feed`에서 중간 클릭 후 스와이프 시 데이터 커서/오프셋 혼용으로 중복/결손 최소화.
- `latest` 기반에 대한 안정적 방향 탐색 성공(다이나믹하게 새 콘텐츠 유입해도 cursor 기반 순서 유지).
- `recommend` 고정순에서 빠른 랜딩과 일관성 유지 (offset 기반) - 사용자에게 인기피드 기준 고정성 체감.
- 컨트롤러 API는 `GET /singit/swipe/feed/{feed_type}/from-position` 형태로 동일하게 유지하며, 내부에서 `feed_type`+`sort` 조합으로 분기.

### 코드 예시 (cursor 조건 개념)
```sql
-- 예: 추천순에서 커서 조건
SELECT *
FROM si_sing ss
WHERE mr_id = :mr_id
  AND (
    ss.best_sing < :start_best
    OR (ss.best_sing = :start_best AND ss.score < :start_score)
    OR (ss.best_sing = :start_best AND ss.score = :start_score AND ss.sing_id < :start_sing_id)
  )
ORDER BY ss.best_sing DESC, ss.score DESC, ss.sing_id DESC
LIMIT :limit
```

### 트레이드오프 (요약)
- OFFSET
  - 장점: 구현 간단, `recommend`에 적합
  - 단점: 변동 데이터에서 일관성 저하, 대규모 offset 비용
- CURSOR
  - 장점: 정확한 위치복원, 페이징 사이 일관성
  - 단점: 비교 로직 복잡, 상태(커서값) 유지 필요
