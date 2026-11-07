# 느린 길 - 기술 설계

> Version: 0.1 (초안)
> 작성일: 2026-11-07
> 선행 문서: `02_REQUIREMENTS.md`(기준선), `04_DATA_MODEL.md`(기준선), `03_UX_SPEC.md`, `07_EXPORT_DESIGN.md`, `docs/spec/context_file.md`, `docs/reference/neurin-naite-profile.md`
> 골격 출처: `helpnara/md_mgmt_tool` 브랜치 `claude/team-task-management-tool-3d91g8`, 커밋 `a23bd75`(2026-11-06, 배포본 v6). 프로파일(10-02) 이후 한 달 사이 팀 공유 기능이 붙었으므로 이 문서는 **커밋 a23bd75 기준으로 다시 확인한** 내용이다.
> 개인정보 규칙은 저장소 루트 `CLAUDE.md`. 골격을 복사할 때도 적용한다.

---

## 1. 전제

| 항목 | 값 | 근거 |
|------|----|------|
| 실행 | 사용자 PC에서 `run.bat` → `127.0.0.1:8000` 단독. 외부 요청 0 | D-006, D-008, NFR-03 |
| 사용자 | 1인. 인증·IP 식별·권한 없음 | 02 §3.1 |
| 백엔드 | Python ≥ 3.10, FastAPI ≥ 0.115, Uvicorn ≥ 0.30(standard 제외), python-frontmatter, markdown-it-py, python-multipart | 느린 나이테 `requirements.txt`에서 Pillow·openpyxl 제외 |
| 프론트 | React 18, TypeScript 5, Vite 5, markdown-it. 상태 라이브러리·UI 라이브러리 없음 | 느린 나이테 `package.json` 그대로 |
| 데이터 | `vault/` Markdown + front matter 원본, `.index/` SQLite 파생 | 04 §6·§7 |
| 배포 | 빌드된 프론트 + wheel 동봉 ZIP, `setup.bat` → `run.bat` | NFR-11 |
| 시험 | pytest + Playwright(`tests/ui`). CI 없음 | NFR-09 |

## 2. 느린 나이테에서 가져올 것

### 2.1 그대로 복사 (이름·문구만 교체)

| 파일 | 역할 | 교체할 것 |
|------|------|-----------|
| `backend/app/clock.py` | 오늘·지금. 시험이 날짜를 고정한다 | 환경 변수 `MD_MGMT_TODAY` → `SLOW_WAY_TODAY` |
| `backend/app/deps.py` | 요청별 커넥션, 쓰기 직렬화 | 없음 |
| `backend/app/db.py` | SQLite 열기, `PRAGMA user_version` 불일치 시 재생성, FTS5 trigram·unicode61 폴백 | `SCHEMA_VERSION = 1`, 버전 주석 비움 |
| `backend/app/vault/markdown.py` | front matter 보존 읽기·쓰기, 관용 YAML, 원자적 저장, 외부 변경 감지(`ensure_unchanged`) | 없음 |
| `backend/app/vault/paths.py` | 안전 파일명, `safe_join`, 날짜 검증, `retry_os`(윈도우 파일 잠김) | `project_dir_name` 제거 |
| `backend/app/services/errorlog.py` | 내용 없는 오류 기록(JSONL, 3개월) | 없음 |
| `backend/app/services/buildinfo.py` | 배포본 이름 읽기 | 없음 |
| `backend/app/services/period.py` | 연도 겹침 계산 | 과제 전용 함수(`effect_year` 등)를 경험 기간용으로 줄임 (NFR-14) |
| `run.py`, `run.bat`, `run.sh`, `setup.py`, `setup.bat` | 실행·설치 | 이름, 환경 변수, **`--share` 옵션 제거** |
| `tools/make_dist.py` | 배포 ZIP | `NAME = "느린길"`, 아이콘 경로 |
| `frontend/src/{api,nav,notify,unsaved,markdown,util,clipboard,storage}.ts`, `main.tsx` | 요청 래퍼·라우팅·토스트·떠나기 경고·마크다운·클립보드 | 메시지의 앱 이름, `nav.ts` 화면 이름표 |
| `frontend/src/components/` 범용 | ErrorBoundary, LoadError, Toast, ScrollTop, SortHeader, TotalRow, CopyTableButton, FolderPicker, PreviewToggle, StatusBadge, PasteOffer, VersionPanel, VersionsCard, BackupCard, ErrorLogCard, TrashCard, Help, ViewTabs, AiPromptPanel | `AiPromptPanel`은 S-03의 ① 칸 출발점 |
| `frontend/` 설정 | `index.html`, `vite.config.ts`, `tsconfig*.json`, `package.json`, `scripts/compress.mjs` | 제목·아이콘 |
| `tests/ui/contrast.mjs`, `tests/ui/screens.mjs` 앞부분(시험 틀·서버 기동·`go`·`contrastOf`) | UI 시험 틀 | 화면 시험 본문은 새로 |
| `.gitignore`(`/vault/`), `.gitattributes`(`*.bat` CRLF) | | 없음 |

