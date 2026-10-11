# FreePli 일정 취소와 GitHub 단독 실행 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 미실행 제작의 안전한 일정 취소를 웹앱에 제공하고, 승인된 일정이 GitHub-hosted Actions에서 PC 실행기 없이 처리되도록 한다.

**Architecture:** `freeplist-mobile-runner`의 기존 상태 브랜치, 스케줄 기록 검증, cloud CLI 및 제작 엔진을 재사용한다. 상태 브랜치에 저장된 일정은 GitHub-hosted 주기 실행에서 조회하고, 취소 여부와 승인 시각을 확인한 후 이미 연결된 동일 request ID만 기존 엔진으로 전달한다. 웹앱은 식별된 편집 가능한 소스에서 일정 상태와 Actions 실행 상태를 표시한다. PC 실행기는 hosted 검증과 미해결 요청 정리 후 마지막 단계에서만 등록 해제한다.

**Tech Stack:** GitHub Actions `ubuntu-latest`, Python 3.11, 기존 `cloud.cli` / 상태 브랜치 기록 스키마, FreePli Studio 기존 웹 기술 및 기존 테스트 도구. 새 프레임워크·데이터베이스·워크플로 엔진은 추가하지 않는다.

**Spec:** [2026-10-11-hosted-schedule-and-runner-removal-design.md](../specs/2026-10-11-hosted-schedule-and-runner-removal-design.md)

## Global Constraints

- 기존 음악·완곡 영상·숏폼 생성 및 YouTube 업로드 엔진은 재작성하지 않는다.
- GitHub-hosted `ubuntu-latest`만 제작 실행 장소로 허용하고 `self-hosted` 실행 경로를 두지 않는다.
- 기존 콘텐츠·요청·산출물·게시 이력은 보존하고 기존 결과물을 재사용한다.
- 외부 제출·게시 ID가 있거나 상태가 불명확한 요청은 일정 삭제만으로 취소하지 않는다.
- 취소 기록은 tombstone으로 보존하며 로컬 스냅샷이 취소 항목을 되살리지 못하게 한다.
- 실제 소스 저장소가 확인되기 전에는 배포 번들 `assets/index-*.js`를 직접 편집하지 않는다.
- 승인 시각이 지났거나 예약 시각 준수가 불가능한 요청을 즉시 공개하지 않는다.
- GitHub API/Actions 충돌에서는 최신 상태를 다시 읽고, 확인 전 재전송·외부 작업 재제출을 하지 않는다.
- PC 실행기 제거는 hosted 경로 통과 및 미해결 작업 확인 이후에만 수행한다.

## Review Focus

- 상태 브랜치의 예상 버전이 달라진 취소 요청은 거부하고 최신 일정을 다시 읽는다. **Task 2**의 CAS 충돌 테스트.
- Kaggle 제출·외부 작업 ID·Drive 파일·YouTube delivery가 존재하는 요청은 취소하지 않는다. **Task 2**의 각 제출 증거별 테스트.
- 승인 전/시작 시각 전 일정은 스케줄러가 시작하지 않는다. **Task 3**의 due 필터 테스트.
- 반복된 `cloud-advance` 실행과 동일 일정의 취소/승인이 중복 Kaggle 제출 또는 업로드를 만들지 않는다. **Task 3/5**의 동일 request ID 멱등 테스트.
- 취소된 일정과 미해결 일정은 웹앱 새로고침·스냅샷 동기화 후 활성 목록에 복구되지 않는다. **Task 4**의 UI/API 테스트.

---

## 파일 책임 지도

현재 FreePli Studio 배포 저장소 `SolidLAB-korea/freeplist-studio` 기본 브랜치에는 정적 HTML과 해시된 자산만 확인된다. 앱 원본 저장소 및 Pages 빌드 입력은 아직 식별되지 않았으므로 먼저 매핑한다. 원본을 찾지 못하면 UI 코드는 수정하지 않고 해당 단계에서 중단한다.

