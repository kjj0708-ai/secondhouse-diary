# 세컨하우스 일지 (Second House Diary) — 외부 배포용 PRD

> **문서 버전**: v1.0  
> **작성일**: 2026-04-13  
> **개발 방식**: Claude Code 단독 개발 기준  
> **배포 형태**: PWA (Progressive Web App) + Netlify + Firebase

---

## 1. 제품 개요

### 1-1. 한 줄 정의
세컨하우스·텃밭 방문 기록, 지출 관리, 전년도 비교를 한 앱에서 해결하는 모바일 우선 PWA

### 1-2. 핵심 가치
- **기록**: 방문할 때마다 30초 안에 활동·지출 입력
- **비교**: 오늘 날짜 열면 작년 이맘때 뭘 했는지 바로 확인
- **수익**: 활동 태그 기반 어필리에이트 광고로 운영 수익 창출

### 1-3. 타깃 사용자
- 세컨하우스·주말농장·텃밭을 보유한 30~60대
- 매주말 방문하며 관리 기록을 남기고 싶은 사람
- 연간 지출을 파악하고 싶은 사람

---

## 2. 기술 스택

| 구분 | 기술 | 비고 |
|---|---|---|
| 프론트엔드 | Vanilla JS + HTML/CSS | 단일 파일 기반, 프레임워크 없음 |
| 인증 | Firebase Authentication | Google OAuth (지메일 로그인) |
| DB | Firebase Realtime Database | 사용자별 분리 저장 |
| 파일 저장 (유료) | Firebase Storage | 사진 업로드용 |
| 배포 | Netlify | 드래그앤드롭 또는 Git 연동 |
| PWA | Web App Manifest + Service Worker | 홈화면 설치, 오프라인 캐시 |
| 광고 | AliExpress Affiliate 링크 | 활동 태그 연동 배너 |

---

## 3. 사용자 플로우

```
앱 첫 접속
  ↓
Google 로그인 (Firebase Auth)
  ↓
개인정보 수집 동의 (필수) — 법적 요건
  ↓
메인 화면 (월 캘린더)
  ↓
날짜 탭 → 일지 입력 모달
  └ 상단: 작년 이맘때 기록 리스트 (텍스트)
  └ 중단: 방문여부 / 날씨 / 활동태그 / 메모 / 지출
  └ 하단: 활동태그 연동 어필리에이트 배너
  ↓
저장 → Firebase에 실시간 저장
```

---

## 4. 화면 구성

### 4-1. 탭 구조 (2개)
```
[📅 캘린더]  [📊 지출분석]
```

### 4-2. 캘린더 탭
- 월 캘린더 (평일 1 : 토·일 3 비율 Grid)
  - `grid-template-columns: repeat(5, 1fr) 3fr 3fr`
- 방문일: 초록 테두리
- 지출 있는 날: 금액 소계 표시 (₩3.2만)
- 활동 태그 이모지 최대 4개 표시 (주말), 2개 (평일)
- 하단 월 요약바: 방문일 수 / 월 지출(만원) / 활동 건수

### 4-3. 지출분석 탭
- 연도 선택 (< 2025 >)
- 카테고리별 도넛 차트 (SVG, 라이브러리 없음)
- 월별 바 차트 (1~12월)
- 연간 요약: 방문일·활동건수·지출합계·월평균

### 4-4. 일지 입력 모달 (날짜 탭 시)
```
[작년 이맘때] — 앞뒤 각 2주, 기록있는 날만 텍스트 리스트
  4월 1일  토  🌱텃밭 고추 모종 심기  ₩3.2만
  4월 8일  토  🔧수리 울타리 보수
  4월 12일 수  🌱텃밭 🧹청소          ← (작년 오늘, 황색 강조)

[방문 여부]  ✅방문 / ❌미방문

[날씨]  ☀️ 🌤️ 🌧️ ❄️

[활동 태그]  복수 선택
  기본값: 🌱텃밭 / 🔧시설관리 / 🍽️식사
  추가 태그: 사용자 직접 설정 가능

[활동 메모]  자유 텍스트

[지출 내역]  항목명 + 금액 + 카테고리 (시설유지/농업/식비/기타)

[어필리에이트 배너]  활동 태그 연동

[저장하기]
```

---

## 5. Firebase 데이터 구조

```json
/users/{uid}/
  profile:
    displayName: "홍길동"
    email: "user@gmail.com"
    plan: "free"            // "free" | "pro"
    createdAt: timestamp
    agreedAt: timestamp     // 개인정보 동의 시각

  settings:
    customTags:
      - { id: "tag_001", emoji: "🌱", label: "텃밭",    order: 0 }
      - { id: "tag_002", emoji: "🔧", label: "시설관리", order: 1 }
      - { id: "tag_003", emoji: "🍽️", label: "식사",    order: 2 }
      - { id: "tag_004", emoji: "🧹", label: "청소",    order: 3 }
      // 사용자가 추가/수정/삭제 가능

  entries:
    "2025-04-12":
      visited: true
      weather: "sunny"       // sunny | cloudy | rainy | snow
      tags: ["🌱텃밭", "🔧시설관리"]
      memo: "고추 모종 심기, 울타리 보수"
      expenses:
        - { label: "모종", amount: 12000, category: "농업" }
        - { label: "점심", amount: 8500,  category: "식비" }
      photos: []             // 유료 플랜에서만 사용
      updatedAt: timestamp
```

