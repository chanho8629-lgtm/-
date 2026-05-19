# BIDEO 웹 서비스 발표자료

> BIDEO 소스 학습 내용을 바탕으로 만든 발표용 캡처 자료입니다.  

## 1. 서비스 소개

![서비스 소개](assets/slide-01.png)

BIDEO는 영상 크리에이터가 작품을 등록하고, 사용자가 피드에서 탐색하며, 경매와 결제까지 이어갈 수 있는 영상 작품 마켓플레이스입니다. Spring Boot와 Thymeleaf 기반의 서버 렌더링 화면 위에 기능별 JavaScript가 REST API를 호출하는 구조입니다.

## 2. 웹 화면 구조

![웹 화면 구조](assets/slide-02.png)

공통 레이아웃은 `templates/layout/layout.html` fragment가 담당합니다. 헤더, 검색, 사이드바, 알림, 채팅, 만들기 모달이 공통으로 제공되고, 각 화면은 기능별 템플릿과 CSS/JS를 조합합니다.

## 3. 작품 등록과 피드

![작품 등록과 피드](assets/slide-03.png)

작품 등록 화면에서는 제목, 설명, 가격 유형, 태그, 썸네일, 미디어 파일을 입력합니다. 등록된 작품은 `/work/feed`와 `/work/detail/{id}`에서 쇼츠형 상세 화면으로 노출되며, 좋아요, 댓글, 공유, 찜, 신고 흐름을 제공합니다.

## 4. 경매와 결제

![경매와 결제](assets/slide-04.png)

작품 상세 화면 안에서 경매 패널이 함께 동작합니다. 사용자는 현재 최고가, 입찰 이력, 마감 시간을 확인하고 다음 입찰가로 입찰할 수 있습니다. 낙찰 이후에는 주문과 결제 화면으로 흐름이 이어집니다.

## 5. 예술관, 공모전, 프로필

![예술관 공모전 프로필](assets/slide-05.png)

예술관은 여러 작품을 하나의 전시 단위로 묶는 기능이고, 공모전은 목록, 등록, 참여 작품 제출 흐름을 제공합니다. 프로필 화면에서는 내 작품, 팔로우, 배지, 보관함 등 크리에이터 활동을 관리합니다.

## 6. 관리자 화면

![관리자 화면](assets/slide-06.png)

관리자 화면은 회원, 작품, 경매, 결제, 신고, 출금 요청을 목록과 상세 화면으로 관리합니다. 운영자는 상태값을 기준으로 처리 대상을 확인하고, 서비스 운영 데이터를 한 화면에서 점검할 수 있습니다.

## 7. 기술 구성

![기술 구성](assets/slide-07.png)

- Backend: Spring Boot 3.5, Java 17, Spring Security, MyBatis, PostgreSQL, Redis, WebSocket
- Frontend: Thymeleaf layout fragment, 기능별 정적 JavaScript, CSS 분리
- API: `/api/works`, `/api/galleries`, `/api/auction`, `/api/payments`, `/api/admin/*`
- 화면: `/main`, `/work/feed`, `/gallery/{id}`, `/contest/list`, `/profile`, `/dashboard`, `/admin/*`

## 발표 순서 요약

1. BIDEO가 어떤 서비스인지 설명합니다.
2. 공통 웹 레이아웃과 화면 구성을 소개합니다.
3. 작품 등록부터 피드 노출까지 사용자 흐름을 설명합니다.
4. 경매 입찰과 결제 전환 흐름을 설명합니다.
5. 예술관, 공모전, 프로필로 확장되는 기능을 설명합니다.
6. 관리자 화면에서 운영 데이터를 관리하는 방식을 설명합니다.
7. 마지막으로 기술 스택과 API 구조를 정리합니다.

## 작성 파일

- `capture/bideo-presentation.html`: 캡처 원본 HTML
- `assets/slide-01.png` ~ `assets/slide-07.png`: 발표용 캡처 이미지
- `README.md`: 발표 문서

