# survey.md — 현황 조사

이 기능은 아직 코드가 없는 그린필드다. 나중 단계가 다시 확인하게 될 환경·전제 사실만 기록한다.

## 코드 현황
- 이 도구의 소스 코드는 아직 작성되지 않았다.
- 워크스페이스는 이슈/문서만 git으로 추적하고, 실제 코드는 gitignore된 `repos/` 아래에 클론한다.
- 현재 `repos/`는 비어 있으며, 이 도구의 코드 레포는 아직 생성되지 않았다.

## 대상 실행 환경
- 하드웨어는 Apple M4 Pro, RAM 48GB, arm64다.
- GPU 가속은 Metal 4다.
- OS는 macOS(darwin 25.5.0)다.
- 이 사양은 large급 로컬 STT 모델도 실용 속도로 실행 가능한 수준이다.

## 기술 스택 방향
- 구현 형태는 Python CLI로 정해져 있다.
- 로컬 실행 STT 후보 비교는 design 단계의 벤치마크 항목이며, 현재까지 후보 조사는 수행되지 않았다.

## 로컬 STT 후보 프레임워크 (아키텍처 사실)
아래는 후보 프레임워크의 실행 방식·바인딩 같은 안정적 아키텍처 사실만 기록한 것이다. 각 후보의 현재 버전·정확도·속도 수치는 여기서 확인할 수 없으므로 implement 벤치마크(작업 1)에서 실측한다.

- 가속 경로 축:
  - whisper.cpp는 GGML 기반 C/C++ 구현으로 macOS에서 Metal 백엔드로 GPU 가속하며, Python 바인딩(pywhispercpp 등)이 있다.
  - MLX 기반 whisper(mlx-whisper 등)는 Apple MLX 배열 프레임워크로 Apple Silicon GPU(Metal)를 네이티브 사용하는 pip 설치형 Python 패키지다.
  - WhisperKit는 CoreML 기반으로 Apple Neural Engine(ANE)·GPU를 쓰며, Swift 패키지라 Python CLI에서 쓰려면 서브프로세스 호출이나 브리징이 필요하다.
  - faster-whisper는 CTranslate2 백엔드로 CPU·CUDA를 대상으로 하며, Apple Silicon에는 Metal GPU 가속 경로가 없어 arm64 CPU로 실행된다.