### Firebase Security Rules (기본 구조)
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

---

## 6. 인증 플로우

### 6-1. Google OAuth 로그인
```
앱 접속
  ↓
비로그인 상태 → 로그인 화면 표시
  ↓
"Google로 시작하기" 버튼
  ↓
Firebase Auth → Google OAuth 팝업
  ↓
최초 로그인 시 → 개인정보 수집 동의 화면
  ↓
동의 완료 → /users/{uid}/profile 생성
  ↓
메인 화면 진입
```

### 6-2. 개인정보 수집 동의 (법적 필수)
```
동의 항목:
□ [필수] 서비스 이용약관 동의
□ [필수] 개인정보 수집·이용 동의
  - 수집 항목: 이메일, 이름, 입력 데이터
  - 수집 목적: 서비스 제공, 서비스 개선
  - 보유 기간: 회원 탈퇴 시까지
□ [선택] 마케팅 활용 동의

→ 필수 미동의 시 앱 사용 불가
→ agreedAt 타임스탬프 Firebase에 저장
```

---

## 7. 활동 태그 시스템

### 7-1. 기본 태그 (앱 내장)
| 이모지 | 라벨 | 변경 가능 여부 |
|---|---|---|
| 🌱 | 텃밭 | 수정/삭제 가능 |
| 🔧 | 시설관리 | 수정/삭제 가능 |
| 🍽️ | 식사 | 수정/삭제 가능 |

### 7-2. 사용자 커스텀 태그
- 태그 추가: 이모지 선택 + 라벨 입력
- 태그 수정: 라벨·이모지 변경
- 태그 삭제: 확인 후 삭제 (기존 기록의 태그는 유지)
- 순서 변경: 드래그 or 위·아래 버튼
- 최대 개수: 무료 12개 / 유료 무제한

### 7-3. 태그 설정 화면 접근
```
헤더 우측 ⚙️ 아이콘 → 태그 설정
```

---

## 8. 어필리에이트 광고 시스템

### 8-1. 광고 위치
- 일지 입력 모달 하단 (저장하기 버튼 위)
- 고정 높이: 80px 배너 1개

### 8-2. 태그 연동 매핑 (활동 태그 → 제품 카테고리)

```javascript
const TAG_AD_MAP = {
  '🌱텃밭':    { label: '텃밭용품 보러가기', url: 'https://s.click.aliexpress.com/...' },
  '🔧시설관리': { label: '공구·자재 보러가기', url: 'https://s.click.aliexpress.com/...' },
  '🍽️식사':   { label: '주방용품 보러가기', url: 'https://s.click.aliexpress.com/...' },
  '🧹청소':   { label: '청소용품 보러가기', url: 'https://s.click.aliexpress.com/...' },
  '🌳조경':   { label: '조경·정원 보러가기', url: 'https://s.click.aliexpress.com/...' },
  '💧수도':   { label: '관개·호스용품 보러가기', url: 'https://s.click.aliexpress.com/...' },
  'default':  { label: '세컨하우스 용품 보러가기', url: 'https://s.click.aliexpress.com/...' }
};
```

### 8-3. 광고 노출 로직
```
선택된 태그 목록에서 첫 번째 매핑 태그 기준으로 광고 노출
태그 미선택 시 → default 광고 노출
```

### 8-4. 광고 UI
```
┌─────────────────────────────────────────┐
│ 🛒 텃밭용품 보러가기 →  aliexpress.com  │
│ 파트너 링크 · 구매 시 소정의 수수료 수익  │
└─────────────────────────────────────────┘
```
- 배경: 연한 amber
- 하단에 "파트너 링크" 명시 (광고 표시 의무)
- 탭 시 새 탭으로 열림

---

## 9. 플랜 구조 (무료 / 유료)

### 9-1. 무료 플랜 (Free)
- 일지 작성·조회
- 지출 분석
- 작년 이맘때 비교
- 커스텀 태그 최대 12개
- 데이터 내보내기 (JSON, CSV)
- **사진 업로드 불가**

### 9-2. 유료 플랜 (Pro) — 향후 구현
- 무료 플랜 전체 포함
- **날짜별 사진 첨부** (최대 5장/일)
- Firebase Storage 사용
- 커스텀 태그 무제한
- 광고 미표시

