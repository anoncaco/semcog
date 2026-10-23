# Implementation Plan — 1차 목표 (Phase 0~5 + Phase 6 대표 예시)

- **문서 버전:** 1.0
- **작성일:** 2026-10-20
- **기준:** `summary_arch.md` v1.4.0-merged (규범), `docs/decisions/ADR-0001~0011.md`
- **지위:** `summary_arch.md`로부터 파생된 **실행 문서**. 충돌 시 `summary_arch.md`가 이긴다 (§1.1)

---

## 0. 1차 목표 범위

### 0.1 목표

**Phase 0~5를 완성하고, Phase 6의 대표 예시(§10.2)를 end-to-end로 재현한다.**

이 목표가 갖는 의미:

1. **인지 능력을 구현하는 것이 아니라, 인지 능력이 측정될 수 있는 기반을 세운다.** (ADR-0009)
2. §10.2 예시가 재현된다는 것은 "반증 가능성 판정 → 반증 근거 연결 → 가설 상태 unresolved 유지 → 사실 승격 차단"이 **전 경로에서** 동작한다는 뜻이다. 이는 §10.2에 명시된 기대 결과 단언으로 자동 판정한다.
3. **1차 목표는 평가 하네스를 포함하지 않는다.** "인지 능력이 측정되었다"는 주장은 하지 않는다. 측정 도구는 Phase 10이며, 1차 목표의 산출물은 **측정 가능한 상태**다.

### 0.2 범위

| Phase | 이름 | 1차 목표 포함 | 비고 |
|---|---|---|---|
| 0 | 환경·임베더 실체 확인 | ✅ | |
| 1 | E 모듈 (임베더) | ✅ | 텍스트 어댑터 포함 |
| 2 | 컨텍스트 스키마·정규화·해시 | ✅ | `docs/DATA_MODEL.md` 산출 |
| 3 | Parse·기호 관계·목표 분류 | ✅ | 라벨 72문장 |
| 4 | Registry + M | ✅ | 검색 100쌍, 이벤트 **인프라** |
| 5 | F/STATE + 이벤트 | ✅ | 상태 이벤트·스냅샷·replay 완성 |
| **6-부분** | R 기본 연산 (대표 예시분) | ✅ **부분** | §10.2에 필요한 연산 + V 최소분 |

### 0.3 범위 밖 (명시적 제외)

| 항목 | Phase | 이관 사유 |
|---|---|---|
| `deduction.is_a_transitivity`·`deduction.instance_class` **구현** | 6 | §10.2 대표 예시에 필요 없음. **단, 연산 레지스트리·선언은 Phase 6에서** |
| `induction.common_relation`·`analogy.graph_match` | 11 | 1차 범위 밖 |
| 재귀 인지 루프·종료 정책 판정 | 8 | 1차 범위 밖. `termination_status` enum만 Phase 5에서 |
| P-v1 / D / CLI 실행기 | 9 | 1차 범위 밖 |
| 평가 하네스·B0~B4 | 10 | 1차 범위 밖 |
| 멀티모달 (이미지·오디오·비디오) | 14, 15 | 1차 범위 밖 |
| META 샌드박스 | 12 | 1차 범위 밖. **개발자 경로로 대체** (ADR-0001) |
| `z_abs`·지속학습·학습형 종료 | 13, 17, 18 | 1차 범위 밖 |

### 0.4 1차 목표의 **비**목표

- §15.3의 4종 ablation 항목(그래프 재순위·RL 인지제어·LLM 내부루프 편입·학습형 projector)을 **구현하지 않는다.** Phase 9까지 코드 경로에 존재하지 않아야 하며, grep으로 검사한다.
- **인라인 학습 모듈을 넣지 않는다.** P8: 결정적이고 설명 가능하게 먼저.
- **아무것도 "작동한다"고 보고하지 않는다.** 실측 없이 통과 판정하지 않는다(§15.7, §18.3).

---

## 1. Phase 게이트 맵

### 1.1 선행 조건

```text
Phase 0 (환경·임베더 실체)
   │
   ├──▶ Phase 1 (E 모듈) ──── needs: 0
   │       │
   │       └──▶ Phase 2 (스키마·해시) ──── needs: 1
   │               │
   │               └──▶ Phase 3 (Parse) ──── needs: 2 + 라벨 52
   │                       │
   │                       └──▶ Phase 4 (기억 M) ──── needs: 3 + 검색 100
   │                               │
   │                               └──▶ Phase 5 (STATE·이벤트) ──── needs: 4
   │                                       │
   │                                       └──▶ Phase 6-부분 (대표 예시) ──── needs: 5
   │
   └──▶ (Phase 10 평가 하네스는 Phase 6-부분 이후) ──── needs: 6 전부
```

### 1.2 게이트 판정

각 Phase의 게이트는 ADR-0009의 **통과 / 실패 / 보류** 3상태로 판정한다.

- **보류는 다음 Phase를 막는다.** 기준을 임의로 낮추지 않는다.
- **통과 시 다음 Phase 착수 가능.**
- 1차 목표 전체의 완료 판정 = **Phase 6-부분 게이트 통과.**

### 1.3 병목 (일정상 최대 리스크)

| 병목 | Phase | 해소 조건 |
|---|---|---|
| **파싱 라벨 52문장** | 3 | §10.2 2문장 포함, 개발/대립/비지원 세트 완성 + 검수 |
| **검색 라벨 100쌍** | 4 | 질의·정답 컨텍스트 쌍 + 검수 |
| 임베더 모델 실측(차원·digest·노름) | 0 | ST/Ollama 양쪽 기능 테스트 |
| Ollama 동일성 판정 | 0 | digest/치수 대조, `undetermined` 가능 |

---

## 2. 개발자 직접 등록 경로 계약 (ADR-0001)

1차 목표는 Phase 0~5 + 6-부분이므로 **전 구간이 개발자 경로 구간**이다. 이 경로의 규칙을 그대로 적용한다.

### 2.1 변경 가능 대상 (선언적 구조만)

