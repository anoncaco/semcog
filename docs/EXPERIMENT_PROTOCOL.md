# EXPERIMENT_PROTOCOL — 평가 프로토콜

- **문서 버전:** 0.1 (Phase 3 착수 전 확정)
- **작성일:** 2026-10-20
- **기준:** `summary_arch.md` v1.3 §15.1.1, §15.2, §15.4, §15.5, §15.6, §15.7, ADR-0007, ADR-0010
- **지위:** 파생 문서. 충돌 시 `summary_arch.md`가 이긴다.

---

## 1. 공통 프로토콜 (§15.4)

모든 주요 평가는 **4요소**를 명시한다.

| # | 요소 | 내용 |
|---|---|---|
| 1 | **데이터셋 및 데이터 분할** | 어떤 데이터셋의 어떤 분할을 쓰는지 |
| 2 | **비교 기준선** | 무엇과 비교하는지 (B0–B7, §15.1) |
| 3 | **측정 지표 및 계산 방법** | 지표 정의 + 계산식 |
| 4 | **난수 시드와 반복 조건** | 시드 값, 반복 횟수, 분산 보고 방식 |

**추가로 각 데이터셋은 7개 항목을 명시한다(§15.1.1).**

1. 평가 목적
2. 입력 형식
3. 정답 라벨 형식
4. 데이터 출처
5. 개발 / 검증 / 테스트 분할
6. 분할 기준과 난수 시드
7. 지표와 계산 방법

**시드 정책(§18.1.1)**: 모든 난수 발생 지점은 **명시적 시드**를 받는다. 기본 시드를 하드코딩하지 않는다. 시드는 `results.json`에 기록한다.

---

## 2. 파싱 라벨 데이터셋

### 2.1 규모와 구성 (ADR-0007)

| 세트 | 파일 | 규모 | Phase |
|---|---|---|---|
| `parsing_dev` | `tests/fixtures/parsing/parsing_dev.jsonl` | **52** (개발 27 + 대립 15 + 비지원 10) | 3 |
| `parsing_holdout` | `tests/fixtures/parsing/parsing_holdout.jsonl` | **20** | 3 (구축), 10 (사용) |

### 2.2 라벨 형식 (JSONL, 1문장 1행)

```json
{
  "id": "pd_001",
  "text": "고양이는 동물이다",
  "split": "dev",
  "pattern_id": "is_a_typical",
  "parse_status": "parsed",
  "relations": [
    {
      "subject": "고양이",
      "predicate": "is_a",
      "object": "동물",
      "quantifier": "typical",
      "polarity": "positive",
      "span": [0, 9]
    }
  ],
  "nodes": [
    {"canonical_label": "고양이", "node_type": "entity"},
    {"canonical_label": "동물", "node_type": "class"}
  ],
  "notes": ""
}
```

| 필드 | 설명 |
|---|---|
| `id` | 세트 내 고유 ID |
| `text` | 원문 |
| `split` | `dev` / `holdout` (파일 단위 분할) |
| `subset` | 문장 역할. `base` / `adversarial` / `out_of_scope` / `holdout` (§7.3 세트) |
| `pattern_id` | §7.3의 9개 문형 행 ID. `unparsed`는 `unparsable` |
| `parse_status` | 기대 `parse_status` |
| `relations` | 기대 관계. **관계가 없으면 빈 배열** |
| `nodes` | 기대 ConceptNode. `canonical_label`은 **정규화된 값** |
| `notes` | 자유 메모 |

- **`subset`은 문군(cluster) 단위로 배정한다.** 같은 문군의 패러프레이즈가 서로 다른 `subset`에 들어가지 않는다(§15.1.1 분리 규칙과 동일 원칙). 한 문장 안에서 `subset`이 갈라지지 않는다.
- **`split`과 `subset`은 다른 축이다.** `split`은 파일 단위 분할(dev/holdout), `subset`은 파일 안에서 문장의 역할을 나타낸다. `parsing_dev.jsonl`은 `split=dev` + `subset` ∈ {`base`,`adversarial`,`out_of_scope`} 52행, `parsing_holdout.jsonl`은 `split=holdout` + `subset=holdout` 20행.

