# 🐧 linux-dev-notes

WSL / Linux / CUDA 개발 환경에서 자주 쓰는 명령어와 설정을 정리한 개인 참고 사전입니다.

## 📁 카테고리

| 폴더 | 내용 |
|------|------|
| [bash/](bash/) | Bash 기본 명령어, 환경변수, 셸 설정 |
| [conda/](conda/) | Conda 환경 생성, 관리, 환경별 설정 |
| [cuda/](cuda/) | CUDA 버전 관리, 경로 설정, 트러블슈팅 |
| [git/](git/) | Git 기본 명령어, GitHub 연동 |
| [wsl/](wsl/) | WSL2 설치, VS Code 연동, GPU 설정 |
| [pip/](pip/) | Pip 패키지 관리, 빌드 격리 |
| [uv/](uv/) | uv 설치, SSL 인증서 설정, pip 대체 사용법 |
| [vscode/](vscode/) | VS Code 설정, 터미널 연동, 단축키 |
| [ffmpeg/](ffmpeg/) | ffmpeg/ffprobe 명령어, SBS 3D 합성, 딥러닝 전처리 |
| [huggingface/](huggingface/) | HuggingFace Hub 캐시 관리, 오프라인 모드, 슈퍼컴 이전 |
| [projects/](projects/) | 프로젝트별 설치/실행 기록 (SHARP 등) |

## 🔍 빠른 검색

GitHub에서 `Ctrl+K` → 파일명이나 키워드로 검색하면 빠르게 찾을 수 있습니다.

## 📝 업데이트 기록

- 2026-04-19: huggingface/ 폴더 추가 (캐시 관리, 오프라인 모드, HF_HUB_OFFLINE/TRANSFORMERS_OFFLINE, 슈퍼컴 rsync 이전 시나리오, hf_transfer 가속)
- 2026-04-18: git/ 에 remote 관리 문서 추가 (다중 원격 저장소, fork 동기화, fast-forward/rebase 패턴)
- 2026-04-18: vscode/ 폴더 추가 (visual_code 레포에서 통합), ffmpeg/ 폴더 추가 (ffmpeg 레포에서 통합), 디스크/캐시 관리 문서 추가
- 2026-04-16: uv 설치 방법 및 SSL 인증서(native-tls) 트러블슈팅 추가
- 2026-03-27: 초기 구조 생성, SHARP 프로젝트 설치 기록 추가
