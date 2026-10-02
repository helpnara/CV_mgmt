# 느린 나이테 (md_mgmt_tool) 프로젝트 프로파일

> 목적: 느린 길(Slow Way)을 같은 스택과 디자인으로 만들기 위한 참조 문서
> 출처: `helpnara/md_mgmt_tool` 저장소 (브랜치 `claude/team-task-management-tool-3d91g8`, 2026-10-02 기준 최신 커밋 "feat: 팀원 면담 기록 … (182)")
> 비밀값 없음. 환경 변수는 이름만 적었다.
> 추출일: 2026-10-02

---

## 1. 개요

**한 줄.** 과제별 수행 이력을 Markdown 파일로 관리하는 **로컬 실행 웹 도구**. 팀장 한 명이 자기 PC에서 `run.bat`으로 켜고 브라우저로 쓴다. 호스팅되는 서비스가 아니다.

**핵심 성격 (느린 길 설계에 직접 영향)**

| 성격 | 내용 |
|------|------|
| 실행 | `127.0.0.1:8000` 단독 실행. 인터넷 불필요. 외부로 나가는 요청 0 |
| 데이터 | `vault/` 폴더의 `.md` 파일 + front matter가 원본. Obsidian, VS Code에서 그대로 열림 |
| 색인 | SQLite는 검색·필터용 **파생 인덱스**. 지우면 재생성. 스키마 버전이 다르면 통째로 다시 만듦 |
| 사용자 | 1인. 인증 없음. 다중 사용자는 M6(선택)으로 구조만 남겨 둠 |
| AI | **AI를 호출하지 않는다.** 붙여넣기용 프롬프트를 만들어 복사만 제공 |
| 배포 | 윈도우 사내 PC(인터넷 차단, Node 없음) 대상. 빌드된 프론트와 wheel을 ZIP에 동봉 |

**규모 (2026-10-01 기준)**: API 끝점 105, 스키마 버전 16, 백엔드 테스트 파일 61개(약 614건), UI 체크 219건, TODO 182건, 개발 기간 2026-09-01 ~ 10-01.

**저장소 구조**

```text
md_mgmt_tool/
├── backend/app/
│   ├── api/        라우터 (activities, attachments, dashboard, drafts, entries, errors, export,
│   │               folders, home, intakes, linkfix, meetings, meta, people, projects, reports,
│   │               roadmap, search, settings, trash, versions)
│   ├── services/   비즈니스 로직 (ai_prompt, backup, buildinfo, errorlog, export, period, settings …)
│   ├── vault/      파일 계층 (markdown.py, paths.py, indexer.py, versions.py, schema.sql)
│   ├── config.py   상수(상태·속성·분류)와 Settings dataclass
│   ├── db.py       SQLite 열기, 스키마 버전, FTS5
│   ├── deps.py     요청별 커넥션, 쓰기 직렬화
│   └── main.py     lifespan, 오류 미들웨어, SPA 서빙
├── backend/tests/  pytest (conftest: 임시 vault + TestClient)
├── frontend/       React 18 + TS + Vite. src/components 70여 개. dist/ 커밋됨
├── tests/ui/       Playwright 화면 시험 (screens.mjs, contrast.mjs)
├── tools/make_dist.py   배포 ZIP
├── docs/           DESIGN, LESSONS-LEARNED, TODO, ROADMAP, 확산 로드맵 등
└── run.py / run.bat / run.sh / setup.py / setup.bat
```

**사용자 플로우 핵심**: 접수 등록 → 검토·판정 → 과제로 승격 → 진행일지 기록(첨부·엑셀 붙여넣기) → 월요일 보고 대상 선정 → 초안 생성 → 엑셀 셀로 복사해 보고 → 보고 확정(스냅샷) → 지시사항 기록 → 홈에서 연도별 성과 조회.

---

## 2. 기술 스택

