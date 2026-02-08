# 🎯 포켓몬 플리마켓 판매 관리 앱 (Flea Market POS)

## 📋 프로젝트 개요

- **프로젝트명**: 플리마켓 판매 관리 앱 (Flea Market POS)
- **배포 URL**: https://sales-data-23e78.web.app
- **기술 스택**: React 18, Firebase (Firestore + Hosting), Tailwind CSS
- **개발 시작일**: 2025-12-16
- **마지막 업데이트**: 2026-02-08

---

## 🏗️ 프로젝트 진화 과정

### Phase 1: 초기 개발 (2025-12-16 ~ 12-17)

**목표**: 플리마켓에서 클리커 등 상품 판매를 관리하는 웹앱 개발

**주요 기능**:
- 캐릭터별 재고 추적
- 판매 기록 (현금/Square 결제)
- 가격 계산 로직 (2+1 할인 알고리즘)
- 판매 수정/취소 기능

**기술 결정**:
- React 기반 SPA
- 초기에는 LocalStorage 사용
- 한국어 UI

### Phase 2: Firebase 연동 (2025-12-17 ~ 12-23)

**목표**: 클라우드 동기화로 여러 기기에서 데이터 공유

**주요 변경**:
- Firebase Firestore 연동
- 탭 네비게이션 (판매/히스토리)
- 일일 마감(Close Day) 기능
- 데이터 복원 워크플로우
- Firebase Hosting 배포

**배포 완료**: https://sales-data-23e78.web.app

### Phase 3: 기능 확장 (2025-12-18 ~ 12-19)

**추가 기능**:
- 카드 결제 옵션 (Venmo, Zelle, Square)
- 세율 설정 기능 (Square만 세금 적용)
- 인기 상품 통계 (1주/1달/커스텀)
- 메모 기능
- CSV 내보내기
- 한/영 언어 전환

### Phase 4: 캐릭터 변경 및 배포 (2025-12-23)

**변경 사항**:
- 원피스 캐릭터 → 포켓몬 21개로 변경
- Firebase Hosting 재배포
- 업데이트 워크플로우 정립

### Phase 5: 가격 시스템 재설계 (2026-01-06)

**주요 변경**:
- 카테고리별 가격 정책 도입
  - `algorithm`: 2+1 할인 적용 (Clicker, Sticker)
  - `fixed`: 개별 가격 (Keyring, Blind Bag)
- 커스텀 가격 입력 기능
- 감사 로그(Audit Log) 시스템 구현

### Phase 6: 데이터 구조 리팩토링 (2026-01-12 ~ 01-14)

**데이터 구조 개선**:

```javascript
// Before (분리된 구조)
characters: [...],
simpleItems: [...],
inventory: {...}

// After (통합 구조)
products: [
  { id, name, category, stock, price? }
]
```

**추가 개선**:
- `itemDetails` 배열로 판매 기록 상세화
- 스티커 할인 알고리즘 추가
- 마이그레이션 도구 개발

### Phase 7: 성능 최적화 (2026-01-21 ~ 01-25)

**React 최적화**:
- `useMemo` 적용 (9개) - 불필요한 재계산 방지
- `useReducer` 적용 (2개) - 상태 그룹화
- Map Lookup (`productsMap`, `categoriesMap`) - O(n) → O(1)
- Custom Hook (`useFirebaseData`) - 데이터 로딩 로직 추출

**상태 변수 축소**: 35개 → 21개

**Vite 마이그레이션 시도**: (진행 중, 현재는 CDN 방식 유지)

### Phase 8: 버그 수정 및 기능 추가 (2026-02-01 ~ 02-08)

**버그 수정**:
- NaN 표시 오류 해결 (신/구 데이터 구조 호환)
- 기록 재계산 기능 (🔄 Recalc 버튼)

**새 기능**:
- 카테고리 추가/수정/삭제
- 카테고리 이름 간소화 (한국어/영어 → 하나로)
- 카드 결제 상세 모달 (Venmo/Zelle/Square 분리)
- 중복 판매 방지 (로딩 상태 + 버튼 비활성화)
- 장소 입력 기능 (세금 보고용)
- 기존 기록에 장소 추가 기능

---

## 📁 현재 파일 구조

```
clicker-deploy/
├── public/
│   └── index.html          ← 메인 앱 (단일 파일, ~2000줄)
├── functions/              ← Square 연동용 (새로 추가 예정)
│   ├── package.json
│   └── index.js
└── firebase.json
```

---

## 🗄️ Firebase 데이터 구조

