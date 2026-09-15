# CodexGPT · Claude · VS Code · Higgsfield 비디오 자동화 시스템 설계

## 1. 문서 목적

이 문서는 핵심 `CLAUDE.md` 원칙인 **Command → Agent → Skill**, 기능별 서브에이전트, 사람의 승인 게이트, 점진적 공개를 비디오 제작 자동화에 적용하기 위한 구현 표준을 정의합니다.

목표는 자연어 브리프를 기획, 프롬프트 작성, Higgsfield 생성, 품질 검토, 결과 정리 단계로 처리하는 일관된 파이프라인을 구축하는 것입니다.

> 중요: VS Code는 개발 및 운영 인터페이스이지 비디오 생성 엔진이 아닙니다. 생성은 Higgsfield API가 수행하며, Claude와 CodexGPT는 기획, 코딩, 검토, 오케스트레이션 책임을 나누어 맡습니다.

## 2. 시스템 목표

### MVP 목표

1. 자연어 비디오 브리프를 구조화된 작업 명세로 변환합니다.
2. 필요한 품질, 라이선스, 개인정보 보호 및 안정성 조건을 충족한다면 무료·오픈 API를 최우선으로 사용합니다.
3. 유료 API를 호출하기 전에 제공업체, 과금 기준, 예상 요청 횟수, 예상 비용 또는 최대 비용, 외부로 전송되는 데이터, 이용 가능한 무료 대안을 사용자에게 자세히 설명하고, 사용자가 명시적으로 승인한 경우에만 실행합니다.
4. 비동기 생성 작업을 제출하고 완료될 때까지 추적합니다.
5. 생성된 비디오를 로컬 프로젝트에 다운로드합니다.
6. 기술 검증과 AI 기반 콘텐츠 검토를 분리합니다.
7. 실패한 샷만 재시도하고 전체 실행 기록을 보존합니다.

### 무료 우선 API 정책

- 검토를 마친 무료·오픈 API 허용 목록과 오픈소스 로컬 도구를 기본 제공자 풀로 사용합니다.
- “무료”라는 이유만으로 안전하거나 제약이 없다고 판단하지 않습니다. 연동 전에 라이선스, 상업적 이용 조건, 호출 한도, 데이터 보존 정책, 개인정보 보호 정책 및 결과물 권리를 확인합니다.
- 적합한 오픈소스 모델이나 도구를 현실적으로 로컬에서 사용할 수 있다면 로컬 처리를 우선합니다.
- 무료 제공자의 할당량 부족, 호출 제한 또는 품질 검사 실패를 이유로 사용자에게 알리지 않고 유료 제공자로 전환하지 않습니다.
- 적합한 무료 선택지가 없다면 작업을 중단하고, 유료 선택지와 검토한 무료 대안을 제시한 뒤 사용자의 명시적 승인을 받습니다.

### MVP에서 제외되는 항목

- 완전 무인 게시
- 승인되지 않은 대량 생성
- 자동 결제 또는 크레딧 구매
- 여러 비디오 모델에 걸친 복잡한 라우팅
- 자동화된 After Effects 후반 작업
- YouTube, Instagram 또는 TikTok 자동 업로드

핵심 파이프라인이 안정화된 후 별도 단계에서 이러한 기능을 추가합니다.

## 3. 도구별 책임

### Claude

- 사용자 인터뷰 및 요구사항 정리
- 크리에이티브 브리프 작성
- 스토리 구조와 샷 목록 설계
- Higgsfield용 프롬프트 초안 작성
- 전체 Command → Agent → Skill 흐름 조율
- 사람의 승인 지점 관리
- 결과물의 의미적 품질과 브랜드 품질 검토

### CodexGPT

이 문서에서 CodexGPT는 OpenAI 계열 코딩 에이전트 또는 Codex CLI를 의미합니다.

- TypeScript/Python 구현
- API 클라이언트, 큐, 상태 추적기 구축
- 단위 및 통합 테스트 작성
- 오류 재현 및 수정
- Claude의 구현 계획을 독립적으로 검토
- FFmpeg/ffprobe를 사용한 기술 검증 코드 작성

Claude와 CodexGPT가 같은 파일을 동시에 편집하지 않도록 작업을 분리합니다. 병렬 개발이 필요하면 Git worktree 또는 별도 브랜치를 사용합니다.

### VS Code

- 소스 코드 및 Markdown 명세 편집
- 통합 터미널에서 오케스트레이터 실행
- 환경 변수 및 실행 로그 관리
- 작업 상태 JSON, 프롬프트, 생성된 비디오 검토
- Claude/Codex 확장 프로그램 또는 CLI를 활용한 운영 콘솔 역할

