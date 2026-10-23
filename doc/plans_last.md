# Semantic Cognition Core — 통합 계획 (plans_last.md)

**버전**: 1.1-last
**작성일**: 2026-10-14
**기반**: `docs/plans_combined.md` (1.0-combined) + 합의 수정안 A–E 반영
**상태**: 아키텍처 방향 합의. Phase 0 구현 전 기존 명세·실제 API·실험 설계 확인 필요
**중요**: 이 문서는 계획 문서다. 목표 수치와 미실행 실험을 측정 결과로 취급하지 않는다.

---

## 0. 두 원본 비교 및 통합 원칙

| 측면 | `plans.md` 강점 | `plans_gpt.md` 강점 | 통합본 선택 |
|------|----------------|-------------------|------------|
| 큰 그림 | ASCII 아키텍처 다이어그램 명확 | 텍스트 플로우만 | **plans.md 다이어그램 채택** |
| 가설 | 없음 (확정처럼 서술) | H1–H6 명시 + "검증 대상, 입증 아님" | **H1–H6 채택** |
| 수치 (768/256/1024, N=3, 100×100) | 구체적, 실행 용이 | "목표치, 실측 전, 임의 가정 금지" | **수치는 목표치로 유지 + Phase 0 검증 의무** |
| 실험 엄밀성 | 보통 | learn=False, 단일-단계 기준선, 저장≠활용 구분 | **gpt 엄밀성 채택** |
| 미결정 관리 | 없음 | 10개 항목 + 임의 확정 금지 | **채택** |
| 기존 결과 | 표 정리 | 표 + 교훈 컬럼 + 원본 대조 요구 | **교훈 컬럼 채택** |
| 파일 구조 | `modules/semantic_core` 중심 | `input_adapters/`, `output_adapters/` 분리 | **분리 구조 채택 (모듈성)** |
| 작업 규칙 | 없음 | 15개 규칙 (venv, 덮어쓰기 금지 등) | **채택** |

어느 하나가 절대적으로 낫지 않다. `plans.md`는 실행 속도, `plans_gpt.md`는 안전·재현성이 강점이다. 통합본은 실행력을 유지하면서 검증을 강제한다.

---

## 1. 목적과 사용 방법

Nemotron 3 / Muse Spark 세션에서 논의한 연구 방향, 확정 설계, 기존 PoC 관찰 결과, 향후 구현 순서를 하나의 기준 문서로 묶는다. 다른 모델이 구현을 이어받아도 방향을 임의로 바꾸지 않고 실제 테스트로 검증하는 것이 목적이다.

### 구현 시작 전 필수 절차

1. `report.md`, `poc/README.md`, `research_spec.md`를 이 순서로 읽는다.
2. `research_spec.md`가 없으면 추측으로 대체하지 말고 사용자에게 요청한다.
3. 이 문서와 기존 명세 사이 충돌을 확인한다.
4. 기존 코드·파일·결과·의존성을 조사한다 (특히 Experiment 02 Peripheral-Foveal 코드).
5. Phase 0부터 순서대로 진행하고 단계를 건너뛰지 않는다.
6. 각 단계는 코드, 테스트, 실제 실행, `results.json`, `REPORT.md`까지 마친 후 다음으로 넘어간다.

신규 검증 구현은 `src/` 폴더 안에서 수행한다. `poc/`는 참조용으로 보존하며 삭제·이동하지 않는다.

---

## 2. 연구 목적과 핵심 가설 (검증 대상)

장기 목적: 사전 학습 멀티모달 임베딩을 고정 입력 표현 계층으로 사용하고, 그 위에 학습 가능한 추상화 계층, 재귀 잠재 처리, 의미 메모리, 경험 축적, 점진 학습, 출력 어댑터를 구성한다. 외부 LLM/이미지 생성 모델 자체를 새로 만드는 것이 아니라, 공통 잠재 표현 변환 + 내부 상태 정제 + 경험 저장·검색 + 외부 모델 입력 변환 **코프로세서와 인터페이스**를 설계한다.