```
Firestore
└── store (컬렉션)
    ├── products (문서)
    │   └── value: [
    │       { id, name, category, stock, price? }
    │   ]
    ├── sales (문서)
    │   └── value: [
    │       { id, timestamp, price, tax, payment, itemDetails, status }
    │   ]
    ├── dailyRecords (문서)
    │   └── value: [
    │       { id, date, dateString, location, sales, summary }
    │   ]
    ├── categories (문서)
    │   └── value: [
    │       { id, name, emoji, priceType }
    │   ]
    ├── taxRate (문서)
    │   └── value: 9.5
    ├── currentLocation (문서)
    │   └── value: "Rose Bowl"
    └── logs (문서)
        └── value: [
            { id, timestamp, action, details }
        ]
```

---

## 🔧 핵심 알고리즘

### 1. 가격 계산 알고리즘 (2+1 할인)

```javascript
const calculatePrice = (qty, basePrice = 12) => {
  // 2개 묶음당 $8 할인
  // 1개: $12, 2개: $20, 3개: $32, 4개: $40...
  const bundles = Math.floor(qty / 2);
  const remainder = qty % 2;
  return (bundles * 20) + (remainder * basePrice);
};
```

**적용 카테고리**: Clicker, Sticker

### 2. 결제 정보 호환 헬퍼

```javascript
// 새/구 데이터 구조 모두 지원
const getPayment = (sale) => ({
  method: sale.payment?.method || sale.paymentMethod,
  type: sale.payment?.type || sale.cardType
});
```

### 3. Firebase 실시간 동기화

```javascript
const useFirebaseData = (collectionName, defaultValue) => {
  const [data, setData] = useState(defaultValue);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const unsubscribe = db.collection('store').doc(collectionName)
      .onSnapshot(doc => {
        if (doc.exists) setData(doc.data().value);
        setLoading(false);
      });
    return () => unsubscribe();
  }, []);

  const saveData = async (newData) => {
    await db.collection('store').doc(collectionName).set({ 
      value: newData, 
      updatedAt: new Date().toISOString() 
    });
    setData(newData);
  };

  return [data, saveData, loading];
};
```

---

## 📊 카테고리 설정

```javascript
const DEFAULT_CATEGORIES = [
  { id: 'clicker', name: 'Clicker', emoji: '🎯', priceType: 'algorithm' },
  { id: 'keyring', name: 'Keyring', emoji: '🔑', priceType: 'fixed' },
  { id: 'sticker', name: 'Sticker', emoji: '✨', priceType: 'algorithm' },
  { id: 'blindbag', name: 'Blind Bag', emoji: '🎁', priceType: 'fixed' }
];

const CARD_OPTIONS = [
  { id: 'venmo', name: 'Venmo', emoji: '💙', hasTax: false },
  { id: 'zelle', name: 'Zelle', emoji: '💜', hasTax: false },
  { id: 'square', name: 'Square', emoji: '⬛', hasTax: true }
];
```

---

## 🎛️ 주요 상태 변수

### 현재 사용 중인 상태 (36개)

```javascript
// UI 상태
const [activeTab, setActiveTab] = useState('sales');
const [lang, setLang] = useState('en');
const [loading, setLoading] = useState(true);

// 모달 상태
const [showCloseModal, setShowCloseModal] = useState(false);
const [showProductModal, setShowProductModal] = useState(false);
const [showCategoryModal, setShowCategoryModal] = useState(false);
const [showTaxModal, setShowTaxModal] = useState(false);
const [showCardDetailModal, setShowCardDetailModal] = useState(false);
const [showResetModal, setShowResetModal] = useState(false);
const [viewingRecord, setViewingRecord] = useState(null);

// 판매 상태 (useReducer)
const [saleState, saleDispatch] = useReducer(saleReducer, initialSaleState);
const [editState, editDispatch] = useReducer(editReducer, initialEditState);

// Firebase 데이터
const [products, saveProducts] = useFirebaseData('products', []);
const [sales, saveSales] = useFirebaseData('sales', []);
const [dailyRecords, saveDailyRecords] = useFirebaseData('dailyRecords', []);
const [taxRate, saveTaxRate] = useFirebaseData('taxRate', 9.5);
const [logs, saveLogs] = useFirebaseData('logs', []);
const [categories, saveCategories] = useFirebaseData('categories', []);
const [currentLocation, saveCurrentLocation] = useFirebaseData('currentLocation', '');

// 로컬 상태
const [localLocation, setLocalLocation] = useState('');
const [isSaleProcessing, setIsSaleProcessing] = useState(false);
```

### useMemo로 최적화된 값 (9개)

```javascript
const categoriesMap = useMemo(() => {...}, [categories]);
const productsMap = useMemo(() => {...}, [products]);
const productsByCategory = useMemo(() => {...}, [products, categories]);
const t = useMemo(() => translations[lang], [lang]);
// ... 등
```

---

## 📱 UI 구조