### Higgsfield

- 이미지 및 비디오 생성 요청 처리
- 비동기 작업 ID 발급
- 작업 상태 조회 또는 webhook 알림 제공
- 완료된 미디어 URL 제공

Higgsfield는 선택적 제공자로 취급합니다. 현재 요금제가 무료 우선 정책을 충족하거나 사용자가 유료 사용을 명시적으로 승인한 경우에만 사용합니다.

인증 정보가 필요한 모든 API는 신뢰할 수 있는 서버 측 프로세스에서 호출합니다. API 키나 비밀 정보를 브라우저 코드, 프론트엔드 번들, 생성된 페이지, 스크린샷, 프롬프트, 터미널 출력 또는 클라이언트에 표시되는 오류 메시지에 절대 넣지 않습니다.

## 4. 전체 아키텍처

```text
[사용자 브리프]
       |
       v
[/video-create Command]
       |
       v
[Claude 오케스트레이터]
       |
       +--> 브리프 Agent --------> brief.json
       +--> 디렉터 Agent --------> storyboard.json
       +--> 프롬프트 Agent ------> generation-plan.json
       |
       v
[사람의 승인: 비용, 프롬프트, 샷 수]
       |
       v
[Higgsfield Skill / API Adapter]
       |
       +--> submit
       +--> poll or webhook
       +--> download
       |
       v
[기술 QA + 크리에이티브 QA]
       |
       +--> pass ----------> outputs/final/
       +--> fail ----------> 영향받은 샷만 재생성
```

## 5. Command → Agent → Skill 설계

### Commands

#### `/video-create [brief-file]`

전체 제작 워크플로의 진입점입니다.

1. 입력 브리프 검증
2. 누락된 정보 질문
3. 기획 에이전트 실행
4. 생성 계획 작성
5. 사람의 승인 요청
6. 승인 후 Higgsfield 작업 제출
7. 완료된 결과 검토
8. 실행 보고서 작성

#### `/video-status [project-id]`

- 모든 샷의 queued/running/completed/failed 상태 표시
- 예상 비용과 실제 요청 횟수 표시
- 다운로드 상태와 QA 결과 표시

#### `/video-retry [project-id] [shot-id]`

- 실패하거나 거부된 샷만 재생성
- 기존의 성공한 결과를 덮어쓰지 않음
- 재시도 사유와 수정된 프롬프트 기록

#### `/video-review [project-id]`

- 생성 API를 호출하지 않고 기존 결과 검토
- 해상도, 길이, 코덱, 파일 무결성 확인
- 프롬프트 준수 여부와 장면 일관성 평가

### 기능별 Agents

#### `brief-analyst`

입력:

- 사용 목적
- 플랫폼
- 타깃 오디언스
- 핵심 메시지
- 비디오 길이
- 화면비
- 브랜드 제약 조건
- 참조 이미지

출력: `brief.json`

필수 정보가 누락되면 추론하지 말고 사용자에게 질문합니다.

#### `creative-director`

- 훅, 전개, 결말 구성
- 각 샷의 목적과 전환 정의
- 요청된 범위 안에서 카메라 움직임 설계
- 캐릭터, 제품, 색상의 연속성 규칙 정의

출력: `storyboard.json`

#### `prompt-engineer`

- 샷 설명을 Higgsfield 입력 형식으로 변환
- 피사체, 동작, 환경, 카메라, 조명, 스타일 분리
- 금지 요소 및 연속성 토큰 적용
- 모델별 옵션 검증

출력: `generation-plan.json`

#### `generation-operator`

- 승인된 명세만 API로 전송
- 요청 ID 저장
- polling 또는 webhook을 통해 상태 업데이트
- 결과 파일 즉시 다운로드
- 재시도 정책 적용

#### `qa-reviewer`

- 기술 검토와 크리에이티브 검토 분리
- 실패 사유 구조화
- 재생성이 필요한 샷만 식별
- 원본 생성 결과 보존

## 6. Skills

### `higgsfield-submit`

- 인증 헤더 구성
- 선택한 모델 endpoint로 요청 전송
- `request_id`, `status_url`, `cancel_url` 저장
- 로깅 전에 요청 본문에서 비밀 정보 제거

### `higgsfield-monitor`

- queued, running, completed, failed, nsfw 상태 처리
- 지수 백오프 및 최대 대기 시간 적용
- 중복 polling 방지
- webhook 사용 시 서명 검증

### `media-download`

