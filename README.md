# BIDEO 웹서비스 핵심 코드 발표 정리

> 기준 문서: `발표/README.md`  
> 기준 프로젝트: `spring/workspace/bideo`  
> 방향: 화면 이해에 필요한 대표 이미지만 남기고, 코드는 핵심 흐름만 짧게 발췌

## 1. 등록 흐름: 예술관 등록 / 작품 등록

![작품 등록과 피드](assets/slide-03.png)

### 예술관 등록

원본: `src/main/java/com/app/bideo/service/gallery/GalleryService.java`

```java
public GalleryCreateResponseDTO write(Long memberId, GalleryCreateRequestDTO requestDTO, MultipartFile coverFile) {
    Long resolvedMemberId = resolveMemberId(memberId);       // 발표용: 로그인 사용자를 확정 / 개발자용: SecurityContext principal을 정규화해서 현재 요청의 작성자 ID로 사용했습니다.
    requestDTO.setMemberId(resolvedMemberId);
    requestDTO.setCoverImage(saveCoverImage(coverFile));     // 발표용: 커버 이미지를 저장 / 개발자용: 업로드된 파일을 S3 object key로 바꿔서 DB에는 경로만 저장했습니다.

    galleryDAO.save(requestDTO);                             // 발표용: 예술관을 등록 / 개발자용: gallery aggregate root를 먼저 저장해서 PK를 확보했습니다.
    saveWorkLinks(requestDTO.getId(), requestDTO.getWorkIds(), resolvedMemberId); // 발표용: 작품을 연결 / 개발자용: N:M 관계를 relation table에 동기화했습니다.
    saveTags(requestDTO.getId(), requestDTO.getTagIds(), requestDTO.getTagNames()); // 발표용: 태그를 연결 / 개발자용: tag association을 정규화해서 저장했습니다.
    galleryDAO.updateWorkCount(requestDTO.getId());          // 발표용: 작품 수를 갱신 / 개발자용: 화면 집계를 위해 denormalized count column을 업데이트했습니다.

    return GalleryCreateResponseDTO.builder()
            .galleryId(requestDTO.getId())
            .redirectUrl("/main?relatedGalleryId=" + requestDTO.getId())
            .build();
}
```

발표 포인트: 예술관은 커버 이미지, 작품 목록, 태그를 묶어 하나의 전시 공간으로 저장하는 기능입니다.

### 작품 등록

원본: `src/main/java/com/app/bideo/service/work/WorkService.java`

```java
public WorkCreateResponseDTO write(Long memberId, WorkCreateRequestDTO requestDTO,
                                   MultipartFile mediaFile, MultipartFile thumbnailFile) {
    Long resolvedMemberId = resolveMemberId(memberId);
    Long galleryId = requireGalleryId(requestDTO.getGalleryId());
    validateGalleryOwner(galleryId, resolvedMemberId);       // 발표용: 내 예술관인지 확인 / 개발자용: gallery owner 검증을 먼저 해서 권한 없는 등록을 막았습니다.

    WorkDTO workDTO = WorkDTO.builder()
            .memberId(resolvedMemberId)
            .title(limitText(requestDTO.getTitle(), 255))
            .price(requestDTO.getPrice())
            .mediaType(resolveMediaType(requestDTO.getCategory(), mediaFile, requestDTO.getFiles()))
            .predictedViews(defaultLong(requestDTO.getPredictedViews())) // 발표용: AI 예측값을 저장 / 개발자용: null이면 기본값으로 바꿔서 inference result를 안전하게 반영했습니다.
            .status("ACTIVE")
            .build();

    workDAO.save(workDTO);                                   // 발표용: 작품을 등록 / 개발자용: work entity를 먼저 insert해서 식별자(PK)를 만들었습니다.
    saveThumbnailFile(workDTO.getId(), thumbnailFile);       // 발표용: 썸네일을 저장 / 개발자용: 썸네일 파일은 S3에 올리고 DB에는 key만 남겼습니다.
    saveMediaFile(workDTO.getId(), mediaFile, 1);            // 발표용: 작품 파일을 저장 / 개발자용: multipart 업로드 파일을 media asset으로 저장했습니다.
    saveTags(workDTO.getId(), requestDTO.getTagIds(), requestDTO.getTagNames()); // 발표용: 태그를 연결 / 개발자용: 태그는 join table로 정규화해서 저장했습니다.
    saveGalleryLink(galleryId, workDTO.getId());             // 발표용: 예술관에 연결 / 개발자용: parent-child 관계를 별도 매핑 테이블로 연결했습니다.
    saveAuctionIfRequested(workDTO.getId(), resolvedMemberId, requestDTO.getPrice(),
            requestDTO.getAuctionEnabled(), requestDTO.getAuctionStartingPrice(),
            requestDTO.getAuctionDeadlineHours());           // 발표용: 선택 시 경매 생성 / 개발자용: 경매 사용 여부에 따라 auction subdomain을 조건부로 생성했습니다.

    return WorkCreateResponseDTO.builder().id(workDTO.getId()).galleryId(galleryId).build();
}
```

