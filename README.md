# BIDEO 웹서비스 핵심 코드 발표 정리


##  예술관 등록 / 작품 등록

![작품 등록과 피드](assets/slide-03.png)

### 예술관 등록

원본: `src/main/java/com/app/bideo/service/gallery/GalleryService.java`

```java
public GalleryCreateResponseDTO write(Long memberId, GalleryCreateRequestDTO requestDTO, MultipartFile coverFile) {
    Long resolvedMemberId = resolveMemberId(memberId);       //  로그인 사용자를 확정 / SecurityContext principal을 정규화해서 현재 요청의 작성자 ID로 사용했습니다.
    requestDTO.setMemberId(resolvedMemberId);
    requestDTO.setCoverImage(saveCoverImage(coverFile));     //  커버 이미지를 저장 / 개발자용: 업로드된 파일을 S3 object key로 바꿔서 DB에는 경로만 저장했습니다.

    galleryDAO.save(requestDTO);                             //  예술관을 등록 / : gallery aggregate root를 먼저 저장해서 PK를 확보했습니다.
    saveWorkLinks(requestDTO.getId(), requestDTO.getWorkIds(), resolvedMemberId); // 작품을 연결 / N:M 관계를 relation table에 동기화했습니다.
    saveTags(requestDTO.getId(), requestDTO.getTagIds(), requestDTO.getTagNames()); // 태그를 연결 / tag association을 정규화해서 저장했습니다.
    galleryDAO.updateWorkCount(requestDTO.getId());          // 작품 수를 갱신 

    return GalleryCreateResponseDTO.builder()
            .galleryId(requestDTO.getId())
            .redirectUrl("/main?relatedGalleryId=" + requestDTO.getId())
            .build();
}
```

 예술관은 커버 이미지, 작품 목록, 태그를 묶어 하나의 전시 공간으로 저장하는 기능입니다.

### 작품 등록

원본: `src/main/java/com/app/bideo/service/work/WorkService.java`

```java
public WorkCreateResponseDTO write(Long memberId, WorkCreateRequestDTO requestDTO,
                                   MultipartFile mediaFile, MultipartFile thumbnailFile) {
    Long resolvedMemberId = resolveMemberId(memberId);
    Long galleryId = requireGalleryId(requestDTO.getGalleryId());
    validateGalleryOwner(galleryId, resolvedMemberId);       // 내 예술관인지 확인 / gallery owner 검증을 먼저 해서 권한 없는 등록을 막았습니다.

    WorkDTO workDTO = WorkDTO.builder()
            .memberId(resolvedMemberId)
            .title(limitText(requestDTO.getTitle(), 255))
            .price(requestDTO.getPrice())
            .mediaType(resolveMediaType(requestDTO.getCategory(), mediaFile, requestDTO.getFiles()))
            .predictedViews(defaultLong(requestDTO.getPredictedViews())) // AI 예측값을 저장 /  null이면 기본값으로 바꿔서 inference result를 안전하게 반영했습니다.
            .status("ACTIVE")
            .build();

    workDAO.save(workDTO);                                   //  작품을 등록 / work entity를 먼저 insert해서 식별자(PK)를 만들었습니다.
    saveThumbnailFile(workDTO.getId(), thumbnailFile);       // 썸네일을 저장 / 썸네일 파일은 S3에 올리고 DB에는 key만 남겼습니다.
    saveMediaFile(workDTO.getId(), mediaFile, 1);            // 작품 파일을 저장 / multipart 업로드 파일을 media asset으로 저장했습니다.
    saveTags(workDTO.getId(), requestDTO.getTagIds(), requestDTO.getTagNames()); // 태그를 연결 
    saveGalleryLink(galleryId, workDTO.getId());             // 예술관에 연결
    saveAuctionIfRequested(workDTO.getId(), resolvedMemberId, requestDTO.getPrice(),
            requestDTO.getAuctionEnabled(), requestDTO.getAuctionStartingPrice(),
            requestDTO.getAuctionDeadlineHours());           //선택 시 경매 생성 /  경매 사용 여부에 따라 auction subdomain을 조건부로 생성했습니다.

    return WorkCreateResponseDTO.builder().id(workDTO.getId()).galleryId(galleryId).build();
}
```

작품 등록은 파일 저장, 태그 저장, 예술관 연결, 경매 생성까지 이어지는 핵심 시작점입니다.

## 거래 흐름: 경매 / 결제

![경매와 결제](assets/slide-04.png)

### 경매 입찰

원본: `src/main/java/com/app/bideo/service/auction/BidCommandService.java`

```java
public BidResponseDTO placeBid(Long memberId, BidRequestDTO requestDTO) {
    AuctionVO auction = auctionDAO.findRawById(requestDTO.getAuctionId());

    if (!"ACTIVE".equals(auction.getStatus())) throw new IllegalStateException("종료된 경매입니다."); // 종료 경매를 막습니다 / 경매 상태값이 ACTIVE인지 먼저 검사해서 잘못된 입찰을 차단했습니다.
    if (auction.getSellerId().equals(memberId)) throw new IllegalArgumentException("자신의 경매에는 입찰할 수 없습니다."); // 자기 입찰을 막습니다 / sellerId와 memberId를 비교해서 self-bid를 방지했습니다.
    if (requestDTO.getBidPrice() < Math.round(auction.getCurrentPrice() * 1.1)) {
        throw new IllegalArgumentException("최소 입찰 단위 이상으로 입찰해야 합니다."); // 최소 입찰 금액을 확인했습니다 / currentPrice의 10% 상승 규칙을 적용했습니다.
    }

    bidDAO.clearPreviousWinning(requestDTO.getAuctionId());  // 기존 winning bid 플래그를 먼저 false로 바꿨습니다.
    bidDAO.save(BidVO.builder()
            .auctionId(requestDTO.getAuctionId())
            .memberId(memberId)
            .bidPrice(requestDTO.getBidPrice())
            .isWinning(true)
            .build());

    auctionDAO.updateCurrentPrice(requestDTO.getAuctionId(), requestDTO.getBidPrice(),
            bidDAO.findBidderIds(requestDTO.getAuctionId()).size()); //  현재가를 갱신합니다 / currentPrice와 bidderCount를 같이 갱신해서 경매 집계를 일관되게 유지했습니다.

    return bidDAO.findHighestBid(requestDTO.getAuctionId()).orElse(null);
}
```

 입찰은 경매 상태, 판매자 본인 여부, 최소 10% 입찰 조건을 검증한 뒤 최고 입찰자를 갱신합니다.