### 2.3 세 구성

| 세트 | `subset` 값 | 규모 | 규칙 |
|---|---|---|---|
| **개발** | `base` | 27 | 9개 문형 × 최소 3문장. **§10.2의 2문장을 반드시 포함** |
| **대립** | `adversarial` | 15 | 부정·수량사(모든/일부)·양태 변형. 주어 조사(`도`/`가`) 변형 포함 |
| **비지원** | `out_of_scope` | 10 | `unparsed`여야 하는 복합문·비지원 문형. **관계·노드 배열은 비어 있어야 한다** |
| **보류** | `holdout` | 20 | 규칙에 노출되지 않음. §15.5 일반화 + Phase 10 전용 |

### 2.4 문형 ID (9개 행 — §7.3 v1.3)

| `pattern_id` | 술어 | 양태 | 극성 |
|---|---|---|---|
| `is_a_typical` | `is_a` | `typical` | positive |
| `is_a_all` | `is_a` | `all` | positive |
| `is_a_some` | `is_a` | `some` | positive |
| `has_property_possible` | `has_property` | `possible` | positive |
| `has_property_typical` | `has_property` | `typical` | positive |
| `is_a_negative` | `is_a` | `typical` | **negative** |
| `is_a_also` | `is_a` | `typical` | positive |
| `causes_instance` | `causes` | `specific_instance` | positive |
| `unparsable` | — | — | — |

### 2.5 측정 지표 (Phase 3)

지표를 **분리** 측정한다. 관계 하나라도 잘못 추출하면 추론 결과가 달라지므로, 문장 단위 정답 여부만으로는 충분하지 않다.

| 지표 | 단위 | 계산 |
|---|---|---|
| `predicate_precision` / `predicate_recall` | 관계 | 예측 술어 집합 vs 기대 술어 집합 |
| `argument_precision` / `argument_recall` | 관계 | subject·object 일치율 |
| `polarity_accuracy` | 관계 | 극성 일치율 |
| `quantifier_accuracy` | 관계 | 양태·수량사 일치율 |
| `span_exact_match_rate` | 관계 | `span`이 원문 부분 문자열과 완전 일치 |
| `unparsed_accuracy` | 문장 | 기대 `parse_status`와 실제 일치 |
| `parse_status_accuracy` | 문장 | 3값 일치 |
| `node_shared_rate` | 문장 쌍 | 같은 속성을 공유하는 노드가 생성됨 |

- **개발 세트와 보류 세트의 지표를 분리 보고한다.** 보류 세트에서 낮고 개발 세트에서 높으면 **과적합**이다.
- `node_shared_rate`는 §10.2의 2문장이 **동일한 `canonical_label`**의 노드를 만드는지 확인한다.

### 2.6 데이터 분리 규칙 (§15.1.1)

- **문군(cluster) 단위** 분할. 문장 단위 분할 금지.
- 같은 문장의 패러프레이즈·표현 변형은 **같은 분할**에 들어간다.
- §10.2의 2문장은 **개발 세트**에 고정. 보류 세트에 동일 문장을 넣지 않는다.

---

## 3. 검색 평가 데이터셋 (`retrieval_pairs`)

### 3.1 규모·구성 (ADR-0007)

| 항목 | 값 |
|---|---|
| 파일 | `tests/fixtures/retrieval/retrieval_pairs.jsonl` |
| 규모 | **100쌍** |
| Phase | 4 (구축·측정), 10 (재측정) |

### 3.2 라벨 형식