- `SolidLAB-korea/freeplist-mobile-runner/record_status.py`: 공유 일정 기록 형식, 버전 검사, 상태 브랜치에 저장할 payload 검증.
- `SolidLAB-korea/freeplist-mobile-runner/.github/workflows/record-status.yml`: 일정 저장/취소 이벤트의 GitHub-hosted 쓰기 진입점.
- `SolidLAB-korea/freeplist-mobile-runner/cloud/store.py`: 기존 상태 브랜치의 일정 및 제작 요청 목록 조회.
- **신규** `SolidLAB-korea/freeplist-mobile-runner/cloud/schedules.py`: 취소/승인/시작 시각을 검사해 due request ID를 고르는 순수 로직.
- `SolidLAB-korea/freeplist-mobile-runner/cloud/cli.py`: 기존 `advance` 동작에 due 일정 후보를 연결하되, 기존 engine/dispatcher 사용.
- `SolidLAB-korea/freeplist-mobile-runner/.github/workflows/cloud-advance.yml`: 이미 `ubuntu-latest`이며 주기 실행됨. 필요한 경우에만 일정 처리에 필요한 최소 변경.
- `SolidLAB-korea/freeplist-mobile-runner/tests/test_record_status.py`, `tests/cloud/test_*.py`, `tests/test_actions_only.py`: 검증.
- FreePli Studio의 **실제 원본 저장소/파일은 Task 1에서 확정**한다. 기존 일정 화면, 상태 API client, 테스트 파일만 수정하며 생성된 정적 번들은 빌드 결과로 갱신한다.

## Task 1: 실행·저장 소스와 일정 데이터 대조

**Files:**
- Read: `SolidLAB-korea/freeplist-studio` 기본 브랜치 정적 엔트리와 배포 workflow/설정
- Read: `SolidLAB-korea/freeplist-mobile-runner/record_status.py`, `cloud/store.py`, `cloud/cli.py`, `cloud/dispatcher.py`, workflow 파일 및 상태 브랜치 기록
- Read-only: 현재 등록된 PC runner, 서비스, 설치 경로 및 GitHub Actions runner 상태

**Interfaces:**
- Produces: source map; 활성 schedule record IDs; 각 release의 request ID / Kaggle external job / Drive artifact / platform delivery 매핑; editable Studio source location; PC runner ID/service/path.
- Consumes: 승인된 설계서와 기존 상태·workflow 정보.

- [ ] **Step 1: 배포 원본 추적**
  `freeplist-studio`의 index 참조 번들, GitHub Pages 배포 설정/워크플로와 연결된 빌드 입력을 따라 편집 가능한 소스 저장소·파일·테스트 명령을 기록한다.
- [ ] **Step 2: 현재 일정의 읽기 전용 대조**
  `freeplist-status`의 `cloud/schedules/*.json`, `cloud/requests/*.json`, `cloud/external/*`, `cloud/deliveries/*`, Actions run 목록, 연결된 Drive/YouTube 참조를 release ID 단위로 비교한다. Kaggle/Drive/YouTube 외부 확인은 조회만 한다.
- [ ] **Step 3: 실행기 현황의 읽기 전용 대조**
  GitHub runner ID·상태·그룹과 PC의 FreePli 전용 서비스/실행 경로를 확인한다. 아직 중지·등록 해제·파일 제거를 하지 않는다.
- [ ] **Step 4: 안전 게이트 작성**
  분류를 `unstarted`, `submitted_or_running`, `published_or_artifacts_exist`, `unknown`으로 기록한다. 데이터 매핑이 없거나 Studio 원본을 찾지 못하면 그 사실을 보고하고 Task 2 이후로 진행하지 않는다.
- [ ] **Verification:** 취소·삭제·dispatch·runner 변경이 발생하지 않았음을 확인하고, 모든 표시 중 일정에 대한 매핑 결과를 검토할 수 있게 남긴다.

## Task 2: 취소 tombstone과 fail-closed 상태 기록

**Files:**
- Modify: `freeplist-mobile-runner/record_status.py`
- Modify: `freeplist-mobile-runner/.github/workflows/record-status.yml`
- Modify: `freeplist-mobile-runner/tests/test_record_status.py`

