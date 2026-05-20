# BIDEO 웹서비스 핵심 코드 발표 정리
 
> 프로젝트: `spring/workspace/bideo`  

## 1. 등록 흐름: 예술관 등록 / 작품 등록

![작품 등록과 피드](assets/slide-03.png)

### 예술관 등록

원본: `src/main/java/com/app/bideo/service/gallery/GalleryService.java`

```java
public GalleryCreateResponseDTO write(Long memberId, GalleryCreateRequestDTO requestDTO, MultipartFile coverFile) {
    Long resolvedMemberId = resolveMemberId(memberId);       // 로그인 사용자 확정
    requestDTO.setMemberId(resolvedMemberId);
    requestDTO.setCoverImage(saveCoverImage(coverFile));     // 커버 이미지는 S3 저장

    galleryDAO.save(requestDTO);                             // tbl_gallery 저장
    saveWorkLinks(requestDTO.getId(), requestDTO.getWorkIds(), resolvedMemberId); // 전시 작품 연결
    saveTags(requestDTO.getId(), requestDTO.getTagIds(), requestDTO.getTagNames()); // 태그 연결
    galleryDAO.updateWorkCount(requestDTO.getId());          // 작품 수 갱신

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
    validateGalleryOwner(galleryId, resolvedMemberId);       // 내 예술관인지 검증

    WorkDTO workDTO = WorkDTO.builder()
            .memberId(resolvedMemberId)
            .title(limitText(requestDTO.getTitle(), 255))
            .price(requestDTO.getPrice())
            .mediaType(resolveMediaType(requestDTO.getCategory(), mediaFile, requestDTO.getFiles()))
            .predictedViews(defaultLong(requestDTO.getPredictedViews())) // AI 예측값 저장
            .status("ACTIVE")
            .build();

    workDAO.save(workDTO);                                   // tbl_work 저장
    saveThumbnailFile(workDTO.getId(), thumbnailFile);       // 썸네일 S3 저장
    saveMediaFile(workDTO.getId(), mediaFile, 1);            // 작품 파일 S3 저장
    saveTags(workDTO.getId(), requestDTO.getTagIds(), requestDTO.getTagNames());
    saveGalleryLink(galleryId, workDTO.getId());             // 예술관에 작품 연결
    saveAuctionIfRequested(workDTO.getId(), resolvedMemberId, requestDTO.getPrice(),
            requestDTO.getAuctionEnabled(), requestDTO.getAuctionStartingPrice(),
            requestDTO.getAuctionDeadlineHours());           // 경매 선택 시 경매 생성

    return WorkCreateResponseDTO.builder().id(workDTO.getId()).galleryId(galleryId).build();
}
```

작품 등록은 파일 저장, 태그 저장, 예술관 연결, 경매 생성까지 이어지는 핵심 시작점입니다.

## 2. 거래 흐름: 경매 / 결제

![경매와 결제](assets/slide-04.png)

### 경매 입찰

원본: `src/main/java/com/app/bideo/service/auction/BidCommandService.java`