```json
{
  "id": "rp_001",
  "query": "뾰족한 귀를 가진 동물",
  "query_profile_id": "context_retrieval_v1",
  "relevant_context_ids": ["ctx_001", "ctx_007"],
  "counter_context_ids": ["ctx_013"],
  "notes": ""
}
```

| 필드 | 설명 |
|---|---|
| `relevant_context_ids` | 정답 컨텍스트. Recall@K의 정답 |
| `counter_context_ids` | **반대 방향 근거.** 검색에서 사라지지 않아야 하는 컨텍스트(§7.6 Phase 4 게이트) |

### 3.3 지표

| 지표 | 정의 |
|---|---|
| `Recall@K` | 정답 컨텍스트 중 K개 안에 들어온 비율 |
| `MRR@K` | 첫 정답의 역순위 평균 |
| `nDCG@K` | 순위 기반 정규화 할인 누적 이득 |
| `counter_survival_rate@K` | `counter_context_ids` 중 검색 결과에 포함된 비율 |

**`counter_survival_rate@K`는 §7.6의 "반대 근거가 검색에서 사라지지 않음" 게이트를 직접 측정한다.** 이 값이 낮으면 통과 불가.

---

## 4. 인지 평가 데이터셋 (Phase 10 — 1차 목표 범위 밖)

### 4.1 `cognition_core` (50)

- 단일 스텝 판단 세트. 결론·근거·반증 연결 라벨.
- **정답은 사람이 라벨링한다**(§15.1.1).
- 목표 지표: 결론 정확도, 근거 부족 판정 정확도, **반증 탐지율**, **반증 오경보율**.

### 4.2 `cognition_loop` (30)

- 다단계 종료 케이스 세트. 근거 부족·반복 고착·모순 미해결·관찰 대기.
- 목표 지표: 종료율, 루프 고착 비율, 평균 스텝 수, `termination_status` 분류 정확도.

### 4.3 `evidence_linking` (미정 — §19 #18)

- (근거, 가설, 방향) 라벨 세트. 지지/반박 출처 연결 정확도.
- **규모는 Phase 7 착수 전에 확정한다.**

---

## 5. 기준선 비교 프로토콜 (ADR-0010)

### 5.1 단일 변인 통제

- 데이터·시드·설정·결정적 정책은 **B0~B7 전부에서 동일**하게 고정한다.
- 각 B(k)는 **자기 이전 단계에서 추가된 구성요소 하나만** 켠 상태로 실행한다.

### 5.2 leave-one-out 대조

| B(k) | 제거 조건 |
|---|---|
| B3 | V의 반증/모순 검사만 제거 |
| B4 | 반복 인지 루프만 제거 |
| B5 | 규칙 후보 학습만 제거 |
| B6 | META 구조 변경만 제거 |
| B7 | 학습된 `z_abs`만 제거 |

- B(k) − B(k-L1) 의 차이가 해당 구성요소의 귀속 기여도다.

### 5.3 지표별 드롭 지점 표시

성능 저하 시 **어느 B(k)부터** 떨어졌는지 지표별로 표시한다. §5.3 진단 순서와 결합해 원인을 좁힌다.

### 5.4 B4 내부 3조건 대조 (§12.2)

| 조건 | 정의 |
|---|---|
| A. Memory Disabled | 메모리 검색 없이 인지 루프만 |
| B. Single Retrieval | 지정된 스텝에서 메모리 1회 검색 |
| C. Repeated Retrieval | 여러 스텝에서 메모리 반복 검색 |

- 공통 측정: 작업 성능, 평균 스텝 수, 검색 횟수, 검색 지연, 전체 지연, 메모리 사용량.
- **반복 검색이 우월하다고 사전 가정하지 않는다.**
- **검색 결과의 저장 여부와 실제 활용 여부를 trace에 기록한다**(P11).

---

## 6. 일반화 평가 (§15.5)

