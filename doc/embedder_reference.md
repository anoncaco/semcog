# embeddinggemma2 레퍼런스 (embedder_reference.md)

**확인일**: 2026-10-14
**출처**: Google Gemma Cookbook 공식 노트북 4종 (`docs/embeddinggemma2/`)
- inference / multimodal / fine-tuning(text) / fine-tuning(multimodal)
- 원문: https://ai.google.dev/gemma/docs/embeddinggemma/
- 모델 카드: https://ai.google.dev/gemma/docs/embeddinggemma/model_card_2

---

## 1. 모델 실체

| 항목 | 값 |
|------|-----|
| 모델 ID | `google/embeddinggemma-2` |
| 파라미터 | 740M (텍스트+비전+오디오 전체 로드 시) |
| 지원 모달리티 | 텍스트·이미지·오디오·비디오 |
| 출력 차원 | 768-d (통합 공간, 모달리티 내·간 유사도 계산 가능) |
| 로드 방식 | `sentence-transformers`의 `SentenceTransformer` 클래스 |

## 2. Task 프롬프트 (필수)

프롬프트 없이 인코딩하면 품질이 떨어진다. 용도별 지정:

| 용도 | 프롬프트 | 비고 |
|------|----------|------|
| 검색 쿼리 | `prompt_name="Retrieval-query"` | `"task: search result \| query: "` |
| 검색 문서 | `prompt_name="Retrieval-document"` | `"title: none \| text: "`, 제목 포함 시 정확도 향상 |
| 문장 유사도 | `prompt_name="STS"` | |
| 일반 분류 | `prompt_name="Classification"` | 표준 라벨셋에만 유효 |
| 임의·도메인 라벨 | `prompt_name="Clustering"` | 커스텀 카테고리 라우팅용 |
| 크로스모달 검색 | 텍스트에만 프롬프트, 이미지는 직접 인코딩 | 컬럼별 매핑 `prompts={"anchor": ...}` |

## 3. 정밀도·메모리 주의

- **float16 비호환.** bfloat16(지원 GPU 기본) 또는 float32 사용.
- 텍스트만 쓰면 `config_kwargs={"vision_config": None, "audio_config": None}` 로 270M 절약.
- 텍스트+이미지만이면 `{"audio_config": None}`.

## 4. 입력 형식

- 이미지: `{"image": 경로|URL|PIL}`. Vision 토큰 예산 70/140/280/560/1120 중 선택 (`processing_kwargs`), 클수록 품질↑ 지연↑.
- 오디오: `{"audio": 경로|URL|파형}`, 16kHz 자동 리샘플.
- 비디오: `{"video": 경로|URL|프레임}`, 기본 1fps 비전 인코더 경유 (FPS 조절 가능).
- 혼합 입력: `{"text","image","audio","video"}` 동시 전달 또는 본문 내 `<|image|>`,`<|audio|>`,`<|video|>` 태그.

## 5. MRL (차원 축소 대안)

- 128~768 사이 임의 차원 지원, 권장 768/512/256/128.
- `truncate_dim=N` + `normalize_embeddings=True` (잘라낸 뒤 재정규화 필수).
- 128-d + `similarity_fn_name="dot"` 조합이 고속 DB용. 정규화 벡터라 dot=cosine.
- 본 프로젝트의 `z_abs`(학습형 256-d)와 별개 기준선으로 사용 가능.

## 6. 유사도 해석 주의

- 텍스트 코사인 유사도는 하한이 높다 (무관 문장도 ~0.68 사례). 임계값 설계 시 실측 분포 먼저 확인.
- 파인튜닝(contrastive triplet, `MultipleNegativesRankingLoss`)은 본 프로젝트 Freeze 원칙과 충돌하므로 기본 제외. 도메인 적응 필요 시 LoRA 1–2층만 허용 (계획서 원칙 유지).

## 7. Phase 0 체크리스트 연결

- [ ] `google/embeddinggemma-2` 로드 (전체 vs 선택 인코더) + 정밀도(bf16/fp32) 확인
- [ ] Task 프롬프트별 출력 차원·dtype·유사도 분포 기록
- [ ] 768-d 실측 확인 (16.1 차원 계약 테스트 입력)
- [ ] MRL 256-d 절단 vs 학습형 Projector 비교는 Phase 1로 이월