- **H1 — 이중 잠재 표현**: `z_raw` + 학습 가능 `z_abs` 병용이 구체 특징과 추상 개념을 동시에 활용하게 한다.
- **H2 — 의미 메모리**: 두 표현을 ANN 메모리에 저장·검색하면 과거 경험을 새 입력에 재사용할 수 있다.
- **H3 — 재귀적 정제**: 반복 정제 + halting head가 난이도별 계산량을 조절한다.
- **H4 — Peripheral-Foveal**: 전체 저해상도 + 원본 기반 고해상도 초점을 병용하면 정보 유지 + 세부 효율을 달성한다.
- **H5 — 지속적 경험 축적**: 에피소드 메모리 + 빠른 가소성 + 느린 통합이 새 입력 성능을 높이면서 망각을 제한한다.
- **H6 — 모달리티 인터페이스**: 이종 입력을 공통 공간에 연결하고 출력 잠상을 외부 입력 형식으로 바꿀 수 있다.

위 항목은 검증 대상이며 입증된 결론이 아니다.

---

## 3. 큰 그림 (확정본)

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEMANTIC COGNITION CORE                       │
│                                                                  │
│  ┌──────────────┐    ┌─────────────────────┐    ┌────────────┐  │
│  │ INPUT ADAPTER│    │    CORE ENGINE      │    │OUTPUT ADAPT│  │
│  │ (Trainable)  │    │  (Trainable)        │    │ (Trainable)│  │
│  │              │    │                     │    │            │  │
│  │ • TextChunk  │───►│  Z₀ → Z₁ → ... → Zₙ │───►│ • Text→LLM │  │
│  │ • ImgMultiSc │    │  (Recursive Cell)   │    │ • Img→Diff │  │
│  │ • AudSeg     │    │  + Halting Head     │    │ • Aud→TTS  │  │
│  │ • ArrTokenize│    │                     │    │ • Act→Pol  │  │
│  │              │    │  ┌───────────────┐  │    │            │  │
│  │ →embeddinggemma2 │    │  │ SEMANTIC MEM  │  │    │ →Ext API   │  │
│  │   (FROZEN)   │    │  │ (HNSW + Paging)│  │    │            │  │
│  └──────────────┘    │  └───────────────┘  │    └────────────┘  │
│       │              └─────────────────────┘         │          │
│       ▼                                             ▼          │
│  [Any Modality]                              [External Models] │
│  Any Size/Shape                              (Black Box)       │
└─────────────────────────────────────────────────────────────────┘
```

### 3.1 설계 원칙

| 원칙 | 내용 |
|------|------|
| Single Core | 학습/최적화 대상은 재귀적 잠재 사고기 하나 |
| Input Adapters | 임의 모달리티/크기 → embeddinggemma2 입력 포맷으로 정규화 |
| embeddinggemma2 고정 | Freeze. Adapter 뒤 LoRA 1–2층만 허용 |
| Dual-Stream | `z_raw` (목표 768-d) + `z_abs` (목표 256-d) |
| Output Adapters | Latent → 외부 API 입력만 변환. 외부 모델 Black Box |
| Online Learning | 추론 시점 학습. 경험 → 추상 임베딩 → 축적 → 파라미터 영속화 |

### 3.2 범위

포함: 입력 어댑터(전 모달리티), embeddinggemma2 래퍼, 추상화 프로젝터, 이중 스트림 재귀 코어, halting head, HNSW Hot/Warm/Cold + 페이징, 에피소드 메모리 + 빠른 가소성 + replay + EWC, 출력 어댑터, 체크포인트·로그·평가.

미포함: 외부 LLM/이미지/음성 모델 구현·파인튜닝, Multi-Hash/Hash bucket/Hash paging, embeddinggemma2 본체 파인튜닝, 긴 임베딩 시퀀스를 코어 입력으로 쓰는 설계.

---

## 4. 확정 아키텍처 결정 (목표치 + 검증 의무)

| 항목 | 결정 (목표) | 주의사항 |
|------|-------------|----------|
| 기본 임베더 | embeddinggemma2 고정 | 출력 차원·API·배치 형식만 Phase 0에서 확인 (이미지·음성·텍스트 지원은 전제) |
| 원본 표현 | `z_raw`, 목표 768-d | 실제 출력 확인, 임의 절단/reshape 금지 |
| 추상 표현 | `z_abs`, 목표 256-d, 2-layer MLP | 차원 축소≠자동 추상화. 유용성 별도 평가 |
| 결합 | `[z_raw, z_abs]`, 목표 1024-d | 실제 차원과 일치 테스트 |
| 메모리 인덱스 | HNSW ANN (Hash 사용 안 함) | FAISS/hnswlib 중 환경 호환 확인 후 선택 |
| 메모리 계층 | Hot/Warm/Cold | 중요도·최근성·모달리티 기반 |
| 재귀 입력 | 단일 벡터 + 내부 상태 갱신 | 긴 시퀀스를 코어 입력으로 삼지 않음 |
| 재귀 종료 | Halting Head + Watchdog | calibration 평가, Min/Max 단계 강제 |
| 시각 입력 | Peripheral-Foveal 포함 | 주변부 저해상도 + 원본 기반 crop |
| Fovea | 100×100 기준, N=3 방향 | 해상도·좌표·경계 처리 검증 필요 |
| 지속 학습 | Fast + Episodic + Slow | 각 수준 독립 검증 후 통합 |
| 출력 | Latent → 외부 입력 어댑터 | 외부 생성 모델 자체는 범위 밖 |

### 4.1 Hash 제거

Multi-Hash Indexing 제안은 최종적으로 제거, HNSW/ANN 채택으로 확정. Hash Bucket Embedding, Hash-based Paging은 구현하지 않는다.

### 4.2 추상 임베딩 방식

`z_raw`를 임베더에 다시 넣지 않는다 (임베딩 벡터는 원문 입력 형식이 아님).

```
원시 입력 → Input Adapter → Frozen embeddinggemma2 → z_raw
  → AbstractionProjector (MLP) → z_abs
