# API 명세서

v0.1 · 2026-11-25 · 구현 예정 계약. 현재 실행 가능한 API는 없다.

## 1. 공통 규약

- Base path: `/api/v1`, JSON UTF-8, 필드 camelCase, 식별자 UUID.
- 시각은 ISO 8601 UTC(`2026-11-25T03:00:00Z`), 날짜는 `YYYY-MM-DD`. 화면은 Asia/Seoul 기준.
- 공개 GET 외 상담 API는 `Authorization: Bearer <sessionToken>` 필수. 토큰과 경로 sessionId가 일치해야 한다. 토큰은 암호학적 난수 256비트 이상이며 서버에는 해시만 저장한다.
- 브라우저는 토큰을 메모리에 보관한다. MVP는 새로고침·탭 종료 후 복구를 보장하지 않는다. sessionId만으로 복구할 수 없다.
- 상담 만료는 생성부터 24시간 고정, 활동으로 연장하지 않는다. 토큰을 URL·로그에 넣지 않는다.
- POST 생성 성공은 201, 조회·수정·추천 성공은 200, 삭제는 204(본문 없음). 생성 응답에 Location 헤더를 제공한다.
- 목록은 `items`, `nextCursor`를 사용한다. `limit` 기본 20, 최대 100; cursor는 필터·정렬에 묶인 불투명 값이다. 마지막 페이지는 nextCursor=null.
- 오류 시 동일한 `error` 구조를 반환한다. 명세에 없는 필드는 422로 거부한다.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "입력값을 확인해주세요.",
    "details": [{"field": "content", "reason": "1~2000자여야 합니다."}],
    "traceId": "opaque-trace-id"
  }
}
```

| HTTP | code | 상황 |
|---|---|---|
| 400 | BAD_REQUEST | JSON 파싱·cursor 오류 |
| 401 | INVALID_SESSION | 누락·위조·삭제·만료된 토큰 |
| 404 | RESOURCE_NOT_FOUND | 대상 없음 또는 다른 세션 소유 객체 |
| 409 | VERSION_CONFLICT / QUESTION_ALREADY_ANSWERED / INVALID_TRANSITION / IDEMPOTENCY_CONFLICT | 동시 수정·중복 답변·잘못된 상태 전이·키 충돌 |
| 422 | VALIDATION_ERROR / CONSENT_REQUIRED | 필드·값·필수 동의 오류 |
| 429 | RATE_LIMITED | 제한 초과, Retry-After 초 단위 제공 |
| 503 | AI_UNAVAILABLE / DATA_UNAVAILABLE | 의존 서비스 사용 불가 |
| 504 | PROCESSING_TIMEOUT | 처리 시간 초과 |

### 재시도와 동시성

세션 생성 외 모든 POST는 `Idempotency-Key`(UUID) 필수다. 범위는 세션+메서드+경로+키이며 만료까지 보관한다. 같은 본문·키의 성공 요청은 동일 상태 코드·본문을 재전송하고, 다른 본문은 409다. 진행 중이면 409 `REQUEST_IN_PROGRESS`와 Retry-After를 반환한다. AI 실패·시간 초과 후 같은 키로 재시도하면 기존 메시지에 처리를 재개한다. 응답은 세션과 함께 삭제한다. 세션 생성 실패 재시도는 새 세션을 생성할 수 있으며 사용하지 않는 세션은 만료 삭제한다.

프로필 변경은 `expectedProfileVersion` 필수다. 질문 응답과 프로필 저장은 한 트랜잭션이며 충돌 시 아무것도 반영하지 않는다. 숫자 version은 1부터 증가한다.

## 2. 엔드포인트 목록

`S`는 경로의 세션 소유자 인증, `공개`는 인증 없음이다. `{sid}`는 sessionId, `{pid}`는 programId다.

| 메서드 | 경로 | 인증 | 성공 | 목적 |
|---|---|---|---|---|
| POST | /sessions | 공개 | 201 | 동의 및 상담 생성 |
| GET | /sessions/{sid} | S | 200 | 상태·만료·프로필 버전 |
| DELETE | /sessions/{sid} | S | 204 | 상담 및 관련 정보 삭제 |
| POST | /sessions/{sid}/messages | S | 201 | 자기소개·자유 응답 분석 |
| GET | /sessions/{sid}/messages | S | 200 | 메시지 시간순 목록 |
| GET | /sessions/{sid}/profile | S | 200 | 확인·제안된 사실 조회 |
| PATCH | /sessions/{sid}/profile | S | 200 | 사실 확인·수정·제거 |
| POST | /sessions/{sid}/questions/next | S | 200 | 다음 질문 또는 종료 사유 |
| POST | /sessions/{sid}/questions/{questionId}/answers | S | 201 | 선택형 응답·모름·건너뛰기 |
| POST | /sessions/{sid}/recommendations | S | 201 | 현재 정보로 추천 평가 |
| GET | /sessions/{sid}/recommendations/{runId} | S | 200 | 추천 스냅샷 조회 |
| GET | /regions | 공개 | 200 | 지역 코드 검색 |
| GET | /programs | 공개 | 200 | 공고 목록 |
| GET | /programs/{pid} | 공개 | 200 | 최신 공고 상세 |
| GET | /programs/{pid}/versions/{versionId} | 공개 | 200 | 근거가 된 과거 공고 |
| POST | /sessions/{sid}/applications | S | 201 | 신청 준비 생성 |
| GET | /sessions/{sid}/applications | S | 200 | 신청 준비 목록 |
| GET | /sessions/{sid}/applications/{applicationId} | S | 200 | 체크리스트 상세 |
| PATCH | /sessions/{sid}/applications/{applicationId} | S | 200 | 사용자 기록 상태 수정 |
| PATCH | /sessions/{sid}/applications/{applicationId}/items/{itemId} | S | 200 | 항목 완료 여부 수정 |

## 3. 상담 생성·조회

`POST /sessions`

```json
{"consent":{"privacyVersion":"2026-11-25-v1","accepted":true}}
```

privacyVersion은 화면에 제공한 현재 안내 버전과 일치해야 한다. 서버가 동의 시각을 기록한다. 아래 안내 버전은 설계 예시이며 배포 전 실제 안내문 버전으로 교체한다.

```json
{
  "sessionId":"11111111-1111-4111-8111-111111111111",
  "sessionToken":"<생성 시에만 반환되는 비밀 토큰>",
  "status":"ACTIVE",
  "profileVersion":1,
  "expiresAt":"2026-11-26T03:00:00Z"
}
```

GET은 토큰을 제외한 같은 구조 및 createdAt을 반환한다. DELETE는 messages, facts, questions, answers, 추천, 신청 준비, 멱등 기록을 함께 삭제한다. 삭제된 토큰의 재요청은 401이다.

## 4. 메시지와 프로필

`POST /sessions/{sid}/messages`

```json
{"content":"경기도 안산시에 사는 25살입니다."}
```

content 1~2000자. 자유 응답으로 특정 질문에 답할 때만 questionId(UUID)를 추가한다. 처리 성공은 사용자·AI 메시지와 제안을 반환한다. 자유 응답은 사실을 자동 확정하거나 질문을 완료하지 않으며 사용자가 PATCH로 확인해야 한다.

```json
{
  "messageId":"22222222-2222-4222-8222-222222222222",
  "assistantMessage":"지역과 나이를 확인해주세요. 만 나이인지도 확인이 필요해요.",
  "proposedFacts":[
    {"field":"ageYears","value":25,"state":"PROPOSED"}
  ],
  "profileVersion":2
}
```

새 제안 저장도 profileVersion을 증가시킨다. 동일 필드의 CONFIRMED 값은 자유 입력으로 덮어쓰지 않고 응답의 `conflicts`에 `{field,currentValue,proposedValue}`로 표시한다. 최종 변경은 PATCH로 한다. GET messages는 `items:[{id,role,content,processingStatus,createdAt}],nextCursor`이며 role은 USER/ASSISTANT, processingStatus는 PENDING/SUCCEEDED/FAILED다.

`GET /sessions/{sid}/profile` → `{profileVersion,facts:[{field,value,state,sourceMessageId,confirmedAt}]}`. value·sourceMessageId·confirmedAt은 null 가능하다.

`PATCH /sessions/{sid}/profile`

```json
{
  "expectedProfileVersion":2,
  "changes":[
    {"field":"ageYears","value":25,"state":"CONFIRMED"},
    {"field":"ageBasis","value":"INTERNATIONAL","state":"CONFIRMED"},
    {"field":"ageAsOf","value":"2026-11-25","state":"CONFIRMED"}
  ]
}
```

응답은 최신 프로필이다. changes는 1~20개, 중복 field 금지. 사용자 PATCH는 CONFIRMED/UNKNOWN/SKIPPED만 허용한다. UNKNOWN/SKIPPED는 value=null이며 기존 값을 제거한다. PROPOSED는 서버 전용이다. `questionId`를 요청 최상위에 넣으면 해당 질문의 field를 확인하는 경우에만 함께 완료 처리한다.

### 프로필 필드 사전

모든 필드는 선택 항목이다. 미제공·모름·건너뜀은 조건 평가에서 UNKNOWN이다. 아래 소득·주거 정보도 후보 판단에 필요한 경우에만 묻는다.

| field | value 타입·허용값 | 검증·의미 |
|---|---|---|
| ageYears | integer 0~120 | ageBasis·ageAsOf 없으면 나이 비교 보류 |
| ageBasis | INTERNATIONAL / UNKNOWN | 만 나이 여부 |
| ageAsOf | date | 연령 확인 기준일 |
| birthDate | date | 미래 금지, 특정 기준일 계산이 필요할 때 선택 입력 |
| gender | MALE / FEMALE / OTHER / UNDISCLOSED | 필요한 조건에만 사용 |
| residenceRegionCode | string | regions에 등록된 실거주 지역 코드 |
| registeredRegionCode | string | 주민등록 지역, 실거주와 구분 |
| livingAlone | boolean | 독립 거주 여부; 1인 가구 판정과 구분 |
| housingType | MONTHLY_RENT / JEONSE / OWNED / OTHER | 주거 유형 |
| monthlyRentKrw | integer >=0 | 월 임대료, 원 |
| employmentStatus | EMPLOYED / SELF_EMPLOYED / JOB_SEEKING / UNEMPLOYED / OTHER | 취업 상태 |
| studentStatus | ENROLLED / ON_LEAVE / GRADUATED / NOT_STUDENT | 재학 상태 |
| householdSize | integer 1~100 | 가구 정의가 공고와 일치하는지 추가 확인 |
| income | object | amountKrw>=0, period=MONTH/YEAR, scope=PERSONAL/HOUSEHOLD, basis=GROSS/NET/RECOGNIZED, asOf=date 모두 필수 |
| interests | string[] | HOUSING / EMPLOYMENT / EDUCATION / LIVING / STARTUP, 중복 금지 |

새 조건에 필요한 필드는 사전 확장·버전 검토 후 추가한다. 모델이 임의의 필드를 저장할 수 없다.

## 5. 추가 질문

`POST /sessions/{sid}/questions/next` 요청: `{"expectedProfileVersion":3}`.

```json
{
  "question":{
    "id":"33333333-3333-4333-8333-333333333333",
    "field":"housingType",
    "text":"현재 어떤 형태로 거주하고 있나요?",
    "reason":"주거 지원 후보의 월세 조건을 확인하기 위해서예요.",
    "inputType":"SINGLE_SELECT",
    "options":[{"value":"MONTHLY_RENT","label":"월세"},{"value":"JEONSE","label":"전세"},{"value":"OWNED","label":"자가"},{"value":"OTHER","label":"기타"}],
    "canSkip":true
  },
  "stopReason":null,
  "profileVersion":3
}
```

현재 미응답 질문이 있으면 같은 질문을 반환한다. 없으면 조건별로 생성한다. inputType은 SINGLE_SELECT/BOOLEAN/NUMBER/DATE/OBJECT이며 field 사전과 동일한 자료형을 쓴다. 질문이 없으면 question=null, stopReason은 ENOUGH_INFO/QUESTION_LIMIT/NO_CANDIDATES/NO_MORE_QUESTIONS 중 하나다. ENOUGH_INFO도 수급 확정을 뜻하지 않는다.

`POST .../questions/{questionId}/answers`

```json
{"expectedProfileVersion":3,"status":"ANSWERED","value":"MONTHLY_RENT"}
```

status는 ANSWERED/UNKNOWN/SKIPPED. ANSWERED는 질문 타입에 맞는 value 필수, 나머지는 null. 응답은 `{answerId,questionId,profileVersion:4}`. ANSWERED는 CONFIRMED 사실로 저장한다. 다른 키로 완료된 질문에 재응답하면 409; 이후 수정은 profile PATCH를 사용한다. 질문 생성 이후 프로필이 바뀌면 해당 질문을 SUPERSEDED로 만들고 답변을 409로 거부한다. 질문 생성 자체는 프로필 버전을 바꾸지 않는다.

## 6. 추천

`POST /sessions/{sid}/recommendations`

```json
{"expectedProfileVersion":4,"includeIneligible":false,"includeClosed":false}
```

두 boolean은 선택이며 기본 false다. 결과 객체는 GET과 동일하다. 다음 예시는 가상 데이터이며 실제 지원사업·접수 기간이 아니다.

```json
{
  "runId":"44444444-4444-4444-8444-444444444444",
  "profileVersion":4,
  "evaluatedAt":"2026-11-25T03:10:00Z",
  "stale":false,
  "coverage":{"sourceNames":["시연용 공식 출처"],"lastSuccessfulSyncAt":null},
  "items":[{
    "programId":"55555555-5555-4555-8555-555555555555",
    "programVersionId":"66666666-6666-4666-8666-666666666666",
    "title":"가상 월세 지원 예시",
    "eligibilityStatus":"NEEDS_INFO",
    "recruitmentStatus":"UNKNOWN",
    "reasons":["월세 거주 정보를 확인했습니다."],
    "missingFields":["income"],
    "conditions":[{"field":"income","result":"UNKNOWN","reason":"소득 정보가 없습니다.","evidenceChunkIds":["77777777-7777-4777-8777-777777777777"]}],
    "officialUrl":"https://example.org/program",
    "applicationUrl":null,
    "lastVerifiedAt":null
  }],
  "nextCursor":null,
  "emptyReason":null
}
```

생성 응답은 첫 20개 결과를 반환한다.
GET은 limit/cursor를 받아 저장된 결과를 페이지 조회한다. 결과가 없으면 items=[], emptyReason=NO_MATCH 또는 NO_DATA다. 출처 전체 장애로 평가 자체가 불가능하면 503이다. stale은 조회 시 현재 프로필·최신 공고 버전과 비교해 계산한다. 과거 평가 내용을 덮어쓰지 않는다. 접수 상태는 evaluatedAt 기준이며 최신 상태는 공고 상세에서 확인한다.

## 7. 지역·공고 조회

`GET /regions?q=안산&parentCode=...&limit=20&cursor=...`

q(1~100자)·parentCode는 선택. 응답 `{items:[{code,name,parentCode}],nextCursor}`. 코드는 서버 지역 사전을 사용하며 클라이언트가 임의 생성하지 않는다.

`GET /programs?q=주거&regionCode=...&category=HOUSING&recruitmentStatus=OPEN&limit=20&cursor=...`

모든 필터는 선택, q 최대 100자. category는 interests와 동일. 정렬은 최신 갱신 내림차순 후 ID 오름차순. 목록 항목은 `{id,title,agency,category,regionCodes,recruitmentStatus,officialUrl,lastVerifiedAt}`. regionCode 지정 시 해당 지역과 상위 지역 및 전국 공고를 포함한다.

`GET /programs/{pid}` 및 `/versions/{versionId}` 응답:

| 필드 | 타입 | 설명 |
|---|---|---|
| id, versionId | UUID | 공고 및 불변 버전 |
| title, agency, category | string | 제목·기관·분야 |
| regionCodes | string[] | 대상 지역, 전국 코드는 지역 사전에 정의 |
| summary, benefitText | string 또는 null | 요약·지원 내용, 미확인 null |
| officialUrl | URL | 공식 원문 |
| applicationUrl | URL 또는 null | 확인된 신청 링크 |
| recruitmentStatus | enum | OPEN/UPCOMING/CLOSED/UNKNOWN |
| applicationPeriod | object | type=FIXED/ROLLING/UNKNOWN, startDate/endDate, startsAt/endsAt, timezone, rawText; 미확인 필드 null |
| requirements | object | verificationStatus=DRAFT/REVIEWED, complete:boolean, root:조건 트리 또는 null |
| documents | array | {id,name,description,required:boolean 또는 null,evidenceChunkId} |
| steps | array | {id,order,title,description,url 또는 null,evidenceChunkId} |
| evidence | array | {id,ordinal,text} |
| source | object | {id,name,lastSuccessfulSyncAt} |
| lastVerifiedAt | datetime 또는 null | 원문 확인 시각 |

조건 그룹은 `{id,type:"ALL" 또는 "ANY",children:[...]}`, 리프는 `{id,type:"RULE",field,operator,value,evidenceChunkId}`다. operator는 EQ/IN/GTE/LTE/BETWEEN, BETWEEN 값은 `{min,max}`이며 양 끝 포함이다. 근거 없는 조건·서류·절차를 생성하지 않는다. 목록이 비어 있으면 정보 미확인으로 표시하고 서류가 불필요하다고 해석하지 않는다.

## 8. 신청 준비

`POST /sessions/{sid}/applications` 요청: `{programId,programVersionId}`. 지정 버전이 해당 공고에 속해야 한다. `(sessionId,programVersionId)`는 유일하며 기존 항목이 있으면 409 `APPLICATION_ALREADY_EXISTS`와 existingApplicationId를 반환한다. 새 공고 버전은 새 준비 항목으로 만들 수 있다.

응답 및 GET 상세: `{id,programId,programVersionId,status,version,stale,officialUrl,applicationUrl,items,createdAt,updatedAt}`. status 초기 PREPARING, version 초기 1. items는 `{id,type:DOCUMENT 또는 STEP,title,description,evidenceChunkId,completed,completedAt}`. 서버가 해당 공고 버전의 서류·절차에서 복사하며 정보가 없으면 빈 배열을 반환한다. 목록 GET은 이 구조에서 items를 제외하고 페이징한다.

`PATCH .../applications/{id}` 요청: `{"expectedVersion":1,"status":"SUBMITTED_BY_USER"}`.

허용 전이: PREPARING → SUBMITTED_BY_USER/CANCELLED, SUBMITTED_BY_USER/CANCELLED → PREPARING. 현재 상태를 다시 요청하면 변경 없이 200. 승인·수급 완료 상태는 MVP에 없다. 응답은 최신 상세다.

`PATCH .../items/{itemId}` 요청: `{"expectedVersion":1,"completed":true}`. 준비 중인 항목만 수정 가능하며 application.version을 함께 증가시킨다. false는 완료 시각을 null로 만든다. 잘못된 버전은 409. 다른 준비 항목의 itemId는 404다.

## 9. 운영 경계

수집·검수는 내부 작업으로만 실행하며 MVP 공개 관리자 API는 제공하지 않는다. 세션 생성 기본 제한은 IP당 분당 10회, AI 요청은 세션당 분당 10회로 제안한다. 운영 부하 측정 후 조정한다. OpenAPI 파일과 Swagger UI는 DTO·컨트롤러 구현 단계에서 이 계약을 바탕으로 추가한다.
