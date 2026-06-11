회의실 예약 키오스크 API 정의서
1. 엔드포인트 전체 목록
일정 (Schedule)
Method	Path	설명
POST	/schedules	일정 생성 (참여자 포함)
GET	/schedules/{id}	일정 단건 조회 (회의실 + 참여자 포함)
PUT	/schedules/{id}	일정 수정 (이름/시간/회의실)
DELETE	/schedules/{id}	일정 삭제 (참여자 연쇄 삭제)


참여자 (Participant)
Method	Path	설명
POST	/schedules/{id}/participants	참여자 추가 (수용 인원·시간 충돌 검증)
DELETE	/schedules/{id}/participants/{user_id}	참여자 제외
2. 도메인 모델
Room (회의실)
 ├─ id                 : INTEGER PK
 ├─ name               : 회의실 이름
 └─ capacity           : 최대 수용 인원

User (사용자)
 ├─ id                 : INTEGER PK
 └─ name               : 사용자 이름

Schedule (일정)
 ├─ id                 : INTEGER PK
 ├─ name               : 일정 이름
 ├─ start_time         : 시작 시각 (DATETIME)
 ├─ end_time           : 종료 시각 (DATETIME)
 └─ room_id            : Room FK

Participant (참여자)
 ├─ id                 : INTEGER PK
 ├─ user_id            : User FK
 ├─ schedule_id        : Schedule FK
 ├─ name               : 사용자 이름 (비정규화 스냅샷)
 ├─ start_time         : 일정 시작 시각 (비정규화 스냅샷)
 └─ end_time           : 일정 종료 시각 (비정규화 스냅샷)
Participant에 시간·이름을 복제한 이유: 사용자별 시간 충돌 검증 시 Schedule JOIN 없이 (user_id, start_time, end_time) 인덱스 단독으로 조회하기 위한 의도적 비정규화.
3. 상태 코드 정책
코드	발생 상황
200	조회/수정/삭제 성공
201	일정 생성, 참여자 추가 성공
400	start_time ≥ end_time, 수용 인원 초과
404	존재하지 않는 일정/회의실/사용자
409	회의실 중복 예약, 참여자 시간 충돌
시간 겹침 정의: A.start < B.end AND A.end > B.start (경계가 맞닿는 경우 — 15:00 종료 / 15:00 시작 — 는 충돌 아님)
4. 엔드포인트 상세
4.1 POST /schedules — 일정 생성
요청 바디:
{
  "name": "팀 회의",
  "start_time": "2025-06-22T15:00:00",
  "end_time": "2025-06-22T16:00:00",
  "room_id": 1,
  "participants": [1, 2, 3]
}
검증 순서:
1. start_time < end_time → 아니면 400 (InvalidTimeRangeException)
2. room 존재 확인 → 없으면 404 (RoomNotFoundException)
3. 모든 user 존재 확인 → 없으면 404 (UserNotFoundException)
4. 참여자 수 ≤ room.capacity → 초과 시 400 (ExceedingRoomCapacityException)
5. 회의실 시간대 중복 검사 → 충돌 시 409 (ScheduleConflictException)
6. 참여자별 시간대 중복 검사 → 충돌 시 409 (ParticipantConflictException)
응답: 201 + 생성된 일정 정보
4.2 GET /schedules/{id} — 일정 단건 조회
응답: 200
{
  "id": 1,
  "name": "팀 회의",
  "start_time": "2025-06-22T15:00:00",
  "end_time": "2025-06-22T16:00:00",
  "room": { "id": 1, "name": "회의실 A", "capacity": 3 },
  "participants": [
    { "user_id": 1, "name": "김철수" },
    { "user_id": 2, "name": "이영희" }
  ]
}
에러: 일정 없음 → 404
4.3 PUT /schedules/{id} — 일정 수정
수정 가능: name, start_time, end_time, room_id (참여자는 별도 API)
규칙:
* 생성과 동일한 검증, 단 중복 검사에서 자기 자신 제외
* 시간/회의실 변경 시 기존 참여자 전원의 시간 충돌 재검증 → 충돌 시 409
* 새 회의실 capacity < 현재 참여자 수 → 400
응답: 200 + 수정된 일정 정보
4.4 DELETE /schedules/{id} — 일정 삭제
연관 Participant 레코드 연쇄 삭제. 응답 200, 일정 없으면 404
4.5 POST /schedules/{id}/participants — 참여자 추가
요청 바디:
{ "user_ids": [4, 5] }
검증 순서:
1. 일정 존재 → 없으면 404
2. 모든 user 존재 → 없으면 404
3. 기존 참여자 수 + 추가 인원 ≤ capacity → 초과 시 400
4. 추가 대상자의 해당 시간대 충돌 검사 → 충돌 시 409
응답: 201 + 갱신된 참여자 목록
4.6 DELETE /schedules/{id}/participants/{user_id} — 참여자 제외
응답 200. 일정이 없거나 해당 사용자가 참여자가 아니면 404
5. 인덱스 / 동시성
INDEX idx_schedule_room_time      : schedule(room_id, start_time, end_time)
INDEX idx_participant_user_time   : participant(user_id, start_time, end_time)
동시 예약 경쟁 방지: 일정 생성/수정 시 회의실 키 단위로 락 획득 후 검증→저장 수행. LockManager 추상 인터페이스로 분리 (기본: InMemoryLockManager, 확장: Redis 분산 락)
6. 초기 데이터
* 회의실 3개: id 1 (3인실), id 2 (5인실), id 3 (10인실)
* 사용자 20명: id 1~20

