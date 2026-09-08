# design.md — 로컬 강의 노트 (STT + PDF 페이지 연계)

- 작성일: 2026-09-08
- 작성자: devingryu@korea.ac.kr
- 상태: Draft
- 관련 티켓: DVRY-1

## 배경 (Background)

강의 중 교수의 구두 설명을 로컬 STT로 남기되 어느 PDF 페이지에 대한 설명이었는지까지 연결해 저장하는 개인용 Python CLI를 만든다. 확정된 요구는 [requirements.md](requirements.md)에, 실행 환경·후보 프레임워크 사실은 [survey.md](survey.md)에 있다.

- 캡처(`record`: 녹음 + 실시간 넘김 로깅)와 STT·노트 생성(`process`)을 분리한 사후 처리 MVP가 이 문서의 완료 범위다.
- 넘김 로그는 라이브 캡처와 사후 태깅 두 경로로 만들어지고, `process`가 이를 STT 세그먼트 시각과 매핑해 페이지별 `.md`를 생성한다.

## 목표 (Goals)

- 완전 로컬·오프라인으로 강의 오디오를 캡처하고, 한국어·영어·혼용(코드 스위칭) 발화를 전사한다.
- 강의 중 페이지 넘김을 타임스탬프로 기록하고, 사후 태깅으로 누락·오조작을 교정하거나 로깅 없는 녹음을 수동 태깅으로 살린다.
- STT 세그먼트를 넘김 로그 시간축과 매핑해 페이지별 마크다운 노트를 남긴다.
- 로컬 STT 후보를 실측 비교해 한 모델을 선정하는 벤치마크 프로토콜을 확정한다.

## 비목표 (Non-Goals)

- 라이브 STT(강의 중 실시간 전사)는 후속 자식 이슈로 다루며 이 문서 범위 밖이다.
- 강의실 마이크의 원거리·잡음 제거나 음성 향상은 MVP에서 구현하지 않는다.
- PDF 슬라이드 내용 자동 감지(슬라이드 이미지·OCR로 페이지 전환 자동 인식)는 하지 않으며, 페이지 전환은 사람이 기록한 넘김 로그에만 의존한다.
- 화자 분리, 요약·번역, 클라우드 STT 연동은 하지 않는다.

## 설계 (Design)

