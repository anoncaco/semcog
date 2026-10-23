# DATA_MODEL — 데이터 모델 및 저장 스키마

- **문서 버전:** 0.1 (Phase 2 초안 — Phase 2에서 완성)
- **작성일:** 2026-10-20
- **기준:** `summary_arch.md` v1.3 §8 (데이터 모델), §9 (인터페이스 계약), §15.1.1
- **지위:** `summary_arch.md`로부터 파생. 충돌 시 `summary_arch.md`가 이긴다.

> **이 문서의 규칙**
> - SQLite 테이블 스키마는 `summary_arch.md`에 정의하지 않고 **여기**에서 정의한다(§17).
> - 필드마다 **생성 모듈**과 **수정 권한**을 명시한다. 권한 밖의 수정은 오류다.
> - 스키마 변경은 §7.5.1 개발자 직접 등록 경로 + ADR로 한다.

---

## 1. 공통 규칙

| # | 규칙 |
|---|---|
| 1 | 모든 레코드는 `created_at`을 가진다. **UTC ISO-8601, `Z` 접미사.** 로컬 시간 금지(§18.1.1) |
| 2 | 모든 ID는 접두어 + ULID/UUID 기반. 접두어로 타입을 구분한다 |
| 3 | `revision`은 증가만 한다. **수정 = 새 revision.** 이전 revision은 보존 |
| 4 | 삭제하지 않는다. 무효화는 `status` 변경(`retracted` 등)으로 |
| 5 | `content_hash`는 **식별용**이다. 검색 스코어·유사도에 쓰지 않는다(§6.2 규칙 3) |
| 6 | 타임스탬프 컬럼은 ISO-8601 TEXT로 저장. epoch 정수로 바꾸지 않는다 |
| 7 | JSON 컬럼은 **키 정렬 직렬화**한다(§7.5 §8.8) |
| 8 | 벡터은 DB에 넣지 않는다. `data/vectors/<profile_id>/` 파일 + 인덱스 참조 |

### 1.1 ID 접두어

| 접두어 | 대상 |
|---|---|
| `ctx_` | ContextRecord |
| `rel_` | RelationRecord |
| `node_` | ConceptNode |
| `hyp_` | Hypothesis |
| `ev_` | EvidenceRecord |
| `trace_` | InferenceTrace |
| `rt_` | RetrievalTrace |
| `meta_` / `meta_dev_` | StructureChangeProposal |
| `fail_` | FailureCase |
| `rc_` | RuleCandidate |
| `sess_` | Session |
| `ev_evt_` | Event |

---

## 2. 필드별 생성 모듈·수정 권한

| 스키마 | 생성 모듈 | 수정 권한 | 다른 모듈이 하는 일 |
|---|---|---|---|
| `ContextRecord` | `context/registry.py` (Parse, E는 결과만 제공) | registry만 | Parse는 relation 후보를, E는 벡터를 **참조**로만 추가 |
| `RelationRecord` | `parsing/parser.py` (후보) → `cognition/transitions.py` (승격) | Parse가 `candidate` 생성, V/전이 경로가 `accepted`/`disputed`/`retracted` | **R은 관계를 만들지 않는다** |
| `ConceptNode` | `parsing/nodes.py` | nodes만 (`merged`/`retracted` 전이 경유) | Parse가 `context_ids`에 참조 추가 |
| `Hypothesis` | `cognition/transitions.py` (F) | F만 | R은 **후보**만 만들고 F가 생성 |
| `EvidenceRecord` | `cognition/transitions.py` (F) | F만. `direction`은 **파생**이라 직접 수정 금지 | V가 방향을 계산해 전달 |
| `InferenceTrace` | `reasoning/*` (R) | R만 | V는 `validation_status`를 갱신 |
| `RetrievalTrace` | `memory/retrieval.py` (M) | M만 | F는 참조만 |
| `StructureChangeProposal` | `meta/proposals.py` 또는 개발자 경로 | META / 개발자 | — |
| `FailureCase` | 각 모듈의 실패 핸들러 | META가 `status`·`diagnosis_*` 갱신 | — |
| `RuleCandidate` | `reasoning/induction.py` | Phase 11 | — |
| `Event` | `cognition/events.py` (기록 전용) | **수정·삭제 금지** | append-only |

