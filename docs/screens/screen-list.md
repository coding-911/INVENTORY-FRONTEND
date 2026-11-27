# 화면 목록

> 원본(노션): https://app.notion.com/p/425b82ca31e581c49f6bd4044e665efa  
> 내보낸 날짜: 2026-11-27. **노션이 원본이다.** 이 파일과 노션이 다르면 노션을 따른다. 17절(API별로 쓰는 화면)은 노션에 없고 이 파일에서 만든 표다.

> 🖥️ 화면마다 무엇을 하고, 어떤 API를 부르고, 누가 쓸 수 있는지를 정한 기능 목록이다 (화면 59개, API 156개). **레이아웃·디자인·화면 배치는 다루지 않는다.** 근거는 노션의 기능별 가이드, [API 목록](../api/api-list.md), [권한 매트릭스](../rules/permissions.md), 요구사항 명세서의 UI 항목이다.

> ⚠️ **확정 전**: 정해진 메뉴는 UI-01(홈·프로젝트 관리·창고 관리, 그리고 제품·거래처·삭제 항목 관리로 들어가는 길)뿐이다. 나머지 화면이 어느 메뉴 아래에 들어갈지, 두 화면을 한 화면으로 합칠지(예: 목록과 상세, 등록과 수정)는 프론트 개발을 시작할 때 정한다. 아래 3절의 메뉴 배치는 제안이다.

## 1. 읽는 법

- 화면 하나 = 사용자가 한 가지 일을 하는 단위. ID는 `S-번호`이고 앞자리가 영역이다(0 공통, 1 홈, 2 기준 정보, 3 프로젝트·거래 문서, 4 입고·재고, 5 배정, 6 출고·납품·문제 처리, 7 차입·대여, 8 정기 업무·처음 시작·금액, 9 관리·운영).
- **들어가는 권한**: 이 화면을 메뉴에 보여 줄 조건. 표기는 권한 매트릭스 1.8절과 같다(모두 = 로그인한 사람 누구나 / 권한 코드 = 그 권한이 있는 사람 / A 그리고 B). 조회가 되는 화면은 모두, 쓰기만 하는 화면은 그 쓰기 권한이다.
- **쓰는 API**: 화면이 부르는 API와 그 API의 권한 조건. 들어가는 권한보다 좁은 API는 해당 버튼·영역을 그 권한이 있는 사람에게만 보여 준다.
- 여러 화면에서 고르기용으로 쓰는 조회 API(제품·품목군·창고·위치·거래처·프로젝트·통화 목록)는 그 API가 주인공인 화면에만 적었다. 다른 입력 화면에서도 선택 목록으로 부른다.
- 근거 칸의 A1·B2 같은 코드는 노션 기능별 가이드, UI-xx는 요구사항이다.

## 2. 모든 화면 공통 규칙

- 화면에서 메뉴·버튼을 숨기는 것은 편의다. 권한은 서버가 요청마다 검사한다(권한 매트릭스 1절).
- 메뉴와 버튼은 `GET /api/v1/user/me`의 permissions(권한 코드 목록)로 정한다. 역할 이름으로 판단하지 않는다(역할은 `USER_MANAGE` 권한자가 바꿀 수 있는 묶음이다). 화면을 새로 열 때 이 API로 로그인 여부를 확인한다.
- `COST_VIEW` 권한이 없으면 응답에 금액 필드 자체가 없다. 화면은 금액 열·칸을 그리지 않는다(INV-83). 값이 비어 있는 것과 권한이 없는 것을 구분한다.
- 업무 날짜를 넣는 모든 화면: 오늘부터 7일 전까지. `RECORD_CORRECT`가 있으면 마감되지 않은 달까지. `STOCK_ADJUST` 동작(재고 조정·초기 재고·기준일 전 납품 장비 등록)은 7일 제한이 없다. 미래 날짜는 안 된다(가이드 I2, 권한 매트릭스 4절, `BUSINESS_DATE_TOO_OLD`).
- 재고를 움직이는 업무 동작은 `Idempotency-Key`가 필수다. 버튼을 한 번 누를 때 키를 하나 만들고, 다시 보낼 때는 같은 키를 쓴다(API 명세서 1.5).
- 수정 요청에는 화면이 받은 `version`을 함께 보낸다. 그사이 다른 사람이 고쳤으면 충돌 오류가 오므로 새로 읽게 한다(API 명세서 1.6).
- 승인이 필요한 동작(폐기, 대여 출고, 동급 모델 정산, 차이가 있는 실사 제출)은 누르면 "요청됨"이 되고 `APPROVE` 권한자가 승인한 뒤에 반영된다. 사후 확인 동작(발주 없이 입고, 대차 기한 변경, 확인된 시리얼 수정)은 누가 하든 바로 반영되고 확인 요청이 따로 생긴다(권한 매트릭스 4절).
- 실사 중인 범위의 제품은 다른 처리가 막힌다(INV-48). 막힌 이유를 오류 코드로 보여 준다.
- **주소(주소창)에 넣는 것**: 목록의 검색 조건(상태·기간·품목군·id·코드·검색어·페이지·정렬)과 상세 화면의 id는 주소에 넣는다. 새로고침·뒤로 가기·링크 공유가 되고, 홈 숫자를 눌러 조건 목록으로 가는 것(UI-02)도 링크로 된다. 주소를 알아도 권한은 서버가 검사하므로 데이터가 열리지 않는다. **토큰·비밀번호·비밀번호 초기화 키·개인정보(전화번호 등)는 주소에 넣지 않는다.** 주소는 방문 기록·서버 로그에 남기 때문이다. 주소가 외부 사이트로 넘어가지 않게 Nginx에 `Referrer-Policy: same-origin`을 둔다.

## 3. 메뉴 (제안)

| 메뉴 | 들어가는 화면 | 근거 |
|---|---|---|
| 홈 (주 메뉴, 확정) | S-10 | UI-01, 가이드 M1 |
| 프로젝트 관리 (주 메뉴, 확정) | S-30–S-36, S-50–S-51(배정), S-70–S-72(차입·대여) | UI-01. 배정·차입·대여를 여기 둘지는 제안 |
| 창고 관리 (주 메뉴, 확정) | S-40–S-49, S-60–S-68, S-80–S-81(실사) | UI-01. 출고·문제 처리·실사를 여기 둘지는 제안 |
| 제품 (진입점, 확정) | S-20–S-24 | UI-01 |
| 거래처 (진입점, 확정) | S-26–S-27 | UI-01 |
| 삭제 항목 관리 (진입점, 확정) | S-96 | UI-01 |
| 관리 (제안) | S-25, S-82–S-85, S-90–S-95, S-97 | 운영 화면 모음. 메뉴 이름·위치 미정 |
| 금액 (제안) | S-86–S-87 | `COST_VIEW` 권한자에게만 보이는 메뉴. 위치 미정 |
| 공통 | S-01–S-03 | 로그인 전·상단 공통 영역 |

UI-06에 따라 제품 선정·경쟁사 비교·추천 메뉴는 만들지 않는다.