### 2.2 고쳐서 가져올 것

| 파일 | 고칠 것 |
|------|---------|
| `backend/app/main.py` | 미들웨어 5개 중 **오류 기록(`record_failures`)과 예외 처리기·`validation_message`·SPA 서빙만** 남긴다. 읽기 캐시(`readcache`)와 접근 제어(`share`·`who`·`permissions`·`edits`) 미들웨어를 뺀다. 라우터 목록 교체 |
| `backend/app/config.py` | 상수(04 §4 상태, 역량 축, 기록 종류, 기준 종류)와 vault 경로(`careers/`, `projects/`, `experiences/`, `entries/`, `attachments/`, `profile.md`, `skills.yaml`, `anonymize.yaml`). 환경 변수 `SLOW_WAY_VAULT` |
| `backend/app/vault/versions.py` | 과제 폴더 단위 → 파일 단위(경험·경력·과제 파일). 색인기 호출부 교체 |
| `backend/app/vault/indexer.py` | 985행 중 골격(스캔·문제 목록·`reindex_all`)만 쓰고 도메인 색인 함수는 새로 |
| `backend/app/services/trash.py` | 접수·과제 복원 규칙을 빼고 경험·경력·과제 파일 복원으로 |
| `backend/app/services/backup.py` | `export.backup_zip` 의존을 vault zip 함수 하나로 |
| `backend/app/services/settings.py` | 918행 중 로드·모양 검증·원자적 저장만. `DEFAULTS`는 02 FR-K1. 명부·팀장·면담 관련 함수 제거, `who` 의존 제거 |
| `backend/app/services/search.py` | 대상 테이블 교체(경험 본문·기록 원문) |
| `backend/app/services/ai_prompt.py` | `build(conn, report_id)` → 기록 정리·분해 두 종류(4.3) |
| `backend/app/services/export.py` | 병합 Markdown 패턴만 참고. 컨텍스트 파일 직렬화는 새로(4.5) |
| `frontend/src/App.tsx` | 라우트 표(03 §3), 헤더 메뉴(03 §2.1), 읽기 전용 배너 제거 |
| `frontend/src/styles.css` | 토큰 블록과 공통 클래스는 유지. 과제·보드·로드맵 블록 제거. 딱지 팔레트를 토큰으로 승격(03 §2.3) |

### 2.3 가져오지 않을 것

| 대상 | 이유 |
|------|------|
| `share.py`, `who.py`, `permissions.py`, `readcache.py`, `services/edits.py`, `services/conflicts.py`, `frontend/src/{me,readonly,people}.ts` | 팀 공유·다중 사용자(TODO 191·233). 느린 길은 1인 도구다 |
| `contributions.py`, `services/{intakes,projects,reports,activities,meetings,people,person_history,precheck,renumber,roadmap,drafts,my_work,dashboard,home,folders,linkfix,attachments}.py` | 과제 관리 도메인 |
| `frontend/src/components/` 도메인 컴포넌트(Project*, Report*, Intake*, Meeting*, Roadmap, Dashboard, PeopleCard 등) | 과제 관리 도메인 |
| `Pillow`, `openpyxl`, `htmlPaste.ts`, `pasteImages.ts`, `XlsxPreview` | 엑셀·이미지 붙여넣기. M0 기록 첨부는 파일 그대로 보관만 한다 |
| `tests/load/` | 다중 사용자 부하 시험 |
| 백엔드 시험 83개 | 도메인 시험. `conftest.py`(임시 vault + TestClient + 날짜 고정)만 가져온다 |

