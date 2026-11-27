# API 명세서

> 원본(노션): https://app.notion.com/p/408b82ca31e580d19b7ee382dacd572b  
> 내보낸 날짜: 2026-11-26. **노션이 원본이다.** 이 파일과 노션이 다르면 노션을 따른다.

> 🔌 자산관리 시스템 백엔드 API의 **설계용 명세**다. 최종본은 코드(springdoc)가 만든 OpenAPI이며, 개발 후 둘을 대조해 차이를 고친다. API 목록의 기준은 [기능별 가이드](https://app.notion.com/p/df42d680754d4861a1c741bff51dedfd)이고, 필드는 [테이블 정의서](https://app.notion.com/p/408b82ca31e58080bd5deb903f3026c6), 동작·검증은 [재고 원장 처리 규칙](https://app.notion.com/p/53340b01efa54f16a6217503509ca182)와 [불변 조건 목록](https://app.notion.com/p/795f92f07cab4e38860d4e9f1e6df2d0), 검수 기준은 [요구사항](https://app.notion.com/p/c6543c33806d46cf9c93e158c42ba650)를 따른다.

> 🎨 이 페이지는 전부 요구사항 정의서 v1.1 이후 새로 작성한 내용이라 파란 글씨로 표기한다.

## 1. 공통 규칙

### 1.1 경로

- 형식은 `/api/v1/{모듈}/{리소스}`, 리소스는 **단수형**, kebab-case다 (2026-11-23 결정).
- 모듈의 대표 리소스는 모듈 경로 자체를 쓴다. 예: 제품 목록 `/api/v1/product`, 제품 하나 `/api/v1/product/{id}`, 품목군 `/api/v1/product/category`. ID는 UUID라 하위 리소스 이름과 헷갈리지 않는다(2026-11-23 숫자 → UUID, 1.13절).
- 모듈 경로: user, product, warehouse, partner, inventory, equipment, project, allocation, delivery, loan, cost, trash, dashboard, importing, admin(승인·확인 대기함, 알림, 정합성 점검 결과, 2026-11-23 추가). common 모듈은 API가 없고 중복 요청 키 테이블만 갖는다 ([디렉토리 구조](https://app.notion.com/p/408b82ca31e5802da847f03723e75a89) 3절).

| 구분 | 메서드 | 경로 예 |
|---|---|---|
| 목록 조회 | GET | `/api/v1/product` |
| 하나 조회 | GET | `/api/v1/product/{id}` |
| 등록 | POST | `/api/v1/product` |
| 수정 (바꾸는 항목만 보냄) | PATCH | `/api/v1/product/{id}` |
| 삭제 (삭제 목록으로 보냄) | DELETE | `/api/v1/project/trade-document/{id}` |
| 업무 동작 (재고·장비·문서가 바뀌는 처리) | POST | `/api/v1/delivery/receipt`, `/api/v1/inventory/transfer` |
| 승인·반려 | POST | `/api/v1/admin/review/{id}/approve`, `…/reject`, `…/confirm` (모든 승인·확인을 여기 하나로) |
| 처리 묶음 조회 | GET | `/api/v1/inventory/stock-transaction?type=RECEIPT` |

### 1.2 쓰기 API는 업무 동작별로 나눈다

- 입고, 출고, 이동, 검수 판정, 공급사 반품, 교체, 폐기처럼 업무 동작마다 API를 따로 둔다 (2026-11-23 결정). 동작마다 권한·필수 항목·검증이 다르기 때문이다.
- 동작 이름은 처리 유형 코드를 kebab-case로 바꾼 것을 기본으로 한다. 예: `SUPPLIER_RETURN` → `supplier-return` ([재고 원장 처리 규칙](https://app.notion.com/p/53340b01efa54f16a6217503509ca182) 4절).
- 조회는 처리 묶음 하나로 모아서 하고, 처리 유형으로 거른다.
- **경로의 모듈은 그 업무를 맡는 모듈이다** (2026-11-23). 거래 문서에 걸린 처리(발주 입고, 출고, 직납, 고객·공급사 반품)는 `delivery`, 재고만 움직이는 처리(이동, 검수 판정, 발주 외 입고, 조정, 취소·정정, 초기 재고)는 `inventory`. inventory는 거래 문서를 볼 수 없으므로(디렉토리 구조 5절) 발주 잔량을 확인하는 처리는 delivery가 맡고 원장은 InventoryCommands로 요청한다. inventory가 취소한 처리가 문서 수량에 영향을 주면 이벤트로 알리고, 받는 모듈이 같은 트랜잭션에서 바꾼다(INV-75).
- 승인·반려·확인은 대기함 API(`/api/v1/admin/review`) 하나로 한다. 대상 모듈은 이벤트를 받아 반영한다 (2026-11-23).
- 기능별 가이드 하나가 API 여러 개가 될 수 있다. 예: 재고 실사 = 시작, 수량 입력, 대조, 승인, 반려.

### 1.3 인증과 권한

- 방식: **서버 세션 로그인** (2026-11-25 결정, OAuth2 + JWT에서 바꿈. ADR-17 "인증 방식: 세션 로그인", 결정마다 후보와 이유는 `docs/adr/adr-17-session-login.md`). 프론트 로그인 화면이 `POST /api/v1/user/login`으로 아이디·비밀번호를 보내면 서버가 세션을 만들어 HttpOnly 쿠키(`SESSION`)로 준다. 그 뒤 요청에는 브라우저가 쿠키를 자동으로 붙인다. 프론트는 토큰을 다루지 않는다. 로그아웃은 `POST /api/v1/user/logout`.
- 세션은 DB(`auth.spring_session`, Spring Session JDBC)에 둔다. 서버를 다시 켜도 로그인이 유지된다. **마지막 요청 뒤 8시간**이 지나면 끝나고 다시 로그인한다.
- 쿠키 `SESSION`은 HttpOnly·SameSite=Lax이고 HTTPS 요청이면 Secure다. 화면과 API는 같은 주소에서 쓴다(Nginx가 `/api`를 백엔드로 넘김). 그래서 CORS 설정이 없다. 개발 중에는 Vite 프록시로 같은 주소처럼 쓴다.
- CSRF: 쿠키를 자동으로 보내는 방식이라, 쓰기 요청(POST·PATCH·DELETE, 로그인·로그아웃 포함)은 `XSRF-TOKEN` 쿠키 값을 `X-XSRF-TOKEN` 헤더에 넣어 보낸다. 서버가 쿠키가 없는 응답마다 넣어 주고, 로그인하면 값을 바꾼다. 헤더가 없거나 다르면 403 `FORBIDDEN`.
- 없는 아이디·틀린 비밀번호는 모두 401 `LOGIN_FAILED`다. 어느 경우인지 알려 주면 있는 아이디를 알아낼 수 있어서 구분하지 않는다. 비밀번호는 맞는데 사용 중지된 계정은 403 `ACCOUNT_INACTIVE`다. 비밀번호를 아는 사람에게만 보이므로 아이디가 드러나지 않고, 다시 입력해도 소용없으니 관리자에게 문의하라고 알린다. 사용 중지 계정도 비밀번호가 틀리면 `LOGIN_FAILED`다 (2026-11-25). 성공하면 `users.last_login_at`을 기록하고 `Me`를 돌려준다.
- 세션에는 **사용자 ID와 로그인 시각만** 저장한다. 서버는 요청마다 DB에서 계정의 역할과 그 역할의 권한을 읽어 판단한다. 그래서 권한은 항상 DB의 최신 값이다 (2026-11-25). 프론트는 `Me`(로그인 응답, `GET /api/v1/user/me`)의 permissions로 메뉴를 그린다.
- `USER_MANAGE` 권한자가 계정의 역할·사용 여부·로그인 ID·비밀번호를 바꾸면 그 사용자의 `tokens_valid_after`(세션 무효 기준 시각, 컬럼 이름은 그대로 둔다)를 지금으로 기록한다(INV-80). 서버는 요청마다 계정을 DB에서 다시 보고, 로그인 시각이 그보다 이르거나 계정이 사용 중지·삭제됐으면 세션을 끊고 401 `UNAUTHENTICATED`로 막는다. 그 사용자는 다시 로그인하면 바뀐 권한으로 일한다. 역할에 넣은 권한을 바꾸는 것은 세션을 끊지 않고 그 역할인 사람들의 다음 요청부터 적용된다 (2026-11-26).
- 첫 관리자: 앱이 켜질 때 사용 중인 관리자가 없으면 환경 변수 `ADMIN_LOGIN_ID`·`ADMIN_PASSWORD`로 관리자 한 명을 만든다(`must_change_password = true`, 1.14). 개발 편의 기본값 admin/1234는 개발 PC 설정(`application-local.yml`, `local` 프로필)에만 있다. 그 밖에서는 관리자가 없는데 값이 없거나 비밀번호가 1234면 앱이 켜지지 않는다. "처음 로그인하면 비밀번호를 바꿔야 한다"는 기본 비밀번호를 아는 사람이 먼저 로그인하는 것을 막지 못하기 때문이다 (2026-11-25).
- 권한은 서버가 요청마다 확인한다(2026-11-26: 권한 13개를 역할로 묶고 계정은 역할 하나, [권한 매트릭스](https://app.notion.com/p/40ea7b147db34b08b8bdfff1001cc500) 1절). 화면에 안 보이는 기능을 다른 방법으로 요청해도 막힌다 ([사용자 계정과 권한](https://app.notion.com/p/40cb82ca31e5811dbdd1f86d13354231)).
- `COST_VIEW`가 없으면 응답에 금액 필드 자체가 없다 (INV-50).

### 1.4 공통 응답 형식과 오류 코드

모든 응답은 성공·실패 모두 같은 틀을 쓴다 (2026-11-23 결정).

| 필드 | 뜻 |
|---|---|
| `status` | HTTP 상태. 실제 HTTP 상태와 항상 같다. 오류를 200으로 보내지 않는다 |
| `code` | 우리 코드. 성공은 모두 `OK`, 실패는 오류 코드(문자열) |
| `message` | 사용자에게 보여 줄 한국어 메시지. 기본은 오류 코드의 기본 메시지이고, 필요하면 구체적으로 바꾼다. 입력 항목 메시지(`errors[].message`)도 한국어로 고정하고 요청의 `Accept-Language`를 따르지 않는다 (2026-11-25) |
| `data` | 성공이면 결과 데이터, 실패면 오류 상세(아래) |
| `warnings` | 참고용 경고가 있을 때만 붙는다. `[{ "code", "message" }]`. 예: 원가 미확인 |

**성공**

```json
{ "status": 200, "code": "OK", "message": "정상 처리했습니다", "data": { "id": 15, "itemCode": "01230850" } }
```

**목록**: `data` 안에 `items`와 페이지 정보를 넣는다 (페이지 규칙은 1.7).

```json
{ "status": 200, "code": "OK", "message": "정상 처리했습니다",
  "data": { "items": [ ], "page": 1, "size": 50, "totalCount": 132 } }
```

**오류**: `data`에 오류 상세를 넣는다.

```json
{ "status": 400, "code": "STOCK_INSUFFICIENT", "message": "재고가 부족합니다",
  "data": {
    "invariant": "INV-03",
    "errors": [ { "field": "lines[0].quantity", "code": "TOO_LARGE", "message": "재고보다 많습니다" } ],
    "details": { "location": "본사 지하 4층", "available": 2, "requested": 3 },
    "traceId": "8f2c…"
  } }
```

| 오류 상세 필드 | 뜻 |
|---|---|
| `invariant` | 불변 조건 위반이면 그 INV 번호. 아니면 없음 |
| `errors` | 입력 항목별 오류 목록. 여러 개를 한 번에 돌려준다. `field`는 `lines[0].quantity`처럼 요청 본문의 위치 |
| `details` | 막힌 이유의 구체 내용. 오류 코드마다 모양을 정해 둔다. 예: 과거 날짜 입력으로 재고가 음수가 되는 날(9.2), 사용 중지를 막은 남은 항목(#78), 실사 승인을 막은 배정 목록(12.3), 이미 있는 문서(#77) |
| `traceId` | 문의할 때 서버 로그를 찾는 번호 |

**HTTP 상태**: 바깥(모니터링·재시도·브라우저)에는 "누구 잘못인지"를, 우리 `code`는 "무슨 일인지"를 알린다. 업무 규칙 위반을 500으로 보내지 않는다. 서버 오류 집계에 섞이고 자동 재시도 대상이 되기 때문이다.

| HTTP 상태 | 쓰는 경우 |
|---|---|
| 200 / 201 | 성공 / 등록 성공 |
| 400 | 입력 오류 + 업무 규칙 위반(재고 부족, 마감된 달 등) + 저장 전 확인 필요 |
| 401 | 로그인 안 됨 |
| 403 | 권한 없음 |
| 404 | 대상 없음 |
| 405 / 415 | 지원하지 않는 메서드(`Allow` 헤더에 허용 메서드) / 지원하지 않는 요청 본문 형식(Content-Type). 개발 중 실수에서만 난다 (2026-11-25) |
| 409 | 다른 사람이 먼저 수정함(version), 같은 요청이 처리 중. 프론트는 최신 내용을 다시 불러온다 |
| 500 | 서버 오류 |

**저장 전 확인이 필요한 경고** (예: 같은 PDF를 다시 올림, #77): 첫 요청은 400 `CONFIRMATION_REQUIRED`를 돌려주고 `data.details.warnings`에 확인할 내용을 담는다. 사용자가 확인하면 프론트가 요청 본문에 `confirmedWarnings: ["FILE_DUPLICATE"]`를 붙여 다시 보낸다. 참고용 경고는 성공 응답의 `warnings`로만 알린다.

**오류 코드 규칙**

- 문자열(영문 대문자 스네이크)이고, 백엔드 `ErrorCode` enum의 이름을 그대로 보낸다. 숫자 코드는 쓰지 않는다. 프론트도 같은 이름을 TypeScript 타입으로 쓴다.
- enum 하나에 HTTP 상태와 기본 메시지를 함께 둔다. 공통 예외 처리기(`@RestControllerAdvice`)가 이 값으로 응답을 만든다.
- 이름으로 어느 영역인지 알 수 있게 한다. 예: `STOCK_…`, `SERIAL_…`, `LOAN_…`. `INV-`는 불변 조건 번호라 쓰지 않는다.
- 한 번 정한 코드 이름은 바꾸지 않는다. 뜻이 달라지면 새 코드를 만들고 예전 것은 "사용 안 함"으로 표시한다.
- 업무 코드는 API를 쓸 때 추가하고 아래 목록에 올린다.

**오류 코드 목록** (공통 코드는 확정, 업무 코드는 예시로 몇 개만 먼저 적음)

| 코드 | HTTP | 기본 메시지 | 구분·관련 |
|---|---|---|---|
| `OK` | 200 / 201 | 정상 처리했습니다 | 공통 (성공) |
| `VALIDATION_FAILED` | 400 | 입력 내용을 확인해 주세요 | 공통. `errors`에 항목별 오류 |
| `CONFIRMATION_REQUIRED` | 400 | 확인이 필요합니다 | 공통. 확인 후 다시 보내기 |
| `UNAUTHENTICATED` | 401 | 로그인이 필요합니다 | 공통 |
| `LOGIN_FAILED` | 401 | 아이디 또는 비밀번호가 맞지 않습니다 | 로그인. 없는 아이디·틀린 비밀번호를 구분하지 않는다 (2026-11-25 추가) |
| `ACCOUNT_INACTIVE` | 403 | 사용 중지된 계정입니다. 관리자에게 문의해 주세요 | 로그인. 비밀번호가 맞을 때만 (2026-11-25 추가) |
| `FORBIDDEN` | 403 | 권한이 없습니다 | 공통. CSRF 토큰이 없거나 틀려도 이것 (1.3) |
| `PASSWORD_CHANGE_REQUIRED` | 403 | 비밀번호를 바꾼 뒤 사용할 수 있습니다 | 공통. 1.14 (2026-11-25 목록에 추가) |
| `NOT_FOUND` | 404 | 대상을 찾을 수 없습니다 | 공통 |
| `METHOD_NOT_ALLOWED` | 405 | 지원하지 않는 요청 방식입니다 | 공통. 2026-11-25 추가 |
| `UNSUPPORTED_MEDIA_TYPE` | 415 | 지원하지 않는 요청 형식입니다 | 공통. 2026-11-25 추가 |
| `VERSION_CONFLICT` | 409 | 다른 사용자가 먼저 수정했습니다. 다시 불러와 주세요 | 공통 |
| `REQUEST_IN_PROGRESS` | 409 | 같은 요청을 처리하고 있습니다 | 공통. INV-08 |
| `REQUEST_KEY_REUSED` | 400 | 이미 다른 요청에 쓴 키입니다 | 공통. INV-08, 2026-11-23 추가 |
| `INTERNAL_ERROR` | 500 | 서버 오류가 발생했습니다 | 공통. `traceId`로 문의 |
| `STOCK_INSUFFICIENT` | 400 | 재고가 부족합니다 | 예시. INV-03 |
| `STOCK_NEGATIVE_ON_PAST_DATE` | 400 | 지난 날짜 재고가 음수가 됩니다 | 예시. INV-14, `details`에 날짜 |
| `PERIOD_CLOSED` | 400 | 마감된 기간입니다 | 예시. INV-15·46 |
| `SERIAL_DUPLICATE` | 400 | 이미 등록된 시리얼입니다 | 예시. INV-17 |
| `DOCUMENT_NO_DUPLICATE` | 400 | 같은 문서번호가 이미 있습니다 | 예시. INV-31, `details`에 기존 문서 |

### 1.5 중복 요청 키

같은 요청이 두 번 와도 한 번만 반영한다 (INV-08, 2026-11-23 결정). 저장 버튼을 두 번 누르거나 네트워크가 끊겨 다시 보내는 경우를 막는다.

- 헤더 `Idempotency-Key`로 보낸다. 프론트가 "저장"을 누를 때 UUID를 하나 만들고, 다시 보낼 때는 같은 값을 쓴다. 서버는 `common.idempotency_keys`에 키·요청자·동작·요청 내용 해시·처음 응답을 저장하고, 재고 처리 묶음을 만드는 동작은 `stock_transactions.request_id`에도 같은 값을 넣는다 (2026-11-23, [테이블 정의서](https://app.notion.com/p/408b82ca31e58080bd5deb903f3026c6) 5.13).
- 재고·장비·문서 상태가 바뀌는 **업무 동작**(입고·출고·교체·승인 등)에는 필수다. 없으면 400 `VALIDATION_FAILED`. 일반 등록·수정에는 선택이다. API 목록의 "중복 요청 키" 칸에 표시한다.
- 같은 키로 다시 오면, 이미 끝난 요청은 처음 결과를 그대로 돌려주고, 처리 중이면 409 `REQUEST_IN_PROGRESS`를 돌려준다.
- 같은 키에 요청자·동작·요청 내용 중 하나라도 다르면 400 `REQUEST_KEY_REUSED`. 프론트는 새 요청마다 키를 새로 만든다.
- 업무 오류(4xx)로 끝난 요청도 그 응답을 저장해 같은 키면 같은 오류를 돌려준다. 서버 오류(500)면 키를 지워 같은 키로 다시 보낼 수 있다. 키는 7일 동안 보관한다.

### 1.6 동시 수정 (version)

- 수정(PATCH)과 삭제(DELETE)는 요청 본문에 `version`을 함께 보낸다. 조회 응답에 들어 있는 값을 그대로 돌려보낸다 (감사 컬럼 `version`, 2026-11-23 결정).
- 그사이 다른 사람이 바꿨으면 409 `VERSION_CONFLICT`. 프론트는 최신 내용을 다시 불러온다.
- 입고·출고 같은 업무 동작에는 필요 없다. 재고 잠금과 불변 조건이 막는다.

### 1.7 목록 조회

| 항목 | 규칙 (2026-11-23 결정) |
|---|---|
| 페이지 번호 | **1부터** (`page=1`이 첫 페이지). Spring은 기본이 0부터라 설정으로 맞춘다(`spring.data.web.pageable.one-indexed-parameters=true`) |
| 한 페이지 건수 | `size` 기본 50, 최대 200 |
| 정렬 | `sort=businessDate,desc` 형식, 여러 개 가능 |
| 검색 조건 | 쿼리 파라미터 camelCase(`productId`, `businessDateFrom`, `businessDateTo`). 여러 값은 반복(`status=CONFIRMED&status=COMPLETED`) |
| 응답 | `data: { items, page, size, totalCount }`. `page`도 1부터 |

### 1.8 표기

| 대상 | 규칙 (2026-11-23 결정) | 예 |
|---|---|---|
| JSON 필드 이름 | camelCase (DB의 snake_case를 바꿔서) | `businessDate`, `unitPriceKrw` |
| 업무 날짜 | 날짜만 | `"2026-11-23"` |
| 기록 시각 | 시각과 시간대 포함 | `"2026-11-23T11:45:00+09:00"` |
| 수량 | 정수 숫자 | `3` |
| 단가·금액·환율 | **문자열**. JavaScript 숫자는 소수를 정확히 담지 못하므로 서버 값을 그대로 전달한다. 단가 소수 4자리, 금액은 그 통화의 소수 자리(최대 4, `currencies.minor_unit`), 원화 환산은 소수 2자리(COST-05, 2026-11-26 변경) | `"7000000.0000"`, `"1"` |
| 상태 같은 코드값 | 영문 코드 그대로. 한글 이름은 프론트 코드표로 표시 | `"RECEIPT"`, `"AVAILABLE"` |

### 1.9 등록·수정·삭제 (2026-11-23 결정)

- 사용자가 등록하는 모든 리소스에 목록·하나 조회·등록·수정·삭제 API를 둔다. 예외: 권한은 코드가 검사하므로 조회만(역할은 2026-11-26부터 만들기·수정·삭제 가능, 관리자 역할은 고정), 통화는 금액 기록이 참조하므로 삭제 없음.
- **삭제는 소프트 삭제**다(`deleted_at`, `deleted_by`). 삭제하면 목록·선택에서 사라지고 삭제 목록(trash)에서 복구한다. 복구하려는 값이 그사이 다른 데이터와 겹치면 복구할 수 없다.
- **이력이 있으면 삭제할 수 없다.** 더 안 쓰는 것은 사용 중지(`isActive: false`)한다. 사용 중지한 것은 목록에 보이고 다시 켤 수 있다. 삭제는 잘못 만든 것을 지울 때 쓴다 (확정 사항 #78과 같은 원칙).
- 코드·번호의 겹침 검사(Item Code, 품목군 코드, 창고 코드, 위치 코드, 사업자등록번호, 로그인 ID)는 삭제된 것을 빼고 한다. 거래 문서번호(INV-31)와 같다.
- 운영 제한은 최대한 열어 둔다 (2026-11-23): 로그인 실패 잠금 없음, 로그인 ID·품목군 코드 변경 가능, 재고가 남은 위치도 사용 중지 가능. 데이터가 틀어지는 변경(이력 있는 위치의 유형·창고·상대 업체, 통화 코드·소수 자리, 이력 있는 제품의 시리얼 관리 여부)과 마지막 관리자 보호만 막는다.
- 목록의 `isActive` 필터는 주지 않으면 둘 다 돌려준다. 고르는 화면(발주 품목 선택 등)은 `isActive=true`를 보낸다.
- 품목군·통화는 `MASTER_MANAGE` 권한자만 드물게 바꾸는 코드 표라서 `version`을 받지 않는다 (1.6의 예외).

**1단계 오류 코드** (모두 400)

| 코드 | 기본 메시지 | 언제 |
|---|---|---|
| `USER_LOGIN_ID_DUPLICATE` | 이미 있는 로그인 ID입니다 | 계정 만들기·수정 |
| `USER_LAST_ADMIN` | 사용 중인 관리자가 한 명은 있어야 합니다 | 마지막 관리자 역할 계정의 역할을 바꾸거나 중지·삭제 |
| `USER_SELF_CHANGE` | 자기 계정의 역할·사용 여부는 바꿀 수 없습니다 | 자기 역할·사용 여부를 바꾸거나 자기를 삭제 |
| `USER_HAS_HISTORY` | 로그인한 적이 있는 계정은 삭제할 수 없습니다. 사용 중지해 주세요 | 계정 삭제 |
| ~~`USER_VIEWER_COST_EDIT`~~ | — | 2026-11-26 없앰(#106). 만들기 전에 없앴으므로 쓰이지 않는다 |
| `ROLE_CODE_DUPLICATE` | 이미 있는 역할 코드입니다 | 역할 만들기 (2026-11-26) |
| `ROLE_NAME_DUPLICATE` | 이미 있는 역할 이름입니다 | 역할 만들기·수정 (2026-11-26) |
| `ROLE_LOCKED` | 관리자 역할은 바꾸거나 지울 수 없습니다 | 관리자 역할 수정·삭제 (2026-11-26, R1) |
| `ROLE_IN_USE` | 이 역할을 쓰는 계정이 있어 지울 수 없습니다 | 역할 삭제 (2026-11-26, R4) |
| `ROLE_COST_EDIT_WITHOUT_VIEW` | 금액 수정에는 금액 보기가 함께 있어야 합니다 | 역할 만들기·수정 (2026-11-26, R3, INV-88) |
| `PASSWORD_MISMATCH` | 현재 비밀번호가 맞지 않습니다 | 본인 비밀번호 변경 |
| `PRODUCT_ITEM_CODE_DUPLICATE` | 이미 있는 Item Code입니다 | 제품 등록·수정 (INV-31과 같은 방식) |
| `PRODUCT_IN_USE` | 남은 재고·문서·배정·대차가 있어 사용 중지할 수 없습니다 | 제품 사용 중지. `details`에 남은 항목 (INV-71) |
| `PRODUCT_HAS_HISTORY` | 이력이 있는 제품은 삭제할 수 없습니다. 사용 중지해 주세요 | 제품 삭제 |
| `PRODUCT_SERIAL_LOCKED` | 이력이 있는 제품은 시리얼 관리 여부를 바꿀 수 없습니다 | 제품 수정 (A3 확정 전이라 막아 둠) |
| `PRODUCT_CATEGORY_INACTIVE` | 사용 중지된 품목군입니다 | 제품 등록·품목군 변경 |
| `CATEGORY_CODE_DUPLICATE` / `CATEGORY_IN_USE` | 이미 있는 품목군 코드입니다 / 이 품목군의 제품이 있어 삭제할 수 없습니다 | 품목군 추가·수정 / 삭제 |
| `CURRENCY_CODE_DUPLICATE` / `CURRENCY_KRW_REQUIRED` | 이미 있는 통화입니다 / 원화는 끌 수 없습니다 | 통화 추가 / 원화 끄기 |
| `WAREHOUSE_CODE_DUPLICATE` / `WAREHOUSE_IN_USE` | 이미 있는 창고 코드입니다 / 이 창고에 위치가 있어 삭제할 수 없습니다 | 창고 등록·수정 / 삭제 |
| `LOCATION_CODE_DUPLICATE` | 이미 있는 위치 코드입니다 | 위치 등록·수정 |
| `LOCATION_TYPE_RULE` | 자사 창고 위치는 창고가, 외부 위치는 상대 업체가 필요합니다 | 위치 등록·수정 (`ck_locations_2`) |
| `LOCATION_HAS_HISTORY` | 이력이 있는 위치는 삭제할 수 없습니다. 사용 중지해 주세요 | 위치 삭제, 이력 있는 위치의 유형·창고·상대 업체 변경 |
| `PARTNER_BIZ_NO_DUPLICATE` | 이미 등록된 사업자등록번호입니다 | 거래처 등록·수정. `details`에 기존 거래처 |
| `PARTNER_HAS_HISTORY` | 이력이 있는 거래처는 삭제할 수 없습니다. 사용 중지해 주세요 | 거래처 삭제 |

비밀번호 규칙(길이·조합)은 두지 않는다(2026-11-23 확정, #100). 그래서 `PASSWORD_POLICY` 오류는 없다. 단, 새로 정하는 비밀번호(계정 만들기·비밀번호 초기화·본인 비밀번호 변경·첫 관리자)는 해시(BCrypt)가 다룰 수 있는 UTF-8 72바이트(영문 72자, 한글 24자)까지만 받는다. 규칙이 아니라 기술 한도이고, 넘으면 400 `VALIDATION_FAILED`다. 로그인은 길이를 보지 않는다(비교는 앞 72바이트로 하므로 오류가 나지 않는다) (2026-11-25). 초기화·새 계정의 비밀번호 변경 강제는 1.14절.

### 1.10 업무 동작 공통 검사와 2단계 오류 코드 (2026-11-23)

재고를 움직이는 모든 업무 동작은 같은 순서로 검사한다: 중복 요청 키(1.5) → 권한 → 업무 날짜(미래 금지, `RECORD_CORRECT` 없으면 7일, 마감된 달 금지) → 실사 잠금 → 재고 행 잠금 후 수량·배정·지난 날짜 음수 검사 → 장비 상태·날짜 순서. 하나라도 걸리면 묶음 전체가 저장되지 않는다(INV-07). 모두 400이다.

| 코드 | 언제 | 관련 |
|---|---|---|
| `BUSINESS_DATE_FUTURE` / `BUSINESS_DATE_TOO_OLD` | 미래 날짜 / `RECORD_CORRECT` 없이 7일보다 지난 날짜(`STOCK_ADJUST` 동작은 제외) | INV-15·16 |
| `PERIOD_CLOSED` | 마감된 달의 날짜로 처리·취소 | INV-15·46 |
| `STOCK_COUNT_LOCKED` | 실사 중인 위치·제품. `details`에 실사 번호 | INV-48 |
| `STOCK_INSUFFICIENT` / `STOCK_NEGATIVE_ON_PAST_DATE` | 재고 부족 / 지난 날짜 이후 어느 날 음수. `details`에 재고 키·날짜 | INV-03·14 |
| `ALLOCATION_SHORTAGE` | 처리 후 정상 재고가 배정보다 적어짐. `details`에 걸린 배정 | INV-35·36 |
| `ALLOCATED_TO_OTHER_PROJECT` | 출고하려는 재고·장비가 다른 프로젝트에 배정됨 | INV-35 |
| `EQUIPMENT_STATE_INVALID` / `EQUIPMENT_DATE_ORDER` | 장비 상태 전이 불가 / 장비 마지막 처리일보다 앞선 날짜 | INV-21·22 |
| `SERIAL_DUPLICATE` / `SERIAL_COUNT_MISMATCH` | 이미 있는 시리얼 / 수량과 장비 수 다름 | INV-17·23 |
| `PRODUCT_INACTIVE` / `LOCATION_INACTIVE` | 사용 중지된 제품 / 사용 중지된 위치를 도착지로 고름 | INV-70, 1.9 |
| `RECEIPT_EXCEEDS_REMAINING` | 발주 잔량보다 많이 연결. `details`에 잔량과 초과분 처리 방법 3가지 | INV-26, 8.2 |
| `TRADE_LINE_MISMATCH` / `TRADE_LINE_CLOSED` | 문서 품목과 제품·구분이 다름 / 마감된 품목·완료·취소 문서 | INV-27·28 |
| `LOAN_NO_PERMANENT_DELIVERY` | ③ 원물 반환 차입 장비를 납품 | INV-40 |
| `REVERSAL_NOT_ALLOWED` | 이미 취소된 줄, 취소 줄의 취소, 뒤 처리가 걸린 줄. `details`에 이유와 뒤 처리 목록 | INV-10·11·12 |
| `OPENING_NOT_ALLOWED` | 기준일과 다른 날짜, 첫 마감 뒤, 대차 문서 없는 타사·대여처 재고 | INV-58·60 |
| `INSPECTION_NOT_ALLOWED` | 불량 → 정상, 검수 대기·정상이 아닌 재고의 판정 | 5.2, #68 |
| `STOCK_COUNT_STATE_INVALID` | 실사 단계에 맞지 않는 요청 (예: 준비 안 끝낸 실사에 수량 입력) | 12.1 |
| `PERIOD_INTEGRITY_OPEN` / `PERIOD_ORDER` | 해결 안 된 점검 결과 / 앞 달이 열려 있는데 뒷달 마감(해제는 반대) | INV-78 |
| `REVIEW_ALREADY_DECIDED` | 이미 처리된 승인·확인 요청 | INV-77 |
| `PROJECT_STOCK_PROTECTED` / `PROJECT_STATUS_BLOCKED` / `PROJECT_HAS_DOCUMENTS` | 스탁 프로젝트 변경 / 완료·취소 조건 미충족 / 문서가 있어 삭제 불가 | INV-65·66 |
| `TRADE_DOCUMENT_STATE_INVALID` | 상태에 맞지 않는 수정·확정·삭제 | 11.1 |

### 1.11 3단계 오류 코드 (2026-11-23)

1.10의 검사 순서를 그대로 따른다. 모두 400이다.

| 코드 | 언제 | 관련 |
|---|---|---|
| `ALLOCATION_EXCEEDS_AVAILABLE` / `ALLOCATION_EXCEEDS_REMAINING` | 보유 배정이 배정 가능 수량을 넘음 / 예정 배정이 발주 잔량을 넘음. `details`에 최신 수량 | INV-35, #88 |
| `EQUIPMENT_ALREADY_ALLOCATED` | 다른 활성 배정이 있는 장비 | E2 |
| `TRADE_LINE_REDUCE_BLOCKED` / `TRADE_DOCUMENT_HAS_PROCESSING` | 처리·배정보다 작게 수량을 줄임 / 처리가 있는 문서의 취소·거래처·프로젝트 변경 | B8 |
| `DELIVERY_EXCEEDS_REMAINING` / `RETURN_EXCEEDS_DELIVERED` | 직납이 발주·매출 잔량을 넘음 / 반품이 납품·입고 수량을 넘음 | F2, G1, G2 |
| `REPAIR_EXCEEDS_SENT` / `DISPOSAL_NOT_ALLOWED` | 보낸 것보다 많이 수리 받음 / 불량·자사 창고가 아닌 재고 폐기 | G3, G6 |
| `LOAN_STATE_INVALID` / `LOAN_EXCEEDS_REMAINING` | 승인 전 대여 출고, 정산 뒤 변경, 방향·방식에 맞지 않는 처리 / 대차 품목의 남은 수량을 넘음 | INV-39, #89 |
| `BUNDLE_COMPONENT_DUPLICATE` | 한 묶음에 같은 구성품 두 줄 | A4 |
| `TRADE_LINE_DUPLICATE_PRODUCT` | 비교 화면 복사 대상 문서에 같은 제품 줄이 이미 있음. `details`에 기존 줄 | B6, #99 |
| `TRASH_RESTORE_CONFLICT` | 복구하면 유니크 값이 겹침. `details`에 겹치는 항목 | B9 |

자동 처리(오류 아님): 검수 불량·잔량 마감으로 배정이 풀리면 응답 `data.releasedAllocations[]`와 알림 `ALLOCATION_RELEASED`로 알린다(#88).

### 1.12 4단계: 금액 응답과 오류 코드 (2026-11-23)

- 금액 필드는 `COST_VIEW` 권한자에게만 보낸다. 권한이 없으면 null이 아니라 필드 자체를 뺀다(INV-83). 금액 넣기·고치기는 그 영역 권한과 `COST_EDIT`을 함께 가진 사람만(권한 매트릭스 1.4).
- 금액은 문자열(1.7). 합계·마진은 원화 환산(KRW) 기준, 품목 줄마다 원 단위 반올림 후 합계.
- 금액 변경은 cost_changes에 남고(#91), 단가·대여료·초기 재고 단가의 비우기는 행 삭제(#92).

| 코드 | 언제 | 관련 |
|---|---|---|
| `CURRENCY_INACTIVE` / `EXCHANGE_RATE_INVALID` | 켜지 않은 통화 / 원화인데 환율이 1이 아님, 환율 ≤ 0 | K4, P3 |
| `RENTAL_LOAN_REQUIRED` | 대여료 부대비용에 차입 대차 문서가 없음 | INV-82 |
| `WARRANTY_DATE_ORDER` | 보증 만료일이 시작일보다 앞선다. (외화인데 환율 기준일이 없으면 `VALIDATION_FAILED`, INV-84) | L1 |

### 1.13 ID는 UUID (2026-11-23)

- 모든 id와 id 참조(`productId`, `{id}` 등)는 UUID 문자열이다. 예: `"0193f5c2-7a1e-7c3d-9b2a-4e6f8d1a2b3c"`. 서버가 UUID v7로 만든다.
- UUID 형식이 아닌 id는 400 `VALIDATION_FAILED`, 형식은 맞지만 없는 id는 404.
- 이 문서의 예시에 남아 있는 짧은 숫자 id는 자리 표시다. 실제로는 UUID다.
- 사람이 보는 번호(프로젝트 코드, 처리 번호, 납품 번호 등)는 id와 별도다. 정렬 기본값은 id가 아니라 createdAt이나 번호다.

### 1.14 비밀번호 변경 필요 상태 (2026-11-23)

- `USER_MANAGE` 권한자가 계정을 만들었거나 비밀번호를 초기화한 사용자는 본인 비밀번호를 바꿀 때까지 `GET /api/v1/user/me`, `POST /api/v1/user/me/password`, 로그아웃만 쓸 수 있다. 나머지는 403 `PASSWORD_CHANGE_REQUIRED` (#100).
- `Me`·`User` 응답에 `mustChangePassword`(boolean)를 더한다. 비밀번호 규칙·주기적 변경·로그인 실패 잠금은 없다.

### 1.15 5단계: 증빙 첨부 (2026-11-24)

- PDF를 읽어 문서를 채우는 기능은 보류했다. 거래 문서는 사람이 입력하고 파일은 증빙으로만 붙는다 (#104). API는 `/api/v1/importing/attachment` 아래 4개(목록·올리기·열기·떼기), 복구는 삭제 목록 API(`entityType=ATTACHMENT`).
- 첫 대상은 거래 문서(`targetType=TRADE_DOCUMENT`) 하나다. 대상은 종류+ID라 나중에 납품·대차 등에 붙일 때는 종류 코드와 권한 규칙만 더한다.
- 형식 PDF·JPG·JPEG·PNG, 파일당 20MB. 확장자와 파일 앞부분 서명이 모두 맞아야 한다.
- 열기·내려받기는 `COST_VIEW`가 있는 사람만(INV-86). 목록(이름·크기·올린 사람)은 문서를 볼 수 있는 사람 모두.
- 같은 내용(SHA-256)이 이미 붙어 있으면 1.4의 저장 전 확인 방식으로 경고한다(`FILE_DUPLICATE`). 확인하면 별도 파일로 저장한다. 파일 이름은 판별에 쓰지 않는다.
- 저장 위치: 리눅스 서버 디스크. 최상위 폴더는 설정 `app.attachment.root`(기본 `/srv/asset/files`), 그 아래 `{대상 종류}/{연}/{월}/{파일ID}.{확장자}`. 예: `trade-document/2026/11/0193f5c2-….pdf`. DB에는 이 상대 경로만 둔다. 영구 보관하고 덮어쓰거나 지우지 않는다. 백업 대상에 이 폴더를 넣는다(ISS-10).

| 코드 | 언제 | 관련 |
|---|---|---|
| `FILE_EMPTY` / `FILE_TOO_LARGE` | 0바이트 파일 / 20MB 초과. `details`에 한도 | #104, `ck_files_3` |
| `FILE_TYPE_NOT_ALLOWED` | PDF·JPG·JPEG·PNG가 아니거나 확장자와 내용이 다름. `details`에 허용 형식 | #104, `ck_files_2` |
| `CONFIRMATION_REQUIRED` (`FILE_DUPLICATE`) | 같은 내용의 첨부가 이미 있음. `details.warnings`에 그 첨부가 붙은 문서 | #77, PO-03 |

## 2. API 목록

API 하나가 한 행이다. 행을 열면 개요 → 요청 → 응답 → 오류 → 하고 나면 바뀌는 것 → 예시 순으로 쓴다. 관련 가이드·요구사항·협의 이슈와 양방향으로 연결돼 있다.

[API 목록](https://app.notion.com/p/421b82ca31e581898e3fe58ea8d40ecb) (노션 데이터베이스)

## 3. 응답 객체

여러 API가 같이 쓰는 응답 모양이다. API 행에서는 이름(예: `Product`)으로 가리킨다. 모든 객체에 `createdAt`, `updatedAt`(기록 시각)이 있고, 수정·삭제할 수 있는 객체에는 `version`이 있다. 다른 객체를 가리킬 때는 `{ id, code 또는 이름 }`만 넣는다.

| 객체 | 필드 |
|---|---|
| `Me` | id, loginId, name, role{id, code, name}, permissions[권한 코드, 역할의 권한], mustChangePassword (2026-11-26 roles[] → role) |
| `User` | id, loginId, name, email, isActive, role{id, code, name}, lastLoginAt, createdAt, updatedAt, version (2026-11-26 roles[]·permissions[] → role) |
| `Role` | id, code, name, description, permissions[권한 코드], createdAt, updatedAt, version (2026-11-26) |
| `Permission` | code, name, description (2026-11-26) |
| `Product` | id, itemCode, productName, modelName, category{id, code, name}, manufacturer, componentRole, specification, unit, serialManaged, isActive, hasHistory, note, createdAt, updatedAt, version. `hasHistory`는 처리 이력이 있는지(삭제·Item Code 변경·시리얼 관리 변경 가능 여부를 화면이 미리 알기 위함) |
| `ProductChange` | id, fieldName, oldValue, newValue, reason, changedBy{id, name}, changedAt |
| `Category` | id, code, name, sortOrder, isActive |
| `Currency` | code, name, minorUnit(소수 자리), isActive |
| `Warehouse` | id, warehouseCode, warehouseName, address, isActive, note, createdAt, updatedAt, version |
| `Location` | id, locationCode, locationName, locationType(`INTERNAL`·`REPAIR`·`LOAN_OUT`·`CUSTOMER_SITE`), warehouse{id, warehouseCode, warehouseName} 또는 null, partner{id, partnerName} 또는 null, zone, rack, shelf, isActive, hasHistory, note, createdAt, updatedAt, version |
| `Partner` | id, partnerName, businessRegNo, address, phone, roles[`CUSTOMER`·`SUPPLIER`], isActive, note, primaryContact(`Contact` 또는 null), createdAt, updatedAt, version. 하나 조회에서는 contacts[`Contact`]도 넣는다 |
| `Contact` | id, name, department, position, phone, email, isPrimary, version |

**2단계 객체** (2026-11-23)

| 객체 | 필드 |
|---|---|
| `Project` | id, projectCode, projectName, customer{id, partnerName}, status(`ACTIVE`·`COMPLETED`·`CANCELLED`), isStock, note, createdAt, updatedAt, version |
| `TradeDocument` | id, project{id, projectCode, projectName}, tradeType(`PURCHASE`·`SALES`·`PRE_ORDER`), partner{id, partnerName}, documentNo, documentDate, status(`DRAFT`·`CONFIRMED`·`COMPLETED`·`CANCELLED`), note, attachmentCount(5단계), lines[`TradeLine`], createdAt, updatedAt, version |
| `TradeLine` | id, lineNo, product{id, itemCode, productName}, quantity, openingFulfilledQuantity, fulfilledQuantity, closedQuantity, remainingQuantity(계산), route(`WAREHOUSE`·`DIRECT`), note, version |
| `StockBalance` | product{id, itemCode, productName}, location{id, locationCode, locationName, locationType}, warehouse{id, warehouseName} 또는 null, stockCondition(`AVAILABLE`·`INSPECTION`·`DEFECTIVE`·`IN_REPAIR`), ownerType(`OWN`·`THIRD_PARTY`), ownerPartner{id, partnerName} 또는 null, quantity |
| `StockTransaction` | id, transactionNo, transactionType, businessDate, status, reasonCode, reason, referenceNo, counterpartyPartner, source{type, id}, relatedTransactionId, approvedBy, approvedAt, note, createdBy{id, name}, createdAt, lines[`LedgerLine`] (하나 조회에서만), reversedBy[{transactionId, lineNo}] |
| `LedgerLine` | id, lineNo, product, location, stockCondition, ownerType, ownerPartner, quantityDelta, equipment{id, serialNo} 또는 null, pairId, tradeLineId, reversalOfEntryId, itemCodeSnapshot, productNameSnapshot, note |
| `Equipment` | id, product, serialNo, serialStatus(`CONFIRMED`·`UNKNOWN`), status, currentLocation, ownerType, ownerPartner, project, deliveredPartner, replacedByEquipmentId, replacesEquipmentId, originType(`LEDGER`·`PRE_CUTOVER_DELIVERY`), lastEventDate, warranty{status, startDate, endDate} 또는 null, version |
| `EquipmentEvent` | id, eventType, businessDate, fromStatus, toStatus, fromLocation, toLocation, oldValue, newValue, source{type, id}, ledgerEntryId, note, createdBy{id, name}, createdAt |
| `StockCount` | id, countNo, scopeType(`LOCATION`·`PRODUCT`), scopes[{locationId, productId}], baseDate, assignee{id, name}, status(`PREPARED`·`IN_PROGRESS`·`REVIEW`·`APPROVED`·`CANCELLED`), startedAt, approvedBy, approvedAt, adjustmentTransactionId, note, lines[{id, product, location, stockCondition, ownerType, ownerPartner, bookQuantity, countedQuantity, difference, recounted, serials[{serialNo, equipmentId, result}]}] (하나 조회에서만), version |
| `PeriodClosing` | period(`"2026-11"`), status(`OPEN`·`CLOSED`), closedAt, closedBy{id, name}, snapshotVersion, events[{action, reason, actedBy, actedAt}] (하나 조회에서만) |
| `ReviewRequest` | id, requestType, kind(`APPROVAL`·`CONFIRM`, 계산), target{type, id}, status, summary, requestedBy{id, name}, requestedAt, decidedBy, decidedAt, decisionReason, version |
| `Notification` | id, notificationType, title, message, target{type, id}, createdAt, readAt |
| `IntegrityResult` | id, checkCode, targetType, targetKey, detail(점검마다 모양이 다른 JSON), firstFoundAt, lastFoundAt, resolvedAt |

**3단계 객체** (2026-11-23)

| 객체 | 필드 |
|---|---|
| `Bundle` | id, bundleCode, bundleName, isActive, components[{product, quantityPerSet}], note, hasHistory(하나 조회), version |
| `Allocation` | id, target{type(`PROJECT`·`LOAN_REPAY`), project 또는 loanDocument}, product, sourceType(`STOCK`·`RECEIVED`·`PLANNED`), sourceTradeLine, ownerType, ownerPartner, equipment, quantity, consumedQuantity, remainingQuantity(계산), status(`ACTIVE`·`RELEASED`·`CONSUMED`), note, createdBy, createdAt, events[](하나 조회), version |
| `Delivery` | id, deliveryNo, project, salesDocument, customer, deliveryDate, place, route(`WAREHOUSE`·`DIRECT`), bundle, bundleSetQuantity, status, transactionId, lines[{id, lineNo, product, quantity, returnedQuantity, salesLine, purchaseLine, loanLine, equipment}], returns[](하나 조회) |
| `LoanDocument` | id, loanNo, direction(`IN`·`OUT`), partner, project, settlementMethod(`IN_KIND`·`CONVERT`·`RETURN_ORIGINAL`), dueDate, overdueDays, status(`OPEN`·`SETTLED`·`CANCELLED`), approval{approvedBy, approvedAt}, convertedTradeDocument, lines[{id, lineNo, product, quantity, movedQuantity, settledQuantity, remainingQuantity, equipment}], settlements[]·changes[](하나 조회), note, version |
| `TrashItem` | entityType, entityId, label, deletedAt, deletedBy, reason, restorable |
| `ChangeHistory` | targetType, targetId, targetLabel, field, oldValue, newValue, reason, changedBy, changedAt |

**4단계 객체** (2026-11-23)

| 객체 | 필드 |
|---|---|
| `LinePrice` | tradeLineId, unitPrice, currency, exchangeRate, exchangeRateDate, unitPriceKrw, amountKrw, taxType |
| `ExtraCost` | id, project, costType(`SHIPPING`·`INSTALL`·`RENTAL`·`OTHER`), amount, currency, exchangeRate, exchangeRateDate, amountKrw, costDate, tradeDocument, loanDocument, note, version |
| `ProjectMargin` | planned{salesKrw, costKrw, extraCostKrw, marginKrw, marginRate}, actual{…}, warnings[], products[{product, purchase, sales, actualCost{basis, unknownQuantity}, marginKrw}] |
| `Warranty` | equipment{id, serialNo, product, status, deliveredPartner, project}, status(`CONFIRMED`·`UNKNOWN`), startDate, endDate, basis(`DELIVERY`·`MANUFACTURER`·`PURCHASE`·`MANUAL`), inheritedFrom{equipmentId, serialNo}, note, changes[](하나 조회), version |
| `CostChange` | targetType(`LINE_PRICE`·`EXTRA_COST`·`LOAN_FEE`·`OPENING_COST`), targetId, targetLabel, field, oldValue, newValue, reason, changedBy, changedAt |
| `Attachment` (5단계) | id, target{type, id, label}, originalName, contentType(`application/pdf`·`image/jpeg`·`image/png`), sizeBytes, uploadedBy{id, name}, uploadedAt, note, canOpen(요청자가 열 수 있는지). 위치 키·해시는 응답에 넣지 않는다 |
| `Home` | projectCounts{purchase, sales, preOrder}, categories[{category, warehouseQuantity, availableQuantity}], pendingReviewCount |