| 영역 | 선택 | 근거 파일 |
|------|------|-----------|
| 백엔드 언어 | Python ≥ 3.10 (배포 대상 PC는 3.14) | `run.py`, `setup.py`, `tools/make_dist.py` |
| 백엔드 프레임워크 | FastAPI ≥ 0.115 + Uvicorn ≥ 0.30 (standard 제외) | `backend/requirements.txt` |
| 저장 | 파일시스템(`.md` + 첨부) + SQLite(WAL, FTS5 trigram, unicode61 폴백) | `db.py`, `schema.sql` |
| Markdown 처리 | python-frontmatter, PyYAML(관용 로더), markdown-it-py | `vault/markdown.py`, `services/export.py` |
| 기타 백엔드 | python-multipart, Pillow, openpyxl, watchdog, httpx(테스트) | `requirements.txt` |
| 프론트 | React 18.3, TypeScript 5.6, Vite 5.4, markdown-it 14 | `frontend/package.json` |
| 라우팅 | 해시 라우팅 직접 구현. 라이브러리 없음 | `App.tsx`, `nav.ts` |
| 상태 관리 | 없음. useState/useEffect + 모듈 싱글턴(notify, unsaved) | `App.tsx`, `notify.ts` |
| 스타일 | 순수 CSS 한 파일(`styles.css`), CSS 변수 토큰. UI 라이브러리 없음 | `frontend/src/styles.css` |
| 편집기 | textarea + 실시간 미리보기. 위지윅 없음 | DESIGN §2 |
| 백엔드 테스트 | pytest, TestClient, 임시 vault fixture | `backend/tests/conftest.py` |
| UI 테스트 | Node 스크립트 + Playwright(전역 설치) + 자체 미니 틀. 픽셀 비교 없음 | `tests/ui/screens.mjs` |
| 린트/포맷 | 설정 파일 없음 (ruff 캐시만 .gitignore에) | `.gitignore` |
| 패키지 매니저 | pip + venv, npm | `setup.py`, `package.json` |

**package.json**: dependencies `markdown-it ^14.1.0`, `react ^18.3.1`, `react-dom ^18.3.1`. devDependencies `@types/markdown-it`, `@types/react`, `@types/react-dom`, `@vitejs/plugin-react ^4.3.4`, `typescript ^5.6.3`, `vite ^5.4.11`. scripts `dev: vite`, `build: tsc -b && vite build`, `preview`.

**tsconfig**: ES2020, moduleResolution bundler, jsx react-jsx, strict, noUnusedLocals, noEmit.

**vite.config**: `server.port 5173`, `proxy {"/api": "http://127.0.0.1:8000"}`, `outDir dist`. 백엔드 CORS는 5173만 허용.

---

## 3. 백엔드와 데이터

### 3.1 백엔드 유형
자체 FastAPI 서버를 **사용자 PC에서** 실행. 서버리스·BaaS·호스팅 없음. `run.py`가 의존성 점검 → 포트 점검(이미 켜져 있으면 브라우저만) → dist 없으면 빌드 → uvicorn 실행 → 브라우저 열기.

### 3.2 인증
없음. 설정의 "작성자" 이름을 모든 기록에 `author`/`created_by`로 남긴다. 나중에 로그인이 생기면 그 자리에 들어오도록 서비스 함수가 `author` 인자를 받는다.

### 3.3 데이터 위치와 폴더 구조 (`config.py` Settings, DESIGN §4.1)

```text
vault/                          (환경 변수 MD_MGMT_VAULT 또는 저장소/vault, --vault 옵션)
├── projects/<연도>-<코드>-<순번>-<슬러그>/
│   ├── index.md                과제 개요 + front matter
│   ├── logs/YYYY-MM-DD-슬러그.md     진행일지
│   ├── reports/YYYY-MM-DD/report.md + assets/   보고 스냅샷
│   └── assets/<일지날짜>/NNN-원본명   첨부
├── people/<이름>/activities/*.md    역량 이력 (사람 단위)
├── people/<이름>/meetings/*.md      면담 (검색·AI·내보내기 제외)
├── intakes/R<연도>-<코드>-<순번>-<제목>/request.md + logs/ + assets/
├── settings.json               설정 (원본, 파생물 아님)
├── .index/index.sqlite3        파생 색인
├── .versions/                  저장 전 이전 내용 (경로 미러)
├── .trash/                     삭제 보관 (manifest.jsonl)
├── .drafts/                    쓰다 만 글
└── .logs/                      오류 기록(JSONL), backup-state.json
```