Core input = concat(z_raw, z_abs)
```

---

## 5. 전체 데이터 흐름

```
[텍스트/이미지/음성/기타 배열] → [Universal Input Adapter]
  → [Frozen embeddinggemma2] → z_raw → [AbstractionProjector] → z_abs
  → [Dual-Stream Recursive Core: 상태 갱신 + 메모리 검색 + 정제]
  → [Halting Head] → [Semantic/Episodic Memory: HNSW + Hot/Warm/Cold]
  → z_combined → [Universal Output Adapter] → Text/Image/Audio/Action용 외부 입력
```

---

## 6. 입력 어댑터

공통 계약: `forward(raw_input) → (z_raw [B, embedding_dim], metadata dict)`. `embedding_dim` 목표 768-d이나 실측 확인 전까지 가정하지 않는다.

- **텍스트**: 긴 텍스트 청킹(예: 512 tokens, overlap 50). 청크별 위치·순서 메타데이터 보존. 집계 방식(평균/가중/학습형) 분리 비교 + 정보 손실 평가.
- **이미지 (Peripheral-Foveal)**: 원본 보존. 전체 → 평균 풀링 저해상도(예: 112×112 → 28×28). 주변부에서 전역 특징+초점 후보 → 원본에서 직접 crop (100×100 × 최대 3개) → 전역·초점·좌표 결합 → 임베더 입력 형식으로 변환. **3채널 의도**(원본 유지, 초점은 원본, 배경은 평균 풀링)는 보존하되, 각 채널 정의와 Experiment 02 실제 처리 방식을 확인 후 명세 고정. 임의 재정의 금지. 이미지 직접 지원은 전제하고, Phase 0에서는 API 입력 형식·출력 차원만 확인. 최소 입력 규칙: 한 변이 100 미만이면 100×100으로 업스케일하고 이미지 밖 영역은 0으로 채운다. 검은색·패딩 구분용 마스크는 두지 않는다.
- **음성**: 샘플레이트·채널·길이 메타데이터 기록. 구간 분할 모듈화. 음성 직접 지원은 전제하고, Phase 0에서는 API 입력 형식·출력 차원만 확인.
- **임의 배열**: shape/dtype/차원/의미 메타데이터 보존. Perceiver-style 후보이나 최소 검증 사례 먼저 정의.

전체/부분 군집: 전체 고양이 vs 부분(얼굴만)은 공통 인코더 + Foveal 불변성 + 학습 Fusion + 사전학습 정렬 덕분에 같은 클러스터 기대. Phase 3에서 클래스 내 거리 < 클래스 간 × 0.5 + t-SNE로 검증 (가정이 아닌 측정).

---

## 7. 추상화 프로젝터와 재귀 코어

- **AbstractionProjector**: 학습 가능 2-layer MLP. Freeze 임베더. 학습 신호(분류·검색·재구성·대조)는 별도 실험 비교. Phase 0에서는 초기화/차원/forward 검증만, 학습은 Phase 1.
- **Dual-Stream Core**: 초기 `concat(z_raw, z_abs)`, 게이팅은 대안 실험. 매 단계 내부 상태 갱신 + 검색 경험 활용 정제. 기존 Stage 3은 개선 미미 + calibration 문제였으므로 재귀 이득 가정 금지, 단일 단계 기준선과 반드시 비교.
- **Halting Head**: 단계별 종료 확률 + Min/Max 단계. 정확도·평균 단계·지연·조기/지연 종료 측정. 신뢰도 보정 나쁘면 임계값만으로 채택하지 않음.
- **Watchdog**: Latency Budget 초과 시 시스템 레벨 강제 중단 + 상태 저장/복구.

### 7.1 재귀 코어와 메모리 검색 정책 비교

다음 세 조건을 비교한다.

- A. Memory Disabled: 메모리 검색 없이 재귀 코어만 사용한다.
- B. Single Retrieval: 지정된 단계에서 메모리를 한 번 검색한다.
- C. Repeated Retrieval: 여러 재귀 단계에서 메모리를 반복 검색한다.

공통 평가 항목: 작업 성능, 평균 재귀 단계 수, 메모리 검색 횟수, 검색 지연 시간, 전체 추론 지연 시간, 메모리 사용량.

동일한 데이터 분할과 평가 조건을 사용한다. 검색 결과의 저장 여부와 실제 추론 활용 여부를 trace에 기록한다.

세 조건의 비교는 재귀 처리와 메모리 활용의 효과를 분리하기 위한 실험이며, 반복 검색이 더 우수하다고 사전 가정하지 않는다.

HNSW 메모리 미구현 시 Phase 1에서는 검색 인터페이스를 모의 구현해 정책만 검증하고, Phase 2에서 실제 ANN 검색으로 다시 평가한다. 모의 구현 결과를 실제 메모리 검색 성능으로 보고하지 않는다. Phase 1 REPORT에는 모의 결과임을 "참고용, 실측 아님"으로 표기한다.

---

## 8. 의미 메모리와 페이징

- `z_raw`, `z_abs` 함께 저장 (목표 결합 1024-d, 실측 확인).
- HNSW ANN k-NN + 메타데이터(출처·시간·모달리티·중요도·신뢰도·검증상태) + importance/recency/modality 필터.
- Hot(최근/빈번) / Warm(중간) / Cold(장기·저빈도). 중요도·최근성·빈도로 승격·강등. 페이징 단위는 벡터 청크+메타데이터 (해시 버킷 아님).
- 저장 항목: 직렬화 벡터, 원본/에피소드 참조 ID, 모달리티·작업 유형, 생성·최근 접근 시점, 중요도·신뢰도·검증상태, trace·reward, 버전·체크포인트 연결. 미검증 추측을 정답과 동등 신뢰도로 저장 금지.

---

## 9. 온라인 평생 학습 (독립 검증 후 통합)

- **L1 Fast Plasticity**: Cell 일부에 제한적 빠른 가중치. Hebbian + neuromodulator(RPE 등 정의 명확히). clipping/bounded/EMA/baseline으로 안정화. 기준선 대비 성능·안정성 비교.
- **L2 Episodic Memory**: 입력→종료 에피소드 저장 (z_raw/z_abs/상태/halt/행동/reward/맥락/중요도). 대표 z_abs 대안(평균/마지막) 비교. 중요도 스코어러 + prioritized replay. 검색 사용 여부를 trace에 기록.
- **L3 Slow Consolidation**: Replay + EWC (Fisher + 기준 파라미터). fast 일부를 slow에 통합. 체크포인트에 모델·optimizer·plasticity·EWC·메모리 메타 포함. 백그라운드 학습이 추론 방해 않는지 평가.
- **학습/평가 분리**: 평가 `learn=False`로 테스트 누출 방지. 학습 전후 + 망각 분리 측정. "저장했다"≠"활용해 개선했다" — 검색 사용 여부 대조로 입증.

---

## 10. 출력 어댑터

`output_adapters/base.py`, `text/image/audio/action_adapter.py`. 계약: `forward(z_combined) → 외부 입력 dict`, `target_modality` 속성. 외부 프롬프트·토큰·조건·메타데이터 구성. 외부 모델 선택·이미지 해상도·행동 표현은 미결정이므로 임의 확정 금지.

학습: Distillation + Task-driven + Contrastive 후보.

출력 어댑터는 외부 모델이 요구하는 입력 계약에 맞는 표현, 조건, 토큰 또는 메타데이터를 생성한다.

평가는 다음 두 항목을 분리한다.

1. 계약 적합성: 출력 형식, 차원, 타입 및 필수 필드가 유효한가?
2. 작업 성능: 출력 어댑터를 사용했을 때 외부 모델의 실제 작업 결과가 기준선보다 개선되는가?

학습을 수행할 경우 정답 데이터 또는 명시적인 학습 신호를 정의한다. 학습 데이터, 손실 함수, 평가 데이터 및 기준선을 기록한다.

유효한 외부 입력을 생성했다는 사실만으로 외부 모델의 결과 품질이 향상되었다고 판단하지 않는다.

---

## 11. 단계별 실행 계획

`src/` 내 각 스테이지 폴더에 본체·테스트·`results.json`·`REPORT.md`. 기존 명명 규칙 조사 후 충돌 없이 생성. `poc/`는 참조용으로 보존한다.

- **Phase 0**: 임베더 실체 확인 → 텍스트 경로 → Peripheral-Foveal 전처리 → 원본/주변부/crop/좌표/3채널 규약 확인 → 실제 API 이미지 임베딩 검증 → Projector shape 테스트 → 100 미만 소형 입력의 업스케일+0패드 경로 테스트 → 음성 경로 확인 + 배열은 미지원/미검증 명시 → 시간·메모리·품질 기록. 완료: 계약 테스트 + 지원/미지원 문서화 + 실측 기록.
- **Phase 1**: Cell + 결합 + 상태 갱신 + Halting + Plasticity + 단일 기준선 비교 + learn/eval 분리. 완료: 기준선 대비 이득/손실 보고.
- **Phase 2**: HNSW + Hot/Warm/Cold + 3종 검색 + 필터 + paging + Episodic/replay. Recall@K·latency·저장량 측정. 완료: 검색·저장·복원 실측 보고.
- **Phase 2.5**: EWC/Fisher + Checkpoint 버전관리 + Replay/consolidation + Split/Permuted MNIST 망각. 완료: 전후 성능·망각 + 저장/복원 검증.
- **Phase 3**: Imagenette-224 → ImageNet-100 단계 평가. 원본 기반 Peripheral-Foveal + 100×100 N=3 + 해상도/종횡비 테스트 + 전역 기준선 비교 + multi-object/distractor (Multi-MNIST 합성 대조 유지). 완료: 원본 crop 테스트 확인 + 실측 차이 보고.
- **Phase 4**: 계약 확정 + 4종 변환기 + 계약 테스트 + 합성/distillation 평가.
- **Phase 5**: 전체 통합 + learn/eval 분리 + trace/검색 기록 + 전후 성능 + 망각 + 체크포인트 + 지연/메모리/실패율. 완료: 로그·results.json·테스트로 입증.

---

## 12. 기존 PoC 결과와 교훈 (새 결과가 아님)

| 기존 실험 | 보고된 결과 | 교훈 |
|-----------|-------------|------|
| MNIST 기준선 | 전체 ~0.986, 14px 가림 ~0.317, 20px ~0.179 | 전체 vs 가림 별도 측정 |
| 재임베딩 | D 가림 ~0.826, R@1 ~0.816 | MNIST 범위 결과로 한정 |
| 메모리 | exemplar≈분류기, 그래프 이득 없음·지연 3.36배 | 그래프 기본 제외, ANN 기준선부터 |
| 재귀 | 큰 개선 없음, calibration 문제 | 단일 기준선 필수 비교 |
| 지속 학습 | finetune 망각, 메모리 일부 완화 | 신구 성능 함께 측정 |
| RL | Q<규칙, 답 회피 | RL 필수 제외 |
| LLM | qwen3:8b 추종, 개선 없음·비용 큼 | LLM은 외부 인터페이스로 제한 |
| 멀티모달 | 합성에서 높으나 일반화 불가 | 합성≠실제 이해 |
| 코프로세서 | 0.858→0.8587, 수정률 0.9% | 복잡도 대비 이득 검증 필요 |
| CLIP zero-shot | Top-1 0.5803, Top-5 0.9591 | 도메인 격차 분리 해석 |

원본 results.json 대조 없이 조건 추정·변경 금지. CIFAR-100은 당시 다운로드 미완료, 현 상태 확인 후 추정 금지.

---

## 13. 평가 목표 수치 (제안, 실측 아님)

Phase 0 Recall@1 ≥0.95, Phase 1 가림 ≥0.90·평균 ≤4단계, Phase 2 Forgetting ≤0.05·page-in ≤5ms, Phase 3 Multi-MNIST ≥0.92·오차 ≤10px, Phase 4 mapping ≤0.1, Phase 5 E2E 통과 — 전부 목표 제안. 현실성·측정법을 각 Phase에서 검토, 불가하면 임의 변경 말고 근거+대안 보고.

이미지 평가 데이터셋(확정): Phase 0–1은 Imagenette-224(10클래스), Phase 2–3은 ImageNet-100으로 확장, Full ImageNet-1K는 Phase 5에서 선택. 224 입력은 Peripheral 112(절반 다운샘플)·Fovea 100 비율에 부합한다. 1K 기준 Recall 목표는 잠정 취급하고 subset 확정 후 재검토한다. MNIST는 샌리티 체크용으로만 유지한다.

### 13.1 평가 프로토콜

모든 주요 평가에는 다음 네 항목을 명시한다.

1. 데이터셋 및 데이터 분할
2. 비교 기준선
3. 측정 지표 및 계산 방법
4. 난수 시드와 반복 조건

데이터셋과 측정법이 정의되지 않은 목표 수치는 잠정 목표로만 취급한다. 평가 조건을 확정하기 전에는 달성 여부를 판정하지 않는다.

### 13.2 의미 메모리 평가

- 평가 데이터: 학습 데이터와 분리된 질의·정답 쌍
- 기준선: 메모리를 사용하지 않는 조건 및 적절한 임베딩 검색 기준선
- 주요 지표: Recall@1, Recall@5, 검색 지연 시간
- 보조 지표: 메모리 사용량, 검색 실패율
- 재현성: 고정된 데이터 분할과 기록된 난수 시드

데이터셋과 정답 정의는 해당 실험의 실행 전에 확정한다.

---

## 14. 파일 구조

```
src/
├── requirements.txt
├── semantic_core/ (embedder, abstraction_projector, recursive_cell,
│   halting_head, memory_hierarchy, memory_paging, plasticity,
│   episodic_memory, consolidation, decoder, coprocessor_v2)
├── input_adapters/ (text/image/audio/array_adapter)
├── output_adapters/ (base/text/image/audio/action_adapter)
└── stages/ (stage0_input / stage1_core / stage2_memory /
    stage2b_continual / stage3_vision / stage4_output / stage5_integration)
