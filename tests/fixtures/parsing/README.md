# 파싱 라벨 데이터셋 (fixtures)

> **여기가 라벨 세트의 유일한 위치다.** `src/` 하위 `fixtures/` 디렉터리를 만들지 않는다 (ADR-0005).

| 파일 | 규모 | Phase | 사용 범위 |
|---|---|---|---|
| `parsing_dev.jsonl` | **52** (`base` 27 + `adversarial` 15 + `out_of_scope` 10) | 3 | 규칙 개발·회귀 |
| `parsing_holdout.jsonl` | **20** (`holdout`) | 3 | **규칙에 노출 금지.** §15.5 일반화 + Phase 10 전용 |

**현재 상태: 0 / 72 (미작성)**

라벨의 `split`(파일 단위 dev/holdout)과 **`subset`**(문장 역할 base/adversarial/out_of_scope/holdout)은 서로 다른 축이다. `subset`은 **문군(cluster) 단위로** 배정한다 — 같은 문군의 패러프레이즈가 서로 다른 `subset`에 들어가지 않는다.

---

## 1. 라벨 스키마

**정식 정의는 [`docs/EXPERIMENT_PROTOCOL.md` §2.2](../../../docs/EXPERIMENT_PROTOCOL.md).** 여기에는 스키마를 복제하지 않는다.

JSONL, 1문장 1행. 필드: `id`, `text`, `split`, `pattern_id`, `parse_status`, `relations[]`, `nodes[]`, `notes`.

---

## 2. 문형 ID (9개 행 — 고정)

`pattern_id`는 아래 9개 중 하나여야 한다. **새 행을 추가하지 않는다** (§19 #13). 새 행 추가는 구조 변경이다 (§7.5.1).

| `pattern_id` | 술어 | 양태 | 극성 | 예시 |
|---|---|---|---|---|
| `is_a_typical` | `is_a` | typical | positive | 고양이는 **동물**이다 |
| `is_a_all` | `is_a` | all | positive | **모든** 새는 날 수 있다 |
| `is_a_some` | `is_a` | some | positive | **일부** 새는 날 수 있다 |
| `has_property_possible` | `has_property` | possible | positive | 개도 뾰족한 귀를 **가질 수 있다** |
| `has_property_typical` | `has_property` | typical | positive | 동물 A의 귀가 뾰족하다 |
| `is_a_negative` | `is_a` | typical | **negative** | 고양이는 동물이 **아니다** |
| `is_a_also` | `is_a` | typical | positive | 고양이**도** 동물이다 |
| `causes_instance` | `causes` | specific_instance | positive | 화재 때문에 연기가 퍼졌다 |
| `unparsable` | — | — | — | 고양이는 개보다 크다 |

**§10.2 대표 예시의 2문장은 개발 세트에 반드시 포함한다.**

```
"동물 A의 귀가 뾰족하다"          → has_property_typical, attribute "뾰족한 귀"
"개도 뾰족한 귀를 가질 수 있다"    → has_property_possible, attribute "뾰족한 귀"
```

두 문장의 `nodes[].canonical_label`은 **동일하게 `"뾰족한 귀"`**여야 한다. 이것이 §10.2 반증 관계 성립의 전제다.

---

## 3. 작성 절차 (ADR-0007)

1. **초안 작성** — 언어 모델이 라벨 초안을 제안해도 된다.
2. **사람이 검수** — **검수되지 않은 라벨은 gold가 아니다.**
3. **기록** — 검수자·검수일·검수 결과를 `data/data_manifest.yaml`의 `reviewed_by`에 남긴다.
4. **분할** — **문군(cluster) 단위** 분할. 같은 문장의 패러프레이즈가 개발 세트와 보류 세트에 **동시에** 들어가면 안 된다 (§15.1.1).

---

## 4. 주의

- **비지원 세트(`unparsable`)** 는 `relations`·`nodes`가 **빈 배열**이어야 한다.
- **`span`** 은 원문 `text` 기준 오프셋이며, 실제로 해당 부분 문자열과 일치해야 한다.
- **`nodes[].canonical_label`은 정규화된 값**이다 (예: `"귀가 뾰족하다"`가 아니라 `"뾰족한 귀"`). 정규화 규칙은 `configs/attributes.yaml`.
- **보류 세트(`parsing_holdout.jsonl`)** 는 규칙을 고칠 때 **절대 열지 않는다.** §15.5 일반화 평가는 Phase 10 전용이다.
