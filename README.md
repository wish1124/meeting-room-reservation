# 회의실 예약 키오스크 API — Python 3.13 + FastAPI 구현 과제

## 과제 개요

업무 공용 공간의 회의실 예약 키오스크에서 사용할 백엔드 API 서버를 **Python 3.13 + FastAPI**로 구현하는 과제입니다.

기존 Spring Boot 기반 구현을 Python 생태계로 전환하며, 동일한 비즈니스 요구사항과 아키텍처 원칙을 준수해야 합니다.

---

## 기술 스택 요구사항

| 항목 | 요구 사양 |
|------|----------|
| 언어 | Python 3.13 |
| 웹 프레임워크 | FastAPI (최신 버전) |
| ORM | SQLAlchemy 2.x (또는 SQLModel) |
| 데이터베이스 | SQLite In-Memory (`:memory:`) |
| API 문서화 | FastAPI 내장 Swagger UI (`/docs`) |
| 의존성 관리 | `pyproject.toml` (poetry) 또는 `requirements.txt` |
| 테스트 | pytest + pytest-cov |
| 서버 | uvicorn |
| 컨테이너 | Docker (선택 사항, 권장) |

---

## 도메인 모델

다음 4개의 테이블을 설계하고 구현해야 합니다.

### `room` — 회의실

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INTEGER PK | 식별자 |
| name | VARCHAR | 회의실 이름 |
| capacity | INTEGER | 최대 수용 인원 |

### `user` — 사용자

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INTEGER PK | 식별자 |
| name | VARCHAR | 사용자 이름 |

### `schedule` — 일정

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INTEGER PK | 식별자 |
| name | VARCHAR | 일정 이름 |
| start_time | DATETIME | 시작 시각 |
| end_time | DATETIME | 종료 시각 |
| room_id | INTEGER FK | 회의실 참조 |

### `participant` — 참여자

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INTEGER PK | 식별자 |
| user_id | INTEGER FK | 사용자 참조 |
| schedule_id | INTEGER FK | 일정 참조 |
| name | VARCHAR | 참여자 이름 (비정규화 캐시) |
| start_time | DATETIME | 참여 시작 시각 |
| end_time | DATETIME | 참여 종료 시각 |

### 인덱스 요구사항

```sql
-- 회의실별 시간 범위 조회 최적화
CREATE INDEX idx_schedule_room_time ON schedule(room_id, start_time, end_time);

-- 사용자별 시간 범위 조회 최적화 (시간 충돌 검증)
CREATE INDEX idx_participant_user_time ON participant(user_id, start_time, end_time);
```

---

## 초기 데이터

애플리케이션 시작 시 다음 데이터를 자동으로 생성해야 합니다.

- **회의실 3개**
  - id=1, 수용 인원 3명
  - id=2, 수용 인원 5명
  - id=3, 수용 인원 10명
- **사용자 20명** (id=1~20, 각자 이름 보유)

---

## API 엔드포인트 요구사항

### 일정 (Schedule)

#### `POST /schedules` — 일정 생성

요청 바디:

```json
{
  "name": "팀 회의",
  "start_time": "2025-06-22T15:00:00",
  "end_time": "2025-06-22T16:00:00",
  "room_id": 1,
  "participants": [1, 2, 3]
}
```

응답: 생성된 일정 정보 (HTTP 201)

비즈니스 규칙:
- `start_time < end_time` 검증
- **회의실 중복 예약 방지**: 요청 시간대에 동일 회의실이 이미 예약되어 있으면 `409 Conflict`
- **참여자 중복 예약 방지**: 요청 시간대에 특정 사용자가 다른 일정에 이미 참여 중이면 `409 Conflict`
- **수용 인원 초과 방지**: `len(participants) > room.capacity`이면 `400 Bad Request`
- 존재하지 않는 `room_id` 또는 `user_id`는 `404 Not Found`

#### `GET /schedules/{id}` — 일정 단건 조회

응답: 일정 정보 + 회의실 정보 + 참여자 목록

#### `PUT /schedules/{id}` — 일정 수정

수정 가능 항목: `name`, `start_time`, `end_time`, `room_id`

일정 생성과 동일한 비즈니스 규칙 적용 (자기 자신 제외)

#### `DELETE /schedules/{id}` — 일정 삭제

삭제 시 연관 참여자 정보도 함께 삭제

#### `POST /schedules/{id}/participants` — 참여자 추가

요청 바디:

```json
{
  "user_ids": [4, 5]
}
```

- 수용 인원 초과 검증
- 참여자 중복 예약 검증

#### `DELETE /schedules/{id}/participants/{user_id}` — 참여자 제외

---

## 아키텍처 요구사항

### 계층형 아키텍처 (Layered Architecture)

다음 4개 계층을 명확히 분리하여 구현해야 합니다.