```

신규 검증 구현은 `src/` 안에서 수행한다. 기존 `poc/` (Stage 0–7 코드·결과)는 참조용으로 보존한다. archive 이동은 제안일 뿐. 참조·기록 확인 후 승인 없이 이동·삭제 금지. 보존 + `src/` 별도 경로 시작 우선.

---

## 15. 필수 작업 규칙

1. report.md → poc/README.md → research_spec.md 순으로 읽기 (없으면 요청).
2. `poc/.venv`만 사용. 시스템 CUDA/드라이버/Ollama 변경 금지.
3. 의존성은 venv pip만, sudo 금지. 전역 설정 변경 시 백업 후 승인 하에.
4. 각 폴더에 본체·테스트·results.json·REPORT.md.
5. results.json은 실제 실행으로만 생성, 수치 창작 금지. 미실행은 미실행 표기.
6. 기존 결과 덮어쓰기 금지. 검증 전 다음 단계 금지. 작고 단계적 변경.
7. 오류당 최대 3회 수정, 이후 로그 기록 중단.
8. 단위 테스트 통과≠가설 입증.

---

## 16. 인터페이스 계약 초안

```python
class InputAdapter:
    def forward(self, raw_input):  # → z_raw, metadata
        raise NotImplementedError

class RecursiveCore:
    def init_state(self, z_raw): raise NotImplementedError
    def step(self, z_raw, state):  # → new_state, halt_prob, z_abs
        raise NotImplementedError
    def plasticity_step(self, pre, post, neuromodulator):
        raise NotImplementedError