발표 포인트: 작품 등록은 파일 저장, 태그 저장, 예술관 연결, AI 예측값 저장, 경매 생성까지 이어지는 핵심 시작점입니다.

## 2. 거래 흐름: 경매 / 결제

![경매와 결제](assets/slide-04.png)

### 경매 입찰

원본: `src/main/java/com/app/bideo/service/auction/BidCommandService.java`

```java
public BidResponseDTO placeBid(Long memberId, BidRequestDTO requestDTO) {
    AuctionVO auction = auctionDAO.findRawById(requestDTO.getAuctionId());

    if (!"ACTIVE".equals(auction.getStatus())) throw new IllegalStateException("종료된 경매입니다."); // 발표용: 종료 경매를 막습니다 / 개발자용: 경매 상태값이 ACTIVE인지 먼저 검사해서 잘못된 입찰을 차단했습니다.
    if (auction.getSellerId().equals(memberId)) throw new IllegalArgumentException("자신의 경매에는 입찰할 수 없습니다."); // 발표용: 자기 입찰을 막습니다 / 개발자용: sellerId와 memberId를 비교해서 self-bid를 방지했습니다.
    if (requestDTO.getBidPrice() < Math.round(auction.getCurrentPrice() * 1.1)) {
        throw new IllegalArgumentException("최소 입찰 단위 이상으로 입찰해야 합니다."); // 발표용: 최소 입찰 금액을 확인했습니다 / 개발자용: currentPrice의 10% 상승 규칙을 적용했습니다.
    }

    bidDAO.clearPreviousWinning(requestDTO.getAuctionId());  // 발표용: 이전 최고 입찰을 해제합니다 / 개발자용: 기존 winning bid 플래그를 먼저 false로 바꿨습니다.
    bidDAO.save(BidVO.builder()
            .auctionId(requestDTO.getAuctionId())
            .memberId(memberId)
            .bidPrice(requestDTO.getBidPrice())
            .isWinning(true)
            .build());

    auctionDAO.updateCurrentPrice(requestDTO.getAuctionId(), requestDTO.getBidPrice(),
            bidDAO.findBidderIds(requestDTO.getAuctionId()).size()); // 발표용: 현재가를 갱신합니다 / 개발자용: currentPrice와 bidderCount를 같이 갱신해서 경매 집계를 일관되게 유지했습니다.

    return bidDAO.findHighestBid(requestDTO.getAuctionId()).orElse(null);
}
```

발표 포인트: 입찰은 경매 상태, 판매자 본인 여부, 최소 10% 입찰 조건을 검증한 뒤 최고 입찰자를 갱신합니다.

### 결제 검증

원본: `src/main/java/com/app/bideo/service/payment/PaymentService.java`

