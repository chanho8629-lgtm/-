# BIDEO 웹서비스 핵심 코드 발표 정리

> 기준 문서: `발표/README.md`  
> 기준 프로젝트: `spring/workspace/bideo`  
> 발표 범위: 예술관 등록, 작품 등록, 결제, 경매, 대시보드, 예술관 상세, 작품 상세

## 1. 예술관 등록

원본 위치: `src/main/java/com/app/bideo/service/gallery/GalleryService.java`

```java
public GalleryCreateResponseDTO write(Long memberId, GalleryCreateRequestDTO requestDTO, MultipartFile coverFile) {
    // 1. 로그인 사용자 ID를 확정한다.
    // 화면에서 memberId가 넘어오지 않아도 SecurityContext 기준으로 보정한다.
    Long resolvedMemberId = resolveMemberId(memberId);

    // 2. 예술관 기본 정보에 작성자와 커버 이미지를 세팅한다.
    // 커버 이미지는 S3에 업로드되고, DB에는 S3 key가 저장된다.
    requestDTO.setMemberId(resolvedMemberId);
    requestDTO.setCoverImage(saveCoverImage(coverFile));

    // 3. 댓글 허용, 유사 예술관 노출 여부는 값이 없으면 기본 true로 처리한다.
    requestDTO.setAllowComment(requestDTO.getAllowComment() != null ? requestDTO.getAllowComment() : true);
    requestDTO.setShowSimilar(requestDTO.getShowSimilar() != null ? requestDTO.getShowSimilar() : true);

    // 4. tbl_gallery에 예술관 본문을 먼저 저장한다.
    // 저장 후 requestDTO.id에 생성된 gallery_id가 들어간다.
    galleryDAO.save(requestDTO);

    // 5. 예술관에 포함할 작품 목록과 태그를 연결 테이블에 저장한다.
    saveWorkLinks(requestDTO.getId(), requestDTO.getWorkIds(), resolvedMemberId);
    saveTags(requestDTO.getId(), requestDTO.getTagIds(), requestDTO.getTagNames());

    // 6. 연결된 작품 수를 다시 계산해서 예술관 통계값을 맞춘다.
    galleryDAO.updateWorkCount(requestDTO.getId());

    // 7. 프론트가 등록 완료 후 이동할 URL과 생성 ID를 반환한다.
    return GalleryCreateResponseDTO.builder()
            .galleryId(requestDTO.getId())
            .memberId(resolvedMemberId)
            .memberNickname(memberRepository.findById(resolvedMemberId)
                    .map(member -> member.getNickname())
                    .orElseThrow(() -> new IllegalStateException("member not found")))
            .redirectUrl("/main?relatedGalleryId=" + requestDTO.getId())
            .build();
}
```

발표 설명:
예술관 등록은 단순히 제목과 이미지만 저장하는 기능이 아니라, 커버 이미지 업로드, 작품 연결, 태그 연결, 작품 수 갱신까지 한 번에 처리하는 큐레이션 생성 흐름입니다.

## 2. 작품 등록

원본 위치: `src/main/java/com/app/bideo/service/work/WorkService.java`