**복사할 때 확인**: 가져온 파일에 사람 이름·회사명·사내 과제명이 없는지 읽고 넣는다(`CLAUDE.md`). a23bd75 기준으로 `settings.DEFAULTS`의 작성자·명부는 비어 있고, 2.1·2.2 대상 파일에서 실명은 발견되지 않았다.

## 3. 저장소 구조

```text
CV_mgmt/  (저장소 이름은 그대로, 앱 이름은 느린 길 / slow-way)
├── backend/
│   ├── app/
│   │   ├── api/            라우터 (얇음)
│   │   ├── services/       로직
│   │   ├── vault/          파일 계층 + schema.sql
│   │   ├── domain/         ★ 순수 함수: parser, serialize, anonymize, consistency, skillmap
│   │   ├── config.py  db.py  deps.py  clock.py  main.py
│   ├── tests/
│   │   ├── fixtures/
│   │   │   ├── vault_fictional/     가상 경력 vault (07 §8)
│   │   │   └── responses/           가상 AI 응답 (형식 변형 재현용, 4.4)
│   │   └── test_*.py
│   └── requirements.txt
├── frontend/   src/ (components/ 평면), dist/ (커밋, NFR-01)
├── tests/ui/   screens.mjs, contrast.mjs
├── tools/      make_dist.py
├── docs/
├── run.py run.bat run.sh setup.py setup.bat
├── CLAUDE.md  .gitignore  .gitattributes
```

`domain/`은 느린 나이테에 없는 층이다. 파일·DB를 모르는 **순수 함수**만 두어, 실험 01에서 손으로 확인한 규칙을 시험 자료로 바로 고정할 수 있게 한다.

## 4. 백엔드 설계

### 4.1 쓰기 경로 (모든 쓰기 공통)

```text
API 입력 검증(pydantic, 한국어 오류 문구)
 → services.<엔티티>.save()
     → vault.markdown.ensure_unchanged()   # 옵시디언 등 외부 수정 감지
     → vault.versions.keep()               # 이전 본문 보관 (FR-E8)
     → vault.markdown.save()               # 원자적 저장
     → indexer.index_one()                 # 그 파일만 색인
     → domain.consistency.check()          # 경고 갱신 (FR-F1)
 → 응답: 저장된 엔티티 + 경고 목록
```

- 소급 불가 필드(`created`, `updated`, 상태 변경 시각, 확정 시각, 경고 확인 시각)는 이 경로에서만 채운다(NFR-13). 화면이 보낸 시각은 믿지 않는다.
- 기록(Entry)은 이 경로 중 `versions.keep`이 없다. 원문은 한 번 쓰면 다시 쓰지 않는다(FR-B2). "무효" 표시만 front matter에 붙는다.

### 4.2 서비스 모듈

| 모듈 | 하는 일 | FR |
|------|---------|----|
| `services/profile.py` | `profile.md` 읽기·쓰기, 총 경력 계산 | A1 |
| `services/careers.py` | 경력 CRUD, 겹침·공백 | A2~A4 |
| `services/projects.py` | 과제 CRUD | E9 |
| `services/entries.py` | 기록 추가·무효·연결, 목록·필터 | B |
| `services/experiences.py` | 경험 CRUD, 상태 전이, **확정 조건 판정**(04 §4), 질문 답 반영, 관련 경험 | E, C8 |
| `services/structure.py` | 프롬프트 만들기(4.3), 응답 받기 → `domain.parser` → 미리보기, 블록 저장(새 경험·이어짐·충돌 비교) | C, D |
| `services/skills.py` | `skills.yaml` 로드·편집, 매핑, 새 역량 후보 | G, C9 |
| `services/export.py` | 범위 선택 → `domain.serialize` → `domain.anonymize` → 미리보기 묶음 | H |
| `services/anonymize_table.py` | `anonymize.yaml` 편집, 후보 추출(07 §5.2) | H3 |
| `services/backup.py` · `trash.py` · `settings.py` · `errorlog.py` · `search.py` · `buildinfo.py` | 2절 | I, K, L |
| `services/home.py` | 홈 숫자 4개·최근 경험·주요 역량. **모든 숫자는 목록 끝점과 같은 질의로 센다** | J |
| `services/counters.py` | 로컬 카운터(02 §9) | 9절 |

