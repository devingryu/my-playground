# qa.md — DVRY-1 테스트 감사 (로컬 강의 노트)

- 작성일: 2026-09-08
- 대상: `repos/lecnote` (커밋 `07d7031` 시점, main)
- 근거: [requirements.md](requirements.md) 테스트 계획 + [design.md](design.md) 항목 1–6
- 감사자 주석: production/test 코드는 수정하지 않았다. 유닛 스위트 실행과 STT 파이프라인 1회 재현만 수행했다.

## 요약

- 유닛 스위트: **49개 전부 통과** (0 실패). 명령: `PYTHONPATH=src .venv/bin/python -m unittest discover -s tests -v` → `Ran 49 tests in 0.009s / OK`.
  - 영역별: `test_mapping` 11, `test_normalize` 11, `test_notes` 9, `test_record` 7, `test_tag` 11.
- STT 종단(E2E): 이 감사에서 직접 재현·관찰함(합성 한/영 혼용 8.17초 클립, `.venv/bin/python -m lecnote.cli process`). 전사·매핑·페이지별 `.md` 생성 모두 정상.
- 결과 집계(체크리스트 13행): **통과 7 / 실패 0 / 미검증 6**.

## 자동 검증 (유닛·통합 테스트가 커버하는 항목)

| # | 점검 항목 | 증명 수단 | 결과 | 증거 |
|---|---|---|---|---|
| A1 | 페이지 매핑: 세그먼트 중점 `(start+end)/2`이 속한 페이지 구간에 배정, 경계·구간 밖·재방문 처리 (설계 항목 6) | `tests/test_mapping.py` (11개) | 통과 | `test_midpoint_decides_page`, `test_segment_straddling_boundary_goes_to_midpoint_page`, `test_midpoint_exactly_on_boundary_goes_to_next_page`(반개구간 `[t_start,next)`), `test_pre_first_turn_goes_to_first_page`/`test_pre_first_turn_respects_start_page`, `test_post_last_turn_goes_to_last_page`, `test_revisited_page_keeps_time_order_noncontiguous`. E2E에서도 중점 2.39s→page1, 6.64s→page2로 배정 확인. |
| A2 | 디바운스 원복(3초 내/후): 임계값 이내 왕복 취소, 초과 복귀는 별개 넘김 유지 (요구 테스트계획 "디바운스 원복", 설계 공통결정·항목 5) | `tests/test_normalize.py` (11개) | 통과 | `test_roundtrip_within_window_cancelled`(1→2→1, 3s내 취소), `test_roundtrip_after_window_kept`(delta 4 유지), `test_roundtrip_boundary_exactly_at_window_cancelled`(delta==3 포함 취소), `test_boundary_just_over_window_kept`(3.001 유지), `test_nested_roundtrips_cancelled`, `test_consecutive_same_page_collapsed`. |
| A3 | 산출물: 페이지별 `notes/page-NNN.md` 생성, zero-pad 파일명, 헤더에 시간 구간, `[HH:MM:SS] text` 나열 (요구 "산출물 검증", 설계 항목 6) | `tests/test_notes.py` (9개) | 통과 | `test_filenames_zero_padded_and_content_written`(`page-007.md`,`page-012.md`), `test_written_ordered_by_page`, `test_segment_lines_use_hms_timestamp`, `test_noncontiguous_ranges_concatenated_in_header`(재방문 구간 헤더 연결), `test_fmt_ts_hms`/`test_fmt_ts_floors_fraction`. E2E에서 실제 `page-001.md`/`page-002.md` 생성 확인. |
| A4 | 사후 태깅 병합 로직: manual 이벤트 추가/삭제/수정, live+manual 시간순 병합, 입력 불변 (요구 "사후 태깅 반영"의 병합 부분, 설계 항목 4) | `tests/test_tag.py` (11개) | 통과 | `test_add_defaults_to_manual_and_sorts_in`, `test_delete_by_sorted_index`, `test_modify_time_reorders_and_marks_manual`, `test_merge`계열(`test_sorts_live_and_manual_by_time`, `test_stable_at_equal_times`, `test_does_not_mutate_input`), `test_full_edit_flow`(add→add→delete→merge). 종단 반영은 M4 참조. |
| A5 | record 넘김 카운터 로직: next/prev(1에서 바닥)/set-page의 절대 페이지·상대 시각·source=live (설계 항목 3의 순수 로직) | `tests/test_record.py` (7개) | 통과 | `test_next_increments_absolute_page`, `test_prev_decrements_and_floors_at_one`, `test_set_page_jumps_absolute`, `test_mixed_sequence`, `test_time_is_relative_to_construction`, `test_events_are_live_source`. 주의: clock·sink를 주입한 `LivePageLog` 순수 로직만 검증 — 실제 마이크·전역 단축키는 미포함(M6 참조). |
| A6 | STT 종단 파이프라인: `process`가 normalize→transcribe(mlx-whisper large-v3)→map→notes를 실제로 수행 (설계 항목 5·6) | CLI 재현 (이 감사에서 1회 실행) | 통과 | 합성 8.17s WAV(16kHz mono)로 `process` 실행: `normalized 2 page events → 2 intervals (debounce=3s)`, `wrote transcript.jsonl (2 segments)`, `wrote 2 page note(s)`. 총 5.2s(모델 캐시 상태). `transcript.jsonl` 2세그먼트, `page-001.md`/`page-002.md` 정상. |
| A7 | 언어 혼용 (a) 한/영 혼용 발화가 유실 없이 전사 (요구 "언어 혼용 처리") | A6 E2E 관찰 | 통과 | 입력 "오늘은 machine learning의 gradient descent 알고리즘…, loss function을 최소화합니다"가 두 세그먼트로 빠짐없이 전사됨(발화 누락 0). **단서**: 영어 단어가 라틴 표기가 아니라 한글 음차("머신 러닝", "그레이디언트 디센트", "로스 펑크션")로 전사됨 — 콘텐츠 유실은 없으나 코드 스위칭의 원 표기는 보존되지 않음. 세그먼트별 정확도/표기는 M2 참조. |

