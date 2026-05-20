# FastAPI AI 핵심 코드 발표 정리

> 기준 문서: `발표/README.md`의 FastAPI AI 섹션  
> 기준 프로젝트: `fastapi/workspaes/basic`  
> 방향: AI 기능별 대표 이미지 1개와 핵심 코드만 짧게 정리

## 1. FastAPI AI 서버 역할

![FastAPI AI 서버 역할](assets/fastapi-slide-08.png)

원본: `main.py`

```python
from router import product, ai, work, gallery, auction_rag

app = FastAPI(lifespan=lifespan)
app.include_router(product.router)   # 기존 상품/기본 AI 기능
app.include_router(ai.router)        # 이미지 생성, 분석, 공용 AI 엔드포인트
app.include_router(work.router)      # 작품 회귀/분류 예측
app.include_router(gallery.router)   # 예술관 유사도 추천
app.include_router(auction_rag.router) # 경매 RAG 분석
```

발표 포인트: Spring Boot가 무거운 AI 작업을 직접 처리하지 않고, FastAPI가 전담하는 구조입니다.

## 2. 이미지 생성 및 분석

![이미지 생성 및 분석 API](assets/fastapi-slide-09.png)

원본: `service/ai_service.py`

```python
async def run_image_pipeline(self, request: ImagePipelineRequest) -> ImagePipelineResponse:
    return await run_in_threadpool(self._run_image_pipeline, request)

def _run_image_pipeline(self, request: ImagePipelineRequest) -> ImagePipelineResponse:
    image_path = self._generate_image_file(  # OpenAI 이미지 모델로 생성
        prompt=request.prompt,
        size=request.size
    )
    description, cache_hit = self._analyze_image_with_cache(str(image_path))  # 생성 결과를 다시 분석
    uploaded_image = self._upload_image_to_s3(image_path)  # S3 업로드 후 key/url 확보

    return ImagePipelineResponse(
        image_path=str(image_path),
        description=description,
        image_key=uploaded_image["key"],
        image_url=uploaded_image["url"],
        file_type=uploaded_image["content_type"],
        file_size=uploaded_image["size"]
    )
```

발표 포인트: 프롬프트를 받아 이미지를 만들고, 바로 분석한 뒤, S3 key와 presigned URL까지 반환합니다.

## 3. 작품 성과 예측

![작품 성과 예측 API](assets/fastapi-slide-11.png)

원본: `service/work_service.py`

```python
async def predict_views(self, features: WorkRegressionFeatures) -> WorkRegressionResponse:
    self.load_work_regressor()  # pkl 모델과 feature 순서를 1회만 로드

    feature_dict = features.model_dump()
    values = [feature_dict[name] for name in self.work_regressor_features]  # 학습 당시 순서에 맞춤
    prediction = self.work_regressor.predict([values])

    return WorkRegressionResponse(
        predicted_views=int(prediction[0]),
        created_datetime=datetime.now(),
        updated_datetime=datetime.now()
    )
```

발표 포인트: 회귀 모델은 저장된 pkl과 feature 순서를 맞춰 예상 조회수를 계산하고, 첫 호출 이후에는 메모리 재사용으로 성능을 확보합니다.

## 4. 유사도 추천

![작품 및 갤러리 추천](assets/fastapi-slide-12.png)

원본: `service/work_recommend_service.py`

```python
work_texts = [
    f"{r['title']} {r['title']} {r['category']} {r['description']} {r['tags']}"
    for r in rows
]

tfidf_v = TfidfVectorizer(
    analyzer="char_wb",     # 한국어 토큰 분리 한계를 보완하는 문자 n-gram 방식
    ngram_range=(2, 4),
    max_features=6000,
)
tfidf_mat = tfidf_v.fit_transform(work_texts)
query_mat = tfidf_v.transform([request.content])
sim_scores = cosine_similarity(query_mat, tfidf_mat)[0]
```

발표 포인트: 제목, 설명, 태그를 하나의 텍스트로 합치고, Kiwi 기반 전처리와 TF-IDF 유사도 계산으로 추천 점수를 만듭니다.

## 5. 경매 RAG 분석

![경매 RAG 분석](assets/fastapi-slide-13.png)

원본: `service/auction_rag_service.py`

```python
async def analyze(self, request: AuctionRagAnalyzeRequest) -> AuctionRagAnalyzeResponse:
    if request.fast_mode:
        return await self._analyze_fast(request)  # RAG 검색을 생략한 빠른 분석

    rag = await self.get_rag()                  # RAGAnything 초기화 및 재사용
    image_data = self._resolve_image_base64(request)
    image_analysis = await rag.vision_model_func(
        self._build_image_prompt(request),
        image_data=image_data,
    )

    rag_query = self._build_rag_query(request, image_analysis)
    auction_report = await rag.aquery(rag_query, mode="hybrid")  # 문서 검색 + LLM 결합

    return AuctionRagAnalyzeResponse(
        image_analysis=str(image_analysis),
        auction_report=str(auction_report),
        used_rag=True,
        indexed_document_hint=str(DEFAULT_REPORT_PATH),
    )
```

발표 포인트: 빠른 모드와 RAG 모드를 분리해서, 즉시 응답과 문서 기반 정밀 분석을 모두 지원합니다.

## 발표 흐름 요약

1. FastAPI는 Spring이 호출하는 AI 전용 백엔드입니다.
2. 이미지 생성은 생성, 분석, S3 저장을 한 번에 묶어 처리합니다.
3. 작품 예측은 저장된 pkl 모델과 feature 순서를 그대로 사용합니다.
4. 추천은 텍스트 유사도 기반으로 후보를 정렬합니다.
5. 경매 RAG는 빠른 분석과 문서 기반 정밀 분석을 분리합니다.