각 작업 항목은 implement 한 세션 크기로 쪼갰고, "구현할 파일"로 스코프를 고정한다. 코드는 gitignore된 `repos/` 아래 새 레포에 두며, 레포 이름은 `lecnote`로 제안한다(확정은 [미결 질문](#미결-질문-open-questions)).

### 공통 결정 (전 항목 전제)

전 항목이 공유하는 데이터 계약과 파일 배치다. 이 결정이 매핑·재실행·재튜닝 동작을 좌우해 먼저 고정한다.

- 세션 디렉토리 레이아웃 — `repos/lecnote/sessions/<session-id>/` 아래에 둔다.
  - `meta.json`: 세션 메타. 필드는 `session_id`, `created_at`(ISO8601), `pdf_path`(원본 PDF 참조 경로, 복사하지 않음), `audio_file`, `sample_rate`, `channels`, `debounce_seconds`(기본 3).
  - `audio.wav`: 캡처 오디오(무압축 WAV, 모노). 포맷·샘플레이트 기본값은 항목 3의 subagent 판단 사항.
  - `pages.jsonl`: 넘김 로그(아래 스키마).
  - `transcript.jsonl`: `process`가 만드는 STT 세그먼트(아래 스키마).
  - `notes/page-NNN.md`: `process`가 만드는 페이지별 노트.
- 넘김 로그 `pages.jsonl` — 이벤트당 한 줄 JSON.
  - 필드: `t`(녹음 시작 기준 초, float), `page`(넘김 후 절대 페이지 번호, int), `source`(`live`|`manual`).
  - `record`는 raw 이벤트를 append-only로 쓰고 절대 되쓰지 않는다. 디바운스는 로그를 변형하지 않는다.
- 디바운스 적용 시점 — 기록 시점이 아니라 `process`(정규화 단계)에서 적용한다.
  - 근거: raw 로그를 불변으로 두면 임계값(3초)을 바꿔도 재녹음 없이 재정규화만으로 결과를 다시 뽑을 수 있다. 요구사항이 임계값을 조정 가능하게 남기라고 했다.
  - 규칙: 어떤 페이지로 넘긴 뒤 `debounce_seconds` 이내에 직전 페이지로 되돌아오면 그 왕복 두 이벤트를 취소한다. 초과 후 복귀는 별개 넘김으로 유지한다.
- STT 세그먼트 `transcript.jsonl` — 세그먼트당 한 줄 JSON.
  - 필드: `start`(초, float), `end`(초, float), `text`(str), `lang`(선택, 모델이 주면 기록).

### 항목 1 — STT 후보 벤치마크·선정

로컬 실행되고 한/영/혼용에 쓸 수 있는 후보를 실측 비교해 한 모델을 고른다. 후보의 현재 버전·정확도·속도는 이 설계 시점에 확인할 수 없으므로(참고: [survey.md](survey.md)) implement 첫 작업에서 설치·측정한다.

- 후보군 — 가속 경로 축으로 묶는다([survey.md](survey.md)에 아키텍처 사실 정리).
  - Metal(GGML): whisper.cpp — Python 바인딩으로 CLI에서 호출.
  - Apple MLX: mlx-whisper 계열 — pip 설치형, Metal 네이티브.
  - CoreML/ANE: WhisperKit — Swift라 서브프로세스·브리징 필요, 설치 비용 항목에 반영.
  - CTranslate2: faster-whisper — Mac에서는 CPU 실행(Metal 경로 없음), CPU 기준선으로 포함.
- 비교 기준 — 축별로 측정한다.
  - 정확도: 한국어 CER, 영어 WER, 혼용 세그먼트 CER를 각각 낸다.
  - 속도·자원: RTF(처리시간/오디오길이), 피크 메모리(RSS).
  - 운영: 설치·유지보수 난이도(Python 바인딩 유무, 모델 다운로드·업데이트 방식), 코드 스위칭 대응(단일 다국어 모델로 혼용 세그먼트를 유실 없이 전사하는지).
- 측정 프로토콜 — 샘플과 커트라인을 고정한다.
  - 샘플: 한국어 강의성 음성, 영어 강의성 음성, 한 발화 안에 둘이 섞인 혼용 음성 각 1개 이상. 각 샘플에 사람이 만든 기준 전사(정답)를 둔다.
  - 지표 산출: 후보×샘플로 위 정확도·속도·자원을 표로 남긴다.
  - 선정 커트라인: RTF < 1.0(실시간보다 빠름)과 피크 메모리 48GB 예산 내를 통과한 후보 중 한국어 CER가 가장 낮은 모델을 고르고, 혼용 CER가 크게 나쁜 모델은 배제한다. 구체 임계 수치는 측정 기준선으로 확정한다.
- 산출물: 벤치마크 스크립트, 샘플·정답 매니페스트, 결과 표와 선정 근거 문서, 선정 모델의 로드·전사 어댑터가 만족할 인터페이스 메모.
- 구현할 파일:
  - `repos/lecnote/benchmarks/run_bench.py`
  - `repos/lecnote/benchmarks/samples/manifest.json`
  - `repos/lecnote/benchmarks/RESULTS.md`

### 항목 2 — 프로젝트 스캐폴딩

레포·CLI 뼈대·의존성과 워크스페이스 steering 규약을 세운다. 이후 항목이 이 뼈대 위에 붙는다.

- Python 프로젝트: `pyproject.toml`로 패키지·엔트리포인트 정의, `lecnote` 콘솔 스크립트가 `record`/`tag`/`process` 서브커맨드를 노출.
- 최상위 구조: `src/lecnote/` 아래 `cli.py`, `session.py`(세션 경로·스키마 헬퍼), 빈 `record.py`/`tag.py`/`process.py` 뼈대.
- steering 규약: 워크스페이스는 per-repo steering을 `.agents/steering/<repo>.md`에 두고 레포 안에는 그 파일을 가리키는 `AGENTS.md` stub을 둔다(AGENTS.md의 Per-repo steering). 이 항목에서 둘 다 만든다.
- 구현할 파일:
  - `repos/lecnote/pyproject.toml`
  - `repos/lecnote/src/lecnote/__init__.py`
  - `repos/lecnote/src/lecnote/cli.py`
  - `repos/lecnote/src/lecnote/session.py`
  - `repos/lecnote/AGENTS.md` (steering stub)
  - `/Users/hyungseok/personal-repos/my-playground/.agents/steering/lecnote.md`

### 항목 3 — `record` (녹음 + 넘김 로깅)

강의 중 마이크를 녹음하며 페이지 넘김을 raw 이벤트로 기록한다. 디바운스는 여기서 적용하지 않는다(공통 결정).

- 넘김 입력 수단: 전역 단축키를 택한다.
  - 근거: 강의 중에는 PDF 뷰어가 포커스를 잡으므로, 포커스를 CLI로 옮겨야 하는 터미널 키 입력은 넘김과 충돌한다. 전역 단축키는 PDF 뷰어를 그대로 두고 넘김을 기록한다. 트레이드오프는 [대안 검토](#대안-검토-alternatives-considered).
  - macOS 접근성(Accessibility) 권한 승인이 필요하며, 이는 사용자가 시스템 설정에서 1회 허용한다. 미허용 시 안내 후 종료한다.
  - 키 매핑: 다음/이전 페이지 키로 내부 카운터를 증감해 절대 페이지를 로그에 남기고, 점프 대비 "현재 페이지 지정" 키를 둔다. 기본 키 조합은 subagent 판단 사항.
- 오디오 캡처: 마이크 입력을 WAV로 스트리밍 저장. 장치·샘플레이트·채널 기본값은 subagent 판단 사항이나 `meta.json`에 실제값을 기록한다.
- 세션 시작 시 `meta.json` 생성(PDF 경로는 인자로 받아 참조만), 종료 시 `audio.wav`·`pages.jsonl` 확정.
- 구현할 파일:
  - `repos/lecnote/src/lecnote/record.py`
  - `repos/lecnote/src/lecnote/audio.py`
  - `repos/lecnote/src/lecnote/hotkey.py`

### 항목 4 — 사후 태깅 편집 (`tag`)

강의 후 오디오를 들으며 넘김 마커를 추가·수정한다. 라이브 로깅의 누락·오조작 교정과, 로깅 없는 녹음의 전량 수동 태깅 폴백 두 용도를 함께 처리한다.

- 기존 세션을 열어 `pages.jsonl`을 편집: 시각을 지정해 넘김 이벤트(`source: manual`)를 추가·삭제·수정한다.
- 재생 위치 기준으로 "지금 시각에 페이지 N으로 넘김"을 찍을 수 있게 하고, 라이브 이벤트와 수동 이벤트를 시간순으로 병합해 저장한다.
- raw append-only 원칙은 유지하되, 수동 교정은 명시적 편집이므로 기존 라인 수정·삭제를 허용한다(라이브 자동 로깅과 구분되는 사람 개입).
- 구현할 파일:
  - `repos/lecnote/src/lecnote/tag.py`

### 항목 5 — `process` STT 연동 + 로그 정규화

선정 STT로 오디오를 전사하고, 넘김 로그에 디바운스를 적용해 페이지 구간을 만든다. 매핑·노트 생성(항목 6)이 이 산출물을 받는다.

- STT 어댑터: 항목 1이 정한 인터페이스로 선정 모델을 로드해 세그먼트별 `start`/`end`/`text`를 얻어 `transcript.jsonl`로 쓴다.
- 로그 정규화: `pages.jsonl`을 시간순 정렬 후 디바운스(공통 결정 규칙)를 적용해 겹치지 않는 페이지 구간 목록 `(t_i, page_i)`를 만든다. `debounce_seconds`는 `meta.json` 값 또는 CLI 플래그로 재지정 가능.
- 구현할 파일:
  - `repos/lecnote/src/lecnote/process.py` (STT 단계·오케스트레이션)
  - `repos/lecnote/src/lecnote/stt.py`
  - `repos/lecnote/src/lecnote/normalize.py` (디바운스·구간화)

### 항목 6 — `process` 매핑 + 페이지별 노트 생성

정규화된 페이지 구간과 STT 세그먼트를 연결해 페이지별 `.md`를 만든다.

- 매핑 규칙: 각 세그먼트는 세그먼트 중점 시각 `(start+end)/2`이 속한 페이지 구간에 배정한다.
  - 근거: 중점 기준이면 경계를 걸친 세그먼트도 단일 페이지로 결정돼 규칙이 단순하다.
  - 경계·겹침: 첫 넘김 이전 발화는 첫 페이지(또는 `meta.json`의 시작 페이지)에 넣고, 마지막 넘김 이후는 마지막 페이지에 넣는다. 경계를 크게 걸친 세그먼트를 두 페이지로 쪼갤지는 subagent 판단 사항(기본은 쪼개지 않음).
- 노트 출력 형식:
  - 파일명: `notes/page-NNN.md` (페이지 번호 3자리 zero-pad).
  - 파일 내부: 상단에 페이지 번호와 그 페이지의 시간 구간(시작–끝), 이어서 세그먼트를 시간순으로 `[HH:MM:SS] 텍스트` 형태로 나열. 세그먼트별 타임스탬프를 표기해 특정 대목을 오디오에서 되짚을 수 있게 한다.
  - 같은 페이지로 여러 번 돌아온 경우(비연속 구간)는 한 파일에 구간별로 이어 붙인다.
- 구현할 파일:
  - `repos/lecnote/src/lecnote/mapping.py`
  - `repos/lecnote/src/lecnote/notes.py`

## 대안 검토 (Alternatives Considered)

주요 갈림길에서 택하지 않은 안과 이유를 남긴다.

- 넘김 입력: 터미널 포그라운드 키 입력 (택하지 않음)
  - 장점: 접근성 권한 불필요, 추가 의존성 없음, 구현 단순.
  - 배제 이유: 강의 중 PDF 뷰어가 포커스를 가져 CLI로 포커스를 옮겨야 넘김을 찍을 수 있어, 넘김 조작 자체와 충돌한다. 전역 단축키는 이 충돌이 없다.
- 디바운스 적용 시점: 기록 시점에 로그를 되쓰기 (택하지 않음)
  - 장점: `process`가 받는 로그가 이미 깨끗함.
  - 배제 이유: raw를 변형하면 임계값을 바꿀 때 재정규화로 되돌릴 수 없어, 조정 가능해야 한다는 요구와 어긋난다.
- 매핑 기준: 세그먼트 시작 시각으로 페이지 결정 (택하지 않음)
  - 배제 이유: 넘김 직전에 시작해 넘김 후까지 이어지는 세그먼트가 이전 페이지로 쏠린다. 중점 기준이 경계 세그먼트를 더 고르게 배정한다.
- 페이지 전환 감지: PDF 슬라이드 이미지·OCR로 자동 감지 (택하지 않음)
  - 배제 이유: 화면 캡처·이미지 처리 비용이 크고 오탐 위험이 있어 MVP 범위를 넘는다. 사람이 기록한 넘김 로그가 더 단순하고 확실하다(비목표에 명시).

## 미결 질문 (Open Questions)

사용자가 직접 정해야 하는 것만 남긴다.

- 코드 레포 이름을 `lecnote`로 확정할지, 다른 이름을 쓸지 (사용자 취향 — 항목 2 스캐폴딩 전에 필요).