```
┌─────────────────┐
│    Router       │  ← FastAPI APIRouter, 요청/응답 처리 (표현 계층)
├─────────────────┤
│    Service      │  ← 비즈니스 로직 (응용 계층)
├─────────────────┤
│   Repository    │  ← 데이터 접근 추상화 (인프라 계층)
├─────────────────┤
│   Domain/Model  │  ← SQLAlchemy 모델 + Pydantic 스키마 (도메인 계층)
└─────────────────┘
```

### 패키지 구조 예시

```
app/
├── main.py
├── database.py
├── domain/
│   ├── schedule/
│   │   ├── model.py        # SQLAlchemy ORM 모델
│   │   ├── schema.py       # Pydantic 요청/응답 스키마
│   │   ├── repository.py   # DB 접근 계층
│   │   ├── service.py      # 비즈니스 로직
│   │   └── router.py       # FastAPI 라우터
│   ├── room/
│   │   └── ...
│   └── user/
│       └── ...
├── core/
│   ├── exceptions.py       # 도메인 예외 클래스
│   └── lock.py             # LockManager 추상 인터페이스
└── seed.py                 # 초기 데이터 생성
```

### SOLID 원칙 준수

- **SRP**: 각 클래스/함수가 하나의 책임만 가질 것
- **OCP**: 확장에는 열려있고 수정에는 닫혀있을 것
- **DIP**: 구체 구현이 아닌 추상화에 의존할 것 (Repository 인터페이스, LockManager ABC)

### 동시성 제어 (LockManager)

분산 락 전략을 교체 가능하도록 추상 인터페이스로 설계해야 합니다.

```python
from abc import ABC, abstractmethod

class LockManager(ABC):
    @abstractmethod
    def acquire_lock(self, key: str, timeout: float) -> bool: ...

    @abstractmethod
    def release_lock(self, key: str) -> None: ...
```

기본 구현체는 `InMemoryLockManager`로 제공하고, 향후 Redis 등으로 교체 가능한 구조여야 합니다.

### 예외 처리

도메인별 예외 계층 구조를 설계하고, FastAPI `exception_handler`를 통해 일관된 HTTP 응답으로 변환해야 합니다.

```
BusinessException
├── ScheduleException
│   ├── ScheduleConflictException     → 409
│   ├── ScheduleNotFoundException     → 404
│   └── InvalidTimeRangeException     → 400
├── RoomException
│   ├── RoomNotFoundException         → 404
│   └── ExceedingRoomCapacityException → 400
└── UserException
    ├── UserNotFoundException         → 404
    └── ParticipantConflictException  → 409
```

---

## 실행 방법 요구사항

### 로컬 실행

```bash
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8080
```

### Docker 실행 (권장)

```bash
docker build -t kiosk-api .
docker run -p 8080:8080 kiosk-api
```

- Dockerfile 작성 필수
- non-root 사용자로 실행
- 타임존: Asia/Seoul

### 접속 확인

| 서비스 | URL |
|--------|-----|
| API 서버 | http://localhost:8080 |
| Swagger UI | http://localhost:8080/docs |
| ReDoc | http://localhost:8080/redoc |

---

## 테스트 요구사항

### 테스트 실행

```bash
pytest
pytest --cov=app --cov-report=html
```

### 테스트 전략

- **단위 테스트 (Unit Test)**: Service 계층의 비즈니스 로직, Repository 계층의 쿼리 로직
- **통합 테스트 (Integration Test)**: FastAPI `TestClient`를 이용한 엔드포인트 시나리오 검증

### 필수 테스트 시나리오

1. 일정 생성 성공
2. 회의실 중복 예약 시 `409` 반환
3. 참여자 중복 예약 시 `409` 반환
4. 수용 인원 초과 시 `400` 반환
5. 존재하지 않는 리소스 접근 시 `404` 반환
6. 잘못된 시간 범위(`start >= end`) 입력 시 `400` 반환

### 커버리지 목표

| 항목 | 목표 |
|------|------|
| 라인 커버리지 | 60% 이상 |
| 브랜치 커버리지 | 50% 이상 |
| 함수 커버리지 | 60% 이상 |

---

## 제출 요구사항

- GitHub 또는 GitLab 레포지토리 링크로 제출
- `README.md` 작성 필수 (실행 방법, API 설명, 아키텍처 설명 포함)
- `pyproject.toml` 또는 `requirements.txt` 포함
- `Dockerfile` 포함 (권장)
- Python 버전은 **3.13** 고정

---

## 참고 사항

- FastAPI의 `Depends`를 활용한 DB 세션 주입을 권장합니다.
- Pydantic v2 스키마와 SQLAlchemy 모델을 명확히 분리하세요.
- `asyncio` 기반 비동기 처리(`async def`)와 동기 처리 중 하나를 선택하되, 일관성 있게 유지하세요.
- N+1 쿼리 문제를 의식하고, `joinedload` 또는 `selectinload` 활용을 고려하세요.
- DTO와 도메인 객체(ORM 모델)의 역할을 명확히 구분하세요.
