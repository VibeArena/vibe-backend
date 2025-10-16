# 🔒 backend/README.md

## KRvibe Backend (Django + DRF + Celery)

**목표:** “프롬프트 활용 과정 + 최종 코드 품질”을 함께 다루는 **Vibe Coding 평가 백엔드**.  
API, 평가 파이프라인, 프롬프트/코드 로그 수집, 스코어링(속도·정확성·프롬프트 효율) 제공.

---

## 0) 팀 규칙 요약 (축약)

문서 선행 ▶ SDS 반영 ▶ 코드/테스트 ▶ PR 리뷰 ▶ README 업데이트.  
컨벤션·보안·주석 “**왜**” 중심, **Free Tier 정신**, 브랜치/커밋 규칙 준수.

---

## 1) 아키텍처 개요

- **Django + DRF**: Public/Private API.  
- **Celery (worker/beat)**: 장기 작업(평가 스코어 계산, 로그 집계, 비동기 수집).  
- **Postgres**: 트랜잭션 데이터(유저, 과제, 제출, 프롬프트/코드 이벤트).  
- **Redis**: Celery broker/result, 캐시/레이트리밋.  
- **MinIO/S3**: 제출 아카이브, 로그 스냅샷.  
- **(선택) OpenTelemetry/Sentry**: 추적/에러 수집.

---

## 2) 폴더 구조(예시)

```
backend/
├─ app/
│  ├─ config/                # settings/{base,dev,prod}.py
│  ├─ api/                   # DRF routers, views, serializers
│  ├─ vibe_api/              # 도메인 앱(유저/과제/제출/평가)
│  ├─ scoring/               # 평가코어(Time-to-Solution, FPA, Prompt Efficiency)
│  ├─ ingestion/             # 프롬프트/코드 편집 로그 수집 파이프
│  ├─ analytics/             # 대시 API(집계, 랭킹)
│  ├─ tasks/                 # Celery tasks (스코어링/집계/크롤링)
│  ├─ utils/                 # 공통 유틸(마스킹, 추적, idempotency)
│  └─ manage.py
├─ requirements/             # base/dev/test
├─ tests/                    # pytest
├─ scripts/                  # manage 스크립트/데이터 씨드
└─ README.md
```

> 과제·평가 정의/로그 보존/스코어 산출의 **일관된 데이터 흐름**이 핵심.

---

## 3) 도메인 모델(핵심)

- **Assignment**: 문제, 카테고리, 제한(시간/프롬프트 횟수), 채점 스펙.  
- **Session/Attempt**: 참가자 시도 단위(시작/종료 시각, 환경/제한).  
- **PromptLog**: 프롬프트/AI응답/사용자 수정 이력(시각, 지표용 메타).  
- **Submission**: 최종 코드/산출물 링크, 테스트 결과.  
- **Scores**:  
  - **Time-to-Solution**(시작→정답까지),  
  - **Prompt Efficiency**(루프 횟수/품질),  
  - **First-pass Accuracy**(첫 제안 테스트 통과율),  
  - **Final Quality vs Time**(효율).

---

## 4) 로컬 개발

### Docker (권장)
```bash
# API + Celery + DB + Redis 통합 기동(Infra 레포에서 up 후 사용)
docker compose -f ../infra/compose/dev/docker-compose.dev.yml up -d

# 마이그레이션 & 슈퍼유저
docker compose -f ../infra/compose/dev/docker-compose.dev.yml exec api python manage.py migrate
docker compose -f ../infra/compose/dev/docker-compose.dev.yml exec api python manage.py createsuperuser
```

### Bare-metal (선택)
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements/dev.txt
export DJANGO_SETTINGS_MODULE=config.settings.dev
export DATABASE_URL=postgres://...
export REDIS_URL=redis://...
python manage.py migrate && python manage.py runserver
celery -A app worker -l info & celery -A app beat -l info &
```

---

## 5) 환경변수

```
DJANGO_SECRET_KEY=...
DJANGO_SETTINGS_MODULE=config.settings.dev
ALLOWED_HOSTS=api.local,localhost

DATABASE_URL=postgres://user:pass@postgres:5432/app
REDIS_URL=redis://redis:6379/0

# Celery
CELERY_BROKER_URL=${REDIS_URL}
CELERY_RESULT_BACKEND=${REDIS_URL}

# Storage
S3_ENDPOINT_URL=http://minio:9000
S3_BUCKET=krvibe
S3_ACCESS_KEY=...
S3_SECRET_KEY=...

# Analytics (예: GA4 Service Account 환경변수)
GA_SVC_JSON_BASE64=...

# Observability (선택)
SENTRY_DSN=
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

---

## 6) API 정책

- **문서화 우선**: OpenAPI 문서 자동 생성(예: drf-spectacular) → `/api/schema`/`/api/docs`.  
- **권한**: JWT/세션 중 하나 선택(프로파일별). 민감 로그 마스킹.  
- **레이트/감사**: 인증 실패/권한 예외는 원인 코드화(추적성).  
- **버전**: `/api/v1/*`에서 시작, deprecation 가이드 필수.

---

## 7) 평가(Scoring) 파이프라인

1. **Ingestion**: 프롬프트·응답·편집 이벤트 수집(IDE 브릿지 or Web IDE).  
2. **Normalization**: 이벤트 정규화/필드 정합성 체크.  
3. **Scoring**:  
   - **Time-to-Solution**, **Prompt Efficiency**, **First-pass Accuracy**, **Final Quality vs Time** 산출,  
   - **가중치/규칙**은 레귤레이션 버전으로 관리(예: `rulesets/v1.yaml`).  
4. **Report**: 랭킹/개인 리포트(요약/세부 흐름) 제공.  
5. **리뷰/리팩터링 기록**: PR 설명/설계노트 연동(선택).

> “프롬프트 활용 과정 + 코드 품질” **복합 평가**가 프로젝트 차별화 포인트.

---

## 8) 테스트

- **pytest + coverage**. PR에는 최소 1개 이상 테스트 추가(핵심 경로).  
- **단위/통합/펑셔널** 순. 외부 의존성은 목/프록시로 격리.  
- 실패 테스트는 즉시 수정 또는 이슈화.

---

## 9) 데이터 거버넌스/보안

- PII/계정/토큰은 저장·표시에 마스킹.  
- 프롬프트/코드 로그는 계약/약관 범위 내 보관, 익명화 옵션 제공.  
- API key/서비스 계정은 ENV/Secret Manager로만 관리.

---

## 10) 커밋·브랜치 규칙

- 커밋은 **무엇/왜/영향/테스트**를 명확히(Conventional Commits 권장).  
- 브랜치: `feature/*`, `fix/*`, `refactor/*`.  
- main에 직접 푸시 금지, PR 리뷰 필수.

---

## 11) 참고 문서

- 프로젝트 기획/범위/성공기준(문제정의·비전·미션·범위·KPI)  
- VIBE CODING 평가 기준(프롬프트+코드 복합평가 흐름/증거 포인트)  
- 개발 속도 & 정확성(Time-to-Solution, Prompt Efficiency, FPA, 효율)  
- 유사 사례 및 시장 조사(포지셔닝/차별화 논리)