7. ai와 대화를 해봐도 이해가 어려운 부분들
POST /schedules — 일정 생성
입력 검증 레벨 (Pydantic이 잡아주는 것 / 못 잡는 것)
* 필수 필드 누락, 타입 오류, 날짜 형식 오류 → Pydantic이 자동으로 422 반환. 여기서 주의: 과제 예외 체계는 400/404/409인데 FastAPI 기본 검증 실패는 422라서, 422를 400으로 통일할지 그대로 둘지 정의서에 명시해야 한다. 안 정하면 에러 응답 형식이 두 갈래로 갈라져(Pydantic의 detail 배열 vs 우리 error/message 형식) 클라이언트가 헷갈린다.
* start_time >= end_time → 400. 그런데 start == end(0분짜리 일정)를 명시적으로 막았는지 확인. < 검증이면 자동으로 막히지만 <=로 잘못 쓰면 통과된다.
* 과거 시간 예약 허용할 건지? 과제엔 없는데 정의서엔 한 줄 있어야 하는 항목. 키오스크 특성상 "지금 바로 시작하는 회의" 등록하다 보면 start_time이 몇 초 과거가 되는 경우가 흔해서, 막으면 오히려 UX 버그가 된다. "과거 시간 허용" 또는 "5분 유예" 같은 정책 결정 필요.
* participants: [] 빈 배열 → 참여자 0명 일정 허용? 그리고 participants: [1, 1, 2] 중복 user_id → 이거 안 거르면 같은 사람이 participant 레코드 2개 생기고, 인원 검증도 3명으로 세서 capacity 판정이 틀어진다. set()으로 dedup하거나 400으로 거절하거나, 어느 쪽이든 명시.
* 비현실적 입력: end_time이 start보다 10년 뒤, name이 10만 자 → max_length, 최대 일정 길이 제한 걸어둘지.
비즈니스 검증 레벨
* room_id 없음 → 404, user_id 일부 없음 → 404. 디테일: user가 [1, 2, 999]면 어떤 ID가 없는지 메시지에 담아줘야 클라이언트가 고칠 수 있다.
* 인원 초과 → 400, 회의실 충돌 → 409, 참여자 충돌 → 409. 참여자 충돌도 누가 충돌인지 응답에 포함.
시스템 레벨 (제일 많이 놓치는 곳)
* 락 획득 타임아웃: 3초 기다려도 못 잡으면? 무한 대기는 절대 안 되고, 503 또는 409로 빠져야 한다. 이 케이스의 상태 코드가 정의서에 빠져 있다.
* 검증 통과 후 INSERT 도중 DB 예외 → 500인데, participant까지 같이 넣다가 중간에 터지면 schedule만 남는 반쪽 데이터가 된다. schedule + participants INSERT는 반드시 하나의 트랜잭션으로 묶고 실패 시 전체 롤백. 그리고 롤백하더라도 락은 finally에서 해제.
GET /schedules/{id} — 단건 조회
제일 단순해 보이지만:
* 존재하지 않는 id → 404. 기본.
* path 파라미터 타입: /schedules/abc 요청 → id: int 타입 힌트면 FastAPI가 422 반환. 이것도 400으로 통일할지 결정 대상.
* id=-1, id=0 → 음수도 형식상 int라 통과해서 DB까지 간다. 결과는 어차피 404라 동작은 맞는데, Path(gt=0)으로 앞단에서 거르면 더 깔끔.
* N+1 문제: 에러는 아니지만 장애 씨앗. room + participants 같이 내려주는데 lazy loading이면 일정 1 + 회의실 1 + 참여자 N 쿼리가 나간다. 과제가 joinedload/selectinload 언급한 게 정확히 이 지점. 그리고 lazy loading + 세션 닫힌 후 직렬화하면 DetachedInstanceError로 500 터지는 게 SQLAlchemy 초심자 단골 사고다.
* 참여자가 가리키는 user가 삭제됐다면? 이 과제엔 user 삭제 API가 없어서 안 터지지만, participant.name 스냅샷 덕분에 user 없어도 응답 가능하다는 게 비정규화의 부수 효과 — 알고는 있어야 함.
PUT /schedules/{id} — 수정 (에러 표면적이 제일 넓다)
생성의 모든 에러 + 수정 고유의 함정:
* 자기 자신 제외 누락: 충돌 검사 쿼리에 schedule.id != {id} 조건을 빼먹으면, 이름만 바꾸는 수정도 자기 자신과 충돌해서 409가 난다. 이 엔드포인트 구현 버그 1순위.
* 더 미묘한 쪽: participant 테이블에도 시간이 복제돼 있으니까, 참여자 충돌 검사에서도 "이 일정의 participant 레코드들"을 제외해야 한다. schedule 쪽만 제외하고 participant 쪽을 빼먹으면 시간 변경 시 자기 일정의 참여자들이 자기 자신과 충돌 판정 난다.
* 비정규화 동기화: 시간을 수정하면 schedule.start/end만 바꾸고 끝이 아니라 participant 레코드 N개의 start/end도 전부 UPDATE해야 한다. 이거 빼먹으면 그 순간부터 참여자 충돌 검증이 옛날 시간 기준으로 돌아가서, 데이터는 멀쩡해 보이는데 검증만 조용히 틀리는 최악의 버그가 된다. 비정규화의 대가가 바로 이거고, 당연히 schedule UPDATE + participant UPDATE는 한 트랜잭션.
* 회의실 변경 시 새 capacity < 기존 참여자 수 → 400.
* 락 키 변화: 회의실을 1→2로 옮기면 기존 room:1과 새 room:2 둘 다 잠가야 하나? 엄밀히는 새 회의실(room:2)의 충돌 검사가 보호 대상이라 room:2는 필수고, room:1은 "그 시간이 비는" 쪽이라 안전. 다만 둘 다 잡는 게 추론 단순해서 나쁘지 않다. 정렬 획득 규칙은 동일 적용.
* PUT 시맨틱: 부분 필드만 보내면? PUT은 원래 전체 교체라 전 필드 필수로 받는 게 정석. name만 바꾸고 싶어도 4개 다 보내라 — 이걸 스키마(필수 필드)로 강제하든 PATCH를 따로 열든 결정.
DELETE /schedules/{id} — 삭제
* 없는 id → 404. 그런데 이미 삭제된 걸 또 삭제하면? 404가 일반적이지만, DELETE는 멱등이라 200/204로 응답하는 유파도 있다. 어느 쪽이든 정의서에 한 줄.
* 연쇄 삭제 구현 방식 선택: 세 가지가 있고 각각 다르게 터진다.
    * DB의 ON DELETE CASCADE → 제일 견고. 단 SQLite는 PRAGMA foreign_keys=ON을 커넥션마다 켜줘야 FK가 작동한다. 기본이 OFF라서, 이거 모르면 CASCADE 걸어놨는데 고아 participant가 조용히 쌓인다. SQLite 함정 중 최상급.
    * SQLAlchemy relationship의 cascade="all, delete-orphan" → ORM 경유 삭제만 동작. session.execute(delete(...)) 같은 bulk delete로 지우면 cascade가 안 탄다.
    * 서비스 레이어에서 participant 먼저 delete, schedule delete → 순서 바뀌면 FK 위반, 두 delete가 한 트랜잭션 아니면 반쪽 삭제.
* 삭제 vs 조회 레이스: A가 GET으로 일정 보는 중에 B가 DELETE → A의 다음 요청은 404. 이건 정상 동작이라 막을 필요 없지만, 삭제 vs 참여자 추가 레이스는 다르다. 참여자 추가가 "일정 존재 확인" 통과한 직후 삭제가 끼어들면 고아 participant가 생긴다. 해결은 참여자 추가/삭제도 같은 락 체계(또는 FK 제약)에 태우는 것 — POST /schedules만 락 거는 게 아니라는 얘기.
* 삭제는 충돌 검사가 필요 없으니 락이 필요 없어 보이는데, 위 레이스 때문에 삭제도 room 락(혹은 schedule 락)을 잡는 게 안전하다.


1. PUT의 비정규화 동기화 — 조용히 틀려서 제일 위험
2. SQLite FK PRAGMA — CASCADE 무력화, 모르면 못 찾음
3. 422 vs 400 정책 통일 — 에러 응답 일관성
4. 락 타임아웃 시 응답(503) — 정의서 빈칸
5. 참여자 충돌 검사의 자기 자신 제외 (participant 쪽) — 함정의 함정
