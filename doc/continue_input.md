# 새 세션 인계 파일 (continue_input)

이 파일은 새 엔진(새 컨텍스트)에서 작업을 이어받을 때 가장 먼저 읽힌다.

## 1. 먼저 읽을 파일 (순서대로)
1. `report.md` — 전 단계 실측 요약 (진척·수치 복원용)
2. `poc/README.md` — 작업 규칙 (단계 폴더 4종 세트, 공용 venv, 단계 건너뛰기 금지)
3. `research_spec.md` — 연구 문서 v1.3 (목표·가설·성공기준). 없으면 사용자한테 요구할 것

## 2. 작업 규칙
- 가상환경은 `poc/.venv`만 사용. 시스템 CUDA·드라이버·Ollama 서비스는 손대지 않는다
- 의존성 추가는 venv pip로만 (sudo 금지)
- 단계 폴더당 4종 세트 유지: `<실험>.py` / `test_<실험>.py` / `results.json` / `REPORT.md`
- 실측하지 않은 수치는 보고하지 않는다. 실패·미실행은 명시한다
- 수정 시도는 오류당 최대 3회, 그 이상은 보고하고 멈춘다
- 전역 설정(`~/.config` 등) 변경은 사용자 명시 요청이 있을 때만, 백업 후 수행

## 3. 환경 (2026-10-13 확인)
- CPU i7-13700 / RAM 62GB / GPU RTX 4060 8GB / NVMe 여유 689GB
- torch 2.14.1+cu130 (venv), CUDA 사용 가능
- Ollama: qwen3:8b(메인) · qwen3.5:4b(보조) · embeddinggemma(임베딩)
- 프로젝트 루트: `/home/anoncaco/projects/opencode-tictactoe-lab`

## 4. 새 엔진 첫 지시문 (복사해서 사용)
```
report.md와 poc/README.md, research_spec.md를 먼저 읽어.
이전 작업은 위 파일 기준으로 이어받아. venv는 poc/.venv를 쓰고,
시스템 CUDA·드라이버는 손대지 마. 실측하지 않은 수치는 보고하지 마.
```