### 9-3. 유료 전환 준비 사항 (지금 코드에 반영할 것)
```javascript
// user.plan === 'pro' 체크 포인트
// 1. 사진 업로드 버튼 → pro만 활성화
// 2. 태그 12개 초과 추가 → pro 유도 팝업
// 3. 광고 배너 → pro면 숨김
// 4. Firebase Storage 업로드 → pro만 허용
```

---

## 10. PWA 설정

### 10-1. manifest.json
```json
{
  "name": "세컨하우스 일지",
  "short_name": "힐링일지",
  "description": "세컨하우스·텃밭 방문 기록 앱",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#f5f0e8",
  "theme_color": "#4a7c59",
  "orientation": "portrait",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

### 10-2. Service Worker 캐시 전략
```
캐시 대상: HTML, CSS, JS, 폰트, 아이콘
Firebase 통신: 네트워크 우선 (Network First)
오프라인 시: 캐시된 앱 UI 표시 + "오프라인 상태입니다" 토스트
```

### 10-3. 홈화면 설치 유도
- 첫 방문 3회 이후 하단 설치 배너 표시
- "홈화면에 추가하면 앱처럼 사용할 수 있어요" 안내

---

## 11. 파일 구조 (Claude Code 개발 기준)

```
/
├── index.html          ← 메인 앱 (단일 파일 or 분리)
├── manifest.json       ← PWA 설정
├── sw.js               ← Service Worker
├── firebase-config.js  ← Firebase 초기화 (gitignore 처리)
├── icons/
│   ├── icon-192.png
│   └── icon-512.png
└── .env                ← Firebase API Key 등 (gitignore)
```

---

## 12. 개발 우선순위 (Phase)

### Phase 1 — MVP (핵심 기능)
- [ ] Firebase 프로젝트 생성 및 설정
- [ ] Google 로그인 연동
- [ ] 개인정보 동의 화면
- [ ] 월 캘린더 (평일 1:토·일 3 비율)
- [ ] 일지 입력 모달 (방문/날씨/태그/메모/지출)
- [ ] 기본 태그 3개 (텃밭/시설관리/식사)
- [ ] Firebase Realtime DB 저장·조회
- [ ] 작년 이맘때 텍스트 리스트
- [ ] 어필리에이트 배너 (태그 연동)
- [ ] PWA manifest + Service Worker
- [ ] Netlify 배포

### Phase 2 — 완성도
- [ ] 지출분석 탭 (도넛·바 차트)
- [ ] 커스텀 태그 추가·수정·삭제
- [ ] JSON / CSV 내보내기
- [ ] 오프라인 지원 강화

### Phase 3 — 유료화
- [ ] 유료 플랜 분기 로직
- [ ] 사진 업로드 (Firebase Storage)
- [ ] 결제 연동 (추후 결정)
- [ ] 광고 제거 (Pro)

---

## 13. 환경변수 / Firebase 설정값

```javascript
// firebase-config.js (절대 Git에 올리지 말 것)
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  databaseURL: "https://YOUR_PROJECT-default-rtdb.firebaseio.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};
```

---

## 14. 주의사항 / 제약조건

| 항목 | 내용 |
|---|---|
| 광고 표시 의무 | 어필리에이트 배너에 "파트너 링크" 명시 필수 |
| 개인정보 동의 | 첫 로그인 시 필수 동의 화면, agreedAt 저장 |
| Firebase 무료 한도 | 동시접속 100명, 저장 1GB, 전송 10GB/월 |
| 사진 업로드 | Phase 3까지 UI는 표시하되 Pro 유도만 |
| 크로스 브라우저 | 크롬·사파리 기준 (모바일 우선) |
| 만든이 표기 | 앱 하단 "만든이 : 강화힐링아지트" 고정 |

---

## 15. 디자인 토큰

```css
:root {
  --bg:           #f5f0e8;  /* 배경 (크림) */
  --bg2:          #ede7d9;  /* 서브 배경 */
  --card:         #fff9f0;  /* 카드 배경 */
  --border:       #d4c9b0;  /* 테두리 */
  --text:         #2c2416;  /* 본문 */
  --text2:        #7a6a50;  /* 서브 텍스트 */
  --green:        #4a7c59;  /* 주색상 (방문/강조) */
  --green-light:  #e8f2ec;  /* 초록 연한 배경 */
  --amber:        #c17f2a;  /* 지출/광고 강조 */
  --amber-light:  #fdf3e3;  /* 앰버 연한 배경 */
  --red:          #c0392b;  /* 일요일/경고 */
  --blue:         #2980b9;  /* 토요일 */
}

/* 폰트 */
--font-display: 'DM Serif Display', serif;   /* 타이틀 */
--font-body:    'Noto Sans KR', sans-serif;  /* 본문 */
```

---

*이 PRD는 Claude Code에서 단독으로 개발할 수 있도록 모든 기획 결정을 포함합니다.*  
*개발 시 각 Phase를 순서대로 진행하고, Phase 1 완료 후 실사용 피드백을 반영하세요.*