| # | 변환 유형 |
|---|---|
| 1 | 동일 문장 패러프레이즈 |
| 2 | 엔티티 이름 교체 조합 |
| 3 | 관계 유형은 같고 표면 내용이 다른 **교차 도메인** |
| 4 | 알려진 규칙의 반례/예외 |
| 5 | 학습에 등장하지 않은 컨텍스트 조합 |

- **분리 규칙은 사전에 정의한다.** 학습과 테스트가 문장 표현만 바뀐 거의 동일한 예시로 중복되지 않게 한다.
- `parsing_holdout`는 1번·2번 축의 보류 세트로 사용한다.

---

## 7. META 승인 기준 (§15.6)

각 구조 변경 종류마다 최소 승인 조건을 **사전에** 정한다. **결과를 본 뒤 유리하게 기준을 바꾸지 않는다.**

- 기존 회귀 세트 정확도가 사전 정의된 허용 범위 이상 떨어지지 않을 것
- 목표 문제 세트에서 측정 가능한 개선
- 반례에서 악화되지 않을 것
- 무한 루프 / 출처 손실 / 알 수 없는 관계를 확정 사실로 승격하는 새 오류를 만들지 않을 것
- 메모리·지연 비용이 허용 범위 안

**수치 기준은 테스트 데이터를 만든 뒤 개발자가 사전에 정한다.**

---

## 8. 보고 규칙 (§18.3)

- **실측하지 않은 수치는 보고하지 않는다.** 실패·미실행은 **명시한다.**
- `REPORT.md`에는 **실측 수치만** 기록한다.
- **목표 수치와 실측 수치를 구분**해서 적는다.
- **단위 테스트 통과 ≠ 가설 입증.**
- §15.7: **수치 목표를 달성 여부 판정으로 쓰지 않는다.** 측정법이 정의되지 않으면 "측정 불가"로 기록한다.

### 8.1 `results.json` 최소 필드

```json
{
  "experiment": "stage3_parse",
  "command": "python experiments/stage3_parse/parse_eval.py",
  "timestamp_utc": "2026-10-20T00:00:00Z",
  "seeds": {"global": 20261020},
  "environment": {"python": "3.12.3", "packages": {}, "git_rev": null},
  "dataset": {"id": "parsing_dev", "version": "...", "split": "dev", "count": 52, "manifest_hash": "..."},
  "metrics": {},
  "per_item": null,
  "verdict": "pass | fail | deferred",
  "verdict_basis": "...",
  "unresolved": [],
  "elapsed_sec": 0.0
}
```

- `verdict`는 ADR-0009의 3상태.
- `unresolved`에는 **미해결 사항**을 적는다. 비워 두지 않는다.
- **기존 `results.json`을 덮어쓰지 않는다**(§18.2).

---

## 9. 데이터셋 관리

### 9.1 `data/data_manifest.yaml`

```yaml
datasets:
  - id: parsing_dev
    path: tests/fixtures/parsing/parsing_dev.jsonl
    version: "1.0.0"
    created_at: "2026-10-20T00:00:00Z"
    created_by: "<name>"
    reviewed_by:
      - name: "<name>"
        reviewed_at: "2026-10-20T00:00:00Z"
        result: approved | corrected | rejected
    generation_method: manual_draft_then_human_review
    sha256: "<파일 해시>"
    cluster_count: <문군 수>
    notes: ""
```

- **검수되지 않은 라벨은 gold가 아니다.** `reviewed_by`가 비어 있으면 gold가 아니다.
- 파일이 바뀌면 `sha256`과 `version`을 갱신한다.

### 9.2 저장 위치

| 데이터셋 | 위치 |
|---|---|
| 파싱 | `tests/fixtures/parsing/` |
| 검색 | `tests/fixtures/retrieval/` |
| §10.2 E2E | `tests/fixtures/e2e/case_10_2/` |
| 원본·가공 데이터 | `data/raw/`, `data/processed/` |

**`src/` 하위에 fixture 디렉터리를 만들지 않는다**(ADR-0005).
