# docs — 프론트엔드용 설계 문서 (노션 사본)

- 내보낸 날짜: **2026-11-27** (`api/`·`rules/`는 백엔드 `docs/`와 같은 내용)
- **노션이 원본이다.** 노션과 다르면 노션을 따르고, 차이를 발견하면 작업을 멈추고 알린다.
- 노션 허브: https://app.notion.com/p/408b82ca31e58069b467c00b694e7eee
- 프론트엔드 코드는 아직 없다. 기술 스택은 정해졌다(ADR-18, `dev/tech-stack.md`).

| 파일 | 노션 원본 | 담긴 내용 |
|---|---|---|
| `dev/tech-stack.md` | 프론트엔드 기술 스택 | 버전, 쓰지 않는 기술, API 타입이 오는 길(openapi.json), 개발 PC 실행, 저장소 구조 |
| `dev/conventions.md` | 프론트엔드 개발 컨벤션 | 서버 데이터·캐시 무효화, 화면 주소, API 타입, 권한 검사, 공통 부품 동작, 오류 메시지, 개발 방식 |
| `screens/screen-list.md` | 화면 목록 | 화면 59개의 하는 일·들어가는 권한·쓰는 API·근거, 메뉴 제안, API별로 쓰는 화면(17절, 이 파일에서 만든 표) |
| `screens/components.md` | 공통 컴포넌트 | 여러 화면이 같이 쓰는 부품 목록 |
| `api/api-spec.md` | API 명세서 | 공통 규칙(경로·세션 로그인과 CSRF·응답 형식·오류 코드·중복 요청 키·목록·첨부 등), 응답 객체 |
| `api/api-list.md` | API 목록 | API 색인(단계별)과 노션 행 링크 |
| `api/rows/` | API 목록의 각 행 | 비어 있음. 노션에서 "Markdown & CSV", 하위 페이지 포함으로 내보내 넣는다 |
| `rules/permissions.md` | 권한 매트릭스 | 권한 13개와 기본 역할, 권한별로 쓸 수 있는 API와 보이는 필드(금액은 `COST_VIEW`) |

- `api/`·`rules/`는 백엔드 저장소(`../backend/docs/`)의 같은 파일과 내용이 같다. 백엔드 쪽을 고치면 이 사본도 같이 맞춘다.
- `screens/screen-list.md`의 API 칸과 17절은 `api/api-list.md`에서 만든다. API 목록이 바뀌면 함께 다시 맞춘다.
- 화면 동작의 근거는 노션의 **기능별 가이드**와 **요구사항**을 본다(사본 없음).
- API 타입은 백엔드 저장소에 커밋된 `docs/api/openapi.json`에서 만든다(`dev/tech-stack.md` 3절).