### 3.4 Front matter 키 (DESIGN §4.2 ~ §4.10)
- **과제 index.md**: id, title, status, group, tags[], partners[{team, people[]}], owners[], type, nature, category, delivery, cost_kind, intake_id, predecessors[], start_date, due_date, completed_at, created_at, updated_at, created_by, effect_expected, effect_verified, no_report, no_effect
- **진행일지**: date, title, author, tags[], attachments[], created_at, updated_at (`META_ORDER`로 키 순서 고정)
- **보고**: report_date, title, covers_from, covers_to, covered_entries[], attachments[], frozen_at, feedback, feedback_done, report_type, audience, author
- **역량 이력**: person, date, end_date, kind, title, host, place, hours, cost, takeaway, link, tags[], author, created_at, updated_at
- **접수**: id(R 접두), title, status, leader, leader_team, 분류 넷, 날짜들, effect_request, priority, picked, received_on, decided_on, decision_note, project_id, merged_into, precheck{…}, tags[]
- 원칙: 사람이 손으로 넣은 키와 순서를 보존한다. 모르는 키는 지우지 않는다. 날짜가 깨진 값(`2026-02-30`)은 그 값만 비우고 파일은 읽는다(`_TolerantLoader`, `_as_date`).

### 3.5 SQLite 색인 (`schema.sql`, `db.py`)
- 표: project, intake, entry, report, report_entry, attachment, project_predecessor, project_owner, project_partner, tag, project_tag, entry_tag, activity, meeting, search_fts(FTS5)
- `PRAGMA user_version = SCHEMA_VERSION(16)`. 다르면 모든 표를 지우고 다시 만들고 파일에서 재색인. 손상되면 파일 삭제 후 재생성
- `file_mtime`으로 증분 재색인. 기동 시 `reindex_all`, 화면의 [다시 읽기]
- 연결: WAL, foreign_keys ON, busy_timeout 5000, 요청마다 새 커넥션(`deps.get_db`), **쓰기 요청은 threading.Lock으로 직렬화**(1인 도구에서도 동시 요청이 흐르기 때문)

### 3.6 접근 제어
없음(1인). 경로 안전만 엄격: `safe_join`으로 vault 밖 경로 거부, 날짜는 `YYYY-MM-DD`만, 슬러그는 한글 보존 + 윈도우 금지 문자·예약어·길이(슬러그 60, 파일명 90) 처리.

### 3.7 브라우저 저장소
의도적으로 **쓰지 않는다**. 쓰다 만 글은 처음 localStorage에 두었다가(TODO 162) 창·프로필마다 달라 `vault/.drafts/`로 옮겼다(TODO 170). 화면 상태(필터, 돌아갈 곳)는 sessionStorage가 아니라 **주소(해시 쿼리)**에 둔다.

### 3.8 내보내기와 백업 (`services/export.py`, `services/backup.py`, `vault/versions.py`)
- 과제 내보내기 `GET /api/projects/{id}/export?format=md|zip|html|backup&assets=zip|inline|link`: 병합 Markdown(제목 → 메타 인용 줄 → 개요 → 수행 이력 날짜순 → 보고 이력), 첨부는 zip 동봉 / base64 인라인(≤5MB) / 로컬 URL 링크. HTML 단일 파일. 파일명은 RFC 5987 + ASCII 대체
- 전체 백업 `GET /api/backup`: vault zip(.index, .trash 제외)
- 자동 백업: 설정 폴더(vault 안 금지, 네트워크·외장 권장)에 zip, **GFS 보관**(일 7 · 주 4 · 월 6), 하루 한 벌만 후보, 30분마다 점검(주기는 설정, 기본 24시간), 상태를 `.logs/backup-state.json`에
- 이전 버전: 저장 직전 내용을 `.versions/<상대경로>/<타임스탬프>.md`로, 365일·문서당 200개, 되돌리기는 **본문만**(front matter는 현재 유지)
- 휴지통: 삭제는 `.trash/` 이동, 복원 가능, 번호 재사용 금지

---

## 4. AI 연동

**AI API를 호출하지 않는다.** (`services/ai_prompt.py`, DESIGN §5.13, TODO 71)

이유 셋: 인터넷 차단된 사내 PC에서 돌아야 함, 과제 내용을 어디로 보낼지는 사람이 정할 일, 나가는 것이 없으면 정책 승인이 필요 없음.

하는 일: `GET /api/reports/{id}/ai-prompt`가 **붙여넣기 좋은 글 한 덩이**를 만든다.