- 결과 URL이 만료되기 전에 다운로드
- 임시 파일로 다운로드한 뒤 성공 시 원자적으로 이름 변경
- SHA-256, 크기, MIME type 기록

### `technical-video-qa`

`ffprobe`를 사용하여 다음을 검증합니다.

- 파일 읽기 가능 여부
- 비디오/오디오 스트림 존재 여부
- 실제 길이
- 해상도 및 화면비
- 프레임 레이트
- 코덱
- 비정상적으로 작은 파일

### `creative-video-qa`

- 브리프와 장면의 일치도
- 제품, 인물, 텍스트 왜곡
- 샷 간 스타일 연속성
- 카메라 움직임 준수 여부
- 금지된 브랜드 요소 노출 여부

## 7. 데이터 계약

### 프로젝트 상태

```json
{
  "projectId": "campaign-20260914-001",
  "status": "awaiting_approval",
  "createdAt": "2026-09-14T16:00:00+09:00",
  "briefPath": "projects/campaign-20260914-001/brief.json",
  "shotCount": 3,
  "approvedBy": null,
  "approvedAt": null
}
```

### 생성 계획

```json
{
  "projectId": "campaign-20260914-001",
  "provider": "higgsfield",
  "modelEndpoint": "${HIGGSFIELD_MODEL_ENDPOINT}",
  "aspectRatio": "9:16",
  "shots": [
    {
      "shotId": "shot-001",
      "purpose": "첫 2초 안에 제품에 대한 관심 유도",
      "prompt": "승인 전에 작성된 실제 생성 프롬프트",
      "durationSeconds": 6,
      "inputImages": [],
      "generateAudio": false,
      "status": "draft",
      "attempt": 0
    }
  ]
}
```

모델 endpoint와 지원 옵션은 하드 코딩하지 말고 모델 레지스트리 또는 설정을 통해 관리합니다. 요청 필드는 Higgsfield 모델마다 다를 수 있으므로 API 호출 전에 schema를 검증합니다.

### 작업 상태

```text
draft
  -> awaiting_approval
  -> approved
  -> queued
  -> running
  -> downloading
  -> qa_pending
  -> completed

오류 분기:
queued/running -> failed | nsfw | timed_out | cancelled
qa_pending     -> rejected -> approved(regeneration)
```

## 8. 권장 프로젝트 구조

```text
video-automation/
├─ CLAUDE.md
├─ README.md
├─ .env.example
├─ .gitignore
├─ .claude/
│  ├─ commands/
│  │  ├─ video-create.md
│  │  ├─ video-status.md
│  │  ├─ video-retry.md
│  │  └─ video-review.md
│  ├─ agents/
│  │  ├─ brief-analyst.md
│  │  ├─ creative-director.md
│  │  ├─ prompt-engineer.md
│  │  ├─ generation-operator.md
│  │  └─ qa-reviewer.md
│  ├─ skills/
│  │  ├─ higgsfield-submit/SKILL.md
│  │  ├─ higgsfield-monitor/SKILL.md
│  │  ├─ media-download/SKILL.md
│  │  ├─ technical-video-qa/SKILL.md
│  │  └─ creative-video-qa/SKILL.md
│  └─ rules/
│     ├─ video-generation.md
│     └─ secrets-and-cost.md
├─ src/
│  ├─ cli/
│  ├─ orchestration/
│  ├─ providers/higgsfield/
│  ├─ validation/
│  ├─ storage/
│  └─ types/
├─ tests/
│  ├─ unit/
│  ├─ integration/
│  └─ fixtures/
├─ projects/
│  └─ <project-id>/
│     ├─ brief.json
│     ├─ storyboard.json
│     ├─ generation-plan.json
│     ├─ jobs.json
│     ├─ qa-report.md
│     ├─ inputs/
│     ├─ outputs/raw/
│     └─ outputs/final/
└─ logs/
```

## 9. Higgsfield API 통합 원칙

공식 문서에 따르면 Higgsfield는 비동기 생성 API를 제공합니다.

1. Higgsfield를 선택하기 전에 제공자 레지스트리에서 요구사항에 맞는 무료·오픈 제공자를 먼저 확인합니다.
2. 선택한 Higgsfield 요청에 비용이 발생할 수 있다면 제출 전에 Gate 2를 완료하고 명시적인 승인을 받습니다.
3. 서버에서 `Authorization: Key KEY_ID:KEY_SECRET`으로 인증합니다.
4. 모델 endpoint에 생성 요청을 제출합니다.
5. 반환된 `request_id`와 상태 URL을 영구 저장합니다.
6. `/requests/{request_id}/status` polling 또는 webhook을 통해 완료 여부를 확인합니다.
7. 완료된 URL의 장기 보존은 보장되지 않으므로 결과를 즉시 로컬 또는 object storage로 복사합니다.

