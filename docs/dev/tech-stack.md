# 프론트엔드 기술 스택

> 원본(노션): https://app.notion.com/p/425b82ca31e5815eba66ff0b66bd1d51  
> 내보낸 날짜: 2026-11-27. **노션이 원본이다.** 이 파일과 노션이 다르면 노션을 따른다.

> 🎨 프론트엔드에서 쓰는 기술과 버전, 쓰지 않는 기술이다 (2026-11-27, ADR-18). 기준은 세 가지다: 사내 업무 시스템(표·입력 폼 중심, 한국어, 한 대의 서버)에 맞을 것, 생소하지 않을 것, 쓰는 사람이 많아 유지보수가 쉬울 것. 코드 규칙은 [프론트엔드 개발 컨벤션](conventions.md), 화면별 기능은 [화면 목록](../screens/screen-list.md)을 본다.

## 1. 기술과 버전

버전은 2026-11-27 기준 최신 안정판이다. 주 버전을 정하고, 정확한 버전은 설치할 때 `package-lock.json`으로 고정한다. 버전을 적지 않은 것은 설치 시점의 최신 안정판을 쓴다.

| 분류 | 기술 | 버전 | 쓰는 이유 |
|---|---|---|---|
| 실행 환경 | Node.js | 26 (Active LTS) | 개발 PC와 GitHub Actions에서 빌드할 때만 쓴다. 운영 서버는 빌드 결과(정적 파일)만 받으므로 Node가 필요 없다 |
| 패키지 관리 | npm | Node 포함 | 따로 설치할 것이 없고 가장 널리 쓴다 |
| 언어 | TypeScript | 6.x (strict) | 7.0은 나왔지만 도구용 API가 아직 없어 ESLint 등 주변 도구가 6을 필요로 한다. 7.1에서 API가 나오고 도구가 따라오면 다시 검토한다 |
| 화면 | React | 19.3 | 가장 많이 쓰는 화면 라이브러리(허브 기술 스택에서 이미 결정) |
| 빌드·개발 서버 | Vite | 8 | 빠른 개발 서버와 빌드. 개발 중 `/api` 요청을 백엔드로 넘긴다(4절) |
| 화면 이동 | React Router (데이터 모드) | 8 | React에서 가장 많이 쓰는 라우터. 브라우저에서만 도는 SPA로 쓰고 프레임워크 모드·서버 렌더링은 쓰지 않는다. 주소의 검색 조건은 Zod로 검사해 읽는다 |
| 서버 데이터 | TanStack Query | 5 | 캐시 공유, 업무 처리 뒤 무효화(컨벤션 1절) |
| 표 | TanStack Table | 8 | 서버 페이지 조회·정렬·열 보이기를 다루고, 그리는 것은 MUI 표 부품으로 한다. 금액 열을 권한에 따라 빼기 쉽다 |
| 화면 부품 | Material UI (MUI) + MUI X Date Pickers(무료 버전) | 9 | React 부품 라이브러리 중 가장 널리 쓴다. 자동완성(Autocomplete)이 "고르기", 날짜 막기(shouldDisableDate)가 "업무 날짜 선택기"에 맞고, 표·입력창을 조밀하게(UI-05) 만들 수 있다. 디자인은 나중에 테마로 입힌다 |
| 날짜 | Day.js | | MUI 날짜 부품의 어댑터. 업무 날짜는 Asia/Seoul 기준 |
| 입력 폼·검증 | React Hook Form + Zod | 7 / 4 | 품목 줄이 많은 폼(거래 문서·입고)을 다루기 쉽고 널리 쓴다. 화면은 형식만 검사하고 업무 규칙은 서버가 검사한다 |
| API 타입·호출 | openapi-typescript + openapi-fetch | | 백엔드 `openapi.json`에서 타입을 만들고, 그 타입으로 fetch를 부른다(3절). 만들어지는 코드가 적고 단순하다 |
| 테스트 | Vitest + React Testing Library, MSW, Playwright | MSW 2 | 부품·화면 테스트, 가짜 서버(백엔드 없이 화면 개발), 검수 시나리오 자동 테스트 |
| 코드 품질 | ESLint + typescript-eslint, Prettier | | 커밋 전 형식 정리, CI에서 검사 |

