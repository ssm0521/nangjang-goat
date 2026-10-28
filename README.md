# 냉장GOAT (Naengjang-GOAT)

> 소상공인 식당 사장을 위한 **AI 기반 재고·발주·가격 비교 플랫폼**
> 상명대학교 소프트웨어학과 2026 캡스톤 프로젝트

> **📌 이 레포는 세 명의 팀 레포를 하나로 묶은 통합 열람용 레포입니다.**
> 실제 개발은 아래 **원본 두 레포**에서 진행되었습니다. 각자 담당 범위는 아래 "팀 구성" 참고.
> - 백엔드·모바일 원본: [gm-15/naengjang-goat_backend](https://github.com/gm-15/naengjang-goat_backend)
> - 웹 프론트엔드 원본: [keonU206/smu_capstone_project](https://github.com/keonU206/smu_capstone_project)

<p align="center">
  <img alt="Spring Boot" src="https://img.shields.io/badge/Spring_Boot-3.5-6DB33F?logo=spring-boot&logoColor=white">
  <img alt="Java" src="https://img.shields.io/badge/Java-21-ED8B00?logo=openjdk&logoColor=white">
  <img alt="React" src="https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black">
  <img alt="React Native" src="https://img.shields.io/badge/React_Native-Expo_57-000020?logo=expo&logoColor=white">
  <img alt="MySQL" src="https://img.shields.io/badge/MySQL-8.0-4479A1?logo=mysql&logoColor=white">
</p>

---

## 1. 프로젝트 개요

식당 사장이 **"내일 뭘 얼마나 사야 하지?"** 라는 매일 반복되는 질문에 대답하지 않아도 되게 하는 서비스.

**해결하는 문제**
1. 재고 파악 — 어떤 재료가 얼마나 남았고 유통기한이 언제까지인지 수기로 챙기기 어려움
2. 가격 비교 — 매번 KAMIS·네이버·쿠팡·식자재왕 돌아다니며 수동 비교 소모적
3. 적정 발주량 — 하루 매출 데이터만으로 "다음 주 몇 kg 사야 할지" 감으로 결정

**해결 방식**
- **공공데이터 자동 수집** — KAMIS(농산물)·EKAPE(축산물) OpenAPI 로 매일 공식 시세 축적
- **쇼핑몰 가격 비교** — 네이버 쇼핑 API + 식자재왕·쿠팡 크롤링 결과를 kg당 단가로 정규화
- **판매 → 재고 자동 차감** — POS 엑셀 업로드만으로 레시피 기반 재료 자동 소진
- **데이터 기반 추천 발주량** — 과거 소진 속도 × 다음 발주일까지 일수 로 재료별 권장량 계산

---

## 2. 팀 구성

| 역할 | 담당자 | 비중 |
|-----|-------|------|
| 백엔드·ML | **박건우** ([gm-15](https://github.com/gm-15)) | 백엔드 전반, ML 가격 예측 |
| 프론트엔드 (웹·모바일) | **김건우** ([keonU206](https://github.com/keonU206)) | React 웹 대시보드, Expo React Native 앱 |
| 외부 데이터 파이프라인 | **심상묵** ([ssm0521](https://github.com/ssm0521)) | KAMIS/EKAPE 수집 파이프라인, 쇼핑몰 가격 정규화 |

### 2.1 담당 범위 상세

**박건우 (park) — 백엔드 아키텍처·ML**
- 도메인 설계 — Ingredient / InventoryBatch / Menu / Recipe / PurchaseOrder / PosUpload 등 전체 JPA 모델
- 비즈니스 로직 — 재고 소진(FIFO), 유통기한 알림, 발주 등록·조회, POS 판매 반영 파이프라인
- 인증/인가 — JWT + Spring Security
- Spring Batch 스케줄링 — KAMIS/EKAPE/네이버 배치 Job 구성
- ML 가격 예측 — 변동성 분석, buySignal 임계값 계산 (`batch/analysis/`)
- 핵심 파일: `src/main/java/.../workflow/`, `.../purchase/`, `.../inventory/`, `.../user/`, `batch/job/`, `batch/writer/`

**김건우 (kim) — 웹·모바일 프론트엔드**
- 웹 전체 — React + Vite + Tailwind + shadcn/ui, React Query v5
- 모바일 앱 전체 — Expo React Native + Expo Router, FCM 알림, DocumentPicker 기반 엑셀 업로드 UI
- 페이지 설계 — 로그인, 메인 대시보드, 재고 관리, 최저가 조회, 발주 관리, 운영(POS 반영), 알림
- 컴포넌트 라이브러리 — `web/src/app/components/`, `backend/mobile/src/components/`
- 상태관리 — React Query 기반 서버 상태 캐싱, 커스텀 훅 (`web/src/app/hooks/`, `backend/mobile/src/hooks/`)
- 핵심 파일: `web/` 전체, `backend/mobile/` 전체

**심상묵 (sim) — 외부 데이터 파이프라인**
- KAMIS 농산물 수집 파이프라인 (`batch/service/KamisApiClient.java`, `KamisPriceProcessor.java`, `MarketPriceSaveService.java`, `KamisPriceDto.java`)
  - KAMIS 공식 OpenAPI 연동, 소매·도매 분기, `p_convert_kg_yn` kg 환산
  - `observationKey` 기반 중복 제거, chunk 트랜잭션 안정화
- EKAPE 축산물 수집 파이프라인 — HTML 파싱 → 공식 OpenAPI 전환, 1년치 백필 서비스 (`batch/ekape/`, `batch/service/EkapeBackfillService.java`)
- `IngredientRepository.findAllBy*` 추가 — 다수 점주 매칭 시 단일 반환 버그 해결
- 모바일 파일 업로드 네이티브 전환 버그 fix (`mobile/src/api/workflow.ts`) — kim 이 만든 앱에서 RN `fetch + FormData` 가 안드로이드 cache URI 에서 즉시 throw 되는 이슈를 `expo-file-system` 네이티브 업로드로 우회
- 발주 총액 kg 환산 버그 수정 (`purchase/service/PurchaseOrderService.java:71`) — 단위 불일치로 1,000배 뻥튀기되던 계산 수정
- 쇼핑몰 가격 정규화 — 네이버/쿠팡/식자재왕 상품 가격 kg당 단가 환산 로직
- 레시피 템플릿 import 스크립트 (`backend/crawler/ewangmart/`)
- 웹 `LowestPriceDetailPage.tsx` 발주 모달 `unitPricePerKg` 전달 보완

---

## 3. 아키텍처

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  React Web  │   │  Expo RN    │   │  Admin CLI  │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │ HTTPS           │ HTTPS           │
       └─────────────────┼─────────────────┘
                         ▼
             ┌────────────────────────┐
             │   Spring Boot API      │
             │   (JWT, JPA, Batch)    │
             └────┬────────────┬──────┘
                  │            │
          ┌───────▼──┐    ┌────▼─────┐
          │  MySQL   │    │  Redis   │
          │ (상태)    │    │ (캐시)   │
          └──────────┘    └──────────┘
                  ▲            ▲
                  │            │
       ┌──────────┴────────────┴─────────────┐
       │        배치 파이프라인                │
       │  KAMIS  │  EKAPE  │  네이버/쿠팡/식자재왕 │
       │  (공공 OpenAPI)   │  (크롤링/API)       │
       └──────────────────────────────────────┘
```

**폴더 구조**
```
nangjang-goat/
├── backend/           # Spring Boot API + 배치 + 모바일(mobile/)
│   ├── src/
│   ├── mobile/        # Expo React Native 앱
│   ├── crawler/       # Python 크롤러 (식자재왕 등)
│   └── docs/          # 설계 문서
├── web/               # React + Vite 웹 대시보드
├── .env.example       # 환경변수 템플릿
└── README.md
```

---

## 4. 핵심 기능

### 4.1 재고 관리
- 재료별 배치 단위 입고/소진 (FIFO)
- 유통기한 임박 알림 (3일 이내 자동 색상 강조)
- 재고 부족 알림 (일 평균 소진량 × N일 기준)

### 4.2 공공 가격 수집
- **KAMIS** — 농산물 소매·도매가 매일 수집, 1년치 축적 완료 (2,263건)
- **EKAPE** — 축산물(소/돼지/닭/계란) 100g 기준 소매가 수집 (1,093건)
- `observationKey` 기반 중복 제거, kg 환산 정규화

### 4.3 최저가 비교
- 재료별 KAMIS 공식 시세와 **네이버/쿠팡/식자재왕** 실시간 상품가 비교
- 30일 가격 추세 차트, 저점 매수 시그널
- 상품 사이트 바로가기 → 돌아오면 발주 자동 등록 플로우

### 4.4 판매 반영 (POS 연동)
- POS 엑셀 양식 다운로드 → 메뉴코드·메뉴명 자동 기입
- 업로드 시 메뉴 → 레시피(BOM) → 재료 소진량 자동 환산
- 모바일/웹 공통 멀티파트 업로드 (RN 은 `expo-file-system` 네이티브 업로드로 안드로이드 FormData 이슈 우회)

### 4.5 추천 발주량
- 하루 평균 소진량 × 다음 발주일까지 일수 → 재료별 권장 발주 kg
- 레시피 미등록 재료는 "판단 대기" 로 분류, 사장이 수동 결정

---

## 5. 로컬 실행

### 사전 요구사항
- Java 21, Node 18+, Python 3.11+, Docker (선택)
- MySQL 8.0, Redis 7+

### 환경변수 세팅
```bash
cp .env.example .env
# .env 열고 실제 값 입력 (KAMIS/EKAPE/네이버 API 키, DB 비밀번호 등)
```

### Firebase (모바일 FCM)
`backend/mobile/google-services.json.example` 를 참고해 본인의 Firebase 프로젝트에서 `google-services.json` 을 받아 `backend/mobile/` 에 배치.

### 데모 데이터 복원 (권장)
팀 내 데모 영상 재현을 위한 전체 DB 덤프 포함. 백엔드 기동 **전에** 복원:
```bash
# MySQL 접속 후
mysql -u root -p < seed/demo_full.sql
# → naengjang_goat_db 데이터베이스가 자동 생성되고
#    재료 13개 / 메뉴 3개 (삼겹살 구이·등심 스테이크·양파튀김) /
#    레시피 BOM / POS 매핑 / KAMIS 1년치 농산물 가격 2,263건 /
#    EKAPE 1년치 축산물 가격 1,093건 / 쇼핑몰 샘플 가격 등 전부 복원됨
```
덤프에는 `demo/demo1234` 데모 계정 하나만 포함 (비밀번호는 bcrypt 해시). FCM 토큰 등 디바이스 고유값은 제거됨.

### 백엔드
```bash
cd backend
./gradlew bootRun
# → http://localhost:8080
```

### 웹
```bash
cd web
npm install
npm run dev
# → http://localhost:5173
```

### 모바일 (Expo dev client)
```bash
cd backend/mobile
npm install
npx expo start --dev-client --tunnel
# 폰에 개발 빌드가 미리 설치돼 있어야 함
# → 폰에서 http://localhost:8081 (adb reverse 사용 시) 접속
```

---

## 6. 기술 스택

**백엔드**
- Spring Boot 3.5 / Java 21 / Spring Batch / Spring Security (JWT)
- JPA + Flyway / MySQL 8.0 / Redis
- Apache POI (엑셀 처리)

**프론트엔드 (웹)**
- React 18 / Vite 6 / TypeScript / Tailwind v4
- React Query v5 / shadcn/ui / Motion
- React Router v6

**모바일**
- React Native / Expo SDK 57 / Expo Router
- React Native FCM

**데이터 파이프라인**
- KAMIS OpenAPI (농산물)
- EKAPE OpenAPI (축산물)
- Naver Shopping Search API
- 식자재왕/쿠팡 Python 크롤러 (BeautifulSoup, requests)

---

## 7. 라이선스

학교 캡스톤 프로젝트 결과물로, 비상업적 교육 목적의 참고용으로만 사용해 주세요.
외부 공공 API(KAMIS/EKAPE/Naver) 는 각 기관의 약관을 따릅니다.