### Sales 탭
```
┌─────────────────────────────────────┐
│  🎯 Flea Market POS                 │
│  Inventory & Sales Tracker          │
├─────────────────────────────────────┤
│  📍 [장소 입력]                      │
├─────────────────────────────────────┤
│  📊 Today                           │
│  $150.00 | 💵 $50 | 💳 $100 ▾      │
│  5건 / 12개         [🌙 Close Day]  │
├─────────────────────────────────────┤
│  💰 상품 선택              [✏️ 관리] │
│  ┌─────┬─────┬─────┬─────┐        │
│  │🎯   │🔑   │✨   │🎁   │ 카테고리│
│  └─────┴─────┴─────┴─────┘        │
│  [Mew +] [Pikachu +] [Ditto +]...  │
├─────────────────────────────────────┤
│  🛒 현재 선택: 3개                  │
│  Mew ×2, Pikachu ×1                │
│  [Custom] [Cash] [Card]            │
├─────────────────────────────────────┤
│  [$32.00 Sale Complete ✓]          │
└─────────────────────────────────────┘
```

### History 탭
```
┌─────────────────────────────────────┐
│  📊 History                         │
│  [🔄 Recalc]                        │
├─────────────────────────────────────┤
│  Sun, Feb 8, 2026         $295.31  │
│  📍 Rose Bowl                       │
│  19건 / 25개                        │
│  💵 $150 | 💳 $145 ▾               │
│  [📋 Detail]                        │
├─────────────────────────────────────┤
│  Sat, Feb 7, 2026         $180.00  │
│  📍 Pasadena                        │
│  ...                                │
└─────────────────────────────────────┘
```

---

## 🔌 Square 연동 (진행 예정)

### 목표
Square POS에서 결제 시 자동으로 Firebase 인벤토리 감소

### 구현 방식
```
Square 결제 (예: "Mew" 2개)
        ↓
Square Webhook 전송
        ↓
Firebase Functions 수신
        ↓
상품 이름 매칭 → 재고 감소
        ↓
Firebase Firestore 업데이트
```

### 필요 파일
- `functions/index.js` - Webhook 처리 로직
- `functions/package.json` - 의존성

### 설정 필요 사항
1. Square Developer Console에서 Webhook 등록
2. Firebase Functions 배포
3. Square 상품 이름 = Firebase 상품 이름 (일치 필요)

---

## 🐛 알려진 이슈 및 해결된 버그

### 해결된 버그
| 버그 | 원인 | 해결 |
|------|------|------|
| NaN 표시 | 신/구 데이터 구조 불일치 | `getPayment` 헬퍼 함수 |
| 중복 판매 | 버튼 연속 클릭 | `isSaleProcessing` 상태 |
| Keyring 가격 수정 불가 | `price !== undefined` 조건 | `isFixedPrice` 조건 추가 |

### 잠재적 이슈
| 이슈 | 상태 | 비고 |
|------|------|------|
| Clicker/Sticker 개별 가격 수정 | 미해결 | algorithm 카테고리 특성 |
| 장소 입력 blur 저장 | 개선 완료 | 매 글자 → blur 시 저장 |

---

## 🚀 배포 방법

### 일반 업데이트
```bash
cd clicker-deploy
firebase deploy
```

### Functions 포함 배포
```bash
cd clicker-deploy
cd functions && npm install && cd ..
firebase deploy
```

---

## 📝 최근 세션 변경 사항 (2026-02-08)

### 이번 세션에서 추가된 기능

1. **Keyring 가격 수정 기능**
   - fixed price 카테고리도 가격 수정 가능

2. **장소 입력 기능**
   - Sales 탭에 장소 입력 필드 추가
   - Close Day 시 장소 저장
   - History에 장소 표시

3. **기존 기록 장소 추가**
   - Detail 모달에서 장소 수정 가능

4. **코드 개선**
   - `getPayment` 함수 전역으로 통합 (중복 4개 → 1개)
   - 장소 입력 최적화 (매 글자 저장 → blur 시 저장)

5. **Square 연동 준비**
   - Firebase Functions 코드 작성
   - Webhook 처리 로직 구현
   - 이름 매칭 방식 설계

---

## 💡 향후 개선 아이디어

1. **Square 연동 완료** - 자동 인벤토리 동기화
2. **GitHub 연동** - 버전 관리 및 자동 배포
3. **PWA 지원** - 오프라인 사용
4. **다중 사용자** - Firebase Auth 추가
5. **대시보드** - 매출 분석 차트

---

## 📞 기술 지원

### 파일 위치
- 메인 앱: `/home/claude/index.html`
- Square 연동: `/home/claude/square-integration/`
- 출력 파일: `/mnt/user-data/outputs/`

### 배포된 앱
- URL: https://sales-data-23e78.web.app
- Firebase 프로젝트: sales-data-23e78

---

*마지막 업데이트: 2026-02-08*