## 2. 쓰지 않는 것

| 기술 | 쓰지 않는 이유 |
|---|---|
| Next.js, React Router 프레임워크 모드, 서버 렌더링 | 로그인해야 쓰는 사내 시스템이라 검색 노출이 필요 없다. Nginx가 정적 파일만 주면 되고 서버에 Node를 두지 않는다 |
| Redux, Zustand 등 전역 상태 라이브러리 | 서버 데이터는 TanStack Query, 검색 조건은 주소, 로그인 정보는 내 정보 API로 충분하다 |
| axios | openapi-fetch가 브라우저 fetch를 쓴다 |
| Tailwind 등 다른 CSS 프레임워크, 다른 부품 라이브러리 | MUI 스타일과 섞으면 모양과 규칙이 두 갈래가 된다 |
| Moment.js | 더 이상 개발되지 않는다. Day.js를 쓴다 |
| 다국어 라이브러리 | 한국어만 쓴다 |
| 차트 라이브러리 | 지금 화면에 차트가 없다(홈은 숫자). 필요해지면 그때 정한다 |
| TypeScript 7 | 1절 이유. 7.1 이후 다시 검토 |

## 3. API 타입이 오는 길

백엔드와 프론트는 GitHub 저장소가 다르다. 프론트는 백엔드 코드를 보지 않고 OpenAPI 파일만 본다 (2026-11-27 확정).

1. 백엔드: API를 바꾸는 PR에 `docs/api/openapi.json`을 함께 커밋한다. 파일은 빌드가 만들고, CI가 커밋된 파일이 최신인지 비교한다. PR 리뷰에서 API 변경이 파일 차이로 보인다.
2. 프론트: `npm run gen:api`가 로컬의 `../backend/docs/api/openapi.json`을 `src/api/openapi.json`으로 복사하고 `src/api/schema.d.ts`를 만든다. 두 파일 모두 커밋한다. 어느 백엔드 버전 기준인지는 커밋 메시지에 백엔드 PR 번호로 적는다.
3. 프론트 CI는 커밋된 타입으로 타입 검사·테스트·빌드를 한다. 백엔드 저장소에 접근할 필요가 없다.

## 4. 개발 PC에서 같이 띄우기

- Vite 개발 서버가 `/api` 요청을 로컬 백엔드로 넘긴다(proxy). 화면과 API가 같은 주소가 되어 로그인 쿠키가 그대로 따라간다. CORS 설정이 필요 없다.
- 백엔드 없이 작업할 때는 MSW 가짜 서버를 켠다.

## 5. 저장소 구조

```txt
frontend/
├── CLAUDE.md
├── docs/                노션 사본 (api, rules, screens, dev)
├── src/
│   ├── main.tsx
│   ├── router.tsx       라우트 정의. 라우트 id = 화면 ID(S-41 등)
│   ├── theme.ts         MUI 테마 (디자인은 나중에)
│   ├── api/             openapi.json, schema.d.ts(생성), client.ts, invalidation.ts(무효화 목록)
│   ├── auth/            내 정보(useMe), 권한 상수, Can·useCan
│   ├── components/      공통 부품 (업무 버튼, 고르기, 업무 날짜 선택기, 오류 메시지, 목록 표)
│   └── screens/         화면 목록의 영역별 폴더
│       ├── common/      S-01–S-03
│       ├── home/        S-10
│       ├── master/      S-20–S-27
│       ├── project/     S-30–S-36
│       ├── inventory/   S-40–S-49
│       ├── allocation/  S-50–S-51
│       ├── delivery/    S-60–S-68
│       ├── loan/        S-70–S-72
│       ├── closing/     S-80–S-87
│       └── admin/       S-90–S-97
├── e2e/                 Playwright (검수 시나리오 AT-xx)
└── package.json
```

## 6. 배포

> ⚠️ **확정 전**: 빌드 결과(정적 파일)를 Nginx에 넣는 방법과 프론트 배포 절차는 첫 배포 전에 정한다(디렉토리 구조 1절). 서버에 Node를 두지 않고 빌드 결과만 올린다는 것까지만 정해져 있다(2절).