필수 환경 변수:

```dotenv
HIGGSFIELD_API_KEY_ID=
HIGGSFIELD_API_KEY_SECRET=
HIGGSFIELD_MODEL_ENDPOINT=
HIGGSFIELD_WEBHOOK_SECRET=
OUTPUT_ROOT=./projects
MAX_CONCURRENT_JOBS=2
MAX_RETRIES=2
POLL_INTERVAL_MS=5000
```

`.env`를 Git에 커밋하지 않습니다. `.env.example`에는 빈 값과 함께 키 이름만 기록합니다.

### API 인증 정보 보안

- 비밀 정보는 서버 측 환경 변수 또는 운영체제·클라우드 비밀 관리 도구에서만 읽습니다.
- `.env`, 인증 정보 파일 및 로컬 비밀 저장소를 버전 관리 대상에서 제외하고 정적 호스팅으로 공개되지 않게 차단합니다.
- API 요청은 백엔드 어댑터를 통해 전송하며, 브라우저에는 정제된 작업 ID, 상태 및 결과 참조만 전달합니다.
- 로그와 오류 보고서에서 인증 헤더, 쿼리 문자열 토큰, 비밀 정보 및 서명된 URL을 마스킹합니다.
- 터미널, 프롬프트, 생성된 Markdown/HTML, 스크린샷, 텔레메트리 또는 분석 도구에 비밀 값을 출력하지 않습니다.
- 각 키의 권한을 필요한 최소 범위로 제한하고, 노출이 의심되면 교체하며, 개발용과 운영용 인증 정보를 분리합니다.
- 자동 비밀 정보 검사를 추가하고 응답과 로그에서 인증 정보가 노출되지 않는지 테스트합니다.

## 10. 사람의 승인 게이트

다음 작업은 사용자 확인 후에만 실행합니다.

### Gate 1 — 브리프 승인

- 목적 및 플랫폼
- 비디오 길이 및 화면비
- 브랜드 및 금지 요소

### Gate 2 — 유료 생성 승인

- 요구사항을 충족하는 적합한 무료·오픈 선택지가 없다는 확인
- 제공업체 및 모델
- 최종 프롬프트
- 샷 수
- 과금 기준, 예상 요청 횟수, 예상 총비용 및 절대 비용 상한
- 제공업체로 전송되는 데이터와 파일
- 검토한 무료 대안과 선택하지 않은 이유
- 참조 이미지 사용 권한
- 타임스탬프와 함께 기록된 사용자의 명시적 승인. 침묵하거나 기본 선택지를 그대로 둔 것은 승인으로 간주하지 않습니다.

### Gate 3 — 재생성 승인

- 실패 사유
- 프롬프트 변경 사항
- 추가 비용

### Gate 4 — 게시 승인

- 최종 선택 결과
- 자막, 음악 저작권, 초상권 검토
- 게시 플랫폼 및 공개 범위

## 11. 오류 및 재시도 정책

- HTTP 429/5xx: 지수 백오프를 적용하여 제한된 횟수만 재시도
- 인증 오류: 즉시 중단하고 키 설정 검증
- 입력 schema 오류: 자동 재제출 없이 계획 수정
- NSFW 거부: 우회 프롬프트를 자동 생성하지 않고 사용자에게 보고
- timeout: 서버 상태를 다시 조회한 뒤 중복 제출 여부 확인
- 다운로드 오류: 동일한 결과 URL을 사용하여 다운로드만 재시도
- QA 실패: 전체 프로젝트가 아니라 영향받은 샷만 재생성

모든 API 제출에 내부 idempotency key를 할당하고, 재시작 후에도 동일한 승인 작업이 중복 제출되지 않도록 방지합니다.

## 12. 검증 전략

### 단위 테스트

- 브리프 schema 검증
- 상태 전이 규칙
- 인증 헤더 생성
- 비밀 정보 마스킹
- 재시도 가능 오류와 재시도 불가능 오류 분류
- 출력 파일 이름 충돌 방지

### 통합 테스트

- mock Higgsfield 서버에 제출
- queued → running → completed 흐름
- failed, nsfw, timeout 흐름
- 프로세스 재시작 후 작업 복구
- 중복 webhook 방지

### 라이브 API 스모크 테스트

- 가장 짧고 저렴한 설정으로 샷 하나 생성
- 생성 전에 사용자로부터 비용 승인 획득
- 결과를 다운로드하고 ffprobe로 검증
- API 응답이 민감한 데이터를 로그에 유출하지 않는지 확인