```java
public WorkCreateResponseDTO write(Long memberId, WorkCreateRequestDTO requestDTO,
                                   MultipartFile mediaFile, MultipartFile thumbnailFile) {
    // 1. 작성자를 확정하고, 작품이 들어갈 예술관을 필수값으로 확인한다.
    Long resolvedMemberId = resolveMemberId(memberId);
    Long galleryId = requireGalleryId(requestDTO.getGalleryId());

    // 2. 남의 예술관에 작품을 등록하지 못하도록 소유자 검증을 한다.
    validateGalleryOwner(galleryId, resolvedMemberId);

    // 3. 실제 업로드 파일 또는 AI 생성 파일 정보가 하나는 있어야 한다.
    if ((mediaFile == null || mediaFile.isEmpty()) && !hasExistingFiles(requestDTO.getFiles())) {
        throw new IllegalArgumentException("업로드할 파일을 선택해주세요.");
    }

    // 4. 파일 타입을 기준으로 이미지/영상 카테고리와 썸네일 여부를 판단한다.
    String category = resolveCategory(requestDTO.getCategory(), mediaFile);
    boolean hasThumbnail = thumbnailFile != null && !thumbnailFile.isEmpty();

    // 5. 작품 본문, 거래 정보, AI 예측값을 WorkDTO로 묶는다.
    WorkDTO workDTO = WorkDTO.builder()
            .memberId(resolvedMemberId)
            .title(limitText(requestDTO.getTitle(), 255))
            .category(category)
            .description(requestDTO.getDescription())
            .price(requestDTO.getPrice())
            .licenseType(limitText(requestDTO.getLicenseType(), 100))
            .isTradable(requestDTO.getIsTradable())
            .mediaType(resolveMediaType(category, mediaFile, requestDTO.getFiles()))
            .tagCount(countTags(requestDTO))
            .thumbnailExists(hasThumbnail)
            .isAiGenerated(resolveAiGenerated(requestDTO, mediaFile))
            .aiQualityScore(defaultDouble(requestDTO.getAiQualityScore()))
            .predictedViews(defaultLong(requestDTO.getPredictedViews()))
            .predictedLikeCount(defaultInteger(requestDTO.getPredictedLikeCount()))
            .predictedPopular(defaultInteger(requestDTO.getPredictedPopular()))
            .predictedPopularProbability(defaultDouble(requestDTO.getPredictedPopularProbability()))
            .status("ACTIVE")
            .build();

    // 6. tbl_work 저장 후, 파일/태그/예술관 연결/경매 설정을 순서대로 저장한다.
    workDAO.save(workDTO);
    saveThumbnailFile(workDTO.getId(), thumbnailFile);
    saveMediaFile(workDTO.getId(), mediaFile, hasThumbnail ? 1 : 0);
    saveExistingFiles(workDTO.getId(), requestDTO.getFiles(), hasThumbnail ? 1 : 0);
    saveTags(workDTO.getId(), requestDTO.getTagIds(), requestDTO.getTagNames());
    saveGalleryLink(galleryId, workDTO.getId());
    saveAuctionIfRequested(workDTO.getId(), resolvedMemberId, requestDTO.getPrice(),
            requestDTO.getAuctionEnabled(), requestDTO.getAuctionStartingPrice(),
            requestDTO.getAuctionDeadlineHours());

    // 7. 등록 완료 후 프로필의 작품 탭으로 이동하도록 응답한다.
    return WorkCreateResponseDTO.builder()
            .id(workDTO.getId())
            .galleryId(galleryId)
            .redirectUrl("/profile?tab=works")
            .build();
}
```

발표 설명:
작품 등록은 BIDEO의 중심 기능입니다. 작품 본문 저장뿐 아니라 S3 파일 저장, 태그 저장, 예술관 연결, AI 예측값 저장, 경매 생성까지 연결되어 있어 이후 피드, 상세, 결제, 대시보드의 출발점이 됩니다.

## 3. 결제

원본 위치: `src/main/java/com/app/bideo/service/payment/PaymentService.java`

```java
public PaymentResponseDTO confirmBootpayPayment(Long buyerId, BootpayConfirmRequestDTO requestDTO) {
    // 1. 결제 ID와 Bootpay receiptId가 모두 있어야 서버 검증을 진행한다.
    if (requestDTO.getPaymentId() == null || requestDTO.getReceiptId() == null || requestDTO.getReceiptId().isBlank()) {
        throw new IllegalArgumentException("부트페이 결제 검증 정보가 누락되었습니다.");
    }

    // 2. DB에 저장된 결제 대기 건을 조회한다.
    PaymentVO payment = paymentDAO.findRawById(requestDTO.getPaymentId())
            .orElseThrow(() -> new IllegalArgumentException("결제를 찾을 수 없습니다."));

    // 3. 결제를 요청한 사용자와 현재 로그인 사용자가 같은지 검증한다.
    if (!payment.getBuyerId().equals(buyerId)) {
        throw new IllegalArgumentException("결제 검증 권한이 없습니다.");
    }

    // 4. 이미 완료/취소된 결제를 중복 처리하지 않도록 PENDING 상태만 허용한다.
    if (!"PENDING".equals(payment.getStatus())) {
        throw new IllegalStateException("이미 처리된 결제입니다.");
    }

    // 5. Bootpay 서버에서 실제 결제 영수증을 조회한다.
    JsonNode receipt = bootpayClient.getReceipt(requestDTO.getReceiptId());
    int paidPrice = receipt.path("price").asInt(-1);
    int status = receipt.path("status").asInt(-1);
    String orderId = receipt.path("order_id").asText("");

    // 6. 프론트 조작 방지를 위해 주문번호, 결제금액, 결제상태를 서버에서 다시 검증한다.
    if (!payment.getPaymentCode().equals(orderId)) {
        throw new IllegalStateException("부트페이 주문번호가 일치하지 않습니다.");
    }
    if (paidPrice != payment.getTotalPrice()) {
        throw new IllegalStateException("부트페이 결제 금액이 일치하지 않습니다.");
    }
    if (status != 1 && status != 2) {
        throw new IllegalStateException("부트페이 결제 상태가 완료가 아닙니다.");
    }

    // 7. 모든 검증을 통과하면 결제 완료 처리와 주문 상태 갱신을 수행한다.
    return completePayment(payment.getId(), buyerId);
}
```