---

## 3. 스키마 상세

### 3.1 ContextRecord — `contexts`

```sql
CREATE TABLE contexts (
    context_id              TEXT PRIMARY KEY,
    content_hash            TEXT NOT NULL,
    canonicalization_version TEXT NOT NULL,
    context_type            TEXT NOT NULL,   -- observation|statement|rule|hypothesis|inference|composite
    modality                TEXT NOT NULL,   -- text|image|audio|video|mixed
    raw_content             TEXT NOT NULL,
    canonical_content       TEXT NOT NULL,
    parent_context_ids      TEXT NOT NULL DEFAULT '[]',
    child_context_ids       TEXT NOT NULL DEFAULT '[]',
    embedding_refs          TEXT NOT NULL DEFAULT '[]',
    relation_ids            TEXT NOT NULL DEFAULT '[]',
    provenance              TEXT NOT NULL DEFAULT '[]',
    adapter_meta            TEXT NOT NULL DEFAULT '{}',
    parse_status            TEXT,            -- parsed|partial|unparsed
    epistemic_status        TEXT NOT NULL,   -- observed|extracted|hypothesized|inferred|verified|rejected|uncertain
    confidence              TEXT,
    importance              REAL,            -- §11.1, 접근/갱신 시점에만 계산
    created_at              TEXT NOT NULL,
    accessed_at             TEXT,
    revision                INTEGER NOT NULL DEFAULT 1
);
CREATE INDEX idx_contexts_hash     ON contexts(content_hash);
CREATE INDEX idx_contexts_epistemic ON contexts(epistemic_status);
CREATE INDEX idx_contexts_type     ON contexts(context_type);
```

**규칙**

- `context_id`는 레코드 **인스턴스** 식별. `content_hash`는 **콘텐츠** 식별. 둘을 혼용하지 않는다.
- **같은 해시라도 출처가 다르면 별도 레코드.** 저장 단계에서 병합하지 않는다(§7.4).
- `provenance`는 **리스트**. 단일 출처를 가정하지 않는다.
- `embedding_refs` 항목: `{profile_id, dimension, backend, model_id, digest, vector_ref, norm, is_finite, index_loaded}`.
  - `vector_ref`는 `data/vectors/<profile_id>/<context_id>.bin` 상대 경로.
  - `index_loaded=false`면 **검색에서 제외**된다(§7.6 §7.4 단계 4).
- `parent_context_ids` / `child_context_ids`는 **양방향**으로 유지한다(§8.9).

### 3.2 RelationRecord — `relations`

```sql
CREATE TABLE relations (
    relation_id        TEXT PRIMARY KEY,
    subject_context_id TEXT NOT NULL REFERENCES contexts(context_id),
    predicate          TEXT NOT NULL,
    object_context_id  TEXT NOT NULL REFERENCES contexts(context_id),
    polarity           TEXT NOT NULL,   -- positive|negative
    quantifier         TEXT NOT NULL,   -- possible|typical|all|exists|some|specific_instance
    validity_scope     TEXT,
    source_context_ids TEXT NOT NULL DEFAULT '[]',
    extraction_method  TEXT NOT NULL,   -- rule_parser_v1
    status             TEXT NOT NULL,   -- candidate|accepted|disputed|retracted
    created_by         TEXT NOT NULL,   -- parse|inference|meta
    span               TEXT,            -- [start, end]
    revision           INTEGER NOT NULL DEFAULT 1,
    created_at         TEXT NOT NULL
);
CREATE INDEX idx_relations_subject ON relations(subject_context_id);
CREATE INDEX idx_relations_object  ON relations(object_context_id);
CREATE INDEX idx_relations_pred    ON relations(predicate);
CREATE INDEX idx_relations_status  ON relations(status);
```

**규칙**