### 4.3 프롬프트 만들기

`services/structure.build_prompt(kind, entry_ids)`:

| kind | 원본 | 채우는 자리 |
|------|------|-------------|
| `entry` | `docs/prompts/structure_entry.md`의 코드 블록 | `{기록 원문}`(여러 건이면 날짜별 구분선), `{기록 날짜}`, `{역량 사전}`(축별 대표 이름), `{기존 경험 목록}`(최근 수정 20건: ID·제목·기간) |
| `decompose` | `docs/prompts/decompose_cv.md`의 코드 블록 | `## 경력기술서 원문` 아래에 기록 원문 |

- 프롬프트 원본은 **빌드 때 `docs/prompts/`에서 코드 블록을 뽑아** `backend/app/prompts/*.txt`로 넣는다(`tools/sync_prompts.py`). 문서와 앱의 프롬프트가 어긋나지 않게 하는 장치이며, 시험이 둘을 비교한다.
- 설정의 접두·접미 문구(FR-K1)를 앞뒤에 붙인다.
- 같은 입력이면 같은 문자열(시각·난수 없음). 화면이 보여 준 문자열과 복사되는 문자열이 같아야 한다(원칙 2).

### 4.4 파서 `domain/parser.py`

스펙 3.3의 규칙 15개를 그대로 구현한다. 입출력:

```python
def parse(text: str) -> ParseResult: ...

@dataclass
class ParsedBlock:
    index: int                       # 응답 안 순서
    fields: dict[str, str]           # 스칼라 키 (정규화된 키 이름)
    lists: dict[str, list[str]]      # 행동, 증빙, 확인_필요
    quants: list[Quant]              # 지표·값·기준·시점 + 경고
    continues: str | None            # "E-027", "E-027?", None
    unknown: dict[str, str]          # 미분류 키
    bad_lines: list[BadLine]         # (줄 번호, 원문, 이유)

@dataclass
class ParseResult:
    blocks: list[ParsedBlock]
    found_fence: bool
    stripped_markers: int            # 제거한 Gemini 인용 표식 수
    duplicates: list[tuple[int, int]]  # 중복 후보 블록 쌍 (규칙 14)
```

- 파일·DB를 모른다. 역량 매핑과 누락 검사는 바깥(`services/structure.py`)에서 한다.
- **시험 자료는 가상 응답**이다. 실험 01의 실제 응답은 저장소에 넣지 않는다(`CLAUDE.md`). 대신 실험에서 관찰한 형식 변형을 하나씩 재현한 가상 응답을 만든다.

| 시험 자료 | 재현하는 변형 (실험 01 관찰) |
|-----------|------------------------------|
| `responses/fenced.md` | 펜스 있음, 빈 목록 `-`, 교훈 `/` 결합과 출처 표시 |
| `responses/no_fence.md` | 펜스 없음, 블록이 빈 줄 없이 이어짐, "기준: 기준 없음" 접두 반복, 교훈 목록형 |
| `responses/span_markers.md` | 모든 줄 끝 `[span_n](start_span)[span_n](end_span)`, 스칼라 키 아래 목록, "- 해당 없음" |
| `responses/broken.md` | 정량 2칸, 모르는 키, U+FFFC 글머리표, 물결표 없는 기간, 블록 0개 응답 |
| `responses/continued.md` | 10개씩 나눠 붙여 넣은 두 응답, 같은 제목 블록 재등장 |

### 4.5 직렬화·익명화 `domain/serialize.py`, `domain/anonymize.py`

- `serialize(snapshot, options) -> str`: 07 §4 규칙. `snapshot`은 서비스가 vault에서 읽어 만든 불변 구조체다. 같은 입력이면 같은 출력(07 §4.2 정렬 안정성).
- `anonymize(text, table) -> (text, report)`: 07 §5.3 적용 순서, §5.4 잔존 검사. `report`는 종류별 치환 횟수와 잔존 위치.
- `cleanup`(결과물 정리, 07 §7)은 **프론트에서만** 돈다(`frontend/src/cleanup.ts`). 서버로 보내지 않는다는 약속을 코드 위치로 지킨다. 같은 규칙의 파이썬 판은 만들지 않는다.