| 대상 | 파일 |
|---|---|
| 관계 유형·문형 규칙 | `configs/relations.yaml` |
| 속성 정규화 규칙 | `configs/attributes.yaml` |
| 목표 분류 규칙 | `configs/goal_rules.yaml` |
| 상태 전이 표·구조 버전 | `configs/structure_registry.yaml` |
| 추론 연산 등록·비활성화 | `src/semantic_cognition/reasoning/registry.py` |

### 2.2 게이트 4조건 (모두 필수)

1. **버전 bump** — 대상에 버전 식별자를 부여하고 올린다.
2. **회귀 green** — `tests/regression/` 전체 + 해당 모듈 단위 테스트 통과.
3. **ADR 기록** — `docs/decisions/`에 사유·영향·측정 결과 기록.
4. **버전 전파** — `STATE.structure_version`과 모든 `InferenceTrace.rule_version`에 기록.

### 2.3 1차 목표 중 발동 예상 변경

| Phase | 예상 변경 | 사유 |
|---|---|---|
| 3 | `relations.yaml` 문형 표면형 보강 | 72문장 precision/recall 미달 시. §7.3 확장 절차 |
| 3 | `attributes.yaml` 정규화 규칙 추가 | 속성 공유가 안 되는 사례 발견 시 |
| 6-부분 | `structure_registry.yaml` 전이 표 보강 | 대표 예시 진행 중 필요한 전이 발견 시 |
| 6-부분 | `reasoning/registry.py` 연산 등록 | Phase 6의 5개 기본 연산 최초 등록 |

**각 변경마다 ADR을 하나 남긴다.** 1차 목표 종료 시 ADR이 최소 4건 추가된다.

---

## 3. Phase별 세부 사양

> 공통 템플릿: 목표 / 선행 조건 / 산출물 / 데이터 계약 / 작업 항목 / 테스트 / 게이트 / 실험 산출물 / 비목표 / 결정 필요 항목

---

### Phase 0 — 환경·임베더 실체 확인

#### 목표
프로젝트가 실행 가능한 상태를 만들고, **임베더가 무엇인지 실측으로 확정**한다.

#### 선행 조건
없음. (문서 골격은 본 Phase에서 만든다)

#### 산출물

```
pyproject.toml
requirements.txt
.venv/                        (사용자 승인 후 생성 — §18.1, §19 #12)
configs/embedding_profiles.yaml
docs/EXPERIMENTS.md            (스켈레톤)
docs/ARCHITECTURE.md
experiments/stage0_probe/
  ├── probe_environment.py
  ├── test_probe_environment.py
  ├── results.json             (실행 산출물)
  └── REPORT.md
```

#### 데이터 계약
`EmbeddingResult` (§7.2) — §9 계약 그대로. Phase 0에서 **필드 존재와 실패 경로만** 확인한다.

#### 작업 항목

| # | 항목 |
|---|---|
| T0.1 | 저장소 골격 생성 (§17 트리의 Phase 0 필요분만) |
| T0.2 | `pyproject.toml` (패키지 `semantic_cognition`, pytest 설정, src layout) |
| T0.3 | `requirements.txt` 기록 |
| T0.4 | venv 생성 (사용자 승인 후) + 의존성 설치 |
| T0.5 | 시드 유틸 — 모든 난수 지점이 명시적 시드를 받도록 (§18.1.1) |
| T0.6 | `configs/embedding_profiles.yaml` 초안 (§6.3 형태, **값은 전부 Phase 0 실측 후 확정**) |
| T0.7 | **ST 백엔드 기능 테스트** — 텍스트 1건·배치·`prompt_name`·`truncate_dim`(256)·dtype 선택·norm·finite·dimension·digest |
| T0.8 | **Ollama 백엔드 기능 테스트** — `POST /api/embed` 텍스트·응답 norm/finite·**모델 식별 정보 확인** |
| T0.9 | **백엔드 간 모델 동일성 판정** (ADR-0002) — `same_model: true\|false\|undetermined` |
| T0.10 | 이미지 입력 수용 여부 기능 테스트 (ST는 허용, Ollama는 확인 후에만) |
| T0.11 | GPU/CPU 실행 프로파일 — GPU 메모리 사용량·소요 시간 기록. **GPU 실패가 임베더 사용 불가 판정은 아니다** |
| T0.12 | dtype 선택 — `bfloat16`은 백엔드가 실제 지원할 때만, 아니면 `float32`. **float16 금지** |
| T0.13 | `results.json` / `REPORT.md` / `docs/EXPERIMENTS.md` 기록 |

#### 테스트
- 단위: 환경 메타데이터 수집 함수의 필드 존재·타입
- 단위: dtype 선택 로직 (지원/미지원 분기)
- 통합: ST encode 1회 성공 + shape/finite 확인
- 통합: Ollama encode 1회 성공 + shape/finite 확인 (실패해도 Phase를 막지 않되 **기록**)
- 회귀: 없음 (첫 Phase)

#### 게이트 (통과 / 실패 / 보류)
- [ ] 사용 가능한 백엔드 ≥ 1 확정
- [ ] 실측 차원·노름·finite·digest 기록 (768 가정 없음)
- [ ] **모델 동일성 판정 결과 기록** (`same_model` 값 + 근거)
- [ ] digest 확인 불가면 **임의 문자열로 대체하지 않고** 확인 불가로 기록
- [ ] 환경 메타데이터(Python·OS·pkg·GPU/CPU·VRAM·소요 시간) 기록
- [ ] 모델 API 오류·잘못된 차원을 자동 감지하는 코드 존재

#### 실험 산출물
`experiments/stage0_probe/` 4종 세트.

#### 비목표
- 실제 임베딩 캐시·프로파일 로더 구현 (Phase 1)
- 이미지/오디오/비디오 처리 (Phase 14·15)

#### 결정 필요 항목
- venv 생성 승인 (§18.1 "생성 경로는 사용자 승인을 받는다")

---

### Phase 1 — E 모듈 (임베더)

#### 목표
**고정 임베더**를 프로파일 계약 아래에 구현하고, 프로파일 격리·캐시·오류 처리를 검증한다.

#### 선행 조건
Phase 0 통과.

#### 산출물

