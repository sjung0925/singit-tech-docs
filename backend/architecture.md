# Backend Architecture: Slim/Laravel → FastAPI 통합

## 1. 기존 구조 문제
- Slim + Laravel 분산 구조
- API 중복 (인증/사용자/미디어 등) 및 비즈니스 로직 중복
- 공통 모듈 공유 어려움, 버전 관리 분산
- 설정, 미들웨어, 에러 핸들링이 프레임워크마다 달라 테스트/배포 복잡
- 리소스 중복(개별 컨테이너/인스턴스), 운영 비용 상승

## 2. 개선 방향
- FastAPI 기반 단일 서버로 통합
- 모듈 기반 구조 (도메인별 분리)
- 공통 미들웨어 일원화: 인증, 로깅, 예외, CORS, Throttling
- 레이어 분리: Controller → Service/Model → Repository
- CI/CD 파이프라인 단순화

## 3. 아키텍처 구조 설명
- 진입점: `app.py`, `main.py` (uvicorn)
- 라우터: `controller/` (APIRouter)
- 도메인: `model/singit/`, `model/search/` 등
- 설정: `config/{local,product}.py`
- 공통: `class/utils.py`

```
app.py
main.py
controller/
  singit/swipe.py
  singit/auth_debug.py
model/
  singit/swipe.py
  ...
config/
  local.py
  product.py
class/
  utils.py
```

### 레이어 역할
- Controller: API 입력 검증 + 인증 + 라우트 분기
- Service/Model: 비즈니스 로직 + 페이징 전략 결정
- Repository: DB 쿼리 실행 (SQL 관리)

## 4. 기술 선택 이유
- FastAPI 선택 이유
  - 비동기 I/O(ASGI) 지원으로 동시성 처리 우수
  - 타입 힌트 기반 데이터 검증 (pydantic)
  - 자동 OpenAPI/Swagger 문서화
  - 경량, 테스트 용이, 확장성
- 단일 서버 장점
  - API 중복 제거, 공통 로직 재사용
  - 유지보수 효율 향상
  - 배포/모니터링/로깅 단일화

## 5. 결과
- API 일관성 확보 및 중복 코드 제거
- 유지보수성과 확장성 향상
- 운영 비용 감소
- 스와이프 UX 전략(`OFFSET` + `CURSOR` 혼용 등) 통합
- 한 곳에서 장애 대응 및 튜닝 가능

## 다이어그램 (텍스트)
- 요청 흐름
  - HTTP → FastAPI `APIRouter` → Controller → Model/Service → DB
- 페이징 전략
  - recommend(recommend) → OFFSET
  - latest/likes/views/etc → CURSOR
  - special feed (following/my/bookmark/same_song) → CURSOR