**Interfaces:**
- Record writer accepts `record_type=schedule_cancel` with `record_id`, `release_id`, `expected_updated_at`, and UTC `cancelled_at`.
- Release schema explicitly allows `request_id`, `production_approved`, `execution_start_at`, and `cancelled_at`; `request_id` is the stable `gh-...` production request key. Legacy releases missing the approval/start fields remain unapproved and are never auto-started.
- Writer returns success only after it verifies the latest schedule revision and confirms the linked request has no Kaggle submission/external job, Drive artifact, or platform delivery ID.
- Cancellation sets the release's `production_status=cancelled` and `cancelled_at`; it keeps the release and schedule record for audit. Existing non-cancelled schedule writes keep their current contract.

- [ ] **Step 1: 취소 실패 사례 테스트 작성**
  `tests/test_record_status.py`에 stale `expected_updated_at`, 연결 요청의 Kaggle/external job, Drive artifact, delivery ID, 식별 불가 상태에서는 취소가 거부되고 원본 schedule이 유지되는 테스트를 추가한다.
- [ ] **Step 2: 테스트가 실패함을 확인**
  Run: `python -m unittest discover -s tests -p 'test_record_status.py'`
  Expected: 새 cancel type/guard 테스트만 실패한다.
- [ ] **Step 3: 취소 스키마 및 기록 검사 구현**
  `record_status.py`에서 release tombstone 필드를 엄격 검증하고, 요청/외부작업/전달 기록을 검사한다. `.github/workflows/record-status.yml`은 선택지에 `schedule_cancel`을 추가한다. 경로는 기존 상태 브랜치의 `schedules/`, `requests/`, `external/`, `deliveries/` 아래로 제한한다.
- [ ] **Step 4: 기존 schedule/timing 테스트 회귀 확인**
  Run: `python -m unittest discover -s tests -p 'test_record_status.py'`
  Expected: 취소 보호 테스트와 기존 schema/CAS/idempotency 테스트 모두 PASS.
- [ ] **Step 5: 변경을 작은 커밋으로 저장**
  `feat: add guarded schedule cancellation tombstones`.

## Task 3: 승인된 due 일정만 기존 요청으로 연결

**Files:**
- Create: `freeplist-mobile-runner/cloud/schedules.py`
- Modify: `freeplist-mobile-runner/cloud/store.py`
- Modify: `freeplist-mobile-runner/cloud/cli.py`
- Create: `freeplist-mobile-runner/tests/cloud/test_schedules.py`
- Modify: `freeplist-mobile-runner/tests/cloud/test_cli.py`

**Interfaces:**
- `due_request_ids(store, now_utc: datetime) -> list[str]` returns unique request IDs for releases with `production_approved=true`, an elapsed UTC `execution_start_at`, `production_status=planned`, no `cancelled_at`, and a linked request that can be safely advanced.
- `RELEASE_FIELDS` gains only `request_id`, `production_approved`, `execution_start_at`, and `cancelled_at`; validation requires request IDs to match the existing safe-ID pattern, `production_approved` to be a boolean, and start/cancel timestamps to be UTC ISO-8601. Existing records without these fields validate as unapproved.
- `FileStore.list_schedules() -> list[dict]` and `GitHubStore.list_schedules() -> list[dict]` read only validated schedule records under the existing status branch `cloud/schedules/`.
- Due releases with missing request payload, conflicting status, or uncertain external submission are excluded and recorded/reported as `needs_attention`; they are never rebuilt from title-only schedule metadata.
- `cloud.cli advance` invokes the existing `advance(request_id, engine, store)` for eligible IDs; it does not add another generation/upload implementation.

- [ ] **Step 1: due selection 실패 사례 테스트 작성**
  `test_schedules.py`에서 미승인, 시작 전, 취소, 누락된 request, needs_attention, dispatch_uncertain 일정이 선택되지 않음을 고정한다.
- [ ] **Step 2: due selection 테스트 실패 확인**
  Run: `python -m unittest discover -s tests/cloud -p 'test_schedules.py'`
  Expected: 모듈/함수 부재로 실패.