### 결제 검증

원본: `src/main/java/com/app/bideo/service/payment/PaymentService.java`

```java
public PaymentResponseDTO confirmBootpayPayment(Long buyerId, BootpayConfirmRequestDTO requestDTO) {
    PaymentVO payment = paymentDAO.findRawById(requestDTO.getPaymentId())
            .orElseThrow(() -> new IllegalArgumentException("결제를 찾을 수 없습니다."));

    if (!payment.getBuyerId().equals(buyerId)) throw new IllegalArgumentException("결제 검증 권한이 없습니다."); // buyerId 비교로 ownership check를 수행했습니다.
    if (!"PENDING".equals(payment.getStatus())) throw new IllegalStateException("이미 처리된 결제입니다."); //  중복 결제를 막습니다 / : PENDING 상태만 처리해서 idempotency를 보장했습니다.

    JsonNode receipt = bootpayClient.getReceipt(requestDTO.getReceiptId()); //  결제 영수증을 다시 확인합니다 /  PG 서버에 직접 조회해서 프론트 응답을 신뢰하지 않았습니다.
    if (!payment.getPaymentCode().equals(receipt.path("order_id").asText(""))) {
        throw new IllegalStateException("부트페이 주문번호가 일치하지 않습니다.");
    }
    if (receipt.path("price").asInt(-1) != payment.getTotalPrice()) {
        throw new IllegalStateException("부트페이 결제 금액이 일치하지 않습니다.");
    }

    return completePayment(payment.getId(), buyerId);        // 결제를 완료합니다 /  PENDING -> COMPLETED state transition을 수행했습니다.
}
```

 프론트 결제 성공만 믿지 않고, 서버에서 Bootpay 영수증의 주문번호와 금액을 다시 확인합니다.

## 3. 상세 화면: 예술관 상세 / 작품 상세

![예술관 공모전 프로필](assets/slide-05.png)

### 예술관 상세

원본: `src/main/java/com/app/bideo/service/gallery/GalleryService.java`

```java
public GalleryDetailResponseDTO getGalleryDetail(Long id) {
    GalleryDetailResponseDTO detail = galleryDAO.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("gallery not found"));

    detail.setCoverImage(s3FileService.getPresignedUrl(detail.getCoverImage())); //  커버 이미지를 보여줍니다 /  DB에 저장한 S3 key를 presigned URL로 변환했습니다.
    detail.setTags(galleryDAO.findTagsByGalleryId(id));
    detail.setWorks(getGalleryWorks(id, detail.getMemberId()));

    Long memberId = resolveAuthenticatedMemberId();
    detail.setIsLiked(memberId != null && galleryDAO.existsLike(memberId, id)); // 좋아요 상태를 보여줍니다 
    detail.setIsBookmarked(memberId != null && bookmarkDAO.exists(memberId, "GALLERY", id)); 
    detail.setIsOwner(memberId != null && memberId.equals(detail.getMemberId())); //  작성자 여부를 보여줍니다 / 현재 사용자와 memberId를 비교해서 owner predicate를 계산했습니다.
    return detail;
}
```

 예술관 상세는 커버, 태그, 포함 작품, 좋아요/북마크/소유자 상태를 한 번에 구성합니다.

### 작품 상세

원본: `src/main/java/com/app/bideo/service/work/WorkService.java`

```java
public WorkDetailResponseDTO getWorkDetail(Long id) {
    Long memberId = resolveAuthenticatedMemberId();
    WorkDetailResponseDTO detail = workDAO.findDetailById(id, memberId)
            .orElseThrow(() -> new IllegalArgumentException("work not found"));

    applyFileUrls(detail);                                   //  파일 경로를 바꿉니다 /  media asset key를 response DTO에서 쓸 URL로 변환했습니다.
    detail.setIsLiked(memberId != null && workDAO.existsLike(memberId, id)); //  좋아요 상태를 보여줍니다 /  사용자 반응 상태를 projection했습니다.
    detail.setIsBookmarked(memberId != null && bookmarkDAO.exists(memberId, "WORK", id)); // 북마크 상태를 보여줍니다 /  bookmark 상태를 DTO에 주입했습니다.
    detail.setIsOwner(memberId != null && memberId.equals(detail.getMemberId())); // 개발자용: owner check를 통해 현재 사용자 소유 여부를 계산했습니다.
    detail.setHasActiveAuction(workDAO.existsActiveAuctionByWorkId(id)); //  경매가 있는지 보여줍니다 /  auction subview 렌더링 조건을 계산했습니다.

    return detail;
}
```

작품 상세는 미디어, 반응 상태, 소유자 여부, 활성 경매 여부를 합쳐 피드형 상세 화면을 만듭니다.

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

 대시보드는 조회수, 경매, 결제, 작품, 예술관 데이터를 모아 사용자의 운영 현황을 한 화면에 보여줍니다.


