# Memoization va Caching Strategiyalari — To'liq Qo'llanma

> JavaScript/React texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [Memoization nima va u qayerdan paydo bo'lgan?](#1-memoization-nima-va-u-qayerdan-paydo-bolgan)
2. [Oddiy memoization'ni noldan yozish](#2-oddiy-memoizationni-noldan-yozish)
3. [`useMemo` ichki mexanizmi qanday ishlaydi?](#3-usememo-ichki-mexanizmi-qanday-ishlaydi)
4. [`useMemo` vs `useCallback` — farqi](#4-usememo-vs-usecallback--farqi)
5. [WeakMap bilan memoization](#5-weakmap-bilan-memoization)
6. [Memoization turlari va strategiyalari](#6-memoization-turlari-va-strategiyalari)
7. [Loyihaning qaysi qismlarida ishlatiladi?](#7-loyihaning-qaysi-qismlarida-ishlatiladi)
8. [Qanday to'g'ri ishlatiladi — amaliy qoidalar](#8-qanday-togri-ishlatiladi--amaliy-qoidalar)
9. [Nega bu muhim — real foydalar](#9-nega-bu-muhim--real-foydalar)
10. [Eng ko'p uchraydigan xatolar](#10-eng-kop-uchraydigan-xatolar)
11. [Interview savollari va qisqa javoblar](#11-interview-savollari-va-qisqa-javoblar)
12. [Xulosa](#12-xulosa)

---

## 1. Memoization nima va u qayerdan paydo bo'lgan?

**Memoization** — bu funksiya chaqiruvining natijasini **eslab qolish** (keshlash) va, agar funksiya **bir xil argumentlar bilan qayta chaqirilsa**, hisoblashni takrorlamasdan, **oldin saqlangan natijani qaytarish** texnikasi.

### Nomi qayerdan kelib chiqqan?

"Memoization" atamasi **1968-yilda**, informatika olimi **Donald Michie** tomonidan kiritilgan (lotincha "memorandum" — "eslab qolinishi kerak narsa" so'zidan). Bu — dasturlashning eng qadimgi va fundamental **optimizatsiya texnikalaridan** biri bo'lib, funksional dasturlash tillarida (Lisp, Haskell) va dinamik dasturlashda (algoritmlarda) keng qo'llaniladi.

### Asosiy g'oya — sodda misol

```javascript
function slowSquare(n) {
  console.log('Hisoblanmoqda...', n);
  // tasavvur qiling, bu juda "og'ir" hisob-kitob
  for (let i = 0; i < 1e9; i++) {} // sun'iy kechikish
  return n * n;
}

console.log(slowSquare(5)); // "Hisoblanmoqda..." chiqadi, sekin ishlaydi
console.log(slowSquare(5)); // YANA "Hisoblanmoqda..." chiqadi, YANA sekin ishlaydi!
```

Bu yerda `slowSquare(5)` **ikki marta**, bir xil natija (`25`) uchun, ikkalasida ham **to'liq hisob-kitob** bilan chaqiriladi — bu **isrofgarchilik**. Memoization aynan shu muammoni hal qiladi: agar funksiya **bir xil kirish** bilan **avval chaqirilgan bo'lsa**, natija **keshdan** olinadi, qayta hisoblanmaydi.

---

## 2. Oddiy memoization'ni noldan yozish

### Asosiy implementatsiya

```javascript
function memoize(fn) {
  const cache = new Map();

  return function (...args) {
    const key = JSON.stringify(args); // argumentlarni "kalit"ga aylantirish

    if (cache.has(key)) {
      console.log('Keshdan olindi:', key);
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
```

### Ishlatish misoli

```javascript
function slowSquare(n) {
  console.log('Hisoblanmoqda...', n);
  for (let i = 0; i < 1e9; i++) {}
  return n * n;
}

const memoizedSquare = memoize(slowSquare);

console.log(memoizedSquare(5)); // "Hisoblanmoqda... 5" - sekin, lekin 1 marta
console.log(memoizedSquare(5)); // "Keshdan olindi: [5]" - DARHOL, hisob-kitobsiz!
console.log(memoizedSquare(10)); // "Hisoblanmoqda... 10" - yangi argument, qayta hisoblanadi
```

### Cheklov: `JSON.stringify` orqali kalit yaratish muammosi

```javascript
// XATO - obyektlarni JSON.stringify orqali solishtirish noaniq bo'lishi mumkin
memoizedFn({ a: 1, b: 2 });
memoizedFn({ b: 2, a: 1 }); // JSON.stringify TURLI natija beradi, garchi mantiqan bir xil obyekt bo'lsa ham!
```

Bu — oddiy memoization'ning eng katta cheklovlaridan biri: **primitive qiymatlar** (son, satr) uchun yaxshi ishlaydi, lekin **obyektlar** uchun to'g'ri "kalit" yaratish murakkabroq masala (bu yerda **WeakMap** yordamga keladi — 5-bo'limga qarang).

---

## 3. `useMemo` ichki mexanizmi qanday ishlaydi?

React'ning `useMemo` — bu **komponent darajasidagi** memoization hook'i bo'lib, u **qimmat hisob-kitobning natijasini**, komponent qayta render bo'lganda, agar **bog'liqliklar (dependencies) o'zgarmagan bo'lsa**, qayta hisoblamasdan saqlab qoladi.

```javascript
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

### Ichki mexanizm — soddalashtirilgan model

React har bir komponent instance'i uchun **ichki xotira** (fiber node ichida, "hooks" ro'yxati) saqlaydi. `useMemo` chaqirilganda, u shu xotiraga **oldingi dependency massivi** va **oldingi hisoblangan qiymat**ni yozib qo'yadi.

```javascript
// React ICHIDA (soddalashtirilgan, tushuntirish uchun model)
function useMemo(calculateValue, dependencies) {
  const hook = getCurrentHook(); // hozirgi komponentning "hooks" xotirasidan

  if (hook.deps && areDepsEqual(hook.deps, dependencies)) {
    // Bog'liqliklar O'ZGARMAGAN - eski qiymatni qaytaramiz
    return hook.value;
  }

  // Bog'liqliklar O'ZGARGAN (yoki bu BIRINCHI render) - qayta hisoblaymiz
  const newValue = calculateValue();
  hook.deps = dependencies;
  hook.value = newValue;
  return newValue;
}

function areDepsEqual(prevDeps, nextDeps) {
  if (prevDeps.length !== nextDeps.length) return false;
  for (let i = 0; i < prevDeps.length; i++) {
    if (!Object.is(prevDeps[i], nextDeps[i])) return false; // Object.is - "===" ga o'xshash
  }
  return true;
}
```

### To'liq misol

```jsx
function ProductList({ products, filter }) {
  // filter yoki products o'zgarmasa, filterlash QAYTA BAJARILMAYDI
  const filteredProducts = useMemo(() => {
    console.log('Filtrlanmoqda...'); // faqat kerak bo'lganda chiqadi
    return products.filter((p) => p.category === filter);
  }, [products, filter]);

  return (
    <ul>
      {filteredProducts.map((p) => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

Agar ota komponent qayta render bo'lsa-yu, lekin `products` va `filter` **o'zgarmagan** bo'lsa, `filteredProducts` **qayta hisoblanmaydi** — React oldingi render'dagi natijani "eslab qoladi" va uni qaytaradi.

### Muhim: dependency solishtirish **reference (havola) bo'yicha** ishlaydi

```javascript
const [obj, setObj] = useState({ count: 0 });

const memoized = useMemo(() => compute(obj), [obj]);

// obj'ni "yangilash" - lekin YANGI OBYEKT yaratiladi (reference o'zgaradi!)
setObj({ count: obj.count + 1 }); // memoized QAYTA hisoblanadi, hatto qiymat "mazmunan" farq qilmasa ham
```

React `Object.is()` (deyarli `===` bilan bir xil) orqali solishtiradi — bu **strukturaviy (deep) solishtirish emas**, balki **reference solishtirish**. Shuning uchun har safar yangi obyekt/massiv yaratilsa (hatto ichidagi qiymatlar bir xil bo'lsa ham), `useMemo` uni "o'zgargan" deb hisoblaydi.

> **Interview uchun muhim eslatma:** React rasmiy hujjatlarida `useMemo` — bu **kafolat emas, balki optimallashtirish maslahati** (performance hint) sifatida ta'riflanadi. React kelajakda (masalan, xotira tejash uchun) keshlangan qiymatni "unutib qo'yishi" va qayta hisoblashi **mumkin** — shuning uchun `useMemo`ga hech qachon **dasturning to'g'ri ishlashi** (masalan, side-effect'lardan qochish uchun) tayanib bo'lmaydi, faqat **performance optimizatsiyasi** sifatida ishlatiladi.

---

## 4. `useMemo` vs `useCallback` — farqi

```javascript
// useMemo - QIYMATNI memoize qiladi
const memoizedValue = useMemo(() => computeValue(a, b), [a, b]);

// useCallback - FUNKSIYANING O'ZINI memoize qiladi
const memoizedFn = useCallback(() => doSomething(a, b), [a, b]);
```

Aslida, `useCallback(fn, deps)` — bu shunchaki `useMemo(() => fn, deps)`ning **qulaylashtirilgan varianti**:

```javascript
// Bu ikkalasi FUNKSIONAL JIHATDAN bir xil
useCallback(fn, deps);
useMemo(() => fn, deps);
```

**Farqi:** `useMemo` — funksiyani **chaqiradi** va uning **natijasini** saqlaydi. `useCallback` — funksiyaning **o'zini** (chaqirmasdan) saqlaydi. `useCallback` odatda callback funksiyalarni **bola komponentlarga** `props` sifatida uzatishda, `React.memo` bilan birga ishlatiladi — bu funksiyaning **reference**i har render'da o'zgarmasligi uchun kerak.

```jsx
const handleClick = useCallback(() => {
  console.log('bosildi');
}, []); // dependencies bo'sh - funksiya HECH QACHON qayta yaratilmaydi

// Bolaga uzatilganda, agar bola React.memo bilan o'ralgan bo'lsa,
// u KERAKSIZ qayta render bo'lmaydi, chunki handleClick reference'i bir xil qoladi
<ChildComponent onClick={handleClick} />
```

---

## 5. WeakMap bilan memoization

Oddiy `Map` bilan memoization qilinganda, agar **kalit sifatida obyektlar** ishlatilsa, bu obyektlarga **strong reference** saqlanadi — ya'ni ular hech qachon Garbage Collector tomonidan tozalanmaydi, hatto boshqa hech kim ularni ishlatmasa ham. Bu — **memory leak**ga olib kelishi mumkin (bu mavzuni "Closure va Xotira" hujjatida batafsil ko'rgan edik).

### Muammoli holat — oddiy `Map` bilan

```javascript
const cache = new Map();

function processUser(userObj) {
  if (cache.has(userObj)) {
    return cache.get(userObj);
  }
  const result = expensiveComputation(userObj);
  cache.set(userObj, result);
  return result;
}

let user = { id: 1, name: 'Ali' };
processUser(user);

user = null; // "user"ni tashladik, deb o'ylaymiz
// LEKIN cache hali ham asl obyektga strong reference saqlab turibdi!
// GC uni tozalay olmaydi - MEMORY LEAK!
```

### Yechim — `WeakMap` bilan memoization

```javascript
const cache = new WeakMap();

function processUser(userObj) {
  if (cache.has(userObj)) {
    console.log('Keshdan olindi');
    return cache.get(userObj);
  }
  const result = expensiveComputation(userObj);
  cache.set(userObj, result);
  return result;
}

let user = { id: 1, name: 'Ali' };
processUser(user); // hisoblanadi, keshga qo'yiladi
processUser(user); // "Keshdan olindi" - bir xil obyekt reference'i

user = null;
// Endi GC obyektni erkin tozalay oladi,
// va u bilan birga cache'dagi mos yozuv ham AVTOMATIK yo'qoladi!
```

### `WeakMap` memoization'da nega ishlaydi?

`WeakMap`ning kalitlari **faqat obyekt** bo'lishi mumkin, va bu kalitlarga **weak reference** sifatida qaraladi. Agar biror obyektga (masalan, `user`ga) boshqa hech qanday strong reference qolmasa, Garbage Collector uni tozalaydi — va shu bilan birga u `WeakMap` ichidan ham **avtomatik** yo'qoladi. Bu, memoization kontekstida, juda **tabiiy va mos** xususiyat: agar hech kim `user` obyektiga endi murojaat qilmasa, uning keshlangan natijasi ham **kerak emas** — va `WeakMap` buni **o'zi, qo'lda tozalashsiz** hal qiladi.

### To'liq amaliy misol — DOM elementlariga bog'liq keshlash

```javascript
const elementDataCache = new WeakMap();

function getElementMetadata(element) {
  if (elementDataCache.has(element)) {
    return elementDataCache.get(element);
  }

  const metadata = {
    computedStyle: getComputedStyle(element),
    boundingRect: element.getBoundingClientRect(),
  };

  elementDataCache.set(element, metadata);
  return metadata;
}

// element DOM'dan olib tashlanganda (element.remove()),
// uning metadata'si ham AVTOMATIK ravishda xotiradan tozalanadi,
// chunki WeakMap uni "ushlab turmaydi"
```

### `Map` vs `WeakMap` memoization uchun — qachon qaysi biri

| | `Map` bilan memoization | `WeakMap` bilan memoization |
|---|---|---|
| **Kalit turi** | Har qanday (primitive, obyekt) | Faqat obyekt |
| **Xotira boshqaruvi** | Qo'lda tozalash kerak (aks holda "abadiy" saqlanadi) | Avtomatik — GC obyekt kerak bo'lmaganda tozalaydi |
| **Iteratsiya (`.size`, `.forEach`)** | Mavjud | Mavjud emas |
| **Qachon ishlatiladi** | Cheklangan, sanoqli argumentlar bilan (masalan, ID, string) | Obyekt-argumentli funksiyalar, ayniqsa umr davomiyligi noaniq bo'lgan obyektlar bilan |

---

## 6. Memoization turlari va strategiyalari

### 6.1. Bitta argumentli memoization (eng oddiy)

```javascript
function memoizeSingle(fn) {
  const cache = new Map();
  return (arg) => {
    if (cache.has(arg)) return cache.get(arg);
    const result = fn(arg);
    cache.set(arg, result);
    return result;
  };
}
```

### 6.2. Ko'p argumentli memoization (kalitni birlashtirish)

```javascript
function memoizeMulti(fn) {
  const cache = new Map();
  return (...args) => {
    const key = args.join('|'); // yoki JSON.stringify(args)
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}
```

### 6.3. LRU (Least Recently Used) Cache — cheklangan hajmli kesh

Cheksiz o'sadigan kesh o'z-o'zidan **memory leak**ga aylanishi mumkin. **LRU Cache** — kesh hajmi chegaraga yetganda, **eng uzoq vaqt ishlatilmagan** yozuvni avtomatik o'chiradigan strategiya.

```javascript
class LRUCache {
  constructor(limit = 100) {
    this.limit = limit;
    this.cache = new Map(); // Map - qo'shilish tartibini saqlaydi
  }

  get(key) {
    if (!this.cache.has(key)) return undefined;
    const value = this.cache.get(key);
    this.cache.delete(key);
    this.cache.set(key, value); // "yangilangan" - oxiriga ko'chiriladi
    return value;
  }

  set(key, value) {
    if (this.cache.has(key)) this.cache.delete(key);
    else if (this.cache.size >= this.limit) {
      const oldestKey = this.cache.keys().next().value; // eng birinchi (eng eski)
      this.cache.delete(oldestKey);
    }
    this.cache.set(key, value);
  }
}
```

### 6.4. TTL (Time To Live) — vaqt bo'yicha eskirgan keshni tozalash

```javascript
function memoizeWithTTL(fn, ttl = 5000) {
  const cache = new Map();

  return (...args) => {
    const key = JSON.stringify(args);
    const cached = cache.get(key);

    if (cached && Date.now() - cached.timestamp < ttl) {
      return cached.value;
    }

    const result = fn(...args);
    cache.set(key, { value: result, timestamp: Date.now() });
    return result;
  };
}
```

Bu — masalan, **API javoblarini** vaqtinchalik keshlash uchun juda foydali (ma'lumot juda "eskirib" ketmasligi uchun).

---

## 7. Loyihaning qaysi qismlarida ishlatiladi?

| Soha | Misol |
|---|---|
| **React komponentlarida qimmat hisob-kitoblar** | `useMemo` — katta ro'yxatni filtrlash, saralash, agregatsiya qilish |
| **Callback'larni bolalarga uzatish** | `useCallback` — `React.memo` bilan birga, keraksiz qayta render'larni oldini olish |
| **API so'rovlarini keshlash** | React Query, SWR kabi kutubxonalar — memoization + TTL strategiyasi bilan |
| **Rekursiv algoritmlar** | Fibonachchi, dinamik dasturlash masalalari — eksponensial murakkablikni chiziqli darajaga tushirish |
| **Selektor funksiyalar (Redux, Reselect)** | State'dan hisoblangan qiymatlarni keshlash — `createSelector` ichida memoization ishlatiladi |
| **DOM elementlarga bog'liq metadata saqlash** | `WeakMap` — element yo'q qilinganda, unga bog'liq ma'lumot ham avtomatik tozalanadi |
| **Grafik/rendering optimallashtiruvchi kutubxonalar** | Katta ma'lumot vizualizatsiyasida qayta hisoblashlarni kamaytirish |

### Amaliy misol — Fibonachchi bilan memoization qanday algoritmni tezlashtiradi

```javascript
// Memoization'siz - EKSPONENSIAL murakkablik O(2^n)
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}
console.time('fib');
fib(35); // bir necha soniya vaqt oladi!
console.timeEnd('fib');

// Memoization bilan - CHIZIQLI murakkablik O(n)
function fibMemo(n, cache = new Map()) {
  if (n <= 1) return n;
  if (cache.has(n)) return cache.get(n);
  const result = fibMemo(n - 1, cache) + fibMemo(n - 2, cache);
  cache.set(n, result);
  return result;
}
console.time('fibMemo');
fibMemo(35); // millisekundlarda!
console.timeEnd('fibMemo');
```

Bu — memoization'ning **algoritmik** ta'sirini ko'rsatuvchi klassik misol: u vaqt murakkabligini **eksponensialdan chiziqliga** tushiradi, chunki har bir noyob kirish qiymati uchun hisob-kitob **faqat bir marta** bajariladi.

---

## 8. Qanday to'g'ri ishlatiladi — amaliy qoidalar

### 8.1. Faqat "qimmat" hisob-kitoblar uchun ishlatish

```javascript
// KERAKSIZ - bu hisob-kitob juda arzon, memoization foyda emas, ortiqcha xarajat qiladi
const sum = useMemo(() => a + b, [a, b]);

// KERAKLI - bu hisob-kitob haqiqatda "qimmat"
const sortedData = useMemo(() => data.sort((a, b) => a.value - b.value), [data]);
```

`useMemo`ning o'zi ham **xarajatga ega** (dependency'larni solishtirish, xotirada saqlash) — juda oddiy hisob-kitoblar uchun ishlatish, aslida, **performance'ni yaxshilamasdan, aksincha yomonlashtirishi** mumkin.

### 8.2. Dependency massivini to'g'ri belgilash

```javascript
// XATO - dependency array TO'LIQ EMAS, eskirgan qiymat qaytarilishi mumkin
const result = useMemo(() => computeValue(a, b), [a]); // b unutilgan!

// TO'G'RI
const result = useMemo(() => computeValue(a, b), [a, b]);
```

### 8.3. Obyekt/massiv referensiyalarini barqarorlashtirish

```javascript
// XATO - options har render'da YANGI obyekt, useMemo hech qachon "kesh"dan foydalanmaydi
function Component({ a, b }) {
  const options = { a, b }; // har safar yangi reference!
  const result = useMemo(() => compute(options), [options]); // hech qachon keshlanmaydi
}

// TO'G'RI - options'ni ham useMemo bilan barqarorlashtirish yoki alohida qiymatlarni dependency qilish
function Component({ a, b }) {
  const result = useMemo(() => compute({ a, b }), [a, b]); // to'g'ridan-to'g'ri primitive qiymatlar
}
```

---

## 9. Nega bu muhim — real foydalar

1. **Performance optimizatsiyasi** — takroriy, og'ir hisob-kitoblarni oldini olish orqali, ilova tezligini sezilarli oshirish
2. **Algoritmik samaradorlik** — rekursiv algoritmlarda (Fibonachchi, dinamik dasturlash masalalari) vaqt murakkabligini eksponensialdan chiziqli darajaga tushirish
3. **React'da keraksiz re-render'larni oldini olish** — `useMemo`/`useCallback` + `React.memo` birgalikda ishlatilganda, katta komponent daraxtlarida sezilarli tezlashtirish
4. **Xotira boshqaruvi** — `WeakMap` orqali, keshlash bilan birga, **avtomatik xotira tozalash**ni ta'minlash, memory leak'larning oldini olish
5. **Tarmoq so'rovlarini kamaytirish** — API javoblarini keshlash orqali (React Query, SWR), server yukini va foydalanuvchi kutish vaqtini kamaytirish

---

## 10. Eng ko'p uchraydigan xatolar

### Xato 1: Har bir hisob-kitobni `useMemo` bilan o'rash

```javascript
// KERAKSIZ - juda ko'p useMemo, kodni murakkablashtiradi, foyda kam
const a = useMemo(() => x + 1, [x]);
const b = useMemo(() => y + 1, [y]);
const c = useMemo(() => a + b, [a, b]);
```
**Yechim:** Faqat haqiqatda "qimmat" (o'lchash mumkin bo'lgan) hisob-kitoblar uchun ishlatish.

### Xato 2: Memoization keshi cheksiz o'sishiga yo'l qo'yish

```javascript
// XATO - agar funksiya juda ko'p turli argument bilan chaqirilsa,
// cache CHEKSIZ o'sadi - bu o'z-o'zidan MEMORY LEAK!
const memoized = memoize(processUniqueUserId); // har foydalanuvchi uchun yangi kalit
```
**Yechim:** LRU Cache yoki TTL strategiyasidan foydalanish, yoki `WeakMap` (agar kalit obyekt bo'lsa).

### Xato 3: Memoization'ni "side effect"lar bilan ishlatish

```javascript
// XATO - useMemo side-effect (masalan, logging, DOM o'zgartirish) uchun EMAS
useMemo(() => {
  console.log('render bo'ldi'); // bu YAXSHI amaliyot emas
  sendAnalytics();
}, []);
```
**Yechim:** Side-effect'lar uchun `useEffect` ishlatish, `useMemo` faqat **qiymat hisoblash** uchun.

### Xato 4: Obyekt kalitlar bilan oddiy `Map`dan foydalanish (memory leak xavfi)

Yuqorida 5-bo'limda ko'rsatilgan — agar kalit sifatida umr davomiyligi noaniq obyektlar ishlatilsa, `WeakMap` o'rniga `Map` ishlatish memory leak'ga olib kelishi mumkin.

---

## 11. Interview savollari va qisqa javoblar

**S: Memoization nima?**
> J: Bu — funksiya chaqiruvining natijasini keshlash va, agar funksiya bir xil argumentlar bilan qayta chaqirilsa, qayta hisoblamasdan, oldin saqlangan natijani qaytarish texnikasi. U og'ir hisob-kitoblarni takrorlashdan saqlab, performance'ni yaxshilaydi.

**S: `useMemo` qanday ishlaydi?**
> J: `useMemo` React'ning fiber node'ida (komponentning ichki xotirasida) oldingi dependency massivi va hisoblangan qiymatni saqlaydi. Har render'da u yangi dependency massivini oldingisi bilan (`Object.is` orqali, reference bo'yicha) solishtiradi — agar bir xil bo'lsa, eski qiymatni qaytaradi, aks holda hisob-kitobni qayta bajaradi.

**S: `useMemo` va `useCallback` orasidagi farq nima?**
> J: `useMemo` funksiyani chaqiradi va uning **natijasini** memoize qiladi. `useCallback` esa funksiyaning **o'zini** (chaqirmasdan) memoize qiladi — bu aslida `useMemo(() => fn, deps)`ga teng.

**S: Nega `WeakMap` memoization uchun foydali?**
> J: Agar memoization kaliti sifatida obyekt ishlatilsa, oddiy `Map` bu obyektga strong reference saqlaydi va u hech qachon Garbage Collector tomonidan tozalanmaydi (memory leak). `WeakMap` esa weak reference saqlaydi — agar obyektga boshqa hech kim murojaat qilmasa, u va uning keshlangan natijasi avtomatik ravishda xotiradan tozalanadi.

**S: `useMemo` har doim performance'ni yaxshilaydimi?**
> J: Yo'q. `useMemo`ning o'zi ham xarajatga ega (dependency solishtirish, xotirada saqlash). Juda oddiy, "arzon" hisob-kitoblar uchun ishlatilsa, bu foyda bermay, aksincha ortiqcha xarajat qo'shishi mumkin. Uni faqat haqiqatda "qimmat" hisob-kitoblar uchun ishlatish tavsiya etiladi.

**S: LRU Cache nima va u qachon kerak bo'ladi?**
> J: LRU (Least Recently Used) Cache — kesh hajmi belgilangan chegaraga yetganda, eng uzoq vaqt ishlatilmagan yozuvni avtomatik o'chirib, keshning cheksiz o'sishining (va shu bilan memory leak'ning) oldini oluvchi strategiya. Bu ko'p, turli xil argumentlar bilan chaqiriladigan funksiyalarni memoize qilishda muhim.

---

## 12. Xulosa

> **Memoization — funksiya natijalarini keshlash orqali takroriy hisob-kitoblardan qochish** uchun ishlatiladigan fundamental optimizatsiya texnikasi bo'lib, u algoritmik samaradorlikdan (Fibonachchi kabi rekursiv masalalarda) tortib, React'dagi komponent performance'igacha (`useMemo`, `useCallback`) keng qo'llaniladi.
>
> **`useMemo`** React'ning ichki "hooks" xotirasida dependency massivini va hisoblangan qiymatni saqlaydi, va har render'da dependency'larni **reference bo'yicha** solishtiradi — bu React hujjatlarida **kafolat emas, balki performance maslahati** sifatida ta'riflanadi.
>
> **`WeakMap` bilan memoization** — obyekt-kalitli keshlashda **avtomatik xotira boshqaruvi**ni ta'minlaydi: agar kalit obyektga boshqa hech kim murojaat qilmasa, uning keshlangan natijasi ham GC tomonidan avtomatik tozalanadi, bu esa memory leak xavfini bartaraf qiladi.
>
> Bu mavzuni chuqur tushunish — nafaqat interview'da, balki real loyihalarda **to'g'ri, o'lchangan** optimizatsiya qarorlarini qabul qilishda (qachon memoization kerak, qachon u ortiqcha xarajat bo'lishi mumkinligini bilishda) muhim ahamiyatga ega.

---

*Tayyorlandi: JS/React texnik intervyu tayyorgarligi uchun*