```text
[설정 ai_prompt_prefix, 비우면 기본 지시문]
  기본: "아래는 한 과제의 지난 보고 이후 진행 내용입니다. 주간 보고용으로 3~5줄로 요약해 주세요.
         규칙 - 원문에 있는 사실만 씁니다. 없는 수치나 판단을 만들어 내지 마세요 / 수치가 있으면 반드시 남깁니다 /
         '~함', '~예정' 체 / 마지막 줄에 다음 계획 한 줄"
과제: 제목 (번호) / 보고일 / 포함 기간 / 피보고자
--- 진행 내용 ---
[보고 본문 마크다운 그대로]      ← 평문으로 누르지 않는다. 표·목록 구조를 AI가 읽는다
[설정 ai_prompt_suffix, 기본 없음]
```

화면(`AiPromptPanel.tsx`)은 글을 **먼저 보여 주고** 복사하게 한다. 무엇이 나가는지 눈으로 확인하지 않고 누르게 하면 안 된다는 판단. 결과를 앱에 되돌리는 기능은 없다(사람이 옮겨 쓴다). 면담 기록은 프롬프트 대상에서 제외(민감).

**다음 단계로 적어 둔 것(TODO 113, 사내AI 검증안)**: 사내 생성형 AI 엔드포인트가 생기면 사내망 주소만 허용, 기본 꺼짐, 보내기 전 전량 공개, 오류 로그에 내용·응답 미저장, 받은 문장은 자동 삽입이 아니라 **옆에 나란히**, AI 실패해도 규칙 기반 초안은 그대로. 착수 신호 = API 주소와 키를 받는 날.

구조화 출력(JSON) 사용, 스트리밍, 사용량 제한: 해당 없음(호출 자체가 없음).

---

## 5. 배포와 운영

- **호스팅 없음.** 사용자 PC에서 실행. 배포본은 ZIP 파일을 전달
- `tools/make_dist.py`: `git archive HEAD`로 추적 파일만 스테이징 → 아이콘을 최상위에 복사 → `pip download --only-binary=:all: --platform win_amd64 --python-version 3.14`로 `vendor/` wheel 동봉 → `배포본-정보.txt`(이름, 날짜, 커밋, 대상 파이썬, wheel 수, 설치법) → `느린나이테-YYYYMMDD-vN.zip`. 같은 날 다시 만들면 판 번호가 오른다. `--no-vendor`, `--python 3.13` 옵션
- `setup.bat` → `setup.py`: Python ≥ 3.10 확인, `.venv` 생성, `vendor/` 있으면 `--no-index --find-links vendor`, 없으면 PyPI, 윈도우면 바탕화면 바로가기(PowerShell COM). 배치 파일은 **순수 ASCII + CRLF**(cmd가 cp949로 읽음), 한글 안내는 파이썬이 출력. `.gitattributes`로 `*.bat` CRLF 고정
- `run.bat` → `run.py`: 위 3.1 참고. 오류 시 창을 닫지 않고 대기
- CI: **없음**(GitHub Actions 등 설정 파일 없음). 시험은 로컬에서 `pytest`와 `node tests/ui/screens.mjs`
- 환경 변수(이름과 용도): `MD_MGMT_VAULT`(데이터 폴더), `PORT`(run.sh), `PYTHONUTF8`(콘솔 인코딩), `UI_TEST_PORT`, `PLAYWRIGHT_PATH`, `CHROMIUM_PATH`(UI 시험), `HTTPS_PROXY`(설치 실패 안내 문구에서만 언급)
- 분석·에러 수집 도구: 외부 서비스 없음. 오류는 `vault/.logs/`에 JSONL로(동작·상태·사유·직전 동작 3개, **과제 내용 미포함**, 404 미기록, 3개월 보관, 파일당 2000줄). 화면 오류도 `/api/errors/client`로 같은 곳에
- 운영 비용: **0원**. 외부 서비스 없음
- 캐시: SPA 진입점 `index.html`만 `Cache-Control: no-cache`, 해시 이름 번들은 캐시. 설정 → 점검에 실행 중 배포본 이름 표시(`/api/meta`의 `build`)
- 업데이트: 켜 둔 서버를 끄고 ZIP을 기존 폴더 위에 덮어쓰기(vault는 ZIP에 없으므로 데이터 보존) → `setup.bat` 한 번 더 → `run.bat`

---

## 6. 디자인 시스템

### 6.1 토큰 (`frontend/src/styles.css` 머리)