발표 설명:
결제는 프론트에서 성공했다고 바로 완료 처리하지 않습니다. Bootpay 영수증을 서버에서 다시 조회하고, 주문번호와 금액을 DB 값과 비교한 뒤에만 완료 처리합니다.

## 4. 경매

원본 위치: `src/main/java/com/app/bideo/service/auction/BidCommandService.java`

```java
public BidResponseDTO placeBid(Long memberId, BidRequestDTO requestDTO) {
    // 1. 입찰 대상 경매가 존재하는지 조회한다.
    AuctionVO auction = auctionDAO.findRawById(requestDTO.getAuctionId());
    if (auction == null) {
        throw new IllegalArgumentException("경매를 찾을 수 없습니다.");
    }

    // 2. ACTIVE 상태이고 마감 시간이 지나지 않은 경매만 입찰 가능하다.
    if (!"ACTIVE".equals(auction.getStatus())) {
        throw new IllegalStateException("종료된 경매입니다.");
    }
    if (auction.getClosingAt() != null && auction.getClosingAt().isBefore(LocalDateTime.now())) {
        throw new IllegalStateException("경매가 종료되었습니다.");
    }

    // 3. 판매자는 자기 작품에 입찰할 수 없다.
    if (auction.getSellerId().equals(memberId)) {
        throw new IllegalArgumentException("자신의 경매에는 입찰할 수 없습니다.");
    }

    // 4. 현재가보다 높고, 최소 10% 이상 높은 금액만 허용한다.
    if (requestDTO.getBidPrice() <= auction.getCurrentPrice()) {
        throw new IllegalArgumentException("현재가보다 높은 금액으로 입찰해야 합니다.");
    }
    if (requestDTO.getBidPrice() < Math.round(auction.getCurrentPrice() * 1.1)) {
        throw new IllegalArgumentException("최소 입찰 단위 이상으로 입찰해야 합니다.");
    }

    // 5. 기존 최고 입찰자를 저장해 두고, 이전 winning 상태를 해제한다.
    BidResponseDTO previousHighest = bidDAO.findHighestBid(requestDTO.getAuctionId()).orElse(null);
    bidDAO.clearPreviousWinning(requestDTO.getAuctionId());

    // 6. 새 입찰을 winning=true로 저장한다.
    bidDAO.save(BidVO.builder()
            .auctionId(requestDTO.getAuctionId())
            .memberId(memberId)
            .bidPrice(requestDTO.getBidPrice())
            .isWinning(true)
            .build());

    // 7. 경매 현재가와 참여자 수를 갱신한다.
    int uniqueBidderCount = bidDAO.findBidderIds(requestDTO.getAuctionId()).size();
    auctionDAO.updateCurrentPrice(requestDTO.getAuctionId(), requestDTO.getBidPrice(), uniqueBidderCount);

    // 8. 판매자와 이전 최고 입찰자에게 알림을 보낸다.
    notificationService.createNotification(
            auction.getSellerId(), memberId, "BID", "AUCTION",
            auction.getWorkId(), String.format("%,d원에 입찰했습니다.", requestDTO.getBidPrice())
    );
    if (previousHighest != null && !previousHighest.getMemberId().equals(memberId)) {
        notificationService.createNotification(
                previousHighest.getMemberId(), memberId, "BID", "AUCTION",
                auction.getWorkId(), String.format("더 높은 금액(%,d원)으로 입찰했습니다.", requestDTO.getBidPrice())
        );
    }

    // 9. 최종 최고 입찰 정보를 반환한다.
    return bidDAO.findHighestBid(requestDTO.getAuctionId()).orElse(null);
}
```

발표 설명:
경매 입찰은 단순 금액 저장이 아니라 경매 상태, 마감 시간, 판매자 본인 여부, 최소 입찰 금액을 모두 검증합니다. 최고 입찰자를 교체하고 알림까지 발생시키는 거래 로직입니다.

## 5. 대시보드

원본 위치: `src/main/java/com/app/bideo/service/dashboard/DashboardService.java`