- `predicate`는 §7.3 술어 사전의 값만. **대응 술어는 `polarity=negative`로 표기**하고 별도 술어로 만들지 않는다.
- `status=candidate`로 시작한다. `accepted`가 되려면 **V의 검증**이 필요하다(§7.3).
- `span`은 원문 `raw_content` 기준 오프셋. **반드시 실제 부분 문자열과 일치**해야 한다.

### 3.3 ConceptNode — `concept_nodes`

```sql
CREATE TABLE concept_nodes (
    node_id              TEXT PRIMARY KEY,
    node_type            TEXT NOT NULL,   -- entity|class|attribute|attribute_value
    canonical_label      TEXT NOT NULL,
    surface_forms        TEXT NOT NULL DEFAULT '[]',
    aliases              TEXT NOT NULL DEFAULT '[]',
    node_kind            TEXT NOT NULL,   -- literal|mention
    context_ids          TEXT NOT NULL DEFAULT '[]',
    resolution_method    TEXT NOT NULL,   -- literal_normalized_match|alias_table|manual|unresolved
    resolution_confidence TEXT NOT NULL,  -- low|medium|high
    status               TEXT NOT NULL,   -- active|merged|retracted
    revision             INTEGER NOT NULL DEFAULT 1,
    created_at           TEXT NOT NULL
);
CREATE UNIQUE INDEX idx_nodes_label ON concept_nodes(canonical_label);
CREATE INDEX idx_nodes_type ON concept_nodes(node_type);
```

**규칙**

- `node_type`은 **관계 위치**로 결정한다(§8.10, ADR-0008). `configs/node_types.yaml`이 있으면 사전이 우선.
- **같은 `canonical_label`은 하나의 노드.** 다른 출처의 동일 라벨도 같은 노드로 묶되 `surface_forms`에 모두 기록.
- `node_kind=mention`(대명사·의역 대치 등)은 v1 미지원 → `status=unresolved`, `node_type=entity`.
- **노드는 삭제하지 않는다.** 오해석은 `status=retracted` + 묶인 관계·근거 재평가.

### 3.4 Hypothesis — `hypotheses`

```sql
CREATE TABLE hypotheses (
    hypothesis_id         TEXT PRIMARY KEY,
    claim                 TEXT NOT NULL,
    supporting_evidence_ids TEXT NOT NULL DEFAULT '[]',
    counter_evidence_ids  TEXT NOT NULL DEFAULT '[]',
    derived_from_context_ids TEXT NOT NULL DEFAULT '[]',
    inference_trace_id    TEXT REFERENCES inference_traces(trace_id),
    status                TEXT NOT NULL,   -- candidate|supported|weakened|unresolved|rejected
    confidence            TEXT NOT NULL,   -- low|medium|high|unresolved
    confidence_method     TEXT NOT NULL,   -- ordinal_v1
    confidence_components TEXT NOT NULL DEFAULT '{}',
    open_questions        TEXT NOT NULL DEFAULT '[]',
    revision              INTEGER NOT NULL DEFAULT 1,
    created_at            TEXT NOT NULL
);
CREATE INDEX idx_hypotheses_status ON hypotheses(status);
```

**`confidence_components` (ADR-0003)**

```json
{
  "support_count": 1,        // 독립 근거 수 (source_group_id 기준)
  "counter_count": 1,        // 독립 근거 수
  "support_count_raw": 3,    // 전체 근거 수
  "counter_count_raw": 3,    // 전체 근거 수
  "applied_rule": "hypothesis.generate_from_feature.v1",
  "rule_version": "1.0.0"
}
```

- `support_count`와 `support_count_raw`는 **다를 수 있다.** 어느 것을 쓰는지 구분 가능해야 한다.
- §11.1 `evidence_weight`·§8.12 `independent_case_count`와 **반드시 같은 값**이어야 한다.

### 3.5 EvidenceRecord — `evidence`