class SemanticMemory:
    def write(self, key, value, metadata): raise NotImplementedError
    def read(self, query, k, filters=None): raise NotImplementedError
    def page_in(self, ids, target_level): raise NotImplementedError

class OutputAdapter:
    def forward(self, z_combined): raise NotImplementedError
    @property
    def target_modality(self): raise NotImplementedError
```

체크포인트: 모델·optimizer·plasticity·EWC/Fisher·메모리 메타·버전. ANN 직렬화는 선택 라이브러리 API 기준.

### 16.1 차원 및 데이터 계약

- 768-d(`z_raw`), 256-d(`z_abs`), 1024-d(`z_combined`)는 설계 목표이며 실측 결과가 아니다.
- Phase 0에서 임베더의 실제 출력 차원, API 입출력 형식, 배치 처리 방식을 확인한다.
- 임베딩의 정규화 여부, dtype, device, 유사도 측정법을 기록한다.
- `z_raw`의 실제 차원이 목표와 다르면 임의 절단, 패딩, reshape로 차원을 맞추지 않는다.
- 차원 불일치 시 인터페이스 수정 또는 대체 방안을 검토하고, 변경 근거와 영향을 기록한다.
- `z_abs`는 실제 `z_raw` 차원을 기준으로 입력 계약을 정의한다.
- `z_combined`의 차원은 실제 `z_raw`와 `z_abs`의 차원 합과 일치해야 한다.
- 차원 계약은 단위 테스트로 검증한다.

---

## 17. 리스크와 완화

임베더 불일치→Phase 0 실측 확인. z_abs 무용→z_raw 기준선 분리 비교. 재귀 무익→단일 대비 정확도·지연 비교. halting 부정확→calibration+Min/Max. 메모리 팽창→용량 제한+pruning. 망각→EWC+replay+재평가. plasticity 발산→clip/bound. 누출→learn=False 분리. 좌표 오류→다해상도/경계 테스트. 출력 불일치→계약 테스트. 유실→보존+승인 후 이동.

### 17.1 최종 평가 능력 분리표

| 능력 | 검증 대상 | 필수 검증 |
|------|-----------|----------|
| 표현 | 입력의 유용한 표현 생성 | 임베딩 품질 및 기준선 비교 |
| 추상화 | 유용한 개념 구조 학습 | 학습 전후 및 차원 축소 기준선 비교 |
| 재귀 처리 | 추가 계산으로 성능 향상 | 단일 단계 대비 성능·지연 비교 |
| 저장 | 경험의 지속적인 보존 | 저장·복원 무결성 |
| 검색 | 관련 경험의 회수 | Recall@K 및 지연 시간 |
| 활용 | 검색 결과가 문제 해결에 기여 | 검색 미사용 조건과 성능 비교 |
| 지속 학습 | 새로운 경험으로 성능 향상 | 학습 전후 평가 |
| 망각 방지 | 기존 작업 성능 유지 | 신구 작업 성능과 망각 측정 |
| 출력 변환 | 외부 모델 입력 계약 충족 | 계약 테스트 |
| 외부 작업 성능 | 실제 결과 품질 향상 | 출력 어댑터 미사용 기준선과 비교 |

핵심: 저장 ≠ 검색 ≠ 활용. 저장 성공이 검색 성공이 아니고, 검색 성공이 문제 해결 능력 향상이 아니다. 각각 독립 검증한다.

---

## 18. 미결정 사항 (임의 확정 금지)

1. embeddinggemma2 모델명/API·출력 차원·배치 형식
2. 3채널 정확한 의미 + Exp02 구현 방식
3. z_abs 학습 목표·loss
4. FAISS vs hnswlib
5. Hot/Warm/Cold 매체·용량·승강 기준
6. Halting 학습 목표·종료 정답
7. 온라인 정답·보상·피드백 신뢰 기준
8. 출력별 외부 입력 계약
9. 기존 폴더 아카이브 여부
10. 목표 수치 현실성·측정 프로토콜

구현 정확도에 영향 시 코드 전 보고. 외에는 미결정 기록 유지.

---

## 19. 다음 실행 순서 + 구현 모델 첫 지시

1. 필수 문서·규칙 읽기 → 2. poc 구조·결과·Exp02 코드 조사 → 3. 임베더 API·출력 차원·배치 형식 확인 → 4. Phase 0 필요 미결정만 질문 → 5. 최소 범위 정의 후 `src/`에 구현 → 6-9. 코드·테스트 분리 작성→실행→results.json→REPORT.md → 10. 완료 후 Phase 1.

> 먼저 report.md, poc/README.md, research_spec.md를 순서대로 읽어. plans_last.md와 기존 구현을 비교해. 충돌·필수 정보 누락이면 구현 전 보고해. 문제없으면 Phase 0부터 `src/` 안에 시작해. 기존 결과 덮어쓰기 금지, poc/.venv만 사용, 시스템 변경 금지. 각 실험은 코드·테스트·실측 results.json·REPORT.md 필수. 미실측 수치 금지, 검증 전 다음 단계 금지, 오류당 3회 제한.