## 13. 구현 단계

### Phase 0 — 의사결정

- 구현 언어: TypeScript 권장
- 운영 모델: CLI 우선
- 상태 저장: MVP에서는 JSON 파일, 다중 사용자 단계에서는 SQLite/PostgreSQL
- polling으로 시작하고 운영 단계에서 webhook 추가

### Phase 1 — 최소 수직 슬라이스

브리프 하나를 받아 샷 하나를 생성하고 다운로드합니다.

완료 기준:

- 저장소 또는 로그에 비밀 정보 없음
- 요청 ID 기록
- 중단 후 상태 polling 재개 가능
- 결과 파일이 기술 검증 통과

### Phase 2 — 다중 샷

- 샷 목록
- 동시 실행 제한
- 부분 실패 및 샷별 재시도
- 프로젝트 상태 보기

### Phase 3 — AI 검토

- Claude 콘텐츠 QA
- CodexGPT 기술 QA 검토
- 실패 사유에 기반한 수정 프롬프트 제안
- 사람의 승인 후 선택적 재생성

### Phase 4 — 후반 작업

- FFmpeg 통합
- 샷 연결
- 자막, 음량 정규화, 썸네일
- 필요할 때 별도의 After Effects adapter 사용

### Phase 5 — 운영

- Webhooks
- 데이터베이스 및 object storage
- 예산 및 사용량 대시보드
- 사용자 권한
- 게시 플랫폼 통합

## 14. Claude와 CodexGPT 협업 규칙

1. Claude가 요구사항과 승인된 구현 계획을 작성합니다.
2. CodexGPT가 코드와 테스트를 포함한 좁은 범위의 구현 단위를 완성합니다.
3. Claude가 브리프와 워크플로 관점에서 결과를 검토합니다.
4. CodexGPT가 테스트, type, 오류 처리를 교차 검토합니다.
5. 한 에이전트가 작성한 코드는 다른 에이전트가 독립적으로 검증합니다.
6. 모든 조사 결과를 메인 컨텍스트에 불러오는 대신 결론과 근거 파일만 전달합니다.
7. 각 기능 작업의 범위는 한 세션 컨텍스트의 절반 이내에서 완료할 수 있을 만큼 작게 설정합니다.

## 15. 완료 기준

다음 조건을 모두 충족하면 MVP가 완료된 것입니다.

- 자연어 브리프가 검증 가능한 JSON으로 변환됩니다.
- 유료 제공자를 사용하기 전에 무료·오픈 제공자를 먼저 평가하고 우선 사용합니다.
- 자세한 비용 설명과 사용자의 명시적 승인 없이 유료 요청을 실행하지 않습니다.
- 모든 Higgsfield 요청의 전체 lifecycle이 저장됩니다.
- 진행 중인 작업은 프로세스 재시작 후 재개할 수 있습니다.
- 성공한 결과는 로컬에 다운로드되고 checksum이 기록됩니다.
- 기술 QA 결과가 자동으로 생성됩니다.
- 실패한 샷만 재시도할 수 있습니다.
- API 키와 비밀 정보가 브라우저, 프론트엔드 번들, 프롬프트, 터미널 출력, 스크린샷, 코드, Git, 로그 또는 결과 보고서에 나타나지 않습니다.
- mock 기반 자동화 테스트와 라이브 one-shot 스모크 테스트를 통과합니다.

## 16. 다음 단계

1. 새 프로젝트 root와 구현 언어를 확정합니다.
2. 200줄 이하의 비디오 자동화 전용 `CLAUDE.md`를 작성합니다.
3. `brief.schema.json`과 `generation-plan.schema.json`을 생성합니다.
4. 공식 Higgsfield SDK와 REST 중 하나를 선택합니다.
5. `/video-create`용 one-shot 수직 슬라이스를 먼저 구현합니다.
6. 라이브 API 호출 전에 mock 통합 테스트를 완료합니다.

## 참고 자료

- 원본 설계 원칙: 현재 디렉터리의 `CLAUDE.md`
- Higgsfield 공식 문서: <https://docs.higgsfield.ai/docs>
- Higgsfield Node.js/TypeScript SDK: <https://github.com/higgsfield-ai/higgsfield-js>

Higgsfield 모델 endpoint와 입력 필드는 모델 및 API 버전에 따라 달라질 수 있습니다. 구현 시점에 최신 공식 schema를 다시 확인하고, 문서화되지 않은 필드는 절대 추론하거나 전송하지 않습니다.