## 4. 공통

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-01 | **로그인** | 아이디·비밀번호로 로그인하고 로그아웃한다 | 로그인 전 | [로그인·로그아웃](https://app.notion.com/p/421b82ca31e5811c9b4bf2988866f304) `POST` · 모두<br>[내 정보](https://app.notion.com/p/421b82ca31e581f5a9dde4ac17f79776) `GET` · 모두 | O2 | 로그인 직후 내 정보로 권한 목록(permissions)을 받아 메뉴를 만든다. 비밀번호 변경이 필요한 사용자는 S-02로 보낸다 |
| S-02 | **비밀번호 변경** | 본인 비밀번호를 바꾼다 | 모두 | [본인 비밀번호 변경](https://app.notion.com/p/421b82ca31e581c4987cdfe99e58da44) `POST` · 모두 | O2 | 새 계정·초기화 직후에는 이 화면과 로그아웃만 쓸 수 있다(나머지 API는 403 PASSWORD_CHANGE_REQUIRED) |
| S-03 | **알림** | 읽지 않은 알림 수를 보여 주고 알림 목록을 열어 읽음 처리한다 | 모두 | [내 알림 목록](https://app.notion.com/p/421b82ca31e581c3ac9cfeb7dac53cc4) `GET` · 모두<br>[알림 읽음 표시](https://app.notion.com/p/421b82ca31e581298181f48b93204922) `POST` · 모두 | O1, O5 | 화면 안 알림만 둔다(#98). 모든 화면 상단에 붙는 공통 영역이다 |

## 5. 홈

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-10 | **홈** | 프로젝트 수(매입·매출·선발주)와 품목군별 재고·배정 가능 수량을 본다. 숫자를 눌러 해당 목록으로 간다 | 모두 | [홈 화면 집계](https://app.notion.com/p/421b82ca31e58126a37ac501216a1268) `GET` · 모두 | M1, UI-01–04 | 보기만 한다. 프로젝트 수를 누르면 S-30, 품목군 재고를 누르면 S-40. 확정 전: UI-02 협의 필요 |

## 6. 기준 정보

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-20 | **제품 목록** | 제품을 찾고 등록 화면으로 간다 | 모두 | [제품 목록](https://app.notion.com/p/421b82ca31e5815281c6ea56861a7f22) `GET` · 모두<br>[품목군 목록](https://app.notion.com/p/421b82ca31e58171a1d3f727dbee00d7) `GET` · 모두 | A1 |  |
| S-21 | **제품 등록** | Item Code·제품명·품목군·시리얼 관리 여부를 넣어 제품을 만든다 | `PRODUCT_MANAGE` | [제품 등록](https://app.notion.com/p/421b82ca31e581bfb5b5ca6688e79ff0) `POST` · `PRODUCT_MANAGE` | A1, A3 |  |
| S-22 | **제품 상세** | 제품 정보를 보고 고치거나 지우고, 사용 중지·다시 사용, 변경 이력을 본다 | 모두 | [제품 하나](https://app.notion.com/p/421b82ca31e58117ac44fab5619de40e) `GET` · 모두<br>[제품 수정 ⚠️확정 전](https://app.notion.com/p/421b82ca31e581539055ff7c030b7cd4) `PATCH` · `PRODUCT_MANAGE`<br>[제품 삭제](https://app.notion.com/p/421b82ca31e5819ca427df70027d6b01) `DELETE` · `PRODUCT_MANAGE`<br>[제품 사용 중지](https://app.notion.com/p/421b82ca31e5811298b9cae9e543895b) `POST` · `MASTER_MANAGE`<br>[제품 다시 사용](https://app.notion.com/p/421b82ca31e58175aa28c80640d9f1a7) `POST` · `MASTER_MANAGE`<br>[제품 변경 이력](https://app.notion.com/p/421b82ca31e581a083e1f7da9d5c95b2) `GET` · `AUDIT_VIEW` | A1, A2, A3, O4 | 이력 있는 제품의 Item Code 변경은 `PRODUCT_MANAGE` 그리고 `RECORD_CORRECT`, 사유 필수(INV-72). 이력 있는 제품은 삭제 대신 사용 중지. 확정 전: 제품 수정 API 확정 전(A3) |
| S-23 | **구성 묶음 목록** | 구성 묶음(세트)을 찾는다 | 모두 | [구성 묶음 목록](https://app.notion.com/p/421b82ca31e581be97a4c5748ef5c78e) `GET` · 모두 | A4 |  |
| S-24 | **구성 묶음 상세** | 묶음을 만들고 구성품·세트당 수량을 고치거나 지운다 | 모두 | [구성 묶음 하나](https://app.notion.com/p/421b82ca31e581b2a5b1df475f37c51c) `GET` · 모두<br>[구성 묶음 만들기](https://app.notion.com/p/421b82ca31e581c388d7d72dddc01e19) `POST` · `MASTER_MANAGE`<br>[구성 묶음 수정](https://app.notion.com/p/421b82ca31e5819a8bdae1e5d6d34ad1) `PATCH` · `MASTER_MANAGE`<br>[구성 묶음 삭제](https://app.notion.com/p/421b82ca31e58140ab5bf6e4fed084bf) `DELETE` · `MASTER_MANAGE` | A4 | 묶음 자체에는 재고가 없다. 구성품을 펼쳐 보여 준다 |
| S-25 | **창고·위치 관리** | 창고와 보관 위치, 외부 위치(수리처·대여처·고객사 현장)를 등록·수정·삭제한다 | 모두 | [창고 목록](https://app.notion.com/p/421b82ca31e581a8bbfbc7e2d036084d) `GET` · 모두<br>[창고 하나](https://app.notion.com/p/421b82ca31e5818b8b34ead618857a12) `GET` · 모두<br>[창고 등록](https://app.notion.com/p/421b82ca31e5819185c8f27f835b468c) `POST` · `MASTER_MANAGE`<br>[창고 수정](https://app.notion.com/p/421b82ca31e5811ba5f6daab90fe27c8) `PATCH` · `MASTER_MANAGE`<br>[창고 삭제](https://app.notion.com/p/421b82ca31e581c494f0cdc060de62d0) `DELETE` · `MASTER_MANAGE`<br>[위치 목록](https://app.notion.com/p/421b82ca31e5815391f9e090560c08c7) `GET` · 모두<br>[위치 하나](https://app.notion.com/p/421b82ca31e581118220e918d73f0271) `GET` · 모두<br>[위치 등록](https://app.notion.com/p/421b82ca31e5813d8937fc4d149a2c4e) `POST` · `MASTER_MANAGE`<br>[위치 수정](https://app.notion.com/p/421b82ca31e581feba10c61fda87f526) `PATCH` · `MASTER_MANAGE`<br>[위치 삭제](https://app.notion.com/p/421b82ca31e581749572c392075524d7) `DELETE` · `MASTER_MANAGE` | A5 | 조회는 모두, 등록·수정·삭제는 `MASTER_MANAGE` |
| S-26 | **거래처 목록** | 거래처를 찾는다 | 모두 | [거래처 목록](https://app.notion.com/p/421b82ca31e581d0b0cfed044585962f) `GET` · 모두 | A6, B7 |  |
| S-27 | **거래처 등록·상세** | 거래처를 구분·담당자와 함께 등록하고, 수정·삭제하고 담당자를 관리한다. 연결된 프로젝트를 본다 | 모두 | [거래처 등록 (구분·담당자 함께)](https://app.notion.com/p/421b82ca31e58116be08e12aea345309) `POST` · `PARTNER_MANAGE`<br>[거래처 하나 (담당자 포함)](https://app.notion.com/p/421b82ca31e58197abe2dea4b504587d) `GET` · 모두<br>[거래처 수정 (구분 포함)](https://app.notion.com/p/421b82ca31e5811c98c8cd5c79a489f0) `PATCH` · `PARTNER_MANAGE`<br>[거래처 삭제](https://app.notion.com/p/421b82ca31e5818aa05aeef51dcbb480) `DELETE` · `PARTNER_MANAGE`<br>[담당자 추가](https://app.notion.com/p/421b82ca31e58172a4fcfed371073aef) `POST` · `PARTNER_MANAGE`<br>[담당자 수정](https://app.notion.com/p/421b82ca31e581e0806ee08cc34e90de) `PATCH` · `PARTNER_MANAGE`<br>[담당자 삭제](https://app.notion.com/p/421b82ca31e581dea469f667a470429d) `DELETE` · `PARTNER_MANAGE`<br>[프로젝트 목록](https://app.notion.com/p/421b82ca31e58173b92dec331201910a) `GET` · 모두 | A6, B7 | 지운 거래처는 지울 때와 같은 `PARTNER_MANAGE`로 되살린다(S-96). 확정 전: 거래처로 프로젝트 찾기(B7)에 연결된 API가 없다. 프로젝트 목록의 거래처 조건으로 할지 확인 필요 |

## 7. 프로젝트·거래 문서

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-30 | **프로젝트 목록** | 프로젝트를 찾고 새로 만든다 | 모두 | [프로젝트 목록](https://app.notion.com/p/421b82ca31e58173b92dec331201910a) `GET` · 모두<br>[프로젝트 만들기](https://app.notion.com/p/421b82ca31e58101af6ef5116ba4774b) `POST` · `DOCUMENT_MANAGE` | B1, UI-02 |  |
| S-31 | **프로젝트 상세** | 프로젝트 정보·상태를 고치거나 지우고, 연결된 거래 문서·배정·납품 진행과 금액·마진을 본다 | 모두 | [프로젝트 하나](https://app.notion.com/p/421b82ca31e5818e891dca89ac6c50c6) `GET` · 모두<br>[프로젝트 수정 (이름·고객사·상태)](https://app.notion.com/p/421b82ca31e58165a80ff7cd0803bb4e) `PATCH` · `DOCUMENT_MANAGE`<br>[프로젝트 삭제](https://app.notion.com/p/421b82ca31e58133810ecafeadf970e7) `DELETE` · `DOCUMENT_MANAGE`<br>[거래 문서 목록](https://app.notion.com/p/421b82ca31e5813c92fcfe3d74a402eb) `GET` · 모두<br>[배정 목록 (프로젝트 배정 현황)](https://app.notion.com/p/421b82ca31e58158918ced8bf7e4d1a3) `GET` · 모두<br>[납품 현황 (회차 목록)](https://app.notion.com/p/421b82ca31e581f4b0f2c4ffdcc6bcbc) `GET` · 모두<br>[프로젝트 금액·마진 (예정·실제, 품목별)](https://app.notion.com/p/421b82ca31e581e08629cb392c145835) `GET` · `COST_VIEW` | B1, K3, M1 | 발주·납품 진행은 이 화면에서 본다(M1). 금액·마진은 `COST_VIEW`가 있을 때만 부르고 보여 준다. 연결 문서가 없을 때만 삭제(INV-66) |
| S-32 | **거래 문서 목록** | 발주서·매출·선발주 문서를 찾는다 | 모두 | [거래 문서 목록](https://app.notion.com/p/421b82ca31e5813c92fcfe3d74a402eb) `GET` · 모두 | B2 | 응답에 금액 필드가 섞인다. `COST_VIEW`가 없으면 금액 열을 그리지 않는다 |
| S-33 | **거래 문서 등록·수정** | 문서를 품목·수량과 함께 등록하고, 확정 전 문서를 고친다 | `DOCUMENT_MANAGE` | [거래 문서 등록 (품목 포함)](https://app.notion.com/p/421b82ca31e5810e8e00fe3a695be147) `POST` · `DOCUMENT_MANAGE`<br>[거래 문서 수정](https://app.notion.com/p/421b82ca31e5817f8dbde2189d324cb4) `PATCH` · `DOCUMENT_MANAGE` | B2, B8, N1 | 등록만으로 재고는 바뀌지 않는다. 다른 프로젝트·거래처로 옮기기는 `RECORD_CORRECT`, 기준일 이전 처리 수량 입력은 `STOCK_ADJUST`가 함께 있어야 한다 |
| S-34 | **거래 문서 상세** | 문서와 품목·잔량을 보고 확정·취소·삭제, 잔량 마감, 매출로 복사, 단가 입력, 증빙 파일 첨부를 한다 | 모두 | [거래 문서 하나 (품목·잔량 포함)](https://app.notion.com/p/421b82ca31e581558cb4ccff6c6c7315) `GET` · 모두<br>[거래 문서 확정](https://app.notion.com/p/421b82ca31e5816d8ce4dd96de2462a8) `POST` · `DOCUMENT_MANAGE`<br>[거래 문서 취소](https://app.notion.com/p/421b82ca31e5816280d9de0921c99c78) `POST` · `DOCUMENT_MANAGE`<br>[거래 문서 삭제](https://app.notion.com/p/421b82ca31e581568b52fbd2dca1b6a3) `DELETE` · `DOCUMENT_MANAGE`<br>[거래 품목 잔량 마감](https://app.notion.com/p/421b82ca31e581d3bb1bc8540a02a6f5) `POST` · `CLOSING`<br>[매입을 매출로 복사](https://app.notion.com/p/421b82ca31e58140834fd1f683204e54) `POST` · `DOCUMENT_MANAGE`<br>[거래 문서 품목 단가 넣기·고치기·비우기](https://app.notion.com/p/421b82ca31e581dda05ecfa0716d31c3) `PATCH` · `DOCUMENT_MANAGE` 그리고 `COST_EDIT`<br>[첨부 파일 목록](https://app.notion.com/p/422b82ca31e58153bff9e05a4496cfd7) `GET` · 모두<br>[첨부 파일 올리기](https://app.notion.com/p/422b82ca31e58105a93af81e2e4f1540) `POST` · `DOCUMENT_MANAGE`<br>[첨부 파일 떼기](https://app.notion.com/p/422b82ca31e581fca0a3d2cab3eb3e0f) `DELETE` · `DOCUMENT_MANAGE`<br>[첨부 파일 열기·내려받기](https://app.notion.com/p/422b82ca31e581ddbb51df921c427a14) `GET` · `COST_VIEW` | B2, B4, B5, B8, B9, K1, K4 | 금액 필드는 `COST_VIEW`가 있을 때만 온다. 첨부 목록의 canOpen이 거짓이면 열기 버튼을 막는다. 완료된 문서는 메모(`DOCUMENT_MANAGE`)와 금액(`DOCUMENT_MANAGE` 그리고 `COST_EDIT`)만 바뀐다(INV-67). 확정 전: 단가 입력(K1) 가이드 확정 전 |
| S-35 | **매입·매출 비교** | 같은 프로젝트의 매입과 매출을 나란히 보고, 한쪽에만 있는 품목을 반대쪽으로 복사한다 | 모두 | [매입·매출 비교](https://app.notion.com/p/421b82ca31e58167a077ff0107ad77ab) `GET` · 모두<br>[한쪽에만 있는 품목 반대쪽으로 복사](https://app.notion.com/p/421b82ca31e581619cebdb97dc41a997) `POST` · `DOCUMENT_MANAGE` | B6 |  |
| S-36 | **부대비용** | 운송·설치비 같은 부대비용을 보고 넣고 고치고 지운다 | `COST_VIEW` | [부대비용 목록](https://app.notion.com/p/421b82ca31e5811d9e7fcded6a9792ac) `GET` · `COST_VIEW`<br>[부대비용 하나](https://app.notion.com/p/421b82ca31e581768495f6a1e9c02867) `GET` · `COST_VIEW`<br>[부대비용 넣기](https://app.notion.com/p/421b82ca31e5819c8450d814110bf9b8) `POST` · `DOCUMENT_MANAGE` 그리고 `COST_EDIT`<br>[부대비용 고치기](https://app.notion.com/p/421b82ca31e5812e8e3fe940a5e99a22) `PATCH` · `DOCUMENT_MANAGE` 그리고 `COST_EDIT`<br>[부대비용 삭제](https://app.notion.com/p/421b82ca31e581fd8f02dd65ea7ca98d) `DELETE` · `DOCUMENT_MANAGE` 그리고 `COST_EDIT` | K2 | 보기는 `COST_VIEW`, 넣기·고치기·삭제는 `DOCUMENT_MANAGE` 그리고 `COST_EDIT`. 확정 전: K2 가이드 확정 전. 따로 둘지 프로젝트 상세 안에 둘지 미정 |

## 8. 입고·재고

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-40 | **재고 목록** | 제품·위치·상태별 재고와 배정 수량을 본다 | 모두 | [재고 목록](https://app.notion.com/p/421b82ca31e581139e11d95d3ff22c03) `GET` · 모두 | D1, M1 | 홈 품목군 숫자와 이 화면의 같은 품목군 합계가 같아야 한다 |
| S-41 | **발주 입고** | 발주에 연결해 여러 품목을 한 번에(부분 입고 포함) 입고하고, 시리얼 제품은 한 대씩 번호를 넣는다 | `STOCK_OPERATE` | [발주 입고 (시리얼·부분 입고 포함)](https://app.notion.com/p/421b82ca31e5812e8d9ed05f9dae4448) `POST` · `STOCK_OPERATE`<br>[거래 문서 목록](https://app.notion.com/p/421b82ca31e5813c92fcfe3d74a402eb) `GET` · 모두 | C1, C2, C3, C5 | 대체품이면 발주 품목을 먼저 바꾼다(C5, S-33). 확정 전: 시리얼 입고(C2) 가이드 확정 전 |
| S-42 | **발주 없이 입고** | 발주 없는 물건·다른 물건·파손된 물건을 입고한다 | `STOCK_OPERATE` | [발주 없이 입고 (비정상 입고)](https://app.notion.com/p/421b82ca31e581108c75fb33ccf50fba) `POST` · `STOCK_OPERATE` | C4 | 누가 하든 사후 확인 요청이 함께 생기고, `APPROVE` 권한자가 S-90에서 확인한다 |
| S-43 | **위치 옮기기** | 창고·위치 사이에서 물건을 옮긴다 | `STOCK_OPERATE` | [위치 옮기기](https://app.notion.com/p/421b82ca31e5810a8305df31610eabf9) `POST` · `STOCK_OPERATE` | D3 |  |
| S-44 | **검수 판정** | 검수 대기 물건을 정상·불량으로 판정한다 | `STOCK_OPERATE` | [검수 판정](https://app.notion.com/p/421b82ca31e581d1a8e4f44cb6f2ff12) `POST` · `STOCK_OPERATE`<br>[재고 목록](https://app.notion.com/p/421b82ca31e581139e11d95d3ff22c03) `GET` · 모두 | D4 |  |
| S-45 | **장비 목록** | 시리얼로 장비를 찾는다 | 모두 | [장비 목록 (시리얼 찾기)](https://app.notion.com/p/421b82ca31e58190abdbf1112f5b48df) `GET` · 모두 | D2 |  |
| S-46 | **장비 상세** | 장비 한 대의 위치·상태·납품처·이력과 보증을 보고, 시리얼·메모·보증을 고치고 현장 수리를 기록한다 | 모두 | [장비 하나 (이력 포함)](https://app.notion.com/p/421b82ca31e5811bb99ce89ed8cf52fa) `GET` · 모두<br>[장비 시리얼·메모 수정](https://app.notion.com/p/421b82ca31e581a8a69af8cfea884055) `PATCH` · `STOCK_OPERATE`<br>[현장 수리 기록](https://app.notion.com/p/421b82ca31e581c9970de60cb6399111) `POST` · `STOCK_OPERATE`<br>[장비 보증 하나 (변경 이력 포함)](https://app.notion.com/p/421b82ca31e581dcaf4ec9ce62ec8ce4) `GET` · 모두<br>[장비 보증 넣기·고치기](https://app.notion.com/p/421b82ca31e5813cb73fea9600446859) `PATCH` · `STOCK_OPERATE` | D2, G5, L1, L2, N3 | 납품한 적 있는 장비의 시리얼은 `STOCK_OPERATE` 그리고 `RECORD_CORRECT`로 고친다. 확인된 시리얼을 바꾸면 누가 고치든 사후 확인 요청이 생긴다(INV-61·77). 확정 전: 시리얼 모르는 장비(N3) 가이드 확정 전 |
| S-47 | **보증 목록** | 만료일이 다가오거나 보증이 미확인인 장비를 찾는다 | 모두 | [보증 목록 (만료일·미확인 찾기)](https://app.notion.com/p/421b82ca31e58133b239d331d63cdaa1) `GET` · 모두 | L1 |  |
| S-48 | **처리 이력** | 처리 묶음별·품목별로 언제 누가 무엇을 처리했는지 찾는다 | 모두 | [처리 묶음 목록 (처리 이력)](https://app.notion.com/p/421b82ca31e5814bbb63dd560a67dd26) `GET` · 모두 | M2 |  |
| S-49 | **처리 상세** | 처리 묶음 하나와 줄을 보고, 잘못 넣은 처리를 취소하거나 정정한다 | 모두 | [처리 묶음 하나 (줄 포함)](https://app.notion.com/p/421b82ca31e581659081f3d08b8d1eda) `GET` · 모두<br>[처리 취소](https://app.notion.com/p/421b82ca31e581ccb512dd0c88ef126c) `POST` · `STOCK_OPERATE`<br>[처리 정정 (취소 + 다시 입력)](https://app.notion.com/p/421b82ca31e581b1a129d48fd320779e) `POST` · `STOCK_OPERATE` | I1, M2 | 지우지 않고 취소·정정한다 |

## 9. 배정

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-50 | **배정 현황** | 프로젝트별 배정을 보고 수량을 바꾸거나 다른 프로젝트로 옮기거나 푼다 | 모두 | [배정 목록 (프로젝트 배정 현황)](https://app.notion.com/p/421b82ca31e58158918ced8bf7e4d1a3) `GET` · 모두<br>[배정 하나 (이력 포함)](https://app.notion.com/p/421b82ca31e581d6a772f9432f5ac29e) `GET` · 모두<br>[배정 수량 바꾸기](https://app.notion.com/p/421b82ca31e581e5bcdffc91f3b8859b) `PATCH` · `STOCK_OPERATE`<br>[배정 다른 프로젝트로 옮기기](https://app.notion.com/p/421b82ca31e5810c9682d9dddd2693ab) `POST` · `STOCK_OPERATE`<br>[배정 해제](https://app.notion.com/p/421b82ca31e581b9be54cff9fb0f3c9a) `POST` · `STOCK_OPERATE` | E1, E3 |  |
| S-51 | **배정하기** | 배정 가능 수량·장비를 보고 보유 재고 또는 발주 예정 물량을 프로젝트에 배정한다 | `STOCK_OPERATE` | [배정 가능 수량·장비 조회](https://app.notion.com/p/421b82ca31e58111bd30c173d0547661) `GET` · 모두<br>[보유 재고 배정 (수량·장비)](https://app.notion.com/p/421b82ca31e581cebaa3e872d1df1c9f) `POST` · `STOCK_OPERATE`<br>[발주 예정 물량 배정](https://app.notion.com/p/421b82ca31e5815a843beea4722faac9) `POST` · `STOCK_OPERATE` | E1, E2, E3, B3 | 발주 예정 물량은 보유 재고와 구분해 보여 준다 |

## 10. 출고·납품

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-60 | **창고 출고** | 창고 재고를 고객사로 내보낸다 | `STOCK_OPERATE` | [창고 출고](https://app.notion.com/p/421b82ca31e58111ac4be64270d6c240) `POST` · `STOCK_OPERATE` | F1 | 확정 전: 출고(F1) 가이드 확정 전 |
| S-61 | **현장 직납** | 공급사에서 고객사로 바로 간 물건을 기록한다(차입 직납 포함) | `STOCK_OPERATE` | [현장 직납 기록 (차입 직납 포함)](https://app.notion.com/p/421b82ca31e5811abf2ff7e3dd61fd16) `POST` · `STOCK_OPERATE` | F2, H1 | 창고 재고는 바뀌지 않는다 |
| S-62 | **납품 현황** | 회차별 납품과 남은 수량을 본다 | 모두 | [납품 현황 (회차 목록)](https://app.notion.com/p/421b82ca31e581f4b0f2c4ffdcc6bcbc) `GET` · 모두<br>[납품 회차 하나](https://app.notion.com/p/421b82ca31e581dea3bdc98a3f3e072e) `GET` · 모두 | F3, F4, G1 | 구성품을 덜 보낸 회차는 "구성품 부족"으로 표시한다 |
| S-63 | **구성 묶음 납품** | 세트 단위로 날짜·장소를 한 번 넣어 납품한다 | `STOCK_OPERATE` | [구성 묶음 납품 (세트 출고)](https://app.notion.com/p/421b82ca31e581628bd9d1f0f9bfccb2) `POST` · `STOCK_OPERATE` | F4 |  |

## 11. 문제 처리

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-64 | **고객 반품 받기** | 고객 반품을 검수 대기로 받는다 | `STOCK_OPERATE` | [고객 반품 받기](https://app.notion.com/p/421b82ca31e581469f12db808f5ef068) `POST` · `STOCK_OPERATE` | G1 | 납품 수량이 줄어든다 |
| S-65 | **공급사에 반품** | 교환 요청 또는 반품 환불로 공급사에 돌려보낸다 | `STOCK_OPERATE` | [공급사에 반품](https://app.notion.com/p/421b82ca31e581ad8214ca79e0898d53) `POST` · `STOCK_OPERATE` | G2 |  |
| S-66 | **수리 보내기·받기** | 수리처로 보내고, 돌아오면(다른 시리얼 포함) 받는다 | `STOCK_OPERATE` | [수리 보내기](https://app.notion.com/p/421b82ca31e581a891ead2f9411a2435) `POST` · `STOCK_OPERATE`<br>[수리 받기 (다른 시리얼 포함)](https://app.notion.com/p/421b82ca31e581119c90e765e4a4a566) `POST` · `STOCK_OPERATE` | G3 |  |
| S-67 | **교체해 주기** | 불량품 회수와 대체품 출고를 한 번에 처리한다 | `STOCK_OPERATE` | [교체해 주기 (회수 + 대체품 출고)](https://app.notion.com/p/421b82ca31e5818db327ffd30ada0fa6) `POST` · `STOCK_OPERATE` | G4 |  |
| S-68 | **폐기 요청** | 사유를 넣고 폐기를 요청한다 | `STOCK_OPERATE` | [폐기 요청](https://app.notion.com/p/421b82ca31e5813cadfdff45a160fa40) `POST` · `STOCK_OPERATE` | G6 | `APPROVE` 권한자가 승인해야 반영된다(S-90) |

## 12. 차입·대여

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-70 | **대차 목록** | 차입·대여 문서를 찾고 기한 지난 대차를 본다 | 모두 | [대차 문서 목록 (기한 지난 목록 포함)](https://app.notion.com/p/421b82ca31e58186b438e10754c19349) `GET` · 모두 | H1, H6 |  |
| S-71 | **대차 문서 만들기·수정** | 차입·대여 문서를 만들고 고친다 | `STOCK_OPERATE` | [대차 문서 만들기 (차입·대여)](https://app.notion.com/p/421b82ca31e581b6b316feca2f7d6de9) `POST` · `STOCK_OPERATE`<br>[대차 문서 수정](https://app.notion.com/p/421b82ca31e5818fab32c5963ecec1df) `PATCH` · `STOCK_OPERATE` | H1, H4 |  |
| S-72 | **대차 문서 상세** | 대차와 정산·변경 이력을 보고, 받기·임시 설치·대여 출고·기한 변경·정산·정산 방식 변경·삭제, 대여료 입력을 한다 | 모두 | [대차 문서 하나 (정산·변경 이력 포함)](https://app.notion.com/p/421b82ca31e5810f92dff346ed5689b9) `GET` · 모두<br>[대차 문서 삭제](https://app.notion.com/p/421b82ca31e581e39133d539c6182c21) `DELETE` · `STOCK_OPERATE`<br>[차입 받기 (차입 입고·직접 임시 설치)](https://app.notion.com/p/421b82ca31e581438e74ea880414b3c8) `POST` · `STOCK_OPERATE`<br>[빌려 온 장비 임시 설치](https://app.notion.com/p/421b82ca31e58143bd10c511521b34c2) `POST` · `STOCK_OPERATE`<br>[대여 출고](https://app.notion.com/p/421b82ca31e58171ac2cf8bb3c17789f) `POST` · `STOCK_OPERATE`<br>[대차 기한 바꾸기](https://app.notion.com/p/421b82ca31e58158b3e5c5a4c6b12ecc) `POST` · `STOCK_OPERATE`<br>[대차 정산 (차입·대여 3가지 방식)](https://app.notion.com/p/421b82ca31e5812f8d55d53734b8ffe9) `POST` · `STOCK_OPERATE`<br>[정산 방식 바꾸기](https://app.notion.com/p/421b82ca31e581d68df7cc02d06ecd61) `POST` · `STOCK_OPERATE` 그리고 `RECORD_CORRECT`<br>[대여료 넣기·고치기·비우기 (빌려준 경우)](https://app.notion.com/p/421b82ca31e58116b02be148a6c1decc) `PATCH` · `STOCK_OPERATE` 그리고 `COST_EDIT` | H1–H6, K2 | 대여 출고·동급 모델 정산은 `APPROVE` 승인 후 반영, 기한 변경은 사후 확인 요청이 생긴다. 대여료는 `COST_VIEW`가 있을 때만 보인다. 확정 전: K2 가이드 확정 전 |

## 13. 정기 업무·처음 시작

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-80 | **실사 목록** | 실사를 찾고 새로 준비한다 | 모두 | [실사 목록](https://app.notion.com/p/421b82ca31e581f586e9e44a7609cc93) `GET` · 모두<br>[실사 준비 (만들기)](https://app.notion.com/p/421b82ca31e5814b9b87c12554bd950b) `POST` · `STOCK_OPERATE` | J1 |  |
| S-81 | **실사 상세** | 실사를 시작(잠금)하고 수량·시리얼을 넣어 제출하거나 취소한다 | 모두 | [실사 하나 (줄·시리얼 포함)](https://app.notion.com/p/421b82ca31e5811493d7feb02bbd04c8) `GET` · 모두<br>[실사 시작 (잠금)](https://app.notion.com/p/421b82ca31e581078c45f5d90f3ede5b) `POST` · `STOCK_OPERATE`<br>[실사 수량 입력](https://app.notion.com/p/421b82ca31e581efa51ae563863ec0b0) `PATCH` · `STOCK_OPERATE`<br>[실사 제출](https://app.notion.com/p/421b82ca31e581fe894edaeb07585868) `POST` · `STOCK_OPERATE`<br>[실사 취소](https://app.notion.com/p/421b82ca31e581069ab5f9804e03bd5d) `POST` · `STOCK_OPERATE` | J1 | 실사 중인 범위는 다른 처리가 막힌다(`RECORD_CORRECT`가 있으면 예외 처리, INV-48). 차이가 있으면 `APPROVE` 승인 후 반영 |
| S-82 | **월 마감** | 달별 마감 상태를 보고 마감·해제하고, 마감 전 정합성 점검을 돌린다 | 모두 | [월 마감 목록](https://app.notion.com/p/421b82ca31e581aaa06bfc1f65a8d51b) `GET` · 모두<br>[월 마감 하기](https://app.notion.com/p/421b82ca31e581c5b5a4e40838092c76) `POST` · `CLOSING`<br>[월 마감 해제](https://app.notion.com/p/421b82ca31e58188bea5cb6acbc03fb3) `POST` · `CLOSING`<br>[정합성 점검 지금 실행](https://app.notion.com/p/421b82ca31e581ac8236c41f39c1754e) `POST` · `AUDIT_VIEW` | J2, N4, I3 | 마감·해제는 `CLOSING`, 점검 실행은 `AUDIT_VIEW` |
| S-83 | **재고 조정** | 마감된 달의 오류를 이번 달 조정으로 바로잡는다 | `STOCK_ADJUST` | [재고 조정 (이번 달 조정)](https://app.notion.com/p/421b82ca31e581bb999cf3743045cb49) `POST` · `STOCK_ADJUST` | I3 |  |
| S-84 | **초기 재고 넣기** | 기준일 현재 재고를 초기 재고로 한 번 넣고 초기 단가를 넣는다 | `STOCK_ADJUST` | [초기 재고 넣기](https://app.notion.com/p/421b82ca31e58137aedee31705e33b4b) `POST` · `STOCK_ADJUST`<br>[초기 재고 단가 넣기·고치기·비우기](https://app.notion.com/p/421b82ca31e581dab6fdc95927b5a3f7) `PATCH` · `STOCK_ADJUST` 그리고 `COST_EDIT` | N2, N3 | 초기 단가는 `STOCK_ADJUST` 그리고 `COST_EDIT`(INV-74) |
| S-85 | **기준일 전 납품 장비 등록** | 보증이 남은 예전 납품 장비를 재고 없이 납품 완료로 등록한다 | `STOCK_ADJUST` | [기준일 전 납품 장비 등록](https://app.notion.com/p/421b82ca31e581149965e7f715074ffc) `POST` · `STOCK_ADJUST` | N5 |  |

## 14. 금액

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-86 | **창고 재고 평가액** | 창고 재고의 금액을 본다 | `COST_VIEW` | [창고 재고 평가액](https://app.notion.com/p/421b82ca31e5817eae09ffe8cafebdc3) `GET` · `COST_VIEW` | K3 |  |
| S-87 | **금액 변경 이력** | 단가·부대비용·대여료를 누가 언제 바꿨는지 본다 | `COST_VIEW` | [금액 변경 이력](https://app.notion.com/p/421b82ca31e5815b90d9e1b65e6043f4) `GET` · `COST_VIEW` | K1, O4 |  |

## 15. 관리·운영

| ID | 화면 | 하는 일 | 들어가는 권한 | 쓰는 API (권한) | 근거 | 알아둘 것 |
|---|---|---|---|---|---|---|
| S-90 | **승인·확인 대기함** | 승인·확인할 일을 모아 보고 승인·반려·확인한다 | 모두 | [승인·확인 대기함 목록](https://app.notion.com/p/421b82ca31e5810b8eb9d82dee799cdf) `GET` · 모두 (`APPROVE`면 전부)<br>[승인·확인 요청 하나](https://app.notion.com/p/421b82ca31e58194a3dad8e4a0f4d7be) `GET` · 모두 (`APPROVE`면 전부)<br>[승인](https://app.notion.com/p/421b82ca31e5813eb0ccf31711b71e1c) `POST` · `APPROVE`<br>[반려](https://app.notion.com/p/421b82ca31e58105a0b7e4b47ba15c4c) `POST` · `APPROVE`<br>[확인 (사후 확인)](https://app.notion.com/p/421b82ca31e581b0be96ca4aaaf1ba67) `POST` · `APPROVE` | O1, UI-07 | 자기가 낸 요청만 보이고 `APPROVE`가 있으면 전부 보인다. 승인·반려·확인은 `APPROVE`. 반려하면 요청자에게 알림. 확정 전: O1 가이드 확정 전, UI-07 협의 필요 |
| S-91 | **사용자 관리** | 계정을 만들고 역할·사용 여부를 바꾸고, 비밀번호를 초기화하고 지운다 | `USER_MANAGE` | [사용자 목록](https://app.notion.com/p/421b82ca31e5814f9187cbe31feb571b) `GET` · `USER_MANAGE`<br>[사용자 하나](https://app.notion.com/p/421b82ca31e5818a92e4d674f278dcda) `GET` · `USER_MANAGE`<br>[역할 목록](https://app.notion.com/p/421b82ca31e581eda913d6ec29659923) `GET` · 모두<br>[계정 만들기](https://app.notion.com/p/421b82ca31e5810f9f5ce2173c86476a) `POST` · `USER_MANAGE`<br>[계정 수정 (역할·사용 여부 포함)](https://app.notion.com/p/421b82ca31e5810db1ebd8793e79eb2c) `PATCH` · `USER_MANAGE`<br>[계정 삭제](https://app.notion.com/p/421b82ca31e581daba0fd798a319516a) `DELETE` · `USER_MANAGE`<br>[비밀번호 초기화](https://app.notion.com/p/421b82ca31e58186aca5c2a5534654a2) `POST` · `USER_MANAGE` | O2 | 자기 역할·사용 여부 변경, 자기 삭제, 마지막 관리자 역할 계정 삭제는 막는다(권한 매트릭스 1.6 R2). 계정의 역할을 바꾸면 그 계정의 세션이 끊긴다(R5). 역할 자체는 S-97에서 관리한다. 확정 전: O2 가이드 확정 전 |
| S-92 | **품목군 관리** | 품목군을 추가하고 이름·순서·사용 여부를 바꾸고 지운다 | `MASTER_MANAGE` | [품목군 목록](https://app.notion.com/p/421b82ca31e58171a1d3f727dbee00d7) `GET` · 모두<br>[품목군 추가](https://app.notion.com/p/421b82ca31e581f4a07ef2a60c9103cf) `POST` · `MASTER_MANAGE`<br>[품목군 수정 (이름·순서·사용 여부)](https://app.notion.com/p/421b82ca31e581338cefd75249c8cf47) `PATCH` · `MASTER_MANAGE`<br>[품목군 삭제](https://app.notion.com/p/421b82ca31e5812297c1cc28b322b7c6) `DELETE` · `MASTER_MANAGE` | O3 |  |
| S-93 | **통화 관리** | 통화를 추가하고 켜고 끈다 | `MASTER_MANAGE` | [통화 목록](https://app.notion.com/p/421b82ca31e581319c6af94efb6c80a4) `GET` · 모두<br>[통화 추가](https://app.notion.com/p/421b82ca31e5814eac28c515d8557d38) `POST` · `MASTER_MANAGE`<br>[통화 수정 (이름·켜기·끄기)](https://app.notion.com/p/421b82ca31e5812b862fdce833d06dcc) `PATCH` · `MASTER_MANAGE` | O3, K4 | 지금은 원화만 쓴다. 외화를 켜면 거래 문서에 통화·환율 칸이 보인다(K4) |
| S-94 | **변경 이력** | 문서·배정·장비·보증·마감을 누가 언제 바꿨는지 찾는다 | `AUDIT_VIEW` | [변경 이력 조회 (통합)](https://app.notion.com/p/421b82ca31e581b5b05ffd2fa61314b6) `GET` · `AUDIT_VIEW` | O4 |  |
| S-95 | **정합성 점검** | 매일 점검 결과를 보고 지금 다시 돌린다 | `AUDIT_VIEW` | [정합성 점검 결과 목록](https://app.notion.com/p/421b82ca31e58192ad6edb7066e78e0f) `GET` · `AUDIT_VIEW`<br>[정합성 점검 지금 실행](https://app.notion.com/p/421b82ca31e581ac8236c41f39c1754e) `POST` · `AUDIT_VIEW` | O5 |  |
| S-96 | **삭제 목록** | 지운 것을 보고 되살린다 | 모두 | [삭제 목록](https://app.notion.com/p/421b82ca31e581f89f95e9a802cbf770) `GET` · 모두<br>[삭제한 것 복구](https://app.notion.com/p/421b82ca31e5814fadabcce614fe47a0) `POST` · 삭제와 같은 권한 (권한 매트릭스 3.5) | B9, UI-01 | 목록은 모두 본다. 되살리기는 그것을 지우는 권한과 같다(거래처 `PARTNER_MANAGE`, 품목군·창고·위치·구성 묶음 `MASTER_MANAGE`, 첨부 `DOCUMENT_MANAGE`, 부대비용 `DOCUMENT_MANAGE` 그리고 `COST_EDIT`). 부대비용은 `COST_VIEW`가 있어야 보인다 |
| S-97 | **역할 관리** | 역할(권한 묶음)을 만들고 이름·설명·권한을 고치고 지운다 | `USER_MANAGE` | [역할 목록](https://app.notion.com/p/421b82ca31e581eda913d6ec29659923) `GET` · 모두<br>[역할 하나](https://app.notion.com/p/424b82ca31e581c9aceccbaddb5357cb) `GET` · 모두<br>[권한 목록](https://app.notion.com/p/424b82ca31e581efa2ebf99c147804ff) `GET` · 모두<br>[역할 만들기](https://app.notion.com/p/424b82ca31e581f4b5aaea20e692c8bd) `POST` · `USER_MANAGE`<br>[역할 수정 (이름·설명·권한)](https://app.notion.com/p/424b82ca31e581d1a006d990ed15cd08) `PATCH` · `USER_MANAGE`<br>[역할 삭제](https://app.notion.com/p/424b82ca31e581f3ab81d321880c2f41) `DELETE` · `USER_MANAGE` | O2, 권한 매트릭스 1.5·1.6 | 관리자 역할(ADMIN)은 고치거나 지울 수 없다(R1, ROLE_LOCKED). `COST_EDIT`을 고르면 `COST_VIEW`도 골라야 한다(R3). 그 역할인 계정이 있으면 지울 수 없다(R4). 권한을 바꾸면 다음 요청부터 적용된다(R5) |

## 16. 화면이 없는 가이드와 확인할 것

| 항목 | 내용 |
|---|---|
| 가이드 I2 지난 날짜로 입력 | 따로 화면이 없다. 업무 날짜를 넣는 모든 화면의 공통 규칙이다(2절). |
| 가이드 B7 거래처로 프로젝트 찾기 | 연결된 API가 없다. S-27에서 프로젝트 목록 API를 거래처 조건으로 부르는 방식인지 API 행 확인이 필요하다. |
| 가이드 O6 백업·복구·내보내기 | 운영 담당이 서버에서 하는 일이라 화면이 없다. 엑셀 내보내기를 화면에 둘지는 미정이다(협의 이슈). |
| 가이드 N1 처음 등록 순서 | 순서 안내다. 쓰는 화면은 S-20–S-27(기준 정보) → S-84(초기 재고) → S-33(진행 중인 거래 문서) → S-85다. |

점검(2026-11-27): API 목록의 156개가 모두 한 화면 이상에 들어 있다. API를 새로 만들거나 화면을 나누고 합치면 이 표를 함께 고친다.

## 17. API별로 쓰는 화면

API 목록 순서다. API를 바꾸면 여기서 영향받는 화면을 찾는다.

| 모듈 | API | 메서드·경로 | 권한 | 쓰는 화면 |
|---|---|---|---|---|
| cost | [통화 목록](https://app.notion.com/p/421b82ca31e581319c6af94efb6c80a4) | `GET` `/api/v1/cost/currency` | 모두 | S-93 |
| cost | [통화 추가](https://app.notion.com/p/421b82ca31e5814eac28c515d8557d38) | `POST` `/api/v1/cost/currency` | `MASTER_MANAGE` | S-93 |
| cost | [통화 수정 (이름·켜기·끄기)](https://app.notion.com/p/421b82ca31e5812b862fdce833d06dcc) | `PATCH` `/api/v1/cost/currency/{code}` | `MASTER_MANAGE` | S-93 |
| partner | [거래처 목록](https://app.notion.com/p/421b82ca31e581d0b0cfed044585962f) | `GET` `/api/v1/partner` | 모두 | S-26 |
| partner | [거래처 등록 (구분·담당자 함께)](https://app.notion.com/p/421b82ca31e58116be08e12aea345309) | `POST` `/api/v1/partner` | `PARTNER_MANAGE` | S-27 |
| partner | [담당자 수정](https://app.notion.com/p/421b82ca31e581e0806ee08cc34e90de) | `PATCH` `/api/v1/partner/contact/{id}` | `PARTNER_MANAGE` | S-27 |
| partner | [담당자 삭제](https://app.notion.com/p/421b82ca31e581dea469f667a470429d) | `DELETE` `/api/v1/partner/contact/{id}` | `PARTNER_MANAGE` | S-27 |
| partner | [거래처 하나 (담당자 포함)](https://app.notion.com/p/421b82ca31e58197abe2dea4b504587d) | `GET` `/api/v1/partner/{id}` | 모두 | S-27 |
| partner | [거래처 수정 (구분 포함)](https://app.notion.com/p/421b82ca31e5811c98c8cd5c79a489f0) | `PATCH` `/api/v1/partner/{id}` | `PARTNER_MANAGE` | S-27 |
| partner | [거래처 삭제](https://app.notion.com/p/421b82ca31e5818aa05aeef51dcbb480) | `DELETE` `/api/v1/partner/{id}` | `PARTNER_MANAGE` | S-27 |
| partner | [담당자 추가](https://app.notion.com/p/421b82ca31e58172a4fcfed371073aef) | `POST` `/api/v1/partner/{id}/contact` | `PARTNER_MANAGE` | S-27 |
| product | [제품 목록](https://app.notion.com/p/421b82ca31e5815281c6ea56861a7f22) | `GET` `/api/v1/product` | 모두 | S-20 |
| product | [제품 등록](https://app.notion.com/p/421b82ca31e581bfb5b5ca6688e79ff0) | `POST` `/api/v1/product` | `PRODUCT_MANAGE` | S-21 |
| product | [품목군 목록](https://app.notion.com/p/421b82ca31e58171a1d3f727dbee00d7) | `GET` `/api/v1/product/category` | 모두 | S-20, S-92 |
| product | [품목군 추가](https://app.notion.com/p/421b82ca31e581f4a07ef2a60c9103cf) | `POST` `/api/v1/product/category` | `MASTER_MANAGE` | S-92 |
| product | [품목군 수정 (이름·순서·사용 여부)](https://app.notion.com/p/421b82ca31e581338cefd75249c8cf47) | `PATCH` `/api/v1/product/category/{id}` | `MASTER_MANAGE` | S-92 |
| product | [품목군 삭제](https://app.notion.com/p/421b82ca31e5812297c1cc28b322b7c6) | `DELETE` `/api/v1/product/category/{id}` | `MASTER_MANAGE` | S-92 |
| product | [제품 하나](https://app.notion.com/p/421b82ca31e58117ac44fab5619de40e) | `GET` `/api/v1/product/{id}` | 모두 | S-22 |
| product | [제품 수정 ⚠️확정 전](https://app.notion.com/p/421b82ca31e581539055ff7c030b7cd4) | `PATCH` `/api/v1/product/{id}` | `PRODUCT_MANAGE` | S-22 |
| product | [제품 삭제](https://app.notion.com/p/421b82ca31e5819ca427df70027d6b01) | `DELETE` `/api/v1/product/{id}` | `PRODUCT_MANAGE` | S-22 |
| product | [제품 다시 사용](https://app.notion.com/p/421b82ca31e58175aa28c80640d9f1a7) | `POST` `/api/v1/product/{id}/activate` | `MASTER_MANAGE` | S-22 |
| product | [제품 변경 이력](https://app.notion.com/p/421b82ca31e581a083e1f7da9d5c95b2) | `GET` `/api/v1/product/{id}/change` | `AUDIT_VIEW` | S-22 |
| product | [제품 사용 중지](https://app.notion.com/p/421b82ca31e5811298b9cae9e543895b) | `POST` `/api/v1/product/{id}/deactivate` | `MASTER_MANAGE` | S-22 |
| user | [사용자 목록](https://app.notion.com/p/421b82ca31e5814f9187cbe31feb571b) | `GET` `/api/v1/user` | `USER_MANAGE` | S-91 |
| user | [계정 만들기](https://app.notion.com/p/421b82ca31e5810f9f5ce2173c86476a) | `POST` `/api/v1/user` | `USER_MANAGE` | S-91 |
| user | [내 정보](https://app.notion.com/p/421b82ca31e581f5a9dde4ac17f79776) | `GET` `/api/v1/user/me` | 모두 | S-01 |
| user | [본인 비밀번호 변경](https://app.notion.com/p/421b82ca31e581c4987cdfe99e58da44) | `POST` `/api/v1/user/me/password` | 모두 | S-02 |
| user | [역할 목록](https://app.notion.com/p/421b82ca31e581eda913d6ec29659923) | `GET` `/api/v1/user/role` | 모두 | S-91, S-97 |
| user | [권한 목록](https://app.notion.com/p/424b82ca31e581efa2ebf99c147804ff) | `GET` `/api/v1/user/permission` | 모두 | S-97 |
| user | [역할 만들기](https://app.notion.com/p/424b82ca31e581f4b5aaea20e692c8bd) | `POST` `/api/v1/user/role` | `USER_MANAGE` | S-97 |
| user | [역할 하나](https://app.notion.com/p/424b82ca31e581c9aceccbaddb5357cb) | `GET` `/api/v1/user/role/{id}` | 모두 | S-97 |
| user | [역할 수정 (이름·설명·권한)](https://app.notion.com/p/424b82ca31e581d1a006d990ed15cd08) | `PATCH` `/api/v1/user/role/{id}` | `USER_MANAGE` | S-97 |
| user | [역할 삭제](https://app.notion.com/p/424b82ca31e581f3ab81d321880c2f41) | `DELETE` `/api/v1/user/role/{id}` | `USER_MANAGE` | S-97 |
| user | [사용자 하나](https://app.notion.com/p/421b82ca31e5818a92e4d674f278dcda) | `GET` `/api/v1/user/{id}` | `USER_MANAGE` | S-91 |
| user | [계정 수정 (역할·사용 여부 포함)](https://app.notion.com/p/421b82ca31e5810db1ebd8793e79eb2c) | `PATCH` `/api/v1/user/{id}` | `USER_MANAGE` | S-91 |
| user | [계정 삭제](https://app.notion.com/p/421b82ca31e581daba0fd798a319516a) | `DELETE` `/api/v1/user/{id}` | `USER_MANAGE` | S-91 |
| user | [비밀번호 초기화](https://app.notion.com/p/421b82ca31e58186aca5c2a5534654a2) | `POST` `/api/v1/user/{id}/password-reset` | `USER_MANAGE` | S-91 |
| user | [로그인·로그아웃](https://app.notion.com/p/421b82ca31e5811c9b4bf2988866f304) | `POST` `/api/v1/user/login, /api/v1/user/logout` | 모두 | S-01 |
| warehouse | [창고 목록](https://app.notion.com/p/421b82ca31e581a8bbfbc7e2d036084d) | `GET` `/api/v1/warehouse` | 모두 | S-25 |
| warehouse | [창고 등록](https://app.notion.com/p/421b82ca31e5819185c8f27f835b468c) | `POST` `/api/v1/warehouse` | `MASTER_MANAGE` | S-25 |
| warehouse | [위치 목록](https://app.notion.com/p/421b82ca31e5815391f9e090560c08c7) | `GET` `/api/v1/warehouse/location` | 모두 | S-25 |
| warehouse | [위치 등록](https://app.notion.com/p/421b82ca31e5813d8937fc4d149a2c4e) | `POST` `/api/v1/warehouse/location` | `MASTER_MANAGE` | S-25 |
| warehouse | [위치 하나](https://app.notion.com/p/421b82ca31e581118220e918d73f0271) | `GET` `/api/v1/warehouse/location/{id}` | 모두 | S-25 |
| warehouse | [위치 수정](https://app.notion.com/p/421b82ca31e581feba10c61fda87f526) | `PATCH` `/api/v1/warehouse/location/{id}` | `MASTER_MANAGE` | S-25 |
| warehouse | [위치 삭제](https://app.notion.com/p/421b82ca31e581749572c392075524d7) | `DELETE` `/api/v1/warehouse/location/{id}` | `MASTER_MANAGE` | S-25 |
| warehouse | [창고 하나](https://app.notion.com/p/421b82ca31e5818b8b34ead618857a12) | `GET` `/api/v1/warehouse/{id}` | 모두 | S-25 |
| warehouse | [창고 수정](https://app.notion.com/p/421b82ca31e5811ba5f6daab90fe27c8) | `PATCH` `/api/v1/warehouse/{id}` | `MASTER_MANAGE` | S-25 |
| warehouse | [창고 삭제](https://app.notion.com/p/421b82ca31e581c494f0cdc060de62d0) | `DELETE` `/api/v1/warehouse/{id}` | `MASTER_MANAGE` | S-25 |
| admin | [정합성 점검 결과 목록](https://app.notion.com/p/421b82ca31e58192ad6edb7066e78e0f) | `GET` `/api/v1/admin/integrity-check` | `AUDIT_VIEW` | S-95 |
| admin | [정합성 점검 지금 실행](https://app.notion.com/p/421b82ca31e581ac8236c41f39c1754e) | `POST` `/api/v1/admin/integrity-check/run` | `AUDIT_VIEW` | S-82, S-95 |
| admin | [내 알림 목록](https://app.notion.com/p/421b82ca31e581c3ac9cfeb7dac53cc4) | `GET` `/api/v1/admin/notification` | 모두 | S-03 |
| admin | [알림 읽음 표시](https://app.notion.com/p/421b82ca31e581298181f48b93204922) | `POST` `/api/v1/admin/notification/read` | 모두 | S-03 |
| admin | [승인·확인 대기함 목록](https://app.notion.com/p/421b82ca31e5810b8eb9d82dee799cdf) | `GET` `/api/v1/admin/review` | 모두 (`APPROVE`면 전부) | S-90 |
| admin | [승인·확인 요청 하나](https://app.notion.com/p/421b82ca31e58194a3dad8e4a0f4d7be) | `GET` `/api/v1/admin/review/{id}` | 모두 (`APPROVE`면 전부) | S-90 |
| admin | [승인](https://app.notion.com/p/421b82ca31e5813eb0ccf31711b71e1c) | `POST` `/api/v1/admin/review/{id}/approve` | `APPROVE` | S-90 |
| admin | [확인 (사후 확인)](https://app.notion.com/p/421b82ca31e581b0be96ca4aaaf1ba67) | `POST` `/api/v1/admin/review/{id}/confirm` | `APPROVE` | S-90 |
| admin | [반려](https://app.notion.com/p/421b82ca31e58105a0b7e4b47ba15c4c) | `POST` `/api/v1/admin/review/{id}/reject` | `APPROVE` | S-90 |
| delivery | [창고 출고](https://app.notion.com/p/421b82ca31e58111ac4be64270d6c240) | `POST` `/api/v1/delivery/issue` | `STOCK_OPERATE` | S-60 |
| delivery | [발주 입고 (시리얼·부분 입고 포함)](https://app.notion.com/p/421b82ca31e5812e8d9ed05f9dae4448) | `POST` `/api/v1/delivery/receipt` | `STOCK_OPERATE` | S-41 |
| equipment | [장비 목록 (시리얼 찾기)](https://app.notion.com/p/421b82ca31e58190abdbf1112f5b48df) | `GET` `/api/v1/equipment` | 모두 | S-45 |
| equipment | [기준일 전 납품 장비 등록](https://app.notion.com/p/421b82ca31e581149965e7f715074ffc) | `POST` `/api/v1/equipment/pre-cutover-delivery` | `STOCK_ADJUST` | S-85 |
| equipment | [장비 하나 (이력 포함)](https://app.notion.com/p/421b82ca31e5811bb99ce89ed8cf52fa) | `GET` `/api/v1/equipment/{id}` | 모두 | S-46 |
| equipment | [장비 시리얼·메모 수정](https://app.notion.com/p/421b82ca31e581a8a69af8cfea884055) | `PATCH` `/api/v1/equipment/{id}` | `STOCK_OPERATE` | S-46 |
| inventory | [재고 조정 (이번 달 조정)](https://app.notion.com/p/421b82ca31e581bb999cf3743045cb49) | `POST` `/api/v1/inventory/adjustment` | `STOCK_ADJUST` | S-83 |
| inventory | [검수 판정](https://app.notion.com/p/421b82ca31e581d1a8e4f44cb6f2ff12) | `POST` `/api/v1/inventory/inspection-result` | `STOCK_OPERATE` | S-44 |
| inventory | [초기 재고 넣기](https://app.notion.com/p/421b82ca31e58137aedee31705e33b4b) | `POST` `/api/v1/inventory/opening` | `STOCK_ADJUST` | S-84 |
| inventory | [월 마감 목록](https://app.notion.com/p/421b82ca31e581aaa06bfc1f65a8d51b) | `GET` `/api/v1/inventory/period-closing` | 모두 | S-82 |
| inventory | [월 마감 하기](https://app.notion.com/p/421b82ca31e581c5b5a4e40838092c76) | `POST` `/api/v1/inventory/period-closing/{period}/close` | `CLOSING` | S-82 |
| inventory | [월 마감 해제](https://app.notion.com/p/421b82ca31e58188bea5cb6acbc03fb3) | `POST` `/api/v1/inventory/period-closing/{period}/reopen` | `CLOSING` | S-82 |
| inventory | [재고 목록](https://app.notion.com/p/421b82ca31e581139e11d95d3ff22c03) | `GET` `/api/v1/inventory/stock` | 모두 | S-40, S-44 |
| inventory | [실사 목록](https://app.notion.com/p/421b82ca31e581f586e9e44a7609cc93) | `GET` `/api/v1/inventory/stock-count` | 모두 | S-80 |
| inventory | [실사 준비 (만들기)](https://app.notion.com/p/421b82ca31e5814b9b87c12554bd950b) | `POST` `/api/v1/inventory/stock-count` | `STOCK_OPERATE` | S-80 |
| inventory | [실사 하나 (줄·시리얼 포함)](https://app.notion.com/p/421b82ca31e5811493d7feb02bbd04c8) | `GET` `/api/v1/inventory/stock-count/{id}` | 모두 | S-81 |
| inventory | [실사 취소](https://app.notion.com/p/421b82ca31e581069ab5f9804e03bd5d) | `POST` `/api/v1/inventory/stock-count/{id}/cancel` | `STOCK_OPERATE` | S-81 |
| inventory | [실사 수량 입력](https://app.notion.com/p/421b82ca31e581efa51ae563863ec0b0) | `PATCH` `/api/v1/inventory/stock-count/{id}/line` | `STOCK_OPERATE` | S-81 |
| inventory | [실사 시작 (잠금)](https://app.notion.com/p/421b82ca31e581078c45f5d90f3ede5b) | `POST` `/api/v1/inventory/stock-count/{id}/start` | `STOCK_OPERATE` | S-81 |
| inventory | [실사 제출](https://app.notion.com/p/421b82ca31e581fe894edaeb07585868) | `POST` `/api/v1/inventory/stock-count/{id}/submit` | `STOCK_OPERATE` | S-81 |
| inventory | [처리 묶음 목록 (처리 이력)](https://app.notion.com/p/421b82ca31e5814bbb63dd560a67dd26) | `GET` `/api/v1/inventory/stock-transaction` | 모두 | S-48 |
| inventory | [처리 묶음 하나 (줄 포함)](https://app.notion.com/p/421b82ca31e581659081f3d08b8d1eda) | `GET` `/api/v1/inventory/stock-transaction/{id}` | 모두 | S-49 |
| inventory | [처리 정정 (취소 + 다시 입력)](https://app.notion.com/p/421b82ca31e581b1a129d48fd320779e) | `POST` `/api/v1/inventory/stock-transaction/{id}/correction` | `STOCK_OPERATE` | S-49 |
| inventory | [처리 취소](https://app.notion.com/p/421b82ca31e581ccb512dd0c88ef126c) | `POST` `/api/v1/inventory/stock-transaction/{id}/reversal` | `STOCK_OPERATE` | S-49 |
| inventory | [위치 옮기기](https://app.notion.com/p/421b82ca31e5810a8305df31610eabf9) | `POST` `/api/v1/inventory/transfer` | `STOCK_OPERATE` | S-43 |
| inventory | [발주 없이 입고 (비정상 입고)](https://app.notion.com/p/421b82ca31e581108c75fb33ccf50fba) | `POST` `/api/v1/inventory/unplanned-receipt` | `STOCK_OPERATE` | S-42 |
| project | [프로젝트 목록](https://app.notion.com/p/421b82ca31e58173b92dec331201910a) | `GET` `/api/v1/project` | 모두 | S-27, S-30 |
| project | [프로젝트 만들기](https://app.notion.com/p/421b82ca31e58101af6ef5116ba4774b) | `POST` `/api/v1/project` | `DOCUMENT_MANAGE` | S-30 |
| project | [거래 문서 목록](https://app.notion.com/p/421b82ca31e5813c92fcfe3d74a402eb) | `GET` `/api/v1/project/trade-document` | 모두 | S-31, S-32, S-41 |
| project | [거래 문서 등록 (품목 포함)](https://app.notion.com/p/421b82ca31e5810e8e00fe3a695be147) | `POST` `/api/v1/project/trade-document` | `DOCUMENT_MANAGE` | S-33 |
| project | [거래 문서 하나 (품목·잔량 포함)](https://app.notion.com/p/421b82ca31e581558cb4ccff6c6c7315) | `GET` `/api/v1/project/trade-document/{id}` | 모두 | S-34 |
| project | [거래 문서 수정](https://app.notion.com/p/421b82ca31e5817f8dbde2189d324cb4) | `PATCH` `/api/v1/project/trade-document/{id}` | `DOCUMENT_MANAGE` | S-33 |
| project | [거래 문서 삭제](https://app.notion.com/p/421b82ca31e581568b52fbd2dca1b6a3) | `DELETE` `/api/v1/project/trade-document/{id}` | `DOCUMENT_MANAGE` | S-34 |
| project | [거래 문서 확정](https://app.notion.com/p/421b82ca31e5816d8ce4dd96de2462a8) | `POST` `/api/v1/project/trade-document/{id}/confirm` | `DOCUMENT_MANAGE` | S-34 |
| project | [프로젝트 하나](https://app.notion.com/p/421b82ca31e5818e891dca89ac6c50c6) | `GET` `/api/v1/project/{id}` | 모두 | S-31 |
| project | [프로젝트 수정 (이름·고객사·상태)](https://app.notion.com/p/421b82ca31e58165a80ff7cd0803bb4e) | `PATCH` `/api/v1/project/{id}` | `DOCUMENT_MANAGE` | S-31 |
| project | [프로젝트 삭제](https://app.notion.com/p/421b82ca31e58133810ecafeadf970e7) | `DELETE` `/api/v1/project/{id}` | `DOCUMENT_MANAGE` | S-31 |
| admin | [변경 이력 조회 (통합)](https://app.notion.com/p/421b82ca31e581b5b05ffd2fa61314b6) | `GET` `/api/v1/admin/change-history` | `AUDIT_VIEW` | S-94 |
| allocation | [배정 목록 (프로젝트 배정 현황)](https://app.notion.com/p/421b82ca31e58158918ced8bf7e4d1a3) | `GET` `/api/v1/allocation` | 모두 | S-31, S-50 |
| allocation | [보유 재고 배정 (수량·장비)](https://app.notion.com/p/421b82ca31e581cebaa3e872d1df1c9f) | `POST` `/api/v1/allocation` | `STOCK_OPERATE` | S-51 |
| allocation | [배정 가능 수량·장비 조회](https://app.notion.com/p/421b82ca31e58111bd30c173d0547661) | `GET` `/api/v1/allocation/available` | 모두 | S-51 |
| allocation | [발주 예정 물량 배정](https://app.notion.com/p/421b82ca31e5815a843beea4722faac9) | `POST` `/api/v1/allocation/planned` | `STOCK_OPERATE` | S-51 |
| allocation | [배정 하나 (이력 포함)](https://app.notion.com/p/421b82ca31e581d6a772f9432f5ac29e) | `GET` `/api/v1/allocation/{id}` | 모두 | S-50 |
| allocation | [배정 수량 바꾸기](https://app.notion.com/p/421b82ca31e581e5bcdffc91f3b8859b) | `PATCH` `/api/v1/allocation/{id}` | `STOCK_OPERATE` | S-50 |
| allocation | [배정 다른 프로젝트로 옮기기](https://app.notion.com/p/421b82ca31e5810c9682d9dddd2693ab) | `POST` `/api/v1/allocation/{id}/move` | `STOCK_OPERATE` | S-50 |
| allocation | [배정 해제](https://app.notion.com/p/421b82ca31e581b9be54cff9fb0f3c9a) | `POST` `/api/v1/allocation/{id}/release` | `STOCK_OPERATE` | S-50 |
| delivery | [납품 현황 (회차 목록)](https://app.notion.com/p/421b82ca31e581f4b0f2c4ffdcc6bcbc) | `GET` `/api/v1/delivery` | 모두 | S-31, S-62 |
| delivery | [구성 묶음 납품 (세트 출고)](https://app.notion.com/p/421b82ca31e581628bd9d1f0f9bfccb2) | `POST` `/api/v1/delivery/bundle` | `STOCK_OPERATE` | S-63 |
| delivery | [고객 반품 받기](https://app.notion.com/p/421b82ca31e581469f12db808f5ef068) | `POST` `/api/v1/delivery/customer-return` | `STOCK_OPERATE` | S-64 |
| delivery | [현장 직납 기록 (차입 직납 포함)](https://app.notion.com/p/421b82ca31e5811abf2ff7e3dd61fd16) | `POST` `/api/v1/delivery/direct` | `STOCK_OPERATE` | S-61 |
| delivery | [교체해 주기 (회수 + 대체품 출고)](https://app.notion.com/p/421b82ca31e5818db327ffd30ada0fa6) | `POST` `/api/v1/delivery/exchange` | `STOCK_OPERATE` | S-67 |
| delivery | [공급사에 반품](https://app.notion.com/p/421b82ca31e581ad8214ca79e0898d53) | `POST` `/api/v1/delivery/supplier-return` | `STOCK_OPERATE` | S-65 |
| delivery | [납품 회차 하나](https://app.notion.com/p/421b82ca31e581dea3bdc98a3f3e072e) | `GET` `/api/v1/delivery/{id}` | 모두 | S-62 |
| equipment | [현장 수리 기록](https://app.notion.com/p/421b82ca31e581c9970de60cb6399111) | `POST` `/api/v1/equipment/{id}/field-repair` | `STOCK_OPERATE` | S-46 |
| inventory | [폐기 요청](https://app.notion.com/p/421b82ca31e5813cadfdff45a160fa40) | `POST` `/api/v1/inventory/disposal` | `STOCK_OPERATE` | S-68 |
| inventory | [수리 받기 (다른 시리얼 포함)](https://app.notion.com/p/421b82ca31e581119c90e765e4a4a566) | `POST` `/api/v1/inventory/repair-in` | `STOCK_OPERATE` | S-66 |
| inventory | [수리 보내기](https://app.notion.com/p/421b82ca31e581a891ead2f9411a2435) | `POST` `/api/v1/inventory/repair-out` | `STOCK_OPERATE` | S-66 |
| loan | [대차 문서 목록 (기한 지난 목록 포함)](https://app.notion.com/p/421b82ca31e58186b438e10754c19349) | `GET` `/api/v1/loan` | 모두 | S-70 |
| loan | [대차 문서 만들기 (차입·대여)](https://app.notion.com/p/421b82ca31e581b6b316feca2f7d6de9) | `POST` `/api/v1/loan` | `STOCK_OPERATE` | S-71 |
| loan | [대차 문서 하나 (정산·변경 이력 포함)](https://app.notion.com/p/421b82ca31e5810f92dff346ed5689b9) | `GET` `/api/v1/loan/{id}` | 모두 | S-72 |
| loan | [대차 문서 수정](https://app.notion.com/p/421b82ca31e5818fab32c5963ecec1df) | `PATCH` `/api/v1/loan/{id}` | `STOCK_OPERATE` | S-71 |
| loan | [대차 문서 삭제](https://app.notion.com/p/421b82ca31e581e39133d539c6182c21) | `DELETE` `/api/v1/loan/{id}` | `STOCK_OPERATE` | S-72 |
| loan | [대차 기한 바꾸기](https://app.notion.com/p/421b82ca31e58158b3e5c5a4c6b12ecc) | `POST` `/api/v1/loan/{id}/due-date` | `STOCK_OPERATE` | S-72 |
| loan | [대여 출고](https://app.notion.com/p/421b82ca31e58171ac2cf8bb3c17789f) | `POST` `/api/v1/loan/{id}/issue` | `STOCK_OPERATE` | S-72 |
| loan | [차입 받기 (차입 입고·직접 임시 설치)](https://app.notion.com/p/421b82ca31e581438e74ea880414b3c8) | `POST` `/api/v1/loan/{id}/receive` | `STOCK_OPERATE` | S-72 |
| loan | [대차 정산 (차입·대여 3가지 방식)](https://app.notion.com/p/421b82ca31e5812f8d55d53734b8ffe9) | `POST` `/api/v1/loan/{id}/settle` | `STOCK_OPERATE` | S-72 |
| loan | [정산 방식 바꾸기](https://app.notion.com/p/421b82ca31e581d68df7cc02d06ecd61) | `POST` `/api/v1/loan/{id}/settlement-method` | `STOCK_OPERATE` 그리고 `RECORD_CORRECT` | S-72 |
| loan | [빌려 온 장비 임시 설치](https://app.notion.com/p/421b82ca31e58143bd10c511521b34c2) | `POST` `/api/v1/loan/{id}/temp-install` | `STOCK_OPERATE` | S-72 |
| product | [구성 묶음 목록](https://app.notion.com/p/421b82ca31e581be97a4c5748ef5c78e) | `GET` `/api/v1/product/bundle` | 모두 | S-23 |
| product | [구성 묶음 만들기](https://app.notion.com/p/421b82ca31e581c388d7d72dddc01e19) | `POST` `/api/v1/product/bundle` | `MASTER_MANAGE` | S-24 |
| product | [구성 묶음 하나](https://app.notion.com/p/421b82ca31e581b2a5b1df475f37c51c) | `GET` `/api/v1/product/bundle/{id}` | 모두 | S-24 |
| product | [구성 묶음 수정](https://app.notion.com/p/421b82ca31e5819a8bdae1e5d6d34ad1) | `PATCH` `/api/v1/product/bundle/{id}` | `MASTER_MANAGE` | S-24 |
| product | [구성 묶음 삭제](https://app.notion.com/p/421b82ca31e58140ab5bf6e4fed084bf) | `DELETE` `/api/v1/product/bundle/{id}` | `MASTER_MANAGE` | S-24 |
| project | [거래 문서 취소](https://app.notion.com/p/421b82ca31e5816280d9de0921c99c78) | `POST` `/api/v1/project/trade-document/{id}/cancel` | `DOCUMENT_MANAGE` | S-34 |
| project | [매입을 매출로 복사](https://app.notion.com/p/421b82ca31e58140834fd1f683204e54) | `POST` `/api/v1/project/trade-document/{id}/copy-to-sales` | `DOCUMENT_MANAGE` | S-34 |
| project | [거래 품목 잔량 마감](https://app.notion.com/p/421b82ca31e581d3bb1bc8540a02a6f5) | `POST` `/api/v1/project/trade-document/{id}/line/{lineId}/close` | `CLOSING` | S-34 |
| project | [매입·매출 비교](https://app.notion.com/p/421b82ca31e58167a077ff0107ad77ab) | `GET` `/api/v1/project/{id}/purchase-sales-comparison` | 모두 | S-35 |
| project | [한쪽에만 있는 품목 반대쪽으로 복사](https://app.notion.com/p/421b82ca31e581619cebdb97dc41a997) | `POST` `/api/v1/project/{id}/purchase-sales-comparison/copy-line` | `DOCUMENT_MANAGE` | S-35 |
| trash | [삭제 목록](https://app.notion.com/p/421b82ca31e581f89f95e9a802cbf770) | `GET` `/api/v1/trash` | 모두 | S-96 |
| trash | [삭제한 것 복구](https://app.notion.com/p/421b82ca31e5814fadabcce614fe47a0) | `POST` `/api/v1/trash/{entityType}/{id}/restore` | 삭제와 같은 권한 (권한 매트릭스 3.5) | S-96 |
| cost | [금액 변경 이력](https://app.notion.com/p/421b82ca31e5815b90d9e1b65e6043f4) | `GET` `/api/v1/cost/change` | `COST_VIEW` | S-87 |
| cost | [부대비용 목록](https://app.notion.com/p/421b82ca31e5811d9e7fcded6a9792ac) | `GET` `/api/v1/cost/extra-cost` | `COST_VIEW` | S-36 |
| cost | [부대비용 넣기](https://app.notion.com/p/421b82ca31e5819c8450d814110bf9b8) | `POST` `/api/v1/cost/extra-cost` | `DOCUMENT_MANAGE` 그리고 `COST_EDIT` | S-36 |
| cost | [부대비용 하나](https://app.notion.com/p/421b82ca31e581768495f6a1e9c02867) | `GET` `/api/v1/cost/extra-cost/{id}` | `COST_VIEW` | S-36 |
| cost | [부대비용 고치기](https://app.notion.com/p/421b82ca31e5812e8e3fe940a5e99a22) | `PATCH` `/api/v1/cost/extra-cost/{id}` | `DOCUMENT_MANAGE` 그리고 `COST_EDIT` | S-36 |
| cost | [부대비용 삭제](https://app.notion.com/p/421b82ca31e581fd8f02dd65ea7ca98d) | `DELETE` `/api/v1/cost/extra-cost/{id}` | `DOCUMENT_MANAGE` 그리고 `COST_EDIT` | S-36 |
| cost | [대여료 넣기·고치기·비우기 (빌려준 경우)](https://app.notion.com/p/421b82ca31e58116b02be148a6c1decc) | `PATCH` `/api/v1/cost/loan-fee/{loanDocumentId}` | `STOCK_OPERATE` 그리고 `COST_EDIT` | S-72 |
| cost | [초기 재고 단가 넣기·고치기·비우기](https://app.notion.com/p/421b82ca31e581dab6fdc95927b5a3f7) | `PATCH` `/api/v1/cost/opening-cost` | `STOCK_ADJUST` 그리고 `COST_EDIT` | S-84 |
| cost | [프로젝트 금액·마진 (예정·실제, 품목별)](https://app.notion.com/p/421b82ca31e581e08629cb392c145835) | `GET` `/api/v1/cost/project/{projectId}/margin` | `COST_VIEW` | S-31 |
| cost | [창고 재고 평가액](https://app.notion.com/p/421b82ca31e5817eae09ffe8cafebdc3) | `GET` `/api/v1/cost/stock-valuation` | `COST_VIEW` | S-86 |
| cost | [거래 문서 품목 단가 넣기·고치기·비우기](https://app.notion.com/p/421b82ca31e581dda05ecfa0716d31c3) | `PATCH` `/api/v1/cost/trade-document/{id}/line-price` | `DOCUMENT_MANAGE` 그리고 `COST_EDIT` | S-34 |
| dashboard | [홈 화면 집계](https://app.notion.com/p/421b82ca31e58126a37ac501216a1268) | `GET` `/api/v1/dashboard/home` | 모두 | S-10 |
| equipment | [보증 목록 (만료일·미확인 찾기)](https://app.notion.com/p/421b82ca31e58133b239d331d63cdaa1) | `GET` `/api/v1/equipment/warranty` | 모두 | S-47 |
| equipment | [장비 보증 하나 (변경 이력 포함)](https://app.notion.com/p/421b82ca31e581dcaf4ec9ce62ec8ce4) | `GET` `/api/v1/equipment/{id}/warranty` | 모두 | S-46 |
| equipment | [장비 보증 넣기·고치기](https://app.notion.com/p/421b82ca31e5813cb73fea9600446859) | `PATCH` `/api/v1/equipment/{id}/warranty` | `STOCK_OPERATE` | S-46 |
| importing | [첨부 파일 목록](https://app.notion.com/p/422b82ca31e58153bff9e05a4496cfd7) | `GET` `/api/v1/importing/attachment` | 모두 | S-34 |
| importing | [첨부 파일 올리기](https://app.notion.com/p/422b82ca31e58105a93af81e2e4f1540) | `POST` `/api/v1/importing/attachment` | `DOCUMENT_MANAGE` | S-34 |
| importing | [첨부 파일 떼기](https://app.notion.com/p/422b82ca31e581fca0a3d2cab3eb3e0f) | `DELETE` `/api/v1/importing/attachment/{id}` | `DOCUMENT_MANAGE` | S-34 |
| importing | [첨부 파일 열기·내려받기](https://app.notion.com/p/422b82ca31e581ddbb51df921c427a14) | `GET` `/api/v1/importing/attachment/{id}/content` | `COST_VIEW` | S-34 |