```java
@Cacheable(value = "dashboard", key = "#memberId")
public DashboardResponseDTO getDashboard(Long memberId) {
    // 1. 대시보드 상단에 표시할 사용자명과 요약 통계를 조회한다.
    String creatorName = defaultIfBlank(dashboardMapper.selectCreatorName(memberId), "크리에이터");
    DashboardSummaryRawDTO summary = defaultSummary(dashboardMapper.selectDashboardSummary(memberId));
    DashboardNoticeSummaryDTO noticeSummary = defaultNoticeSummary(dashboardMapper.selectNoticeSummary(memberId));

    // 2. 작품, 예술관, 공모전, 카드 등 주요 개수를 각각 조회한다.
    int bookmarkedWorkCount = nullSafe(dashboardMapper.selectBookmarkedWorkCount(memberId));
    int ownedWorkCount = nullSafe(dashboardMapper.selectOwnedWorkCount(memberId));
    int ownedGalleryCount = nullSafe(dashboardMapper.selectOwnedGalleryCount(memberId));
    int hostedContestCount = nullSafe(dashboardMapper.selectHostedContestCount(memberId));
    int registeredCardCount = nullSafe(dashboardMapper.selectRegisteredCardCount(memberId));

    // 3. 최근 기간별 추이 그래프에 사용할 날짜 범위를 만든다.
    LocalDate endDate = LocalDate.now();
    LocalDate startDate = endDate.minusDays(MAX_LOOKBACK_DAYS - 1L);

    // 4. 일자별 찜/작품등록/예술관등록 데이터를 조회해 그래프용 Map으로 변환한다.
    Map<LocalDate, Long> bookmarkedDailyMap =
            toDailyMap(dashboardMapper.selectDailyBookmarkedWorkCounts(memberId, startDate, endDate));
    Map<LocalDate, Long> createdWorkDailyMap =
            toDailyMap(dashboardMapper.selectDailyCreatedWorkCounts(memberId, startDate, endDate));
    Map<LocalDate, Long> createdGalleryDailyMap =
            toDailyMap(dashboardMapper.selectDailyCreatedGalleryCounts(memberId, startDate, endDate));

    // 5. 프론트에서 바로 차트로 그릴 수 있는 analyticsMetrics 구조를 만든다.
    Map<String, DashboardMetricSeriesDTO> analyticsMetrics = new LinkedHashMap<>();
    analyticsMetrics.put("favorites", buildMetricSeries("favorites", "찜한 작품 추이",
            String.valueOf(bookmarkedWorkCount), bookmarkedDailyMap, true, "찜한 작품", "개", endDate));
    analyticsMetrics.put("works", buildMetricSeries("works", "내가 만든 작품 추이",
            String.valueOf(ownedWorkCount), createdWorkDailyMap, true, "내 작품", "개", endDate));
    analyticsMetrics.put("galleries", buildMetricSeries("galleries", "내가 만든 예술관 추이",
            String.valueOf(ownedGalleryCount), createdGalleryDailyMap, true, "내 예술관", "개", endDate));

    // 6. 화면의 모든 탭에 필요한 목록과 요약 정보를 한 번에 응답한다.
    return DashboardResponseDTO.builder()
            .creatorName(creatorName)
            .totalViewsText(formatCompactNumber(nullSafe(summary.getTotalViews())))
            .activeAuctionsText(formatTwoDigits(nullSafe(summary.getActiveAuctionCount())))
            .pendingPaymentsText(formatCurrencyCompact(nullSafe(summary.getPendingPaymentAmount())))
            .analyticsMetrics(analyticsMetrics)
            .summaryTables(buildSummaryTables(summary, noticeSummary, bookmarkedWorkCount,
                    ownedWorkCount, ownedGalleryCount, hostedContestCount, registeredCardCount))
            .reactionChart(buildChart(dashboardMapper.selectReactionTrend(memberId)))
            .myAuctions(withFallback(dashboardMapper.selectMyAuctionItems(memberId), "등록된 경매가 없습니다.", ""))
            .bookmarkedWorks(withFallback(dashboardMapper.selectBookmarkedWorkItems(memberId), "찜한 작품이 없습니다.", ""))
            .ownedWorks(withFallback(dashboardMapper.selectOwnedWorkItems(memberId), "등록된 작품이 없습니다.", ""))
            .galleries(withFallback(dashboardMapper.selectGalleryItems(memberId), "예술관이 없습니다.", ""))
            .paymentHistory(withFallback(dashboardMapper.selectPaymentHistoryItems(memberId), "결제 이력이 없습니다.", ""))
            .wishlistNotifications(buildWishlistNotifications(noticeSummary))
            .build();
}
```

발표 설명:
대시보드는 여러 화면의 데이터를 따로 호출하지 않고, 사용자 기준 운영 현황을 한 번에 모아 응답합니다. 캐시를 적용해서 반복 조회 부담을 줄이고, 프론트는 받은 JSON으로 탭과 차트를 렌더링합니다.