```sql
CREATE TABLE evidence (
    evidence_id    TEXT PRIMARY KEY,
    context_id     TEXT NOT NULL REFERENCES contexts(context_id),
    relation_id    TEXT REFERENCES relations(relation_id),
    direction      TEXT NOT NULL,   -- supports|counters|neutral (파생값)
    source_kind    TEXT NOT NULL,   -- user_input|observation|llm_generated|external
    strength       TEXT NOT NULL,   -- weak|moderate|strong
    scope_match    INTEGER NOT NULL, -- 0/1
    lineage        TEXT NOT NULL,   -- §3.5.1
    created_at     TEXT NOT NULL
);
CREATE INDEX idx_evidence_context  ON evidence(context_id);
CREATE INDEX idx_evidence_direction ON evidence(direction);
```

#### 3.5.1 `lineage` (ADR-0003)

```json
{
  "source_group_id": "srcgrp_...",
  "derivation_kind": "direct | copy | paraphrase | derived",
  "derived_from_evidence_ids": []
}
```

| 규칙 | 내용 |
|---|---|
| 그룹 판정 | `provenance.source_id`가 같으면 같은 `source_group_id`. 다르면 다른 그룹 |
| `derived` | `derived_from_evidence_ids`가 **비어 있으면 안 된다** |
| 독립 개수 | `source_group_id`의 **서로 다른 값 개수** |
| 수동 병합 | v1 미구현(§19 #24). 두 그룹을 합치는 API는 Phase 7까지 없다 |

**`direction` 파생 규칙 (§7.8)**

| 근거 관계의 술어 | `direction` |
|---|---|
| `supports` | `supports` |
| `contradicts` | `counters` |
| 그 외 (`has_property`, `is_a`, …) | `neutral` |

**`counters`가 되는 경로는 2개뿐이다.** (1) 술어가 `contradicts` (2) distinctiveness 판정을 V가 반증으로 승격. **이 외 경로는 코드에 존재하지 않아야 하며, 회귀 테스트로 검사한다.**

### 3.6 InferenceTrace — `inference_traces`

```sql
CREATE TABLE inference_traces (
    trace_id              TEXT PRIMARY KEY,
    operator_id           TEXT NOT NULL,
    premise_ids           TEXT NOT NULL DEFAULT '[]',
    conclusion_context_id TEXT REFERENCES contexts(context_id),
    rule_version          TEXT NOT NULL,   -- 비어 있을 수 없다
    validation_status     TEXT NOT NULL,   -- pending|passed|failed|needs_more_evidence
    scope                 TEXT,            -- §3.6.1
    created_at            TEXT NOT NULL
);
CREATE INDEX idx_traces_operator ON inference_traces(operator_id);
```

#### 3.6.1 `scope` (ADR-0003)

```json
{
  "scope_type": "store | validity_scope | explicit",
  "scope_refs": ["node_..."],
  "description": "현재 SymbolicStore에 존재하는 주어 전체"
}
```

- `distinctive` 판정에는 **`scope`가 필수**이다. 없으면 `distinctive`는 허용되지 않는다.
- `not_distinctive`·`unknown`은 `scope` 없이도 유효하다.

### 3.7 RetrievalTrace — `retrieval_traces`

```sql
CREATE TABLE retrieval_traces (
    retrieval_id      TEXT PRIMARY KEY,
    step              INTEGER NOT NULL,
    query_context_ids TEXT NOT NULL DEFAULT '[]',
    profile_id        TEXT NOT NULL,
    top_k             INTEGER NOT NULL,
    hits              TEXT NOT NULL DEFAULT '[]',
    hits_not_used     TEXT NOT NULL DEFAULT '[]',
    latency_ms        REAL,
    created_at        TEXT NOT NULL
);
```

**`hits` 항목**: `{context_id, score, rank, used, use_reason}`.
**`hits_not_used`는 "축약"이 아니라 "사용하지 않음" 표시다.** 삭제·무시 중 하나로 표기하지 않는다(§7.6).

### 3.8 StructureChangeProposal — `structure_change_proposals`

```sql
CREATE TABLE structure_change_proposals (
    proposal_id             TEXT PRIMARY KEY,
    proposed_by             TEXT NOT NULL DEFAULT 'developer',  -- developer|meta
    trigger_cases          TEXT NOT NULL DEFAULT '[]',
    diagnosis              TEXT,
    diagnosis_category     TEXT,   -- 7종 고정 (§8.11)
    change_type            TEXT NOT NULL,
    proposed_change        TEXT NOT NULL,
    expected_benefit       TEXT,
    regression_suite       TEXT NOT NULL,
    status                 TEXT NOT NULL,   -- proposed|sandbox_testing|accepted|rejected|rolled_back
    before_metrics         TEXT NOT NULL DEFAULT '{}',
    after_metrics          TEXT NOT NULL DEFAULT '{}',
    gate_evidence          TEXT NOT NULL DEFAULT '{}',
    approved_structure_version TEXT,
    created_at             TEXT NOT NULL
);
```

**`proposed_by=developer`인 레코드는 `gate_evidence`에 게이트 4조건 증빙을 담는다**(ADR-0001):
`{version_bump, regression, adr, version_propagation}`.

### 3.9 FailureCase — `failure_cases`

```sql
CREATE TABLE failure_cases (
    failure_id         TEXT PRIMARY KEY,
    session_id         TEXT,
    step               INTEGER,
    failure_kind       TEXT NOT NULL,
    expected           TEXT,
    actual             TEXT,
    failed_stage       TEXT NOT NULL,   -- §3.9.1
    context_ids        TEXT NOT NULL DEFAULT '[]',
    hypothesis_ids     TEXT NOT NULL DEFAULT '[]',
    trace_ids          TEXT NOT NULL DEFAULT '[]',
    diagnosis_category TEXT,            -- 7종 고정
    diagnosis_status   TEXT NOT NULL DEFAULT 'suspected',  -- suspected|confirmed
    diagnosis          TEXT,
    status             TEXT NOT NULL DEFAULT 'new',  -- new|diagnosed|proposed|resolved|dismissed
    created_at         TEXT NOT NULL
);
CREATE INDEX idx_failures_stage ON failure_cases(failed_stage);
CREATE INDEX idx_failures_diag  ON failure_cases(diagnosis_category);
```

#### 3.9.1 `failed_stage` (ADR-0003)

`adapter` | `encoder` | `parser` | `registry` | `retriever` | `reasoner` | `verifier` | `policy` | `store` | `runtime`

**`failed_stage`(발생 지점)와 `diagnosis_category`(근본 원인)는 별개 축이다.** 둘 다 기록한다. `diagnosis_category`는 7종에서 **벗어나지 않는다.**

### 3.10 RuleCandidate — `rule_candidates` (Phase 11 대비 계약)

```sql
CREATE TABLE rule_candidates (
    candidate_id          TEXT PRIMARY KEY,
    rule_form             TEXT NOT NULL,
    relation_pattern      TEXT NOT NULL,
    source_case_ids       TEXT NOT NULL DEFAULT '[]',
    independent_case_count INTEGER NOT NULL DEFAULT 0,
    independent_source_count INTEGER NOT NULL DEFAULT 0,
    applicability_scope   TEXT,
    exception_case_ids    TEXT NOT NULL DEFAULT '[]',
    counter_case_ids      TEXT NOT NULL DEFAULT '[]',
    algorithm_version     TEXT NOT NULL,
    min_case_threshold    INTEGER NOT NULL,
    status                TEXT NOT NULL,  -- candidate|validated|rejected|promoted
    validated_on_case_ids TEXT NOT NULL DEFAULT '[]',
    created_at            TEXT NOT NULL
);
```

**규칙**: `counter_case_ids`가 비어 있지 않으면 `status=promoted`로 갈 수 없다. `independent_case_count`는 `source_group_id` 기준.

### 3.11 Event — `events` (append-only)

```sql
CREATE TABLE events (
    event_id    TEXT PRIMARY KEY,
    event_type  TEXT NOT NULL,
    session_id  TEXT NOT NULL,
    step        INTEGER NOT NULL,
    actor       TEXT NOT NULL,   -- 모듈명
    subject_ids TEXT NOT NULL DEFAULT '[]',
    reason      TEXT,
    payload     TEXT NOT NULL DEFAULT '{}',
    created_at  TEXT NOT NULL
);
CREATE INDEX idx_events_session ON events(session_id, step);
CREATE INDEX idx_events_type    ON events(event_type);
```

**규칙**

- **append-only.** 수정·삭제 금지.
- 쓰기는 **멱등**하다. 같은 `event_id` 재시도 시 중복 삽입되지 않는다.
- `payload`는 키 정렬 JSON. **이벤트 테이블의 컬럼은 9개로 고정**이며 `event_type`별 추가 필드는 `payload`에 넣는다(§7.5, ADR-0011). `payload`의 타입별 스키마는 **Phase 5에서 정의**한다.

**Phase 4에서 만드는 것 (ADR-0006)**: 테이블·인덱스·멱등 쓰기 API + **기록 인프라**.
**Phase 5에서 완성하는 것**: 상태 전이 이벤트 + **원자적 트랜잭션(상태 + 이벤트)** + 스냅샷 + replay.

### 3.12 STATE 스냅샷 — 파일

```
data/sqlite/state/<session_id>/<step>.json
```

- JSON, **키 정렬 직렬화**.
- 스키마는 `summary_arch.md` §7.5의 직렬화 스키마 그대로.
- 마지막 스냅샷이 복원 지점. **이벤트 로그가 재현 경로.**

### 3.13 벡터 저장 — 파일

```
data/vectors/<profile_id>/<context_id>.bin
```

- `profile_id` 디렉터리로 분리(§17). **프로파일 간 벡터를 섞지 않는다.**
- 파일 포맷: `float32` 순수 배열 (헤더 없이) + 메타는 `contexts.embedding_refs`에.
- **인덱스 적재와 파일 저장은 별개 단계**(§7.6). 파일만 있고 인덱스 미적재 → `orphaned`.

---

## 4. 버전·호환성 규칙

| 대상 | 버전 필드 | 규칙 |
|---|---|---|
| 정규화 규칙 | `canonicalization_version` | 변경 시 **새 해시 버전.** 기존과 섞지 않는다 |
| 임베딩 프로파일 | `profile_id` | 모델·차원·정규화·task prefix 변경 시 **새 프로파일 + 새 인덱스**. 병합 금지 |
| 상태 직렬화 | `state_version` | 필드 추가 시 **추가로만** 한다. 제거·의미 변경은 새 major |
| 구조 | `structure_version` | §7.5.1 게이트 4조건으로 bump |
| 규칙 | `rule_version` | 모든 `InferenceTrace`에 기록. **비어 있을 수 없다** |
| 상태 해시 | `hash_version` | 규칙 변경 시 기존 해시와 섞지 않는다 |
| SQLite 스키마 | `schema_version` | `PRAGMA user_version`에 기록. 마이그레이션은 Phase 2에서 정의 |

**하위 호환 규칙**: 필드 추가는 항상 허용(기존 레코드는 `null`/기본값). 필드 제거·타입 변경·의미 변경은 금지하며, 새 `state_version`/`schema_version`으로 분리한다.

---

## 5. Phase 2에서 완료할 것

- [ ] 위 DDL 전체를 실제 SQLite로 생성하고 **pragma user_version** 기록
- [ ] 각 스키마의 Python dataclass 구현 (`context/schemas.py`)
- [ ] 필드별 권한을 **코드로 강제** (수정 권한 밖 필드 변경 시도 → 오류)
- [ ] `upsert_*` 멱등성 + **실패 시 예외 대신 실패 레코드 반환** (§9)
- [ ] `canonicalization_version` / `state_version` / `schema_version` 관리
- [ ] `lineage.source_group_id` 자동 파생 (`provenance.source_id` 기준)
- [ ] `node_type` 결정 규칙 구현 (ADR-0008 표)
- [ ] `direction` 파생 규칙 구현 + **counters 2경로 전용 검사** (회귀 테스트 등록)
- [ ] `confidence_components` 독립 개수 계산이 §11.1·§8.12와 **동일한 값**을 내는지 검사