```css
:root {
  --bg: #f6f7f9;
  --surface: #ffffff;
  --border: #e3e6ea;
  --text: #1c2024;
  --muted: #6b7280;
  --accent: #2f6feb;
  --accent-soft: #eaf1fe;
  --accent-strong: #255ccc;   /* 연한 바탕(#fcfcfd, #eaf1fe) 위 파란 글자. --accent는 그 위에서 AA 미달 */
  --muted-strong: #626a78;    /* hover 줄(#eaf1fe) 위 흐린 글자. --muted는 4.26으로 미달 */
  --danger: #c0392b;
  --warn: #b45309;
  --radius: 10px;
  color-scheme: light;
}
```

- **다크 모드 없음.** `color-scheme: light` 고정
- 런타임 변수: `--header-h`(헤더 실제 높이, ResizeObserver, 기본 67px), `--rm-label`(로드맵 라벨 폭)
- 서체: `"Pretendard", "Apple SD Gothic Neo", "Malgun Gothic", system-ui, -apple-system, sans-serif`. 고정폭 `ui-monospace, "D2Coding", Menlo, monospace`. 웹폰트 로딩 없음(시스템 설치 폰트 의존)
- 크기: 본문 14px / 1.6, 보조 12~13px, 칩 11px, 화면 제목 20px, 상세 제목 22px, 카드 h2 15px
- 모서리: 카드·배너·모달 10px, 버튼·입력 8px, 칩 999px
- 토큰 밖 하드코드 색이 많다: 상태·속성 딱지 팔레트(Tailwind 계열 `#ede9fe/#5b21b6`, `#dcfce7/#166534`, `#fef3c7/#92400e`, `#fee2e2/#991b1b` 등), 연한 바탕 `#fcfcfd`, `#fafbfc`, `#f1f5f9`. 새 앱에서는 토큰으로 승격할 후보
- 아이콘: 나무 단면 `favicon.svg` 하나가 원본, ico(16·32·48), icon-180/512 png. `theme-color #f2e1c6`. 아이콘 라이브러리 없음(이모지 📎 등 사용)

### 6.2 레이아웃
`div.app` > `header.app-header`(sticky, 브랜드 + 메뉴 + 검색창 + 데이터 경로) > 조건부 배너 > `main`(max-width 1180px, 1440px 이상에서 1720px, padding 24px 28px 64px) > ErrorBoundary > 화면 > Toast > ScrollTop.

반응형 기준점: 1440 / 1500 / 1180 / 1100 (min-width), 1240 / 1200 / 1000 / 900 / 720 (max-width). **기준 뷰포트 1093px**(1366×768 윈도우 125% 배율). 폰 전용 레이아웃 없음. 가로 스크롤은 `.table-scroll` 표 안에서만.

### 6.3 명암비 규칙
WCAG 2.1 AA(4.5, 큰 글자 3.0)를 **UI 시험이 실제 렌더 색으로 측정**(`contrast.mjs`가 조상 배경까지 합성). 상태 조합(기본/hover/선택/선택+hover) 전부. 색만으로 구분하지 않음(diff는 +/− 기호, 기간 내 보고는 색 + ●).

### 6.4 공통 컴포넌트 (`frontend/src/components/`)
범용: ErrorBoundary, LoadError(사유 + 다시 시도), Toast(하단 중앙, 최대 3, 10초), ScrollTop, SortHeader, TotalRow(합계 줄), CopyTableButton(TSV 클립보드), FolderPicker, PreviewToggle, StatusBadge, TagSuggestions, PasteOffer, AttachmentList, XlsxPreview, VersionPanel/VersionsCard, BackupCard, ErrorLogCard, TrashCard, LinkFixCard, Help.
도메인: Home, Dashboard, ProjectList/Board/Detail/Form, EntryEditor, ReportEditor/Candidates/History/Diff/DayCard, IntakePool/Detail/Form, Roadmap, Skills, MeetingPanel, Settings 등.

공통 CSS 클래스 묶음: `.card/.card-head/.page-head/.page-desc`, `.toolbar/.filters/.view-switch`, `.grid`(+tfoot, th.sortable), `.status-*/.type-badge/.tag/.chip`, `.form-row/.form-actions/.form-error`, `.split/.preview/.markdown/.table-scroll`, `.modal-backdrop/.modal`, `.toast-stack/.toast`, `.load-error/.render-error/.empty/.hint`, `.timeline/.entry`, `.board*`, `.home-stat/.bar-row`.