```
src/semantic_cognition/contracts/embedding.py
src/semantic_cognition/encoder/base.py            # Encoder Protocol
src/semantic_cognition/encoder/profiles.py        # 프로파일 로더 + 검증
src/semantic_cognition/encoder/st_embed.py
src/semantic_cognition/encoder/ollama_embed.py
src/semantic_cognition/adapters/base.py
src/semantic_cognition/adapters/text_adapter.py   # 청킹 (A)
configs/runtime.yaml                               # 청킹 파라미터 등
experiments/stage1_encoder/  (4종 세트)
```

#### 데이터 계약
- `EmbeddingResult` (§7.2): `vector`, `dimension`, `profile_id`, `backend`, `model_id`, `digest`, `modality`, `normalization`, `norm`, `is_finite`, `status`, `input_fingerprint`, `elapsed_ms`
- `status`: `ok` / `input_too_long` / `modality_unsupported` / `model_error` / `invalid_dimension` / `non_finite`
- `AdaptedPayload` (§7.1): `{ payload, modality, metadata }`

#### 작업 항목

| # | 항목 |
|---|---|
| T1.1 | `contracts/embedding.py` — `EmbeddingResult` dataclass, `EmbeddingStatus` enum |
| T1.2 | `encoder/base.py` — `Encoder` Protocol. `encode_context(input_payload, profile_id) -> EmbeddingResult` |
| T1.3 | `encoder/profiles.py` — 프로파일 로드 + **검증**(프로파일 ID ↔ model/digest/dimension/normalization/task prefix 일관성). **불일치는 경고가 아니라 오류** (§6.2 규칙 5) |
| T1.4 | `encoder/st_embed.py` — `prompt_name` / `prompt` / `default_prompt_name` 3중 지원. `config_kwargs={"vision_config": None, "audio_config": None}` (텍스트 전용). `truncate_dim` + `normalize_embeddings=True` (MRL) |
| T1.5 | `encoder/ollama_embed.py` — `POST {host}/api/embed`. **노름·finite를 코드에서 가정하지 않고 검사** |
| T1.6 | **캐시** — 키 = `(profile_id, payload_hash)`. 프로파일 변경 시 자동 무효화. `z_raw`는 **영구 보존** |
| T1.7 | 배치 처리 |
| T1.8 | 오류 처리 — **빈 벡터를 정상 결과로 반환하지 않는다**. 실패는 `status`로 |
| T1.9 | `adapters/text_adapter.py` — 청킹. **E의 토크나이저 사용** (max 512 tok / overlap 50 tok — §19 #20, 실측 조정 가능). 청크별 `char_span`, `chunk_index`, `chunk_total` 기록 |
| T1.10 | **유사도 분포 5범주 측정** — 고정 세트로 기록: (1) 동일/거의 동일 (2) 의미 유사 (3) 주제 유사·관계 상이 (4) 의미 무관 (5) 지지-반박 쌍. **임계값은 여기서 정하지 않는다** (§19 #3) |
| T1.11 | `z_mrl` 프로파일 등록 — `truncate_dim` + 재정규화. 별도 프로파일 ID (`..._mrl256`) |

#### 테스트
- 단위: CPU 반복 호출 → **동일 결과** (비트 단위)
- 단위: GPU 호출 → 허용 오차 내 일치
- 단위: 출력 차원 == 프로파일 `dimension`. **하드코딩된 768이 아님**
- 단위: NaN/Inf 없음
- 단위: 오류 경로 — 빈 벡터 반환 안 함, `status` 정확
- 단위: **캐시 격리** — 프로파일 A의 캐시가 프로파일 B에서 재사용되지 않음. 프로파일 변경 시 자동 무효화
- 단위: 프로파일 불일치 → **오류** (경고 아님)
- 단위: 청킹 경계 — 오프셋 재조립 시 원문 복원, 토큰 수 ≤ 한도
- 통합: 5개 task prefix로 인코딩 후 profile_id가 요청과 일치
- 회귀: `tests/regression/` 생성

#### 게이트
- [ ] CPU 반복 동일 결과
- [ ] 차원 정확 (프로파일 값 기준)
- [ ] NaN/Inf 없음
- [ ] 프로파일·프리픽스 기록됨
- [ ] 캐시 동작 + **프로파일 격리** 검증
- [ ] **유사도 분포 5범주 기록** (임계값 미확정 상태로)
- [ ] 오류 시 빈 벡터 반환 안 함
- [ ] 청크는 E가 벡터만 반환 (집계는 M 책임, §7.1)

#### 실험 산출물
`experiments/stage1_encoder/` 4종 세트. `results.json`에 분포 수치와 프로파일 실측값 포함.

#### 비목표
- 청크 집계 (Phase 4)
- 검색 (Phase 4)
- 이미지/오디오 (Phase 14·15)

#### 결정 필요 항목
- 청킹 파라미터 확정 여부 (§19 #20)

---

### Phase 2 — 컨텍스트 스키마·정규화·해시

#### 목표
**이후 모든 기능이 의존할 데이터 계약을 확정**한다. `docs/DATA_MODEL.md`를 산출한다.

#### 선행 조건
Phase 1 통과.

#### 산출물

```
docs/DATA_MODEL.md                     # §8 확장 + SQLite 스키마 (핵심 산출물)
src/semantic_cognition/contracts/context.py
src/semantic_cognition/context/canonicalize.py
src/semantic_cognition/context/hashing.py
src/semantic_cognition/context/schemas.py
src/semantic_cognition/context/registry.py
experiments/stage2_schema/  (4종 세트)
```

#### 데이터 계약
`docs/DATA_MODEL.md`에 **필드별 생성 모듈·수정 권한**을 명시한다.

| 스키마 | 신규/변경 (v1.3) |
|---|---|
| `ContextRecord` | 변경 없음 |
| `RelationRecord` | 변경 없음 |
| `Hypothesis` | `confidence_components`에 **독립 개수 + 원시 개수** (§8.3) |
| `Evidence` | **`lineage`** 신설 (§8.4) |
| `InferenceTrace` | **`scope`** 신설, `rule_version` 필수 (§8.5) |
| `StructureChangeProposal` | **`proposed_by`** 신설 (§8.6) |
| `RetrievalTrace` | 변경 없음 |
| `ConceptNode` | `node_type` 결정 규칙 (§8.10) |
| `FailureCase` | **`failed_stage`**, **`diagnosis_status`** 신설 (§8.11) |
| `RuleCandidate` | **신규** (§8.12, Phase 11 대비 계약만) |

#### 작업 항목

| # | 항목 |
|---|---|
| T2.1 | **`docs/DATA_MODEL.md` 작성** — §8 전체 + SQLite 테이블 DDL + 필드별 생성/수정 권한 + 버전/호환성 규칙 |
| T2.2 | `canonicalize.py` — `canon_v1`: Unicode NFC, 줄바꿈 통일, 앞뒤 공백 정리. **문장부호·대소문자·숫자 표기 제거 금지** |
| T2.3 | `hashing.py` — SHA-256(canonical_content) + `canonicalization_version` |
| T2.4 | `schemas.py` — §8 전체 dataclass |
| T2.5 | `registry.py` — `register_context` / `get_context` / `link_parent_child` / provenance 리스트 / revision |
| T2.6 | **동일 해시 자동 병합 금지** — 같은 해시라도 출처·시점이 다르면 별도 레코드 + provenance |
| T2.7 | 부모/자식 컨텍스트 — 분해 기록 보존 (§8.9) |
| T2.8 | `canon_v1` 규칙을 **`data/canon_v1` 규칙 목록으로 외부화** (코드 하드코딩 금지) |

#### 테스트
- 단위: 동일 정규화 입력 → 동일 해시 (결정성)
- 단위: **정규화가 의미를 바꾸지 않는다** — "모든 새는 날 수 있다" / "일부 새는 날 수 있다" / "어떤 새는 날 수 없다"의 해시가 **서로 다름**
- 단위: 같은 문장·다른 출처 → **별도 레코드 + 별도 provenance**
- 단위: revision 증가 — 원본이 조용히 덮어써지지 않음
- 단위: `canonicalization_version` 변경 시 해시 버전이 섞이지 않음
- **단위: 해시를 유사도/검색 스코어로 쓰는 코드가 0건** — grep + import 그래프 검사 (§6.2 규칙 3)
- 단위: `lineage` 기본값 — `source_group_id`가 `provenance.source_id`에서 파생됨
- 단위: `node_type` 결정 규칙 — §8.10 표의 각 위치 규칙 검증
- 통합: registry round-trip — 등록 → 조회 → revision → 부모/자식 연결

#### 게이트
- [ ] 동일 정규화 → 동일 해시
- [ ] 의미가 달라질 수 있는 차이가 제거되지 않음
- [ ] 같은 문장·다른 출처 → 별도 source 보존
- [ ] 수정이 revision으로 남음
- [ ] **해시를 유사도 계산에 쓰는 코드가 없음**
- [ ] **`docs/DATA_MODEL.md` 완성** — §8 + §9 계약 + SQLite 스키마 + 필드별 권한

#### 실험 산출물
`experiments/stage2_schema/` 4종 세트.

#### 비목표
- 벡터 저장·검색 (Phase 4)
- SQLite 실사용 저장소 (Phase 4) — 이 Phase는 스키마 계약까지만

#### 결정 필요 항목
없음.

---

### Phase 3 — Parse·기호 관계·목표 분류

#### 목표
원문에서 명시적 관계 후보를 추출하고, **속성 노드가 공유되도록** 만든다. §10.2의 2문장이 파싱되어야 한다.

#### 선행 조건
Phase 2 통과 + **파싱 라벨 52문장(개발 27 + 대립 15 + 비지원 10) 검수 완료**.

#### 산출물

```
configs/relations.yaml            # 술어 11종 + 9개 문형 행 + 대응 술어 + 양태 순서
configs/attributes.yaml           # 속성 정규화 규칙 (ADR-0004)
configs/goal_rules.yaml           # 5개 goal_type + 우선순위
tests/fixtures/parsing/
  ├── parsing_dev.jsonl           # 52문장 (27+15+10)
  └── parsing_holdout.jsonl       # 20문장
src/semantic_cognition/parsing/parser.py
src/semantic_cognition/parsing/relation_schema.py
src/semantic_cognition/parsing/goal_rules.py
src/semantic_cognition/parsing/nodes.py
experiments/stage3_parse/  (4종 세트)
```

#### 데이터 계약
`ParseResult`: `relation_candidates`, `node_candidates`, `parse_status` (`parsed`/`partial`/`unparsed`), `spans`

#### 작업 항목

| # | 항목 |
|---|---|
| T3.1 | **`configs/relations.yaml`** — 술어 11종, 문형 9행(§7.3 v1.3 표), 대응 술어 쌍, 양태 판정 순서, 주체 조사 허용(`는/은/도/가`) |
| T3.2 | **`configs/attributes.yaml`** — "귀가 뾰족하다" → "뾰족한 귀" 등 정규화 규칙. `literal_normalized_match` → `alias_table` → `unresolved` 순 시도, 적용 규칙 기록 |
| T3.3 | **`configs/goal_rules.yaml`** — `fact_query`/`feature_identification`/`relation_query`/`generalization`/`unknown` + 우선순위 + `goal_source` |
| T3.4 | `relation_schema.py` — `RelationRecord` 생성, `status=candidate`, `created_by=parse` |
| T3.5 | `parser.py` — `rule_parser_v1`. **정규식 + 패턴 매칭. LLM 미사용.** (형태소 분석기는 §19 #14에 따라 정규식만으로 9행 처리가 안 될 때만 Phase 3 착수 전에 추가) |
| T3.6 | `nodes.py` — ConceptNode 생성. **관계 위치 기반 `node_type` 결정** (ADR-0008). 속성 정규화 적용 후 동일성 판정 |
| T3.7 | `goal_rules.py` — 목표 분류기. **LLM 미사용** |
| T3.8 | **라벨 세트 구축** — 72문장 수작업 라벨링 + 검수. 라벨은 `subject/predicate/object/quantifier/polarity/span` 구조화 + **`subset`**(`base`/`adversarial`/`out_of_scope`/`holdout`, 문군 단위 배정). **§10.2 2문장 포함** |
| T3.9 | **부세 지표 산출** — 술어 / 주어·목적어 / 극성 / 수량사·양태 / span 일치 / `unparsed` 판정 정확도를 **분리** 측정 |
| T3.10 | `parse_status` 판정 — 파싱 실패 문장은 `unparsed`로 **남긴다.** 억지 사실을 만들지 않는다 |
| T3.11 | `goal_source` 기록 — `explicit` / `inferred` |

#### 테스트
- 단위: 9개 문형 행 각각에 대해 매칭/불매칭
- 단위: **§10.2 2문장이 파싱됨** (ADR-0004)
- 단위: 대응 술어 — 부정형이 `polarity=negative`로 (별도 술어 아님)
- 단위: 양태 판정 순서 — `all` → `some` → `possible` → `typical`
- 단위: 주체 조사 `도`/`가` 허용, 주어 불변
- 단위: `unparsed` — 비지원 세트 10문장이 모두 `unparsed`
- 단위: `span` — 원문 오프셋이 실제로 해당 부분 문자열과 일치
- 단위: **속성 노드 공유** — §10.2 2문장의 속성이 **동일한 node_id**
- 단위: `node_type` 결정 규칙 (ADR-0008 표)
- 단위: 목표 분류 — 5개 타입 + `unknown` + `goal_source`
- 통합: `parse_context(context_id, raw_text)` → `ParseResult` + 노드 후보 + relation 후보

#### 게이트
- [ ] 사람이 라벨링한 참조 집합에서 **precision/recall 기록** (세부 지표 분리)
- [ ] **추출 못 하는 문장을 억지로 사실로 만들지 않음**
- [ ] 원문 span·출처 추적 가능
- [ ] **속성 노드가 공유되어 §10.2 예시가 재현 가능** ("뾰족한 귀" 1개 노드)
- [ ] **§10.2 2문장이 파싱됨**
- [ ] **보류 세트 20문장은 규칙 개발에 사용되지 않음** (검증: 보류 세트로 규칙을 고친 흔적 없음)

#### 실험 산출물
`experiments/stage3_parse/` 4종 세트. `results.json`에 세부 지표 + 개발/보류 세트 분리 측정값.

#### 비목표
- LLM 파서 (별도 Phase)
- 형태소 분석기 (기본 제외, §19 #14)
- ConceptNode 고급 코리퍼런스 (§8.10 v1 미지원)

#### 결정 필요 항목
- 파싱 지표가 기준에 못 미치면 → §7.5.1 개발자 경로로 규칙 보강 (ADR 기록)

---

### Phase 4 — Registry + M (하이브리드 의미 기억)

#### 목표
벡터 기억과 기호 기억을 **정합성 있게** 저장·검색한다. **반대 근거가 검색에서 사라지지 않아야 한다.**

#### 선행 조건
Phase 3 통과 + **검색 라벨 100쌍 검수 완료**.

#### 산출물

```
src/semantic_cognition/contracts/memory.py
src/semantic_cognition/memory/vector_index.py     # VectorIndex Protocol
src/semantic_cognition/memory/brute_force.py
src/semantic_cognition/memory/symbolic_store.py
src/semantic_cognition/memory/aggregate.py         # 청크 집계 (§7.1)
src/semantic_cognition/memory/importance.py        # §11.1
src/semantic_cognition/memory/retrieval.py
src/semantic_cognition/memory/tiers.py             # Hot/Warm/Cold (스텁, §19 #7)
src/semantic_cognition/memory/paging.py
src/semantic_cognition/memory/events_store.py      # 이벤트 스토어 인프라 (ADR-0006)
tests/fixtures/retrieval/retrieval_pairs.jsonl     # 100쌍
experiments/stage4_memory/  (4종 세트)
```

#### 데이터 계약
`MemoryHit`: `context_id`, `score`, `profile_id`, `dimension`, `source`, `epistemic_status`, `tier`, `snippet`

#### 작업 항목

| # | 항목 |
|---|---|
| T4.1 | `vector_index.py` — `VectorIndex` Protocol. **`(profile_id, dimension)` 쌍으로 인덱스 분리** (§6.2 규칙 1) |
| T4.2 | `brute_force.py` — 정확한 코사인. 기본 구현 |
| T4.3 | `symbolic_store.py` — SQLite. `docs/DATA_MODEL.md` DDL 반영 |
| T4.4 | **`aggregate.py`** — 청크 집계. v1: **정규화 후 평균 → 재정규화**. 청크 1개면 그대로. **프로파일이 다른 청크를 섞지 않는다** |
| T4.5 | `importance.py` — §11.1 식. **근거 수는 독립 근거 기준** (ADR-0003). 가중치 합 1.0 검증 |
| T4.6 | `retrieval.py` — §7.6 검색 절차 6단계 + 중복 축약 + `hits_not_used` 기록 |
| T4.7 | **벡터 쓰기 경로 5단계** (§7.6) — 등록 → 인코딩 → **파일 저장** → **인덱스 적재** → 기호 연결. **각 단계 성공/실패 개별 기록** |
| T4.8 | **부분 실패 격리** — 단계 3 실패 → 인덱스 미적재. 파일만 있고 인덱스 미적재 → `orphaned` 기록 + 재적재 대상 |
| T4.9 | **인덱스 정합성 검사 5케이스** (§7.6 표) |
| T4.10 | `tiers.py`/`paging.py` — Hot/Warm/Cold 스텁. **승강 기준은 §19 #7 미정**이므로 v1은 정책만 검증 (§11.3 "모의 인터페이스로 정책만 검증") |
| T4.11 | **이벤트 스토어 인프라** (ADR-0006) — SQLite 테이블·인덱스·멱등 쓰기 API + **기억/검색 이벤트** |
| T4.12 | **검색 라벨 100쌍** 구축 + 검수 |

#### 테스트
- 단위: `MemoryHit` 8필드 존재
- 단위: **프로파일 불일치 → 오류** (§6.2 규칙 5)
- 단위: **새 프로파일로 전환 후 이전 벡터 참조 → 오류** (§6.2 규칙 2)
- 단위: 청크 집계 — 1개면 그대로, N개면 정규화→평균→재정규화
- 단위: 다른 프로파일의 청크 혼합 금지
- 단위: importance — 가중치 합 1.0, 독립 근거 기준, 접근 시점에만 계산(저장)
- 단위: 중복 축약 — 같은 `context_id` 1개로 접히고 `hits_not_used`에 기록
- 단위: **서로 다른 출처의 동일 해시는 축약하지 않음**
- 단위: 방향 보존 — `hits`에 지지/반박 근거가 **모두** 포함
- 통합: 쓰기 경로 5단계 — **부분 실패 시 검색에 나오지 않음**
- 통합: 인덱스 정합성 5케이스
- 통합: 이벤트 스토어 — 멱등 쓰기, 인프라 이벤트 기록
- 회귀: 검색 결과가 이전과 동일한가

#### 게이트
- [ ] 검색 결과에 `context_id`·score·profile·source·epistemic status 포함
- [ ] **벡터 버전 불일치 감지 (오류)**
- [ ] **부분 실패한 데이터가 검색에 나오지 않음**
- [ ] 관련 컨텍스트 top-K 포함율 측정 (`retrieval_pairs` 100쌍, Recall@K·MRR@K·nDCG@K)
- [ ] **반대 근거가 검색에서 사라지지 않음** (독립 테스트)
- [ ] 이벤트 스토어 **인프라** 동작 (Phase 5에서 완성)

#### 실험 산출물
`experiments/stage4_memory/` 4종 세트.

#### 비목표
- HNSW (Phase 10 이후, §19 #6)
- Hot/Warm/Cold 실제 매체 (스텁만)
- 이벤트 replay (Phase 5)

---

### Phase 5 — F/STATE + 이벤트

#### 목표
**인가적 상태의 권위 있는 소유자**를 구현한다. 모든 상태 변경이 사유·출처와 함께 기록되고, 재현 가능하다.

#### 선행 조건
Phase 4 통과.

#### 산출물

```
src/semantic_cognition/contracts/reasoning.py
src/semantic_cognition/cognition/state.py
src/semantic_cognition/cognition/transitions.py
src/semantic_cognition/cognition/state_hash.py
src/semantic_cognition/cognition/events.py
src/semantic_cognition/cognition/termination.py
configs/structure_registry.yaml                   # 전이 표 + 구조 버전
data/sqlite/state/<session_id>/<step>.json        # 스냅샷 (런타임 생성)
experiments/stage5_state/  (4종 세트)
```

#### 데이터 계약
`STATE` (§7.5 직렬화 스키마 그대로). `termination_status`에 `awaiting_observation`·`timeout` 포함(판정 동작은 Phase 8).

#### 작업 항목

| # | 항목 |
|---|---|
| T5.1 | `state.py` — STATE dataclass + JSON 직렬화. **키 정렬 직렬화** |
| T5.2 | `transitions.py` — §7.5 가설 상태 전이 표. **`configs/structure_registry.yaml`에서 로드**(코드 하드코딩 금지). 표에 없는 전이는 **거부** |
| T5.3 | `apply_transition(state, transition, reason)` — **`reason`이 비면 거부** |
| T5.4 | `state_hash.py` — §7.5 해시 규칙. `hash_version` 기록. **제외 필드 3개** 정합성 검증 |
| T5.5 | `events.py` — **상태 전이 이벤트** (Phase 4 인프라 위에). 이벤트 스키마 **9필드(최소)** — `event_type`별 추가 필드는 `payload`로 |
| T5.6 | **원자성** — 상태 변경 + 이벤트 기록을 **단일 SQLite 트랜잭션**. 실패 시 부분 반영을 완료로 취급하지 않음 |
| T5.7 | 스냅샷 저장/복원 — `data/sqlite/state/<session_id>/<step>.json` |
| T5.8 | **이벤트 replay** — 스냅샷 이후 이벤트 재적용으로 상태 재구성 |
| T5.9 | `termination.py` — `termination_status` enum + 저장/직렬화. **판정 동작은 Phase 8** |
| T5.10 | `configs/structure_registry.yaml` — 전이 표 + 연산 목록 + `structure_version` |
| T5.11 | **개발자 직접 등록 경로 구현** (ADR-0001) — `proposed_by=developer` 레코드 생성 + 게이트 4조건 검사 |
| T5.12 | `TransitionRejected` 이벤트 기록 |
| T5.13 | 철회 이벤트 — `RelationRetracted`, `EvidenceRetracted`, `NodeRetracted` |

#### 테스트
- 단위: **합법 전이 허용** — 표의 각 행
- 단위: **불법 전이 거부** — `supported → rejected` 직행, `rejected → supported` (사유 없이)
- 단위: `reason` 빈 전이 거부 — **전역 규칙**(`global_transition_rules.require_reason`)이 개별 규칙에 의존하지 않고 적용됨
- 단위: **`reason_kinds` 검증** — `rejected`에서 되돌아오는 전이는 사유가 `existing_evidence_retracted` 또는 `new_evidence_added`를 명시해야 통과. 그 외 사유는 거부
- 단위: **`rejected → supported` 허용** — 사유 내용 조건 충족 시 허용 (ADR-0011)
- 단위: 거부 시 `TransitionRejected` 이벤트 기록 + **상태 무변경**
- 단위: 상태 해시 결정성 — 동일 스냅샷 → 동일 해시
- 단위: **제외 필드 정합성** — `inference_traces`만 추가 → **동일 해시** / `hypotheses`·`evidence_map` 변경 → **상이한 해시**
- 단위: `hash_version` 변경 시 해시 미혼합
- 단위: **원자성** — 이벤트 쓰기 실패 시 상태 변경이 **완료로 취급되지 않음**
- 단위: 직렬화 — 키 정렬, `state_version`·`structure_version` 포함
- 단위: `awaiting_observation`·`timeout` enum 존재
- 단위: 개발자 경로 — 버전 bump 없이 변경 시도 → 거부 / ADR 없음 → 거부 / 회귀 실패 → 거부
- 통합: 저장 → 재시작 시뮬레이션 → 복원 → **복원 상태 == 저장 상태**
- 통합: 이벤트 replay → 스냅샷 이후 상태 재구성
- 회귀:非法 전이 시 상태 훼손 없음

#### 게이트
- [ ] **임의 텍스트가 곧바로 사실이 되지 않음**
- [ ] 모든 상태 변경에 사유·출처 남음
- [ ] 재시작 후 복원, 이벤트 replay 가능
- [ ] **불법 전이가 거부되고 `TransitionRejected`로 기록됨**
- [ ] **상태 전이 + 이벤트 기록이 원자적** (부분 반영 없음)
- [ ] 상태 해시 결정성 + 제외 필드 정합성
- [ ] 개발자 직접 등록 경로 게이트 4조건 동작

#### 실험 산출물
`experiments/stage5_state/` 4종 세트.

#### 비목표
- 인지 루프 스텝 (Phase 8)
- 세션 실행기·CLI (Phase 9)
- `needs_observation` 판정 (Phase 7)

---

### Phase 6-부분 — R 기본 연산 (대표 예시 검증분)

#### 목표
**§10.2 대표 예시를 end-to-end로 재현**한다. 반증 가능성 판정 → 반증 근거 연결 → 가설 unresolved 유지 → 사실 승격 차단.

#### 선행 조건
Phase 5 통과.

#### 산출물

```
src/semantic_cognition/contracts/verification.py
src/semantic_cognition/reasoning/registry.py       # 연산 레지스트리
src/semantic_cognition/reasoning/hypothesis.py      # generate_from_feature, compare_evidence
src/semantic_cognition/reasoning/distinctiveness.py # attribute.distinctiveness_check.v1
src/semantic_cognition/verification/evidence.py     # direction 파생 + counters 2경로
src/semantic_cognition/verification/confidence.py    # 범주형 + 독립 근거
src/semantic_cognition/verification/contradictions.py
tests/fixtures/e2e/case_10_2/                       # §10.2 픽스처
experiments/stage6_reasoning/  (4종 세트)
```

#### 데이터 계약
`InferenceResult`: `conclusion_candidates`, `premise_ids`, `trace`, `status`, `uncertainty`
`VerificationResult`: `status`, `confidence`(범주형), `confidence_method`, `components`, `contradicting_refs`, `needs_observation`

#### 작업 항목

| # | 항목 |
|---|---|
| T6.1 | `reasoning/registry.py` — 연산 레지스트리. 각 연산은 **입력·출력 형식, 전제조건, 실패 유형, 버전** 보유 (§7.7). Phase 6의 5개 연산 **최초 등록** (`proposed_by=developer`) |
| T6.2 | `hypothesis.py` — `hypothesis.generate_from_feature.v1` (관찰 속성 + 후보 주어로 가설 후보 생성), `hypothesis.compare_evidence.v1` (지지/반박 근거 연결) |
| T6.3 | `distinctiveness.py` — `attribute.distinctiveness_check.v1`. **기호 조회** 사용(벡터 검색 아님). **`scope` 기록**. `distinctive`는 `scope` 없으면 불가 (ADR-0003) |
| T6.4 | `verification/evidence.py` — `Evidence.direction` **파생 규칙**. **counters 2경로만** (술어 `contradicts` / distinctiveness 판정). 그 외 경로 금지 |
| T6.5 | `verification/confidence.py` — **범주형만** (`low`/`medium`/`high`/`unresolved`). **숫자 확률 금지.** 독립 근거 카운트 (§8.3 기준과 동일) |
| T6.6 | `verification/contradictions.py` — 모순/범위 불일치 **최소** 판정 |
| T6.7 | **§10.2 픽스처** — 시드 컨텍스트 집합 + 기대 결과 단언 (§10.2 표) |
| T6.8 | **검증 상태 판정 순서** (§7.8 v1.3 표) — `configs/structure_registry.yaml`의 `verification_precedence`에서 로드 |

#### §10.2 픽스처 상세

**시드 컨텍스트 집합** (최소 3건, §10.2 실행 전제 3):

| # | 문장 | 역할 |
|---|---|---|
| S1 | "고양이는 뾰족한 귀를 가질 수 있다" | 후보 분류 "고양이" + 속성 "뾰족한 귀" |
| S2 | "동물 A의 귀가 뾰족하다" | 관찰 (입력1) |
| S3 | "개도 뾰족한 귀를 가질 수 있다" | 반대 근거 (입력2) |

**기대 결과 단언** (§10.2 표 그대로):
- H1 "동물 A는 고양이다"의 `status == unresolved`, `confidence == unresolved`
- **어떤 가설도 `supported`가 아님**
- **"고양이"가 사실로 승격되지 않음** (`observed`/`verified` 컨텍스트 미생성)
- `counter_evidence`에 S3 포함
- `open_questions`에 "귀 이외의 구분 가능한 특징이 있는가?" 포함
- `unknown`과 `not_distinctive` 구분됨
- H1의 반증 연결 (`counters`) 확인

#### 테스트
- 단위: 연산 선행조건 미충족 → `invalid_inference`
- 단위: **수량사/양태 보존** — "모든"/"일부"/"가능성"이 관계에 남음
- 단위: `distinctiveness` — `not_distinctive` 판정 시 `scope` 기록
- 단위: **`distinctive`는 `scope` 없으면 불가**
- 단위: `unknown` ≠ `not_distinctive`
- 단위: `Evidence.direction` 파생 — `has_property` → `neutral`
- 단위: **counters 2경로** — (1) 술어 `contradicts` (2) distinctiveness 판정. **그 외 경로로 `counters`가 되는 코드 없음**
- 단위: **숫자 확률 없음** — `confidence` 값이 4개 범주형 값 중 하나
- 단위: R은 **DB를 변경하지 않음**
- 단위: `rule_version`이 모든 trace에 기록됨
- **통합(핵심): §10.2 end-to-end** — 기대 결과 단언 전체 통과
- 회귀: §10.2 픽스처가 `tests/regression/`에 등록되어 이후 Phase에서 깨지면 실패

#### 게이트
- [ ] 결론이 `InferenceTrace`를 갖고 실행 경로 재현 가능
- [ ] 잘못된 전제·부정·범위 차이·예외 케이스 포함
- [ ] **§10.2 대표 예시가 end-to-end로 재현됨** (기대 결과 단언 전체 통과)
- [ ] **검증 전 상태 전이 없음** (R은 DB를 바꾸지 않음)
- [ ] `rule_version`이 모든 trace에 기록됨
- [ ] `counters`가 2경로 외에서 생성되지 않음

#### 실험 산출물
`experiments/stage6_reasoning/` 4종 세트. `results.json`에 §10.2 단언 결과가 명시적으로 기록.

#### 비목표
- `deduction.is_a_transitivity`·`deduction.instance_class` **구현** (1차 범위 밖, 레지스트리 선언만)
- `induction.*`·`analogy.*` (Phase 11)
- `causal.what_if`·`planning.subgoal` (연구 백로그)
- 검증 상태 7종의 완전한 판정 (V는 최소분)

---

## 4. 1차 목표 밖으로 미룰 항목

| Phase | 핵심 산출물 | 선행 조건 |
|---|---|---|
| 6 (나머지) | 연역 연산 2종 구현 | Phase 6-부분 통과 |
| 7 | V 완성 (검증 상태 7종 판정 순서 전체, 모순/범위 불일치) | Phase 6 전체 |
| 8 | 인지 루프·종료 정책·`awaiting_observation` 판정 | Phase 7 |
| 9 | P-v1·D·CLI 실행기 | Phase 8 |
| 10 | 평가 하네스·B0~B4·leave-one-out 대조 | Phase 9 + 데이터셋 |
| 11 | `RuleCandidate` 구현·귀납 | Phase 10 |
| 12 | META-v1 + **개발자 경로 이관** | Phase 11 |
| 13 | `z_abs`·MRL 비교 | Phase 12 |
| 14~18 | 멀티모달·지속학습·학습형 종료 | Phase 13 |

---

## 5. 라벨링 부담 및 산출 일정

### 5.1 1차 목표 범위 라벨링

| 데이터셋 | 파일 | 규모 | Phase | 상태 |
|---|---|---|---|---|
| `parsing_dev` (개발 27 + 대립 15 + 비지원 10) | `tests/fixtures/parsing/parsing_dev.jsonl` | **52** | 3 | **착수 전 필요** |
| `parsing_holdout` | `tests/fixtures/parsing/parsing_holdout.jsonl` | **20** | 3 | Phase 10 전용 (1차 목표 밖이지만 구축) |
| `retrieval_pairs` | `tests/fixtures/retrieval/retrieval_pairs.jsonl` | **100** | 4 | **착수 전 필요** |
| §10.2 E2E 픽스처 | `tests/fixtures/e2e/case_10_2/` | 시드 3 + 단언 | 6-부분 | 착수 전 필요 |

**합계: 라벨 172건 (파싱 72 + 검색 100) + §10.2 E2E 픽스처 1세트 (시드 3 + 기대 결과 단언)**

### 5.2 절차 (ADR-0007)

1. 초안 작성 (언어 모델이 라벨 초안을 제안 가능)
2. **사람이 검수** — 검수되지 않은 라벨은 gold가 아님
3. `data/data_manifest.yaml`에 **검수자·검수일·검수 결과** 기록
4. §15.1.1 분할 규칙 — **문군(cluster) 단위** 분할. 패러프레이즈 변형이 개발/보류 세트에 동시에 들어가지 않음

### 5.3 일정상 순서

```
[병목 1] 파싱 라벨 52문장  ──┐
[병목 2] 검색 라벨 100쌍  ──┼──▶ Phase 4 착수
                             └──▶ Phase 3 착수
```

- **병목 1이 Phase 3의 착수 조건**이다. 라벨이 없으면 §7.3 통과 기준("사람이 라벨링한 참조 집합")을 충족할 수 없다.
- **병목 2가 Phase 4의 착수 조건**이다.
- **두 병목은 병렬로 진행 가능하다.** 라벨링이 코드 진행보다 느리면, 병목 1·2에 해당하는 산출물만 선행 구축하고 나머지 Phase를 진행한다. 이 경우 **해당 Phase는 "보류"로 기록한다**(ADR-0009).

---

## 6. 1차 목표에서 미결정으로 남기는 항목

| # | 항목 | 해소 시점 | 1차 목표에서의 처리 |
|---|---|---|---|
| §19 #3 | 유사도 임계값 | Phase 1 | Phase 1에서 **5범주 분포만 기록.** 임계값 미확정. 검색 시 필터로 쓰지 않음 |
| §19 #20 | 청킹 파라미터 | Phase 1 | 제안값(512/50) 사용. 실측 조정 가능 |
| §19 #21 | 사이클 판정 N | Phase 8 | Phase 5에서는 해시만 구현. N 미사용 |
| §19 #22 | 보상 신호 | Phase 17 | v1 미도입. `reward` 미저장 |
| §19 #23 | `node_types.yaml` 어휘 범위 | Phase 3 | **선택 사항.** 없어도 동작 (ADR-0008) |
| §19 #24 | `source_group_id` 수동 병합 절차 | Phase 7 | v1 자동 판정(`source_id` 동일 → 같은 그룹)만 사용 |
| §19 #6 | FAISS vs hnswlib | Phase 10 이후 | `BruteForceIndex`만 |
| §19 #7 | Hot/Warm/Cold 승강 기준 | Phase 10 이후 | 스텁. 정책만 |
| §19 #1b | 백엔드 모델 동일성 | Phase 0 | **Phase 0에서 판정.** 판정 결과에 따라 프로파일 분리 |
| §19 #18 | `evidence_linking` 규모 | Phase 7 | Phase 7 착수 전 확정 (1차 범위 밖) |
| §19 #11 | 목표 수치 현실성 | 각 Phase | **수치 목표를 달성 판정으로 쓰지 않는다**(§15.7) |
| §19 #5·#8·#9 | 학습 관련 | Phase 13·17·18 | 1차 범위 밖 |

---

## 7. 문서 관리

| 규칙 | 내용 |
|---|---|
| 규범 변경 | `summary_arch.md`를 버전 올려 수정하고 `docs/decisions/`에 ADR 추가 |
| 파생 문서 변경 | 이 문서·`docs/DATA_MODEL.md`·`docs/EXPERIMENT_PROTOCOL.md`는 규범을 변경하지 않는다. 규범과 충돌하면 규범 승리 |
| 실행 로그 | `docs/EXPERIMENTS.md`는 **실측만.** 여기에 규범을 쓰지 않는다 |
| Phase 산출물 | `experiments/stageN_*/` 4종 세트. **기존 결과 덮어쓰기 금지**(§18.2) |
| ADR | 개발자 경로로 구조를 변경할 때마다 1건 추가 (§7.5.1 게이트 3) |