```java
public PaymentResponseDTO confirmBootpayPayment(Long buyerId, BootpayConfirmRequestDTO requestDTO) {
    PaymentVO payment = paymentDAO.findRawById(requestDTO.getPaymentId())
            .orElseThrow(() -> new IllegalArgumentException("결제를 찾을 수 없습니다."));

    if (!payment.getBuyerId().equals(buyerId)) throw new IllegalArgumentException("결제 검증 권한이 없습니다."); // 발표용: 결제 권한을 확인합니다 / 개발자용: buyerId 비교로 ownership check를 수행했습니다.
    if (!"PENDING".equals(payment.getStatus())) throw new IllegalStateException("이미 처리된 결제입니다."); // 발표용: 중복 결제를 막습니다 / 개발자용: PENDING 상태만 처리해서 idempotency를 보장했습니다.

    JsonNode receipt = bootpayClient.getReceipt(requestDTO.getReceiptId()); // 발표용: 결제 영수증을 다시 확인합니다 / 개발자용: PG 서버에 직접 조회해서 프론트 응답을 신뢰하지 않았습니다.
    if (!payment.getPaymentCode().equals(receipt.path("order_id").asText(""))) {
        throw new IllegalStateException("부트페이 주문번호가 일치하지 않습니다.");
    }
    if (receipt.path("price").asInt(-1) != payment.getTotalPrice()) {
        throw new IllegalStateException("부트페이 결제 금액이 일치하지 않습니다.");
    }

    return completePayment(payment.getId(), buyerId);        // 발표용: 결제를 완료합니다 / 개발자용: PENDING -> COMPLETED state transition을 수행했습니다.
}
```

발표 포인트: 프론트 결제 성공만 믿지 않고, 서버에서 Bootpay 영수증의 주문번호와 금액을 다시 확인합니다.

## 3. 상세 화면: 예술관 상세 / 작품 상세

![예술관 공모전 프로필](assets/slide-05.png)

### 예술관 상세

원본: `src/main/java/com/app/bideo/service/gallery/GalleryService.java`

```java
public GalleryDetailResponseDTO getGalleryDetail(Long id) {
    GalleryDetailResponseDTO detail = galleryDAO.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("gallery not found"));

    detail.setCoverImage(s3FileService.getPresignedUrl(detail.getCoverImage())); // 발표용: 커버 이미지를 보여줍니다 / 개발자용: DB에 저장한 S3 key를 presigned URL로 변환했습니다.
    detail.setTags(galleryDAO.findTagsByGalleryId(id));
    detail.setWorks(getGalleryWorks(id, detail.getMemberId()));

    Long memberId = resolveAuthenticatedMemberId();
    detail.setIsLiked(memberId != null && galleryDAO.existsLike(memberId, id)); // 발표용: 좋아요 상태를 보여줍니다 / 개발자용: 현재 사용자 기준 reaction state를 계산했습니다.
    detail.setIsBookmarked(memberId != null && bookmarkDAO.exists(memberId, "GALLERY", id)); // 발표용: 북마크 상태를 보여줍니다 / 개발자용: bookmark 여부를 projection으로 주입했습니다.
    detail.setIsOwner(memberId != null && memberId.equals(detail.getMemberId())); // 발표용: 작성자 여부를 보여줍니다 / 개발자용: 현재 사용자와 memberId를 비교해서 owner predicate를 계산했습니다.
    return detail;
}
```

발표 포인트: 예술관 상세는 커버, 태그, 포함 작품, 좋아요/북마크/소유자 상태를 한 번에 구성합니다.

### 작품 상세

원본: `src/main/java/com/app/bideo/service/work/WorkService.java`

```java
public WorkDetailResponseDTO getWorkDetail(Long id) {
    Long memberId = resolveAuthenticatedMemberId();
    WorkDetailResponseDTO detail = workDAO.findDetailById(id, memberId)
            .orElseThrow(() -> new IllegalArgumentException("work not found"));

    applyFileUrls(detail);                                   // 발표용: 파일 경로를 바꿉니다 / 개발자용: media asset key를 response DTO에서 쓸 URL로 변환했습니다.
    detail.setIsLiked(memberId != null && workDAO.existsLike(memberId, id)); // 발표용: 좋아요 상태를 보여줍니다 / 개발자용: 사용자 반응 상태를 projection했습니다.
    detail.setIsBookmarked(memberId != null && bookmarkDAO.exists(memberId, "WORK", id)); // 발표용: 북마크 상태를 보여줍니다 / 개발자용: bookmark 상태를 DTO에 주입했습니다.
    detail.setIsOwner(memberId != null && memberId.equals(detail.getMemberId())); // 발표용: 소유자 여부를 보여줍니다 / 개발자용: owner check를 통해 현재 사용자 소유 여부를 계산했습니다.
    detail.setHasActiveAuction(workDAO.existsActiveAuctionByWorkId(id)); // 발표용: 경매가 있는지 보여줍니다 / 개발자용: auction subview 렌더링 조건을 계산했습니다.

    return detail;
}
```