---

## 7. 공통 패턴

- **폴더 규칙**: 백엔드 `api/`(라우터, 얇음) → `services/`(로직) → `vault/`(파일 I/O). 프론트 `components/` 평면 + 루트에 `api.ts`, `nav.ts`, `types.ts`, `markdown.ts`, `notify.ts`, `unsaved.ts`, `util.ts`, `upload.ts`, `table.ts`, `tsv.ts`, `plaintext.ts`
- **데이터 접근**: `api.ts`의 `request<T>()` 래퍼 하나. 네트워크 실패는 "프로그램에 연결하지 못했습니다 — 느린 나이테 창(run.bat)이 켜져 있는지 확인하세요", 서버 오류는 `errorText()` 한 곳에서 한국어 문구로(422 목록은 항목명 모아서, 500은 "설정 › 최근 오류에 남았습니다")
- **로딩/오류**: 컴포넌트마다 `load` useCallback + useEffect, 실패는 `LoadError`로 빈 표와 구분. 저장·삭제 실패는 `attempt()` → Toast. 처리 안 된 예외는 main.tsx가 Toast + 서버 기록. 떠나기 경고 `useUnsaved`/`confirmLeave`. Ctrl+S 저장 통일
- **라우팅**: `nav.ts`에 `projectLink/screenLink/backTarget/syncQuery/useAddressBar`. 조건 변경은 `replaceState`, 이동은 hash 대입. 돌아갈 곳은 `?back=` 파라미터로 주소에 실음. `setPageTitle`
- **날짜/한국어**: `YYYY-MM-DD` 문자열만, 서버에서 검증. 슬러그는 한글 보존(NFC 정규화). 표 셀은 `word-break: keep-all`. 검색은 FTS5 trigram(2글자 이하 LIKE 폴백). "연도" 기준은 `services/period.py` 한 곳(수행기간 겹침, 금액은 끝나는 해)
- **PWA/오프라인**: service worker·manifest 없음. 로컬 서버라 오프라인 개념 자체가 없음
- **접근성**: 토스트 `role=alert aria-live=assertive`, 검색 `aria-label`, focus-visible 일부. 체계적 처리는 명암비 중심
- **설계 원칙(DESIGN·LESSONS)**: 파일이 원본·DB는 파생물 / 소급 불가 데이터(누가·언제)는 화면에 없어도 먼저 저장 / 세는 수 = 거르는 수(모든 숫자는 눌리고 목록 길이와 같다) / `null`≠`0`≠`[]` / 상태는 코드 상수, 속성·분류는 설정 / 쪼개야 하는 데이터는 서버가 구조로 / 식별자는 재사용 금지 / 미리보기(GET)와 실행(POST) 쌍 / 실패는 침묵하지 않는다 / 같은 일은 한 함수·한 부품에

---

## 8. 재사용 가능 자산 (그대로 복사해 시작할 수 있는 것)

**백엔드 골격 (`backend/app/`)**
| 파일 | 역할 | 수정 필요 |
|------|------|-----------|
| `config.py` | Settings dataclass, 환경 변수 vault, 상수 | 상수를 커리어 도메인으로 교체 |
| `db.py` | SQLite 열기, 스키마 버전 재생성, FTS5 폴백 | 스키마 파일 교체 |
| `deps.py` | 요청별 커넥션, 쓰기 직렬화, RowId | 그대로 |
| `main.py` | lifespan, 오류 미들웨어, 422 안전 변환, SPA 서빙 | 라우터 목록 교체 |
| `vault/markdown.py` | front matter 보존 읽기/쓰기, 관용 YAML, 원자적 저장, 외부 변경 감지 | 그대로 |
| `vault/paths.py` | 슬러그, 윈도우 안전 파일명, safe_join, move | 그대로 |
| `vault/versions.py` | 이전 버전 보관·복원 | 그대로 |
| `services/backup.py` | GFS 백업 | 파일명 접두어만 |
| `services/errorlog.py` | 내용 없는 오류 기록 | 그대로 |
| `services/settings.py` | JSON 설정 로드·모양 검증·저장 | DEFAULTS 교체 |
| `services/export.py` | 병합 Markdown·zip·HTML, RFC 5987 | **커리어 컨텍스트 파일의 출발점** |
| `services/ai_prompt.py` | prefix + 사실 + 본문 + suffix | **구조화 프롬프트의 출발점** |
| `tests/conftest.py` | 임시 vault + TestClient | 그대로 |