- [ ] **Step 3: 기존 store에 schedule list reader 추가**
  `cloud/store.py`의 FileStore/GitHubStore가 기존 `cloud/schedules` 기록만 읽고 정렬 가능한 데이터를 반환하게 한다. 경로 검증을 재사용한다.
- [ ] **Step 4: 순수 due selector 구현**
  `cloud/schedules.py`에서 UTC 시각과 기존 record schema로 후보만 계산하고 중복 ID를 제거한다. 날짜 문자열은 명시 시간대가 아니면 거부한다.
- [ ] **Step 5: 기존 advance 흐름에 후보 전달**
  `cloud/cli.py`의 `advance` 경로에서 due selector를 호출하되, 같은 request ID에 대해 기존 `advance`를 한 번만 실행한다. 기존 직접 request 실행/재개 의미를 회귀시키지 않는다.
- [ ] **Step 6: CLI/store/engine 테스트**
  Run: `python -m unittest discover -s tests/cloud`
  Expected: due 후보만 처리되고 기존 요청 상태 전이와 중복 방지 테스트가 PASS.
- [ ] **Step 7: 작은 커밋으로 저장**
  `feat: advance approved due schedules on hosted actions`.

## Task 4: 웹앱 일정 삭제와 PC 상태 의존 제거

**Files:**
- Modify: Task 1에서 확인한 Studio 원본의 일정 목록/상태 client
- Test: 같은 원본 저장소의 일정 UI/API 테스트
- Build output: 저장소의 기존 Pages 빌드 절차가 생성한 파일만 반영

**Interfaces:**
- Cancel action dispatches the existing record-status workflow with `schedule_cancel`, expected revision, and exact release ID.
- Status display reads the current GitHub schedule + request/delivery state and the latest Actions run summary; PC runner availability does not gate schedule save, display, or execution.
- The screen hides only a confirmed cancelled release; unknown/in-flight releases remain visible with a reason.

- [ ] **Step 1: UI 실패 상태 테스트 작성**
  미실행 취소 성공 시 목록에서 사라지고, 진행 중·외부 ID 존재·unknown·409/422 시 일정은 남고 안전한 이유를 표시하는 테스트를 추가한다.
- [ ] **Step 2: 테스트 실패 확인**
  원본 저장소의 공식 UI test command로 새 테스트만 실행해 실패를 확인한다.
- [ ] **Step 3: UI와 client 변경**
  확인된 원본 파일에서 release 단위 취소, 최신 revision fetch, 취소 후 목록 재조회, 실패 상태 표시를 구현한다. PC 실행기 온라인 여부를 진행 불가 메시지나 저장 차단으로 사용하지 않게 한다.
- [ ] **Step 4: 저장소의 정식 빌드/test 수행**
  기존 스크립트만 사용하고 생성 자산을 직접 수정하지 않는다. bundle에 PC runner gating 문구가 제거되고 일정 취소 API에 올바른 ID/revision이 들어가는지 확인한다.
- [ ] **Step 5: 작업 내용을 커밋**
  해당 웹앱 저장소의 기존 커밋 규칙을 따른다.

## Task 5: GitHub-hosted dry-run과 비운영 검증

**Files:**
- Modify if needed: `freeplist-mobile-runner/.github/workflows/cloud-advance.yml`
- Modify: `freeplist-mobile-runner/tests/test_actions_only.py`
- Modify: 위 Task 2–4에서 추가한 테스트

**Interfaces:**
- Production runner stays `ubuntu-latest`; scheduled action reads approved due schedule records from the status branch.
- Dry-run accepts no external side effects and does not submit Kaggle, create Drive artifacts, or upload to YouTube.
- Test fixtures use a dedicated schedule/request ID and mock store/adapters; they do not write to the user's production status branch.

- [ ] **Step 1: Actions-only 회귀 검사 추가**
  `tests/test_actions_only.py`에서 workflows에 `runs-on: ubuntu-latest`가 있고 `self-hosted` 및 로컬 scheduler 경로가 없음을 검사한다.
