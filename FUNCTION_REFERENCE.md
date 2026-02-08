# 🔧 함수 및 컴포넌트 레퍼런스

## 📋 목차

1. [Firebase 관련 함수](#firebase-관련-함수)
2. [판매 처리 함수](#판매-처리-함수)
3. [카테고리/상품 관리 함수](#카테고리상품-관리-함수)
4. [통계 및 유틸리티 함수](#통계-및-유틸리티-함수)
5. [UI 헬퍼 함수](#ui-헬퍼-함수)

---

## Firebase 관련 함수

### `useFirebaseData(collectionName, defaultValue)`

Firebase Firestore와 실시간 동기화하는 커스텀 훅

```javascript
const useFirebaseData = (collectionName, defaultValue) => {
  const [data, setData] = useState(defaultValue);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // 실시간 구독
    const unsubscribe = db.collection('store').doc(collectionName)
      .onSnapshot(doc => {
        if (doc.exists) {
          setData(doc.data().value);
        }
        setLoading(false);
      });
    return () => unsubscribe();
  }, [collectionName]);

  const saveData = async (newData) => {
    try {
      await db.collection('store').doc(collectionName).set({ 
        value: newData, 
        updatedAt: new Date().toISOString() 
      });
      setData(newData);
    } catch (error) {
      console.error('Save error:', error);
    }
  };

  return [data, saveData, loading];
};
```

**사용 예시:**
```javascript
const [products, saveProducts, productsLoading] = useFirebaseData('products', []);
const [sales, saveSales, salesLoading] = useFirebaseData('sales', []);
```

---

### `addLog(action, details)`

감사 로그 추가

```javascript
const addLog = async (action, details) => {
  const newLog = { id: Date.now(), timestamp: new Date().toISOString(), action, details };
  await saveLogs([newLog, ...logs].slice(0, 1000)); // 최근 1000개만 유지
};
```

**액션 종류:**
- `sale_created` - 판매 생성
- `sale_cancelled` - 판매 취소
- `sale_edited` - 판매 수정
- `square_sync` - Square 동기화

---

## 판매 처리 함수

### `handleSale()`

판매 완료 처리 (중복 방지 포함)

```javascript
const handleSale = async () => {
  if (totalQty === 0 || isSaleProcessing) return;
  
  setIsSaleProcessing(true); // 중복 클릭 방지
  
  try {
    // 1. 판매 데이터 생성
    const newSale = {
      id: Date.now(),
      timestamp: new Date().toISOString(),
      price: finalPrice,
      tax: taxAmount,
      payment: { method, type },
      category: { id, name, isMixed },
      itemDetails: [...],
      totalQty,
      status: 'completed'
    };
    
    // 2. 재고 감소
    await updateMultipleStock(stockChanges);
    
    // 3. 판매 저장
    await saveSales([newSale, ...sales]);
    
    // 4. 로그 기록
    await addLog('sale_created', { ... });
    
    // 5. 상태 초기화
    saleDispatch({ type: 'CLEAR' });
  } finally {
    setIsSaleProcessing(false);
  }
};
```

---

### `handleCancelSale(sale)`

판매 취소 (재고 복원)

```javascript
const handleCancelSale = async (sale) => {
  // 1. 재고 복원
  const stockChanges = sale.itemDetails.map(item => ({
    itemId: item.id,
    delta: item.qty // 양수로 복원
  }));
  await updateMultipleStock(stockChanges);
  
  // 2. 판매 상태 변경
  await saveSales(sales.map(s => 
    s.id === sale.id 
      ? { ...s, status: 'cancelled', cancelledAt: new Date().toISOString() }
      : s
  ));
  
  // 3. 로그 기록
  await addLog('sale_cancelled', { ... });
};
```

---

### `handleSaveEdit()`

판매 수정 (가격/결제방법 변경)

```javascript
const handleSaveEdit = async () => {
  const oldSale = editState.editingSale;
  
  // 1. 재고 차이 계산 (기존 - 새로운)
  const stockChanges = calculateStockDiff(oldSale.itemDetails, newItemDetails);
  
  // 2. 가격 재계산
  const updatedSale = {
    ...oldSale,
    price: newPrice,
    tax: newTax,
    payment: newPayment,
    itemDetails: newItemDetails,
    editedAt: new Date().toISOString()
  };
  
  // 3. 저장
  await updateMultipleStock(stockChanges);
  await saveSales(sales.map(s => s.id === oldSale.id ? updatedSale : s));
  await addLog('sale_edited', { ... });
};
```

---

### `handleDayClose()`

일일 마감 (판매 → 기록 이동)

```javascript
const handleDayClose = async () => {
  const activeSales = sales.filter(s => s.status !== 'cancelled');
  
  // 1. 요약 통계 계산
  const summary = {
    totalRevenue: activeSales.reduce((s, x) => s + x.price, 0),
    totalTax: activeSales.reduce((s, x) => s + (x.tax || 0), 0),
    cashRevenue: activeSales.filter(x => getPayment(x).method === 'cash').reduce(...),
    cardRevenue: activeSales.filter(x => getPayment(x).method === 'card').reduce(...),
    venmoRevenue: ...,
    zelleRevenue: ...,
    squareRevenue: ...,
    totalItems: ...,
    transactionCount: activeSales.length
  };
  
  // 2. 새 기록 생성
  const newRecord = {
    id: Date.now(),
    date: new Date().toISOString(),
    dateString: new Date(sales[0].timestamp).toDateString(),
    location: localLocation || '',
    sales: [...sales],
    summary,
    productsSnapshot: products.map(p => ({ id: p.id, name: p.name, stock: p.stock }))
  };
  
  // 3. 저장 및 초기화
  await saveDailyRecords([newRecord, ...dailyRecords]);
  await saveSales([]);
};
```

---

## 카테고리/상품 관리 함수

### `handleAddCategory()`

새 카테고리 추가

```javascript
const handleAddCategory = async () => {
  const newCategory = {
    id: `cat_${Date.now()}`,
    name: newCategoryName.trim(),
    emoji: newCategoryEmoji || '📦',
    priceType: 'fixed' // 새 카테고리는 기본 fixed
  };
  
  await saveCategories([...categories, newCategory]);
  resetCategoryForm();
  setShowCategoryModal(false);
};
```

---

### `handleEditCategory()`

카테고리 수정

```javascript
const handleEditCategory = async () => {
  const updated = categories.map(c => 
    c.id === editingCategory.id 
      ? { 
          ...c, 
          name: newCategoryName.trim(),
          emoji: newCategoryEmoji
          // priceType은 변경하지 않음 (기존 유지)
        }
      : c
  );
  
  await saveCategories(updated);
  resetCategoryForm();
};
```

---

### `handleDeleteCategory(categoryId)`

카테고리 삭제 (기본 4개 보호)

```javascript
const handleDeleteCategory = async (categoryId) => {
  // 기본 카테고리는 삭제 불가
  if (['clicker', 'sticker', 'keyring', 'blindbag'].includes(categoryId)) {
    return;
  }
  
  // 해당 카테고리 상품도 삭제
  await saveProducts(products.filter(p => p.category !== categoryId));
  await saveCategories(categories.filter(c => c.id !== categoryId));
};
```

---

### `handleAddProduct()`

새 상품 추가

```javascript
const handleAddProduct = async () => {
  const category = categories.find(c => c.id === selectedCategoryForProduct);
  
  const newProduct = {
    id: `item_${Date.now()}`,
    name: newProductName.trim(),
    category: selectedCategoryForProduct,
    stock: parseInt(newProductStock) || 0,
    ...(category?.priceType === 'fixed' && { price: parseFloat(newProductPrice) || 0 })
  };
  
  await saveProducts([...products, newProduct]);
};
```

---

## 통계 및 유틸리티 함수

### `getCurrentStats()`

오늘 판매 통계 계산

```javascript
const getCurrentStats = () => {
  const activeSales = sales.filter(s => s.status !== 'cancelled');
  
  return {
    totalRevenue: activeSales.reduce((s, x) => s + x.price, 0),
    totalTax: activeSales.reduce((s, x) => s + (x.tax || 0), 0),
    cashRevenue: activeSales.filter(x => getPayment(x).method === 'cash').reduce((s, x) => s + x.price, 0),
    cardRevenue: activeSales.filter(x => getPayment(x).method === 'card').reduce((s, x) => s + x.price, 0),
    venmoRevenue: ...,
    zelleRevenue: ...,
    squareRevenue: ...,
    totalItems: activeSales.reduce((s, x) => s + (x.totalQty || x.totalQuantity || 0), 0),
    count: activeSales.length,
    dateLabel: '...'
  };
};
```

---

### `calculatePrice(qty, basePrice)`

2+1 할인 가격 계산

```javascript
const calculatePrice = (qty, basePrice = 12) => {
  // 2개 묶음당 $8 할인
  // 1개: $12, 2개: $20, 3개: $32, 4개: $40...
  const bundles = Math.floor(qty / 2);
  const remainder = qty % 2;
  return (bundles * 20) + (remainder * basePrice);
};
```

**가격표:**
| 수량 | 가격 |
|------|------|
| 1개 | $12 |
| 2개 | $20 |
| 3개 | $32 |
| 4개 | $40 |
| 5개 | $52 |

---

### `recalculateAllRecords()`

모든 기록 재계산 (🔄 Recalc 버튼)

```javascript
const recalculateAllRecords = async () => {
  if (dailyRecords.length === 0) return;
  
  const updatedRecords = dailyRecords.map(record => {
    const salesList = record.sales || [];
    const activeSales = salesList.filter(s => s.status !== 'cancelled');
    
    const newSummary = {
      totalRevenue: activeSales.reduce((s, x) => s + (x.price || 0), 0),
      // ... 모든 통계 재계산
    };
    
    return { ...record, summary: newSummary };
  });
  
  await saveDailyRecords(updatedRecords);
  alert('Records recalculated!');
};
```

---

## UI 헬퍼 함수

### `getPayment(sale)`

결제 정보 추출 (새/구 구조 호환)

```javascript
const getPayment = (sale) => ({
  method: sale.payment?.method || sale.paymentMethod,
  type: sale.payment?.type || sale.cardType
});
```

---

### `formatCurrency(amount)`

통화 포맷팅

```javascript
const formatCurrency = (amount) => {
  return '$' + (amount || 0).toFixed(2);
};
```

---

### `formatDate(dateString, lang)`

날짜 포맷팅

```javascript
const formatDate = (dateString, lang) => {
  const date = new Date(dateString);
  return date.toLocaleDateString(
    lang === 'ko' ? 'ko-KR' : 'en-US', 
    { weekday: 'short', month: 'short', day: 'numeric', year: 'numeric' }
  );
};
```

---

### `getCategoryEmoji(categoryId)`

카테고리 이모지 반환

```javascript
const getCategoryEmoji = (categoryId) => {
  const cat = categoriesMap.get(categoryId);
  return cat?.emoji || '📦';
};
```

---

### `getPaymentLabel(sale)`

결제 방법 라벨

```javascript
const getPaymentLabel = (sale) => {
  const p = getPayment(sale);
  return p.method === 'cash' ? '💵' : '💳';
};
```

---

### `getPaymentTypeName(sale)`

카드 타입 이름

```javascript
const getPaymentTypeName = (sale) => {
  const p = getPayment(sale);
  if (p.method !== 'card') return '';
  
  const option = CARD_OPTIONS.find(o => o.id === p.type);
  return option?.name || '';
};
```

---

## 🔄 useReducer 상태 관리

### saleReducer (판매 상태)

```javascript
const initialSaleState = {
  selectedItems: {},     // { [productId]: qty }
  paymentMethod: 'cash',
  cardType: 'venmo',
  customPrice: '',
  customPriceHasTax: false,
  showMemo: false,
  memo: ''
};

const saleReducer = (state, action) => {
  switch (action.type) {
    case 'SET_ITEM':
      return { ...state, selectedItems: { ...state.selectedItems, [action.id]: action.qty } };
    case 'SET_PAYMENT':
      return { ...state, paymentMethod: action.method };
    case 'SET_CARD_TYPE':
      return { ...state, cardType: action.cardType };
    case 'SET_CUSTOM_PRICE':
      return { ...state, customPrice: action.price };
    case 'SET_MEMO':
      return { ...state, memo: action.memo };
    case 'CLEAR':
      return initialSaleState;
    default:
      return state;
  }
};
```

---

### editReducer (수정 상태)

```javascript
const initialEditState = {
  editingSale: null,
  editSelectedItems: {},
  editPaymentMethod: 'cash',
  editCardType: 'venmo',
  editCustomPrice: '',
  editMemo: ''
};

const editReducer = (state, action) => {
  switch (action.type) {
    case 'START_EDIT':
      return {
        editingSale: action.sale,
        editSelectedItems: /* 기존 아이템 복원 */,
        editPaymentMethod: /* 기존 결제 방법 */,
        // ...
      };
    case 'SET_EDIT_ITEM':
      return { ...state, editSelectedItems: { ...state.editSelectedItems, [action.id]: action.qty } };
    case 'CLEAR_EDIT':
      return initialEditState;
    default:
      return state;
  }
};
```

---

*마지막 업데이트: 2026-02-08*