## 수동 검증 (자동 테스트가 닿지 않는 항목)

| # | 점검 항목 | 절차(대화 없이 재현 가능) | 결과 | 증거/사유 |
|---|---|---|---|---|
| M1 | STT 선정 벤치마크 (요구 "STT 선정 벤치마크") | 원래 계획은 한/영/혼용 라벨 음성으로 후보별 WER·속도 측정. **실행하지 않음** — 설계 항목 1에서 사용자가 로컬 실측 대신 공개 벤치마크 기반 선정으로 결정(2026-09-08). 재현하려면 라벨 샘플셋을 만들고 후보(large-v3 / turbo / SenseVoice 등)를 돌려 WER·RTF 비교. | 미검증 | 로컬 벤치마크 미실시(사용자 결정). 선정 결과·근거·폴백 조건은 `repos/lecnote/docs/stt-selection.md`에 문서화됨(large-v3 주력, turbo 폴백). 통과/실패로 판정할 대상 아님. |
| M2 | 언어 혼용 (b) 세그먼트별 `lang` 라벨링 정확도 (설계 transcript 스키마 `lang`) | 한/영 비중이 다른 여러 클립을 `process`로 돌려 각 세그먼트 `lang`이 실제 발화 언어를 따르는지 확인. | 미검증 | A6 E2E에서 혼용 클립의 두 세그먼트가 모두 `lang:"ko"`로 라벨됨. Whisper는 클립당 지배 언어 하나만 감지(설계·`stt.py` docstring에 명시된 알려진 한계) — 세그먼트 단위 정확 라벨링은 보장되지 않음. 자동 테스트 없음. 개선 시 WhisperX(단어 단위) 검토 여지(설계 항목 1 "열어둔 개선"). |
| M3 | 실제 강의 STT 정확도 (요구 제약: 원거리·잡음 환경) | 실제 강의실에서 `record`로 녹음 후 `process` 전사 → 수기 정답과 WER 비교. | 미검증 | 합성 `say` 클립(근거리·무잡음)만 관찰. 원거리·잡음 정확도는 MVP 범위 밖(요구 명시)이며 이 감사에서 측정 안 함. |
| M4 | 사후 태깅 반영 종단 (요구 "사후 태깅 반영": 편집이 `process` 매핑 결과에 반영) | ① 세션에 `tag`(또는 `pages.jsonl` 직접 편집)로 manual 넘김 추가/수정 → ② `process --skip-transcribe` 재실행 → ③ 변경한 시각 기준으로 페이지별 `.md` 세그먼트 배정이 바뀌는지 diff 확인. | 미검증 | 병합 로직은 A4에서 유닛 커버되나, "편집→재매핑→노트 반영" 종단 경로는 자동 테스트 없음. `--skip-transcribe`로 STT 없이 재현 가능(빠름). 이 감사에서는 미실행. |
| M5 | `tag` 인터랙티브 afplay 재생 흐름 (설계 항목 4: 재생 위치 기준 마커 찍기) | `lecnote tag <session-dir>` 실행 → afplay 재생 중 재생 위치 기준으로 "지금 시각에 page N" 마커를 찍고, `help`/add/delete/modify 명령이 동작하는지 확인. | 미검증 | 오디오 재생·stdin 상호작용은 자동화 대상 아님. 유닛 테스트는 순수 편집 함수만 커버(A4). 사람이 터미널에서 1회 실행 필요. |
| M6 | `record` 마이크 캡처 + 전역 단축키 (요구 "실시간 넘김 로깅" MVP 필수, 설계 항목 3) | ① macOS 시스템 설정 > 개인정보 보호 및 보안에서 터미널 앱에 **마이크**(TCC 프롬프트 승인) 및 **손쉬운 사용/입력 모니터링** 권한 부여. ② `lecnote record <pdf-path>` 실행. ③ 기본 단축키로 넘김 기록: 다음 `<ctrl>+<alt>+n`, 이전 `<ctrl>+<alt>+p`, 페이지 지정 `<ctrl>+<alt>+g`, 종료 `<ctrl>+<alt>+q`(또는 Ctrl-C). ④ 종료 후 `audio.wav`·`pages.jsonl`·`meta.json`이 생성되고 넘김 이벤트가 append-only로 기록됐는지 확인. | 미검증 | 실 마이크·전역 단축키·TCC 권한은 헤드리스 자동화 불가. 순수 넘김 카운터 로직만 A5에서 커버. 권한 미허용 시 안내 후 종료가 설계된 동작(설계 항목 3). |

## 미검증 6행을 닫으려면

- **M1 STT 벤치마크**: 판정 대상 아님(사용자가 공개 벤치마크 선정으로 대체). 닫으려면 라벨 음성 샘플셋 + 후보 WER/RTF 측정이 필요하나 범위 밖.
- **M2 세그먼트 lang 라벨링**: 알려진 모델 한계. 세그먼트 단위 언어 라벨이 요구되면 WhisperX 등 단어 단위 경로 도입 후 검증 필요.
- **M3 실제 강의 정확도**: 실 강의실 녹음 1건 + 정답 대조. MVP 범위 밖.
- **M4 태깅 반영 종단**: `tag` 편집 후 `process --skip-transcribe` 재실행·노트 diff 1회면 닫힘(빠르게 재현 가능).
- **M5 afplay tag 흐름**: 사람이 `tag` 세션 1회 대화 실행.
- **M6 record 캡처**: 마이크·손쉬운 사용 권한 부여 후 `record` 1회 실행·산출물 확인.