- [ ] **Step 2: 테스트 실패 확인**
  Run: `python -m unittest discover -s tests -p 'test_actions_only.py'`
  Expected: 새 schedule execution invariant 검사만 실패.
- [ ] **Step 3: 최소 workflow 연결 수정**
  `cloud-advance.yml`은 현재 이미 주기 `7,22,37,52 * * * *`와 `ubuntu-latest`를 사용한다. scanner 연결에 필요한 최소 차이만 반영하고 인증 secret이나 generation engine을 변경하지 않는다.
- [ ] **Step 4: hosted CI 전체 테스트**
  Run: `python -m unittest discover -s tests` in GitHub Actions CI.
  Expected: 모든 cloud, status, timing, Actions-only 테스트 PASS.
- [ ] **Step 5: dry-run/pilot 검증**
  전용 fixture와 read-only status 조회로 시작 전/미승인/취소/외부 작업 있음/중복 실행을 확인한다. 실제 production 요청의 Kaggle submit, Drive write, YouTube upload는 하지 않는다.
- [ ] **Step 6: 검토 가능한 PR 제공**
  runner repo와 Studio 원본 repo의 관련 변경을 각각 작은 PR로 만들고, 양쪽 CI 및 배포 빌드 증거를 모은다. PR merge 및 실제 제작 승인 없이는 운영 main에 실행을 적용하지 않는다.

## Task 6: 실행기 제거 컷오버

**Files / Systems:**
- Read-only verify first: GitHub self-hosted runner registration and pending jobs
- Remove only: identified FreePli-specific runner service, registration, installation directory and runner-only credential
- Keep: shared GitHub CLI, unrelated credentials, request/status history, Drive files and media

**Interfaces:**
- Removal precondition: hosted workflow and website status path verified; no self-hosted job reference; no in-flight/unknown production request; future schedule is represented in GitHub status records.
- Removal operation is idempotent and records runner ID, stopped service, removed installation path, and completion time without logging credential values.

- [ ] **Step 1: 컷오버 전 게이트 실행**
  모든 workflow가 hosted runner인지, 미래 승인 일정과 현재 요청이 GitHub에 보이는지, 저장된 delivery IDs와 외부 Kaggle job 상태가 일치하는지 다시 조회한다.
- [ ] **Step 2: hosted 결과 read-only 확인**
  테스트 workflow/요청 상태만 조회한다. 기능상 운영 요청 실행이 필요해지는 경우 별도 승인 없이 submit/upload 하지 않는다.
- [ ] **Step 3: 전용 runner 제거**
  확인된 FreePli runner registration을 unregister하고 해당 PC service를 중지한 뒤 전용 설치 경로만 제거한다. 경로가 예상과 다르거나 공용 요소와 분리가 안 되면 중단한다.
- [ ] **Step 4: 제거 후 독립 동작 확인**
  웹앱이 PC runner 상태 없이 schedule list/status를 읽고, Actions history가 hosted 상태를 표시하는지 확인한다. 실제 산출/공개 검증은 별도 운영 실행이 승인된 경우에만 수행한다.
- [ ] **Step 5: 최종 요약**
  제거 증거, CI 결과, schedule 상태, 미검증 운영 연동을 구분해 보고한다.

## Completion Verification

- `python -m pytest tests/test_record_status.py tests/cloud/test_schedules.py tests/cloud/test_cli.py tests/cloud/test_engine.py tests/cloud/test_dispatcher.py tests/test_actions_only.py -q` passes on the runner repository.
- GitHub Actions full test suite passes on `ubuntu-latest`.
- UI test/build passes in the confirmed Studio source repository.
- Read-only schedule inventory has zero unexplained releases; unresolved releases remain visible and unmodified.
- No test generated a new Kaggle job, Drive artifact, or YouTube upload.
- Before runner removal, hosted status and dispatch are proven without relying on the PC runner.
- After runner removal, the PC service/registration is absent while GitHub-hosted scheduling and history remain accessible.