```java
public BidResponseDTO placeBid(Long memberId, BidRequestDTO requestDTO) {
    AuctionVO auction = auctionDAO.findRawById(requestDTO.getAuctionId());

    if (!"ACTIVE".equals(auction.getStatus())) throw new IllegalStateException("종료된 경매입니다.");
    if (auction.getSellerId().equals(memberId)) throw new IllegalArgumentException("자신의 경매에는 입찰할 수 없습니다.");
    if (requestDTO.getBidPrice() < Math.round(auction.getCurrentPrice() * 1.1)) {
        throw new IllegalArgumentException("최소 입찰 단위 이상으로 입찰해야 합니다.");
    }

    bidDAO.clearPreviousWinning(requestDTO.getAuctionId());  // 이전 최고 입찰 해제
    bidDAO.save(BidVO.builder()
            .auctionId(requestDTO.getAuctionId())
            .memberId(memberId)
            .bidPrice(requestDTO.getBidPrice())
            .isWinning(true)
            .build());

    auctionDAO.updateCurrentPrice(requestDTO.getAuctionId(), requestDTO.getBidPrice(),
            bidDAO.findBidderIds(requestDTO.getAuctionId()).size());

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

    if (!payment.getBuyerId().equals(buyerId)) throw new IllegalArgumentException("결제 검증 권한이 없습니다.");
    if (!"PENDING".equals(payment.getStatus())) throw new IllegalStateException("이미 처리된 결제입니다.");

    JsonNode receipt = bootpayClient.getReceipt(requestDTO.getReceiptId()); // Bootpay 서버 검증
    if (!payment.getPaymentCode().equals(receipt.path("order_id").asText(""))) {
        throw new IllegalStateException("부트페이 주문번호가 일치하지 않습니다.");
    }
    if (receipt.path("price").asInt(-1) != payment.getTotalPrice()) {
        throw new IllegalStateException("부트페이 결제 금액이 일치하지 않습니다.");
    }

    return completePayment(payment.getId(), buyerId);        // 검증 성공 후 결제 완료
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

    detail.setCoverImage(s3FileService.getPresignedUrl(detail.getCoverImage())); // S3 key -> URL
    detail.setTags(galleryDAO.findTagsByGalleryId(id));
    detail.setWorks(getGalleryWorks(id, detail.getMemberId()));

    Long memberId = resolveAuthenticatedMemberId();
    detail.setIsLiked(memberId != null && galleryDAO.existsLike(memberId, id));
    detail.setIsBookmarked(memberId != null && bookmarkDAO.exists(memberId, "GALLERY", id));
    detail.setIsOwner(memberId != null && memberId.equals(detail.getMemberId()));
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

    applyFileUrls(detail);                                   // 작품 파일 S3 URL 변환
    detail.setIsLiked(memberId != null && workDAO.existsLike(memberId, id));
    detail.setIsBookmarked(memberId != null && bookmarkDAO.exists(memberId, "WORK", id));
    detail.setIsOwner(memberId != null && memberId.equals(detail.getMemberId()));
    detail.setHasActiveAuction(workDAO.existsActiveAuctionByWorkId(id)); // 경매 패널 노출 여부

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
            true, "내 작품", "개", endDate));
    analyticsMetrics.put("galleries", buildMetricSeries("galleries", "내가 만든 예술관 추이",
            String.valueOf(ownedGalleryCount),
            toDailyMap(dashboardMapper.selectDailyCreatedGalleryCounts(memberId, startDate, endDate)),
            true, "내 예술관", "개", endDate));

    return DashboardResponseDTO.builder()
            .totalViewsText(formatCompactNumber(nullSafe(summary.getTotalViews())))
            .activeAuctionsText(formatTwoDigits(nullSafe(summary.getActiveAuctionCount())))
            .pendingPaymentsText(formatCurrencyCompact(nullSafe(summary.getPendingPaymentAmount())))
            .analyticsMetrics(analyticsMetrics)
            .ownedWorks(withFallback(dashboardMapper.selectOwnedWorkItems(memberId), "등록된 작품이 없습니다.", ""))
            .galleries(withFallback(dashboardMapper.selectGalleryItems(memberId), "예술관이 없습니다.", ""))
            .paymentHistory(withFallback(dashboardMapper.selectPaymentHistoryItems(memberId), "결제 이력이 없습니다.", ""))
            .build();
}
```

 대시보드는 조회수, 경매, 결제, 작품, 예술관 데이터를 모아 사용자의 운영 현황을 한 화면에 보여줍니다.

1. 예술관을 만들고 작품을 등록한다.
2. 등록된 작품은 상세 화면과 피드에서 소비된다.
3. 작품에 경매가 열리면 입찰이 진행된다.
4. 낙찰 또는 구매는 결제 검증을 거쳐 완료된다.
5. 대시보드에서 내 작품, 예술관, 거래 현황을 확인한다.