**프론트 골격 (`frontend/src/`)**: `styles.css`(토큰과 공통 클래스 블록), `api.ts`(request/errorText), `notify.ts` + `Toast.tsx`, `unsaved.ts`, `nav.ts`, `App.tsx`(라우트 파싱·헤더 높이·스크롤·ErrorBoundary 배치), `main.tsx`, `markdown.ts`(html:false, 새 창 링크, 표 스크롤 래퍼, base 경로), `util.ts`, `upload.ts`, 범용 컴포넌트(ErrorBoundary, LoadError, ScrollTop, SortHeader, TotalRow, CopyTableButton, FolderPicker, PreviewToggle, VersionPanel, BackupCard, ErrorLogCard, TrashCard, StatusBadge, PasteOffer, AiPromptPanel/AiPromptCard), `index.html`, `vite.config.ts`, `tsconfig.json`, `package.json`.

**루트**: `run.py`, `run.bat`, `run.sh`, `setup.py`, `setup.bat`, `tools/make_dist.py`, `tests/ui/contrast.mjs`, `tests/ui/screens.mjs` 앞 270행(시험 틀·서버 기동·go·contrastOf), `.gitignore`, `.gitattributes`.

**공통 패키지로 분리할 후보**: vault 계층(markdown/paths/versions) + db/deps + backup/errorlog/settings 골격을 "느린 시리즈 로컬 도구 코어"로. 프론트는 styles.css 토큰 + api/notify/unsaved/nav + 범용 컴포넌트. 느린 나이테 문서는 공통 패키지를 만들지 않았고 복사 방식이다.

---

## 9. 알려진 문제와 교훈

**열 가지 반복된 실수 (LESSONS-LEARNED 0절)**: ① 세는 곳과 보여 주는 곳이 다르다 ② 같은 일을 하는 코드가 둘 ③ 날짜·연도 기준이 화면마다 다르다 ④ 사용자 PC 환경을 개발 환경으로 착각(윈도우·1093px·오프라인·파이썬 3.14·캐시) ⑤ 소급 불가 데이터를 나중에 ⑥ 시험이 있는데 못 잡는다 ⑦ 1인 도구라고 동시성 무시 ⑧ 화면이 숫자만 있고 갈 곳이 없다 ⑨ 문서와 화면이 어긋난다 ⑩ 작업 환경에서 낸 사고(셸 죽이기, 포트 겹침). 가장 비싼 것은 ①③⑤, 가장 싸게 막을 수 있던 것은 ④⑨.

**미해결·보류(TODO 머리 "조건이 맞을 때까지")**: watchdog 증분 색인(T22, [다시 읽기] 잦을 때), 묶음 보고 서버 저장(T10·T40), 읽기 전용 정적 HTML 공유(T15, 확산 2.5단계), 사내 AI 호출(71-나·113, API 주소·키 수령 시), 대용량(T23), setup.bat 점검(사용자 PC 3.14 확인). 다중 사용자(M6, R2)는 결정 넷(저장·접속·로그인·권한)이 먼저.

**다음 앱에서 다르게 할 점(문서 근거)**: 첫날 `환경.md` 한 쪽(OS·파이썬·해상도·인터넷·설치 권한·함께 쓰는 프로그램) / 원본·파생물 구분과 파생물 버전 번호를 첫 커밋에 / 소급 불가 필드(누가·언제·완료일·상태 변경 이력)를 설계 첫 주에 / 시간 기준 표를 문장으로 / 첫 시험은 "모든 숫자 = 그 링크의 목록 길이" / 시험은 일부러 깨뜨려 실패를 보고, 실패를 끼워 넣고, 실제 모양(한글 식별자·긴 이름)의 시험 자료 / `.gitattributes` CRLF·ASCII 배치·`index.html` no-cache·배포본 이름 표시를 첫 커밋부터 / 다섯 건쯤 묶어 반영, 빌드마다 번호 붙은 체크리스트(결함 절반이 여기서 나옴) / 지난 기록은 고쳐 쓰지 않고 각주만 / 문서의 수치는 명령으로 돌려 맞춘다.