### 4.6 정합성 검사 `domain/consistency.py`

`check(snapshot, experience_id | None) -> list[Warning]`. 04 §8의 X-01~X-09. 각 경고는 `code`, `targets`(ID 목록), `message`(한국어 한 문장), `key`(확인함 기록용 안정 키). 확인한 경고는 경험 front matter의 `acknowledged: [{key, at}]`에 남고, 같은 key가 다시 나와도 띄우지 않는다.

### 4.7 색인 `vault/schema.sql` v1

04 §7 표 그대로. 추가로 `warnings(experience, code, key, message, acknowledged_at)`. 색인 재생성은 vault 전체 스캔 → 경험마다 정합성 검사 → `warnings` 채움.

## 5. API

경로 접두 `/api`. 미리보기(GET)와 실행(POST) 쌍을 지킨다.

| 메서드 | 경로 | 하는 일 |
|--------|------|---------|
| GET | `/meta` | 배포본, 스키마 버전, vault 경로, 첫 실행 여부 |
| GET·PUT | `/profile` | 프로필 |
| GET·POST | `/careers` | 경력 목록·추가 |
| GET·PATCH·DELETE | `/careers/{id}` | |
| GET·POST | `/projects`, GET·PATCH·DELETE `/projects/{id}` | |
| GET·POST | `/entries` | 목록(필터: 정리 안 됨·종류·기간·검색)·추가 |
| GET | `/entries/{id}` | 원문 |
| POST | `/entries/{id}/void`, `/entries/{id}/link`, `/entries/{id}/unlink` | 무효·연결 |
| GET·POST | `/experiences` | 목록(필터 6종·확인 필요 있음·경고 있음)·수동 생성 |
| GET·PATCH·DELETE | `/experiences/{id}` | 상세·저장·휴지통 |
| GET | `/experiences/{id}/confirmable` | 확정 조건 판정과 남은 조건 목록 |
| POST | `/experiences/{id}/confirm`, `/experiences/{id}/unconfirm` | 상태 전이 |
| POST | `/experiences/{id}/answer` | 확인 필요 질문에 답 반영 |
| POST | `/experiences/{id}/acknowledge` | 경고 확인함 |
| GET | `/experiences/{id}/versions`, POST `/experiences/{id}/versions/{v}/restore` | 버전 |
| GET | `/structure/prompt?kind=entry&entries=…` | 프롬프트 전문 |
| POST | `/structure/parse` | 응답 원문 → AI 응답 기록 저장 + 파싱 결과(미리보기). 경험은 만들지 않는다 |
| POST | `/structure/commit` | 블록별 선택(새·이어짐·버림·필드별 선택)을 받아 경험 저장 |
| GET·PUT | `/skills` | 사전 |
| GET | `/skills/index` | 역량 → 경험 수 |
| GET | `/export/preview?scope=…&status=…&anon=…&ids=…&purpose=…` | 파일·프롬프트·합본·요약·경고 |
| GET·PUT | `/anonymize`, GET `/anonymize/candidates` | 치환표·후보 |
| GET | `/home` | 홈 숫자와 목록 |
| GET | `/search?q=` | 통합 검색 |
| GET·PUT | `/settings` | 설정 |
| GET·POST | `/backup`, `/backup/run`, `/backup/restore` | 백업 (느린 나이테 끝점 모양) |
| GET·POST | `/trash`, `/trash/{id}/restore` | 휴지통 |
| GET·POST | `/errors`, `/errors/client` | 최근 오류, 화면 오류 기록 |
| POST | `/index/rebuild` | 색인 다시 만들기 |

`/export/preview`가 파일 문자열을 돌려주고, 저장·복사는 브라우저가 그 문자열로 한다(07 §2). 내보내기 POST 끝점은 M1 이력용으로만 둔다.

## 6. 프론트 설계

- 라우트는 03 §3 표. `nav.ts`의 화면 이름표를 교체한다.
- 화면별 새 컴포넌트:

| 화면 | 컴포넌트 |
|------|----------|
| S-00 | `Welcome` (4단계) |
| S-01 | `Home`, `QuickEntry`(헤더 Alt+N 겸용) |
| S-02 | `EntryComposer`, `EntryList`, `EntryViewer` |
| S-03 | `StructureView`, `PromptPane`(AiPromptPanel 확장), `PastePane`, `PreviewPane`, `BlockCard`, `MergeCompare`, `MissingCandidates` |
| S-04 | `ExperienceList` |
| S-05 | `ExperienceDetail`, `QuestionsCard`, `WarningsCard`, `QuantTable`, `SkillChips`, `EvidenceList`, `ConfirmButton` |
| S-06 | `CareerTable`, `ProjectTable`, `EditDrawer` |
| S-07 | `SkillIndex`, `SkillDictionaryDialog` |
| S-08 | `ExportView`, `ExportOptions`, `ExportPreview`, `AnonymizeDialog`, `CleanupTool`(+ `cleanup.ts`) |
| S-09 | `Settings`(느린 나이테 카드 재사용 + `PromptTextCard`, `AnonymizeTableCard`, `CountersCard`) |

- 상태 딱지·기준 종류 칩·AI 채움 표시는 `StatusBadge` 확장과 CSS 클래스 세 개(`.status-draft/.status-structured/.status-confirmed`, `.basis-*`, `.ai-filled`)로.

## 7. 시험 계획

| 묶음 | 내용 | 자료 |
|------|------|------|
| `test_parser.py` | 스펙 3.3 규칙 15개 각각 + 4.4 시험 자료 5종의 블록 수·필드·경고 | 가상 응답 |
| `test_serialize.py` | 07 §8 직렬화 스냅샷, 범위, 상태, 빈 필드, 크기 경고 | 가상 vault |
| `test_anonymize.py` | 길이순 치환, 조사, 단어 경계, ID 보호, 잔존 경고(일부러 깨뜨리기) | 가상 치환표 |
| `test_consistency.py` | X-01~X-09 각각 걸리는 경우·안 걸리는 경우 | 가상 vault 변형 |
| `test_experiences.py` | 확정 조건 5개, 상태 전이, 질문 답 반영, 버전 | 가상 vault |
| `test_structure.py` | 프롬프트 = `docs/prompts` 코드 블록(동기화 검사), parse→commit, 이어짐 충돌, 원문 기록 보존 | |
| `test_entries.py` | 원문 불변, 무효, 연결 | |
| `test_home.py` | **숫자 = 목록 길이** (느린 나이테 첫 시험) | |
| `test_safety.py` | 경로 탈출, 깨진 파일 하나, 동시 저장, 외부 수정 감지 | |
| `tests/ui/screens.mjs` | 03 §5 | 가상 vault |

모든 시험은 날짜를 `SLOW_WAY_TODAY`로 고정한다. 시험을 쓴 뒤 일부러 깨뜨려 실패를 확인한다(느린 나이테 교훈).

## 8. 배포

- `tools/make_dist.py`: `NAME = "느린길"` → `느린길-YYYYMMDD-vN.zip`. `vendor/`에 win_amd64 wheel, `배포본-정보.txt`.
- 환경 변수: `SLOW_WAY_VAULT`(데이터 폴더), `SLOW_WAY_TODAY`(시험 전용), `PORT`(run.sh), `PYTHONUTF8`.
- 포트: 느린 나이테와 동시에 켤 수 있게 기본 **8010**. 느린 나이테는 8000.
- 업데이트: 서버 끄기 → ZIP 덮어쓰기(vault는 ZIP에 없음) → `setup.bat` → `run.bat`.

## 9. 미결

- [ ] `vault/indexer.py`를 골격만 가져올지, 새로 쓰고 느린 나이테 것은 참고만 할지 (985행 중 골격 비율 확인 후)
- [ ] 프롬프트 동기화(`tools/sync_prompts.py`)를 빌드 단계에 넣을지, 시험만으로 어긋남을 잡을지
- [ ] 첨부 파일 용량 상한
- [ ] 공통 코어 패키지 분리(느린 시리즈): 두 앱이 다 돌아간 뒤 판단
