# Sell2ndHand — 개발 가이드

## 앱 개요
중고물품 판매 관리 모바일 웹앱. Firebase Spark 플랜(Hosting + Firestore + Storage) 위에서
React CDN + Babel 방식으로 동작하며 **빌드 단계 없음(no npm, no build step)**.

- **배포 URL**: https://sell2ndhand-1045.web.app/
- **Firebase 프로젝트**: `sell2ndhand-1045`
- **개발 브랜치**: `claude/firebase-marketplace-app-TRV6b` → `main` 머지 시 자동 배포

---

## 인증 (Firebase Auth)

| 사용자 | 이메일 | 로그인 방식 |
|--------|--------|------------|
| Shaun  | shaun.yoo.ao@gmail.com  | Google Sign-In |
| Lee    | lee.yeonvely@icloud.com | Apple Sign-In  |

- 두 사용자는 **동일한 Firestore 데이터를 공유** (사용자별 격리 없음)
- `ALLOWED_EMAILS` 상수(`public/index.html` 상단)로 allowlist 관리
- Firestore/Storage 보안 규칙도 동일 이메일 목록으로 이중 보호

---

## Firebase 자격증명 (public/index.html에 포함)

```js
apiKey:            "AIzaSyB8_shUHBshfFRh-kC-V0PqHeoK8q5wBuQ"
authDomain:        "sell2ndhand-1045.firebaseapp.com"
projectId:         "sell2ndhand-1045"
storageBucket:     "sell2ndhand-1045.firebasestorage.app"
messagingSenderId: "238372371247"
appId:             "1:238372371247:web:dfca88b3e5a09a8659bfda"
```

---

## 파일 구조

```
sell2ndhand/
├── CLAUDE.md                  ← 이 파일
├── public/
│   └── index.html             ← 전체 앱 (단일 파일, React CDN + Firebase compat SDK)
├── firestore.rules            ← Firestore 보안 규칙 (allowlist 기반)
├── storage.rules              ← Storage 보안 규칙
├── firestore.indexes.json     ← 복합 인덱스 없음
├── firebase.json              ← Hosting/Firestore/Storage 설정
├── .firebaserc                ← 기본 프로젝트 = sell2ndhand-1045
└── .github/
    └── workflows/
        └── deploy.yml         ← main 브랜치 push 시 Firebase Hosting 자동 배포
```

---

## index.html 코드 구조

```
<head>
  React 18 CDN + Babel standalone CDN
  Firebase compat SDK v10 CDN (app, auth, firestore, storage)
  CSS (전역 스타일)
</head>
<body>
  <div id="root"></div>
  <script>  ← 일반 JS: Firebase 초기화 + provider 선언
  <script type="text/babel">  ← 전체 React 앱
    ── 상수 & 유틸
       CURRENCIES, DEFAULT_CATEGORIES, DEFAULT_LOCATIONS
       ALLOWED_EMAILS
       formatPrice(), statusColor(), priorityColor(), categoryIcon()
       resizeImageToBlob()     ← Canvas로 ≤500KB JPEG 반환
       uploadPhotos()          ← Storage 업로드 → URL 배열 반환
       fetchRates()            ← open.er-api.com, localStorage 1h 캐시
       convertPrice()          ← USD 기준 통화 환산
    ── 공통 UI 컴포넌트
       Tag, Card, SectionLabel, Row, Toggle, CheckItem
       PhotoPlaceholder        ← 빈 슬롯 시각 (변경 없음)
       PhotoUploader           ← 4슬롯 업로드 컴포넌트 (신규)
       AddCustomInput
    ── 화면 컴포넌트
       LoginScreen             ← Google/Apple 로그인 버튼 (신규)
       TabBar
       ListScreen              ← 환율 환산 총 수익 표시
       DetailScreen + InfoTab, LeadsTab, SellingTab, CopyTab
       AddScreen
       ChecklistScreen
       CurrencyPicker
    ── 앱 진입점
       App({ user })           ← Firestore onSnapshot, CRUD, rates
       Root                    ← auth.onAuthStateChanged 래퍼 (신규)
       ReactDOM.createRoot(...).render(<Root />)
</script>
</body>
```

---

## Firestore 데이터 모델

### `items/{autoId}`
| 필드 | 타입 | 설명 |
|------|------|------|
| title | string | 물품명 |
| category | string | 카테고리 |
| status | string | 새것/양호/수리필요 |
| location | string | 보관 위치 |
| price | number | 희망가격 |
| currency | string | KRW/USD/ZAR |
| marketMemo | string | 시장가 메모 |
| condition | string | 상태 메모 |
| brand | string | 브랜드 |
| year | number | 구매연도 |
| dimensions | string | 크기 |
| weight | string | 무게 |
| memo | string | 기타 메모 |
| sellPriority | string | 높음/보통/낮음 |
| stored | boolean | 보관중 여부 |
| photos | string[] | Storage 다운로드 URL (최대 4개) |
| leads | object[] | 관심자 목록 (embedded) |
| checklist | object | { upload, contact, negotiate, prepare } |
| createdAt | Timestamp | 생성시각 (정렬용) |
| updatedAt | Timestamp | 수정시각 |

### `settings/config`
| 필드 | 타입 | 설명 |
|------|------|------|
| categories | string[] | 카테고리 목록 |
| locations | string[] | 위치 목록 |

---

## 사진 업로드 정책
- **최대 4장**, 슬롯별 개별 업로드
- 클라이언트에서 Canvas API로 **≤500KB JPEG**으로 자동 리사이즈
- Storage 경로: `items/{itemId}/photo_{index}_{timestamp}.jpg`
- Storage 규칙: size < 600KB & contentType image/* (서버측 이중 검증)

---

## 환율 환산 (총 수익 표시)
- API: `https://open.er-api.com/v6/latest/USD` (무료, API 키 불필요)
- `localStorage` 키 `sell2ndhand_rates` 에 1시간 캐시
- 개별 물품 가격의 통화 단위는 등록 시 고정; 총 수익만 선택 통화로 환산
- API 실패 시 원래 금액 그대로 표시 (graceful degradation)

---

## CI/CD
- **트리거**: `main` 브랜치 push
- **GitHub Secret 필요**: `FIREBASE_SERVICE_ACCOUNT` (Firebase Console → 프로젝트 설정 → 서비스 계정 → 새 비공개 키 생성)
- `GITHUB_TOKEN`은 자동 제공

---

## 수정 시 유의사항
1. **모든 코드는 `public/index.html` 단일 파일** — 별도 JS/CSS 파일 생성 불필요
2. Firebase compat SDK v10 사용 중 (`firebase.firestore()`, `firebase.auth()` 등 네임스페이스 방식)
3. 새 인증 제공자 추가 시 `ALLOWED_EMAILS`, `firestore.rules`, `storage.rules` 모두 업데이트
4. Firestore 쓰기는 항상 `updatedAt: firebase.firestore.FieldValue.serverTimestamp()` 포함
5. 카테고리/위치 변경은 `settings/config`에 `arrayUnion`으로 저장 (두 사용자 동시 편집 안전)