---

## 10. 같은 스택으로 새 앱을 시작할 때 체크리스트

1. `docs/환경.md`: 사용자 PC(개인 PC인지 사내 PC인지), OS, 파이썬 버전, 해상도·배율, 인터넷 가능 여부, 함께 쓰는 프로그램. 데이터가 밖으로 나가면 안 되는가
2. 저장소 뼈대 복사: `backend/app/{config,db,deps,main}.py`, `vault/`, `services/{backup,errorlog,settings,export,ai_prompt}.py`, `tests/conftest.py`, `frontend/src/{styles.css,api,notify,unsaved,nav,markdown,util,upload,App,main}`, 범용 컴포넌트, `run.*`, `setup.*`, `tools/make_dist.py`, `.gitignore`(`/vault/` 최상위 기준), `.gitattributes`
3. 원본/파생물 선언: `vault/` 폴더 구조와 front matter 키 표, `schema.sql`, `SCHEMA_VERSION = 1`
4. 소급 불가 필드 목록을 먼저 확정하고 모든 쓰기 경로에 넣는다(author, created_at, updated_at, 상태 변경 시각)
5. 시간 기준 표: 어떤 날짜 열로 연도·기간을 자르는가
6. `config.py` 상수(상태 등)와 `settings.py` DEFAULTS 교체. 이름·아이콘(`favicon.svg` 하나가 원본)
7. `nav.ts` 화면 이름표, `App.tsx` 라우트, 메뉴 순서
8. 첫 시험: 서버(숫자=목록, 경로 탈출, 깨진 파일 하나, 동시 요청) + 화면(1093px 가로 넘침 0, 명암비, 메뉴 이동) 작성 후 **일부러 깨뜨려 실패 확인**
9. `make_dist.py`의 NAME·아이콘 경로 교체, 새 폴더에 클론해서 `setup → run`이 되는지 확인
10. TODO.md를 1번부터, 운영 원칙(모아서 "착수하자", 번호는 추가만, 번호 붙은 체크리스트) 그대로

---

## 11. 느린 길 설계에 미치는 영향 (프로파일 추출의 결론)

1. **"웹"은 호스팅이 아니라 로컬 실행이다.** 서버 비용, 로그인, 사용량 할당량, 제공자 결제 상한(D-007)은 모두 해당 없음. 무료 전용(D-005)이 자연스럽게 성립한다.
2. **AI는 프롬프트 복사 방식이 기본값이다.** 느린 나이테가 이미 그렇게 쓰고 있고, 사용자(1차 페르소나 본인)의 환경이 인터넷 차단 사내 PC다. 느린 길의 "기록 단계 구조화"도 앱 내 API 호출이 아니라 **프롬프트 왕복**(앱이 구조화 프롬프트 생성 → 사용자가 본인 AI에 붙여넣기 → 결과를 앱에 붙여넣기 → 앱이 정해진 형식을 파싱해 필드에 채움)으로 설계해야 한다. 데스크톱 브라우저에서는 탭 전환이므로 iOS에서 우려했던 마찰이 크게 줄어든다. 느린 나이테에 없는 부분은 "결과를 되돌려 받아 파싱하는 단계"다.
3. **커리어 데이터가 이미 Markdown 폴더다.** 컨텍스트 파일 내보내기는 `services/export.py`의 병합 Markdown과 같은 패턴이다. 사용자는 vault 폴더를 Obsidian으로 열어도 된다.
4. **데이터 위치 질문이 생긴다.** 커리어 기록은 개인 자산이므로 사내 PC의 vault에 두면 퇴사·전배 시 문제가 된다. 개인 PC 실행, 또는 vault를 개인 외장·클라우드 동기화 폴더(SQLite 색인은 깨질 수 있으므로 백업 폴더만)에 두는 안내가 필요하다.
5. **재사용 범위가 매우 크다.** 백엔드 골격, 안전망 세트(버전·휴지통·백업·오류 기록), 프론트 골격, 배포 도구를 그대로 가져오면 MVP-0은 "도메인 모델 + 화면 몇 개 + 프롬프트 두 종류(구조화, 컨텍스트 파일 사용 안내)"로 줄어든다.
6. **다크 모드·모바일은 없다.** 느린 길도 데스크톱 1093px 기준으로 시작하는 것이 시리즈 일관성에 맞다.