발표 포인트: 작품 상세는 미디어, 반응 상태, 소유자 여부, 활성 경매 여부를 합쳐 피드형 상세 화면을 만듭니다.

## 4. 대시보드

원본: `src/main/java/com/app/bideo/service/dashboard/DashboardService.java`

```java
@Cacheable(value = "dashboard", key = "#memberId")
public DashboardResponseDTO getDashboard(Long memberId) {
    DashboardSummaryRawDTO summary = defaultSummary(dashboardMapper.selectDashboardSummary(memberId));

    int ownedWorkCount = nullSafe(dashboardMapper.selectOwnedWorkCount(memberId));
    int ownedGalleryCount = nullSafe(dashboardMapper.selectOwnedGalleryCount(memberId));
    LocalDate endDate = LocalDate.now();
    LocalDate startDate = endDate.minusDays(MAX_LOOKBACK_DAYS - 1L);

    Map<String, DashboardMetricSeriesDTO> analyticsMetrics = new LinkedHashMap<>();
    analyticsMetrics.put("works", buildMetricSeries("works", "내가 만든 작품 추이",
            String.valueOf(ownedWorkCount),
            toDailyMap(dashboardMapper.selectDailyCreatedWorkCounts(memberId, startDate, endDate)),
            true, "내 작품", "개", endDate)); // 발표용: 작품 추이 그래프를 만듭니다 / 개발자용: 날짜별 집계를 time-series로 만들어 차트에 썼습니다.
    analyticsMetrics.put("galleries", buildMetricSeries("galleries", "내가 만든 예술관 추이",
            String.valueOf(ownedGalleryCount),
            toDailyMap(dashboardMapper.selectDailyCreatedGalleryCounts(memberId, startDate, endDate)),
            true, "내 예술관", "개", endDate)); // 발표용: 예술관 추이 그래프를 만듭니다 / 개발자용: 작품과 예술관 series를 분리해서 집계했습니다.

    return DashboardResponseDTO.builder()
            .totalViewsText(formatCompactNumber(nullSafe(summary.getTotalViews()))) // 발표용: 조회수를 보기 좋게 보여줍니다 / 개발자용: 숫자를 compact format으로 바꿔 KPI 텍스트로 사용했습니다.
            .activeAuctionsText(formatTwoDigits(nullSafe(summary.getActiveAuctionCount()))) // 발표용: 경매 수를 보여줍니다 / 개발자용: 운영 지표를 두 자리 형태로 정규화했습니다.
            .pendingPaymentsText(formatCurrencyCompact(nullSafe(summary.getPendingPaymentAmount()))) // 발표용: 결제 금액을 요약해 보여줍니다 / 개발자용: 금액을 compact currency format으로 변환했습니다.
            .analyticsMetrics(analyticsMetrics)
            .ownedWorks(withFallback(dashboardMapper.selectOwnedWorkItems(memberId), "등록된 작품이 없습니다.", "")) // 발표용: 작품이 없으면 안내 문구를 보여줍니다 / 개발자용: empty list에 fallback 메시지를 넣었습니다.
            .galleries(withFallback(dashboardMapper.selectGalleryItems(memberId), "예술관이 없습니다.", "")) // 발표용: 예술관이 없으면 안내 문구를 보여줍니다 / 개발자용: collection empty-state를 처리했습니다.
            .paymentHistory(withFallback(dashboardMapper.selectPaymentHistoryItems(memberId), "결제 이력이 없습니다.", "")) // 발표용: 결제 내역이 없으면 안내 문구를 보여줍니다 / 개발자용: history list에 fallback을 적용했습니다.
            .build();
}
```

발표 포인트: 대시보드는 조회수, 경매, 결제, 작품, 예술관 데이터를 모아 사용자의 운영 현황을 한 화면에 보여줍니다.

## 발표 흐름 요약

1. 예술관을 만들고 작품을 등록한다.
2. 등록된 작품은 상세 화면과 피드에서 소비된다.
3. 작품에 경매가 열리면 입찰이 진행된다.
4. 낙찰 또는 구매는 결제 검증을 거쳐 완료된다.
5. 대시보드에서 내 작품, 예술관, 거래 현황을 확인한다.




