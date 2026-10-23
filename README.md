# Semantic Cognition

> **인지 코어 `F`가 컨텍스트를 저장하고, 기억을 검색하고, 추론·검증·상태 갱신을 수행하는 시스템.**
> `E`(임베더)와 `D`(생성기)는 입출력 모듈이다. **지속적 지식 상태의 권위 있는 소유자는 `F`뿐이다.**

`summary_arch.md` v1.4.0-merged를 **구현의 유일한 기준**으로 삼는다. 이 저장소의 코드는 그 문서를 구현한 것이다.

---

## 빠른 시작

### 문서 읽기 순서

| 순서 | 문서 | 읽는 이유 |
|---|---|---|
| 1 | [`summary_arch.md`](summary_arch.md) | **규범.** 전체 설계. 처음엔 §0(한 페이지 요약) → §5(아키텍처) → §7(모듈 상세)만 읽어도 전체 그림이 잡힌다 |
| 2 | [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | 아키텍처 요약 (10분 요약본) |
| 3 | [`docs/IMPLEMENTATION_PLAN.md`](docs/IMPLEMENTATION_PLAN.md) | **지금 뭘 만드는가.** 1차 목표의 단계별 사양·선행 조건·게이트 |
| 4 | [`docs/DATA_MODEL.md`](docs/DATA_MODEL.md) | 데이터 모델 + SQLite 스키마 + 필드별 수정 권한 |
| 5 | [`docs/EXPERIMENT_PROTOCOL.md`](docs/EXPERIMENT_PROTOCOL.md) | 데이터셋 명세·지표·분할 규칙 |
| 6 | [`docs/decisions/`](docs/decisions/) | ADR 10건. **왜 이렇게 정했는지** |

### 현재 실행할 수 있는 것

```bash
# 문서·config 검증 (의존성 없이 가능)
python3 -c "import yaml,pathlib; [yaml.safe_load(p.read_text()) for p in pathlib.Path('configs').glob('*.yaml')]; print('configs OK')"

# pytest (venv 생성 후)
pytest -q
```

Phase 0 착수 전이라 **아직 실행할 코드와 테스트가 없다**(`src/`는 패키지 뼈대만 존재).

---

## 현재 상태 (2026-10-20)

**Phase 0 착수 전.** 1차 목표 = *Phase 0~5 완성 + Phase 6 대표 예시(§10.2) 선행 검증*.

| 항목 | 상태 | 비고 |
|---|---|---|
| 규범 문서 | ✅ v1.4.0-merged | C11–C14 포함. 문서 간 모순 정합 완료 (ADR-0011) |
| 파생 문서 | ✅ 5편 | ARCHITECTURE / IMPLEMENTATION_PLAN / DATA_MODEL / EXPERIMENT_PROTOCOL / EXPERIMENTS |
| 결정 기록 (ADR) | ✅ **11건** | `docs/decisions/` |
| 선언적 구조 (`configs/`) | ✅ 7개 | **미검증** — `status`는 `proposed`/`unverified`/`confirmed`가 섞여 있다 (embedding_profiles는 `unverified`, runtime의 정규화 섹션만 `confirmed`) |
| 구현 코드 (`src/`) | ⬜ **0건** | 패키지 15개만 생성 (`__init__.py`) |
| 테스트 (`tests/`) | ⬜ **0건** | 디렉터리만 존재 |
| 라벨링 데이터셋 | ⬜ **0 / 172** | 파싱 72 + 검색 100. §10.2 E2E 픽스처는 별도(시드 3 + 단언) |
| 실험 산출물 (`experiments/`) | ⬜ 0건 | `stage0_probe/` 디렉터리만 |
| venv | ⬜ **미생성** | §19 #12 — 저장소 루트 `.venv`로 확정. **생성 승인 대기** |
| 실행 로그 | ⬜ 미실행 | `docs/EXPERIMENTS.md` |

**다음 두 가지가 승인을 기다리고 있다.**

1. **venv 생성** — §18.1 "생성 경로는 사용자 승인을 받는다". `.venv` 생성 + torch/transformers/sentence-transformers/pytest 설치.
2. **라벨링 검수자 지정** — 라벨 172건. ADR-0007에 따라 *초안 작성 → 사람 검수* 절차. 검수자는 `data/data_manifest.yaml`에 기록된다.

---

## 폴더·파일 설명

**목표 저장 구조는 [`summary_arch.md` §17](summary_arch.md)에 있다.** 여기서는 **현재 실제로 있는 것만** 적는다.

| 경로 | 상태 | 설명 |
|---|---|---|
| `summary_arch.md` | ✅ | **규범.** 유일한 기준. 충돌 시 이긴다 |
| `doc/` | ✅ | 원본 참고 문서 5종 + 노트북 4종. **읽기 전용. 수정·이동·삭제 금지** |
| `docs/` | ✅ | 이 프로젝트가 생성하는 산출물 문서. `doc/`와 **역할이 다르다** (이름이 비슷) |
| `docs/decisions/` | ✅ | ADR. `summary_arch.md`의 결정 근거 |
| `docs/EXPERIMENTS.md` | ✅ | 실행 로그. **실측만 기록** (§18.3) |
| `configs/` | ✅ | 선언적 구조 7개. **코드 하드코딩 금지** (§6.3·§7.3·§7.5) |
| `src/semantic_cognition/` | ⬜ | 15개 패키지. contracts / adapters / encoder / context / parsing / memory / cognition / reasoning / verification / meta / learning / output / evaluation / runtime |
| `tests/` | ⬜ | `unit/` `integration/` `regression/` `fixtures/` — 회귀는 §7.5.1 개발자 경로 게이트가 실행 |
| `tests/fixtures/parsing/` | ⬜ | 파싱 라벨 세트. **파싱 라벨 fixture의 유일한 위치** (ADR-0005). 검색·E2E는 `tests/fixtures/retrieval/`·`e2e/`에 따로 둔다 |
| `experiments/` | ⬜ | `stageN_*/` 4종 세트: `<주제>.py` / `test_<주제>.py` / `results.json` / `REPORT.md` |
| `data/` | ⬜ | `raw/` `processed/` `vectors/` `sqlite/` + `data_manifest.yaml` (데이터셋 버전·검수 기록) |
| `pyproject.toml` | ✅ | 패키지 정의 + pytest 설정 |
| `requirements.txt` | ✅ | **실제 설치된 의존성의 기록** (§18.1) |

---

## 가장 중요한 규칙 5개

1. **`E`와 `D`는 입출력 모듈이다. 지속적 지식 상태의 권위 있는 소유자는 `F`뿐이다.**
2. **`content_hash`(식별) · `z_raw`(의미) · `z_mrl`(축소) · `z_abs`(학습)는 네 개가 서로 다른 것이다.** 같은 필드·같은 인덱스에 섞지 않는다.
3. **벡터 유사성은 검색 신호이며 진위 판정이 아니다.** 판단은 관계·출처·검증(`V`)이 한다.
4. **`D`가 만든 문장은 자동으로 사실이 되지 않는다.** `llm_generated` 출처의 **가설 후보**로만 저장한다.
5. **구조 변경과 파라미터 학습을 한 번에 동시에 하지 않는다.** 각각 별도 버전·별도 실험으로 평가한 뒤 조합한다.

전체 설계 원칙은 [`summary_arch.md` §4](summary_arch.md), 요약본은 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

---

## 작업 규칙

| 규칙 | 출처 |
|---|---|
| 가상환경은 **저장소 루트 `.venv`만.** 시스템 CUDA·드라이버·Ollama 서비스를 손대지 않는다 | §18.1 |
| 의존성 추가는 **venv pip로만.** sudo 금지 | §18.1 |
| **실측하지 않은 수치를 보고하지 않는다.** 실패·미실행은 명시한다 | §18.3 |
| **오류당 최대 3회 수정 시도.** 그 이상이면 로그를 남기고 멈춘다 | §18.4 |
| 타임스탬프는 **UTC ISO-8601(`Z` 접미사)**. 로컬 시간 금지 | §18.1.1 |
| 모든 난수 지점은 **명시적 시드**를 받는다. 기본 시드 하드코딩 금지 | §18.1.1 |
| `doc/` 원본 참고 문서는 **읽기 전용.** 이동·삭제 금지 | §17 |
| **기존 `results.json`을 덮어쓰지 않는다** | §18.2 |
| **Phase를 건너뛰지 않는다.** 코드·테스트·`results.json`·`REPORT.md` 4종 세트가 모두 끝나야 다음 Phase | §16 |
| Phase 판정은 **통과 / 실패 / 보류** 3상태. 보류를 통과로 취급하지 않는다 | §16 (ADR-0009) |

---

## 다음 단계

```
Phase 0 환경·임베더 실체 확인
  └─ venv 생성 (승인 대기) → ST/Ollama 기능 테스트 → 백엔드 간 모델 동일성 판정 (ADR-0002)
       └─ Phase 1 E 모듈 → Phase 2 스키마·해시 → Phase 3 Parse → Phase 4 기억 → Phase 5 STATE → Phase 6-부분 §10.2 재현
```

상세는 [`docs/IMPLEMENTATION_PLAN.md`](docs/IMPLEMENTATION_PLAN.md).

**병목:** Phase 3 착수에 파싱 라벨 52건, Phase 4 착수에 검색 라벨 100쌍이 필요하다. 두 병목은 병렬 진행 가능하다.

---

## 문서 권위 지도

| 문서 | 지위 |
|---|---|
| `summary_arch.md` | **규범. 유일한 기준.** 충돌 시 이긴다 |
| `docs/IMPLEMENTATION_PLAN.md` · `docs/DATA_MODEL.md` · `docs/EXPERIMENT_PROTOCOL.md` | 규범에서 파생된 실행 문서. **규범을 변경하지 못한다** |
| `docs/EXPERIMENTS.md` | 실행 로그. **규범이 아니다** |
| `docs/ARCHITECTURE.md` · **이 README** | **요약·진입점. 규범이 아니다** |
| `docs/decisions/` | 결정 근거. 규범 변경은 `summary_arch.md`를 버전 올려 수정하고 ADR을 남긴다 |
| `doc/` | 원본 참고 문서. **읽기 전용** |

충돌 시의 최종 판정 순서는 [`summary_arch.md` §1.1](summary_arch.md).