## 6. 예술관 상세

원본 위치: `src/main/java/com/app/bideo/service/gallery/GalleryService.java`

```java
public GalleryDetailResponseDTO getGalleryDetail(Long id) {
    // 1. 예술관 기본 정보를 조회한다. 없으면 예외를 발생시킨다.
    GalleryDetailResponseDTO detail = galleryDAO.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("gallery not found"));

    // 2. DB에는 S3 key가 저장되어 있으므로, 화면 표시용 presigned URL로 변환한다.
    detail.setCoverImage(s3FileService.getPresignedUrl(detail.getCoverImage()));
    detail.setMemberProfileImage(s3FileService.getPresignedUrl(detail.getMemberProfileImage()));

    // 3. 예술관 태그와 예술관에 포함된 작품 목록을 붙인다.
    detail.setTags(galleryDAO.findTagsByGalleryId(id));
    detail.setWorks(getGalleryWorks(id, detail.getMemberId()));

    // 4. 현재 로그인 사용자를 기준으로 좋아요, 북마크, 팔로우, 소유자 여부를 계산한다.
    Long memberId = resolveAuthenticatedMemberId();
    detail.setIsLiked(memberId != null && galleryDAO.existsLike(memberId, id));
    detail.setIsBookmarked(memberId != null && bookmarkDAO.exists(memberId, "GALLERY", id));
    detail.setIsFollowing(
            memberId != null
                    && detail.getMemberId() != null
                    && !memberId.equals(detail.getMemberId())
                    && followService.isFollowing(memberId, detail.getMemberId())
    );
    detail.setIsOwner(memberId != null && memberId.equals(detail.getMemberId()));

    // 5. 템플릿에서 바로 사용할 수 있는 상세 DTO를 반환한다.
    return detail;
}
```

발표 설명:
예술관 상세는 예술관 본문, 커버 이미지, 작성자 프로필, 태그, 포함 작품, 사용자별 상호작용 상태를 한 번에 구성합니다. 같은 화면이라도 로그인 사용자에 따라 좋아요/북마크/팔로우 버튼 상태가 달라집니다.

## 7. 작품 상세

원본 위치: `src/main/java/com/app/bideo/service/work/WorkService.java`

```java
@Transactional(readOnly = true)
public WorkDetailResponseDTO getWorkDetail(Long id) {
    // 1. 현재 로그인 사용자를 확인한다.
    // 비로그인 사용자도 상세 조회는 가능하므로 null일 수 있다.
    Long memberId = resolveAuthenticatedMemberId();

    // 2. 작품 기본 정보, 작성자, 파일, 댓글 등을 조회한다.
    WorkDetailResponseDTO detail = workDAO.findDetailById(id, memberId)
            .orElseThrow(() -> new IllegalArgumentException("work not found"));

    // 3. 작품 파일도 S3 key에서 화면 표시용 URL로 변환한다.
    applyFileUrls(detail);

    // 4. 사용자별 좋아요, 북마크, 소유자 여부를 계산한다.
    detail.setIsLiked(memberId != null && workDAO.existsLike(memberId, id));
    detail.setIsBookmarked(memberId != null && bookmarkDAO.exists(memberId, "WORK", id));
    detail.setIsOwner(memberId != null && memberId.equals(detail.getMemberId()));

    // 5. 작품 상세 안에서 경매 패널을 보여줄지 판단한다.
    detail.setHasActiveAuction(workDAO.existsActiveAuctionByWorkId(id));

    // 6. 댓글도 현재 사용자의 좋아요/권한 상태를 반영한다.
    if (detail.getComments() != null) {
        detail.getComments().forEach(comment -> applyCommentState(comment, memberId));
    }

    // 7. 작품 상세 템플릿에서 사용할 완성 DTO를 반환한다.
    return detail;
}
```

발표 설명:
작품 상세는 BIDEO의 핵심 소비 화면입니다. 작품 미디어, 작성자 정보, 댓글, 좋아요, 북마크, 소유자 여부, 활성 경매 여부를 조합해 피드형 상세 화면과 경매 흐름을 동시에 지원합니다.

## 발표 마무리 멘트

BIDEO의 웹서비스 코드는 기능별로 Controller, Service, Mapper가 나뉘어 있습니다. Controller는 요청을 받고, Service는 검증과 비즈니스 흐름을 처리하며, Mapper/DAO는 DB 저장과 조회를 담당합니다. 발표에서는 “등록 → 상세 조회 → 거래 → 대시보드 확인” 순서로 설명하면 서비스 흐름이 자연스럽게 이어집니다.