.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
.
................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................v
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
app.include_router(product.router)   # 발표용: 기본 AI 기능을 연결합니다 / 개발자용: legacy product domain과의 하위 호환 라우팅입니다.
app.include_router(ai.router)        # 발표용: 이미지 생성과 분석 엔드포인트를 묶었습니다 / 개발자용: image generation, vision analysis, orchestration router입니다.
app.include_router(work.router)      # 발표용: 작품 성과 예측을 연결했습니다 / 개발자용: regression, classification inference endpoint입니다.
app.include_router(gallery.router)   # 발표용: 예술관 추천을 연결했습니다 / 개발자용: content-based similarity recommendation router입니다.
app.include_router(auction_rag.router) # 발표용: 경매 분석을 연결했습니다 / 개발자용: auction domain RAG pipeline router입니다.
```

발표 포인트: Spring Boot가 무거운 AI 작업을 직접 처리하지 않고, FastAPI가 전담하는 구조입니다.

## 2. 이미지 생성 및 분석

![이미지 생성 및 분석 API](assets/fastapi-slide-09.png)

원본: `service/ai_service.py`

```python
async def run_image_pipeline(self, request: ImagePipelineRequest) -> ImagePipelineResponse:
    return await run_in_threadpool(self._run_image_pipeline, request)

def _run_image_pipeline(self, request: ImagePipelineRequest) -> ImagePipelineResponse:
    image_path = self._generate_image_file(  # 발표용: 이미지를 생성합니다 / 개발자용: OpenAI image generation model을 호출해서 local artifact를 만들었습니다.
        prompt=request.prompt,
        size=request.size
    )
    description, cache_hit = self._analyze_image_with_cache(str(image_path))  # 발표용: 생성된 이미지를 분석합니다 / 개발자용: vision inference 결과를 cache-aware하게 재사용했습니다.
    uploaded_image = self._upload_image_to_s3(image_path)  # 발표용: 이미지를 S3에 저장합니다 / 개발자용: object storage에 업로드하고 key와 presigned URL 메타데이터를 확보했습니다.

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
    self.load_work_regressor()  # 발표용: 예측 모델을 불러옵니다 / 개발자용: joblib/pkl serialized estimator를 lazy-load해서 재사용했습니다.

    feature_dict = features.model_dump()
    values = [feature_dict[name] for name in self.work_regressor_features]  # 발표용: 입력값 순서를 맞춥니다 / 개발자용: training schema order에 맞춰 feature vector를 정렬했습니다.
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
    analyzer="char_wb",     # 발표용: 한국어 유사도를 계산합니다 / 개발자용: 형태소 분해 대신 character n-gram 기반 vectorizer를 사용했습니다.
    ngram_range=(2, 4),
    max_features=6000,
)
tfidf_mat = tfidf_v.fit_transform(work_texts)
query_mat = tfidf_v.transform([request.content])
sim_scores = cosine_similarity(query_mat, tfidf_mat)[0]  # 발표용: 추천 점수를 계산합니다 / 개발자용: query-document similarity를 dense score로 산출했습니다.
```

발표 포인트: 제목, 설명, 태그를 하나의 텍스트로 합치고, Kiwi 기반 전처리와 TF-IDF 유사도 계산으로 추천 점수를 만듭니다.

## 5. 경매 RAG 분석

![경매 RAG 분석](assets/fastapi-slide-13.png)

원본: `service/auction_rag_service.py`

```python
async def analyze(self, request: AuctionRagAnalyzeRequest) -> AuctionRagAnalyzeResponse:
    if request.fast_mode:
        return await self._analyze_fast(request)  # 발표용: 빠른 분석 경로를 사용합니다 / 개발자용: retrieval step을 생략한 low-latency path입니다.

    rag = await self.get_rag()                  # 발표용: RAG 분석기를 준비합니다 / 개발자용: RAG runtime을 lazy-init하고 connection pool처럼 재사용했습니다.
    image_data = self._resolve_image_base64(request)
    image_analysis = await rag.vision_model_func(
        self._build_image_prompt(request),
        image_data=image_data,
    )

    rag_query = self._build_rag_query(request, image_analysis)
    auction_report = await rag.aquery(rag_query, mode="hybrid")  # 발표용: 문서 기반 정밀 분석을 수행합니다 / 개발자용: dense/sparse retrieval과 generation을 결합한 hybrid RAG를 사용했습니다.

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
