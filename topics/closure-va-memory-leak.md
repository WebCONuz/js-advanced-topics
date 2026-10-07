# Closure va Xotira (Memory Leak) — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [Closure nima?](#1-closure-nima)
2. [Closure xotirani qanday "ushlab turadi"?](#2-closure-xotirani-qanday-ushlab-turadi)
3. [Memory Leak — real misollar](#3-memory-leak--real-misollar)
   - [3.1. Ishlatilmayotgan katta ma'lumotni ushlab turish](#31-ishlatilmayotgan-katta-malumotni-ushlab-turish)
   - [3.2. Event Listener orqali sizish](#32-event-listener-orqali-sizish)
   - [3.3. Cache/Map orqali obyektlarni "unutib qo'yish"](#33-cachemap-orqali-obyektlarni-unutib-qoyish)
4. [Yechim: WeakMap](#4-yechim-weakmap)
5. [Yechim: WeakRef va FinalizationRegistry](#5-yechim-weakref-va-finalizationregistry)
6. [WeakMap vs Map — taqqoslash jadvali](#6-weakmap-vs-map--taqqoslash-jadvali)
7. [Best Practice'lar — Leak'larning oldini olish](#7-best-practicelar--leaklarning-oldini-olish)
8. [Interview savollari va qisqa javoblar](#8-interview-savollari-va-qisqa-javoblar)
9. [Xulosa](#9-xulosa)

---

## 1. Closure nima?

**Closure** — bu funksiyaning o'zi yaratilgan **lexical scope**dagi o'zgaruvchilarni "eslab qolish" qobiliyati, hatto tashqi funksiya ishini tugatgandan keyin ham.

```javascript
function outer() {
  const message = "Salom, dunyo!";

  function inner() {
    console.log(message); // tashqi scope'dagi o'zgaruvchiga murojaat
  }

  return inner;
}

const greet = outer(); // outer() ishini tugatdi
greet(); // "Salom, dunyo!" — message hali ham xotirada!
```

`outer()` funksiyasi allaqachon ishlab bo'lgan bo'lsa-da, `message` o'zgaruvchisi xotiradan o'chmaydi — chunki `inner` funksiyasi unga hali ham havola (reference) saqlab turibdi. Aynan shu — **closure**.

**Nega bu kerak?** Closure orqali biz:
- Private o'zgaruvchilar yaratamiz (encapsulation)
- Funksiya fabrikalarini (function factories) quramiz
- Callback va event handler'larda holatni (state) saqlaymiz

Lekin xuddi shu kuchli xususiyat — **memory leak**larga sabab bo'lishi ham mumkin.

---

## 2. Closure xotirani qanday "ushlab turadi"?

JavaScript'da **Garbage Collector (GC)** ishlatilmay qolgan obyektlarni avtomatik tozalaydi. Biroq GC bir obyektni faqat unga hech qanday **"strong reference"** (kuchli havola) qolmagan taqdirdagina tozalaydi.

Closure yaratilganda, JS dvigateli (engine) odatda butun **lexical scope**ni saqlab qoladi — nafaqat closure ichida ishlatilgan o'zgaruvchini, balki ba'zan butun scope zanjirini ham. Shu sababli, agar closure ichida katta hajmdagi ma'lumotga ishora bo'lsa-yu, u boshqa hech qachon ishlatilmasa ham — xotira band bo'lib qoladi.

```
outer() scope
    │
    ├── message (kichik) ──┐
    └── hugeData (katta) ──┼── inner() closure orqali ULARGA REFERENCE saqlaydi
                            │
                     Garbage Collector buni tozalay olmaydi,
                     chunki "kimdir" hali ham foydalanmoqda deb hisoblaydi
```

> **Muhim tushuncha:** Memory leak — bu JavaScript'ning "xatosi" emas, balki dasturchining e'tiborsizligi natijasida obyektlarga bo'lgan keraksiz strong reference'larni saqlab qolishidir.

---

## 3. Memory Leak — real misollar

### 3.1. Ishlatilmayotgan katta ma'lumotni ushlab turish

```javascript
function createHandler() {
  const hugeData = new Array(1_000_000).fill('katta malumot'); // ~ko'p xotira band qiladi

  return function handler() {
    console.log('Handler ishladi');
    // hugeData bu yerda umuman ishlatilmayapti!
  };
}

const myHandler = createHandler();
```

**Muammo:** `hugeData` `handler` funksiyasi ichida hech qachon ishlatilmaydi, lekin ba'zi holatlarda (murakkab scope zanjirlarida) u closure orqali xotirada saqlanib qolishi mumkin, chunki funksiya butun tashqi scope'ga bog'liq.

**Qanday oldini olish mumkin:** Closure ichida faqat kerakli ma'lumotlarni saqlang, keraksiz katta o'zgaruvchilarni scope'dan chiqarib tashlang yoki `null` qiling.

---

### 3.2. Event Listener orqali sizish

Bu — amaliyotda **eng ko'p uchraydigan** memory leak turi.

```javascript
function attachListener() {
  const largeData = fetchHugeDataFromSomewhere(); // katta hajmdagi ma'lumot

  document.getElementById('btn').addEventListener('click', function () {
    console.log('Bosildi', largeData.length);
  });
}

attachListener();
```

**Muammo zanjiri:**

```
DOM element (#btn) → event listener closure → largeData
```

Agar `#btn` elementi keyinchalik DOM'dan olib tashlansa (`btn.remove()`), lekin listener oldindan `removeEventListener` bilan olib tashlanmagan bo'lsa — brauzer bu elementni **to'liq** tozalay olmaydi, chunki listener closure hali ham unga (va demak, `largeData`ga ham) ishora qilib turadi.

**To'g'ri yechim:**

```javascript
function attachListener() {
  const largeData = fetchHugeDataFromSomewhere();
  const btn = document.getElementById('btn');

  function onClick() {
    console.log('Bosildi', largeData.length);
  }

  btn.addEventListener('click', onClick);

  // Element kerak bo'lmay qolganda:
  // btn.removeEventListener('click', onClick);
}
```

> **Qoida:** Har bir `addEventListener` uchun, komponent yoki element yo'q qilinganda, mos `removeEventListener` chaqirilishi kerak. React kabi freymvorklarda bu odatda `useEffect`ning **cleanup funksiyasida** amalga oshiriladi.

---

### 3.3. Cache/Map orqali obyektlarni "unutib qo'yish"

```javascript
const cache = new Map();

function processUser(user) {
  cache.set(user, user.data); // user obyekti KEY sifatida saqlanadi
}

let user = { id: 1, data: 'katta malumot...' };
processUser(user);

user = null; // "user"ni tashladik, deb o'ylaymiz
// LEKIN cache hali ham o'sha obyektga strong reference saqlab turibdi!
// GC uni tozalay olmaydi.
```

`user = null` qilinganiga qaramay, obyekt xotiradan ketmaydi — chunki oddiy `Map` o'z kalitlariga **strong reference** bilan bog'lanadi. Demak, `cache` mavjud ekan, obyekt ham mavjud bo'lib qoladi — bu esa **xotira sizib chiqishiga** olib keladi.

---

## 4. Yechim: WeakMap

`WeakMap` — bu `Map`ning maxsus turi bo'lib, uning **kalitlari faqat obyekt bo'lishi mumkin** va bu kalitlarga **"weak reference"** (zaif havola) sifatida qaraladi.

**Bu nimani anglatadi?** Agar biror obyektga `WeakMap`dan boshqa hech qanday strong reference qolmasa, Garbage Collector uni avtomatik tozalab tashlaydi — va shu bilan birga u `WeakMap` ichidan ham yo'qoladi.

3.3-misolni tuzatamiz:

```javascript
const cache = new WeakMap(); // oddiy Map o'rniga WeakMap

function processUser(user) {
  cache.set(user, user.data);
}

let user = { id: 1, data: 'katta malumot...' };
processUser(user);

user = null;
// Endi GC obyektni erkin tozalay oladi,
// chunki WeakMap "weak reference" saqlaydi —
// boshqa hech kim unga ishora qilmasa, u avtomatik xotiradan ketadi.
```

### WeakMap qachon ishlatiladi?

| Vaziyat | Tushuntirish |
|---|---|
| **Obyektga bog'liq metadata saqlash** | Masalan, DOM elementiga bog'liq qo'shimcha holat (state) saqlash |
| **Private fields (eski usul)** | ES2022'dan oldin class'larda private property yaratish uchun ishlatilgan |
| **Cache** | Obyekt hali ishlatilayotgan bo'lsagina keshda saqlanishi kerak bo'lganda |

```javascript
// Private field misoli (ES2022'dan oldingi usul)
const _privateData = new WeakMap();

class User {
  constructor(name) {
    _privateData.set(this, { name });
  }

  getName() {
    return _privateData.get(this).name;
  }
}
```

### Cheklovlari

- Kalit faqat **obyekt** bo'lishi mumkin (`string`, `number` kabi primitivlar bo'la olmaydi)
- **Iteratsiya qilib bo'lmaydi** — `.keys()`, `.values()`, `.forEach()`, `.size` mavjud emas, chunki GC istalgan vaqtda elementni tozalab yuborishi mumkin va bu holat oldindan aytib bo'lmaydigan (non-deterministic) xususiyatga ega

---

## 5. Yechim: WeakRef va FinalizationRegistry

### WeakRef

`WeakRef` — bitta obyektga **zaif havola** yaratishga imkon beradi. Obyektga murojaat qilish uchun `.deref()` metodidan foydalaniladi.

```javascript
let user = { name: 'Ali' };
const weakRef = new WeakRef(user);

console.log(weakRef.deref()); // { name: 'Ali' }

user = null; // asosiy (strong) reference olib tashlandi

// Qachondir GC ishlagandan keyin:
console.log(weakRef.deref()); // undefined bo'lishi mumkin
```

### FinalizationRegistry

Obyekt Garbage Collector tomonidan tozalanganda biror amalni bajarish uchun ishlatiladi (masalan, logging yoki resurslarni tozalash):

```javascript
const registry = new FinalizationRegistry((value) => {
  console.log(`${value} obyekti tozalandi`);
});

let user = { name: 'Ali' };
registry.register(user, 'user obyekti');

user = null;
// Qachondir keyinroq: "user obyekti tozalandi" konsolga chiqishi mumkin
```

> ⚠️ **Diqqat — interview uchun juda muhim jihat:**
> `WeakRef` va `FinalizationRegistry` **ehtiyotkorlik bilan** ishlatiladigan ilg'or (advanced) API'lar hisoblanadi, chunki:
> - GC qachon ishlashi **kafolatlanmagan** va **noaniq** (non-deterministic)
> - Ko'pchilik amaliy holatlarda oddiy `WeakMap` yetarli bo'ladi
> - `WeakRef` asosan juda maxsus holatlar uchun (masalan, katta obyektlarni ixtiyoriy cache qilish) mo'ljallangan

---

## 6. WeakMap vs Map — taqqoslash jadvali

| Xususiyat | `Map` | `WeakMap` |
|---|---|---|
| Kalit turi | Har qanday tur (string, number, obyekt...) | Faqat obyekt |
| Reference turi | Strong (kuchli) | Weak (zaif) |
| Garbage Collection | Kalit obyektni to'sib qo'yadi | Kalit obyektni to'smaydi |
| Iteratsiya (`forEach`, `keys`, va h.k.) | Bor | Yo'q |
| `.size` property | Bor | Yo'q |
| Asosiy ishlatilish o'rni | Umumiy ma'lumotlar to'plami | Obyektga bog'liq vaqtinchalik metadata |

---

## 7. Best Practice'lar — Leak'larning oldini olish

1. **Event listener'larni har doim tozalang** — element yo'q qilinganda `removeEventListener` chaqiring (yoki React'da `useEffect` cleanup funksiyasidan foydalaning).
2. **Global o'zgaruvchilardan saqlaning** — global scope'ga tasodifan qo'shilib qolgan o'zgaruvchilar hech qachon GC qilinmaydi.
3. **Katta ma'lumotlarni closure ichida keraksiz saqlamang** — faqat aynan kerakli qismini oling, butun obyektni emas.
4. **Timer'larni tozalang** — `setInterval`/`setTimeout` ichida closure orqali katta obyektlarga ishora qilinsa, `clearInterval`/`clearTimeout` chaqirishni unutmang.
5. **Obyekt-key'li kesh uchun `WeakMap` ishlating** — oddiy `Map` emas, agar obyekt boshqa joyda ishlatilmay qolsa, avtomatik tozalanishi kerak bo'lsa.
6. **DevTools Memory Profiler'dan foydalaning** — Chrome DevTools'dagi "Memory" bo'limida heap snapshot'larni solishtirib, qaysi obyektlar xotirada "qolib ketayotganini" aniqlash mumkin.

---

## 8. Interview savollari va qisqa javoblar

**S: Closure nima va u xotira bilan qanday bog'liq?**
> J: Closure — funksiyaning o'z yaratilgan lexical scope'idagi o'zgaruvchilarga bo'lgan havolasini saqlab qolish qobiliyati. Bu o'zgaruvchilarga strong reference saqlanib qoladi, shu sababli GC ularni tozalay olmaydi — agar diqqat qilinmasa, bu memory leak'ga olib kelishi mumkin.

**S: Memory leak'ning eng ko'p uchraydigan sababi nima?**
> J: Eng ko'p uchraydigani — DOM elementiga qo'shilgan event listener'ni element olib tashlangandan keyin ham `removeEventListener` bilan tozalamaslik. Bu listener ichidagi closure orqali bog'langan barcha ma'lumotlarni ham xotirada "muzlatib" qo'yadi.

**S: WeakMap oddiy Map'dan nimasi bilan farq qiladi?**
> J: WeakMap kalitlari faqat obyekt bo'lishi mumkin va ularga **weak reference** sifatida qaraladi — ya'ni kalit obyektga boshqa joyda reference qolmasa, GC uni avtomatik tozalaydi. Oddiy Map esa strong reference saqlaydi, shu sababli kalit doim xotirada "ushlab turiladi", hatto boshqa hech kim uni ishlatmasa ham.

**S: WeakMap nega iteratsiya qilinmaydi?**
> J: Chunki GC istalgan vaqtda WeakMap ichidagi biror elementni (agar obyektga boshqa reference qolmasa) tozalab yuborishi mumkin. Agar iteratsiya imkoniyati bo'lganida, natija oldindan aytib bo'lmaydigan (non-deterministic) bo'lardi — shu sababli dizayn jihatidan bu funksiyalar ataylab olib tashlangan.

**S: WeakRef qachon kerak bo'ladi?**
> J: Juda kam holatlarda — masalan, katta obyektni ixtiyoriy (optional) cache qilish kerak bo'lganda, lekin bu obyekt xotirani band qilib turishini istamasangiz. Ammo GC ishlashi kafolatlanmagani sababli, buni asosiy strategiyaga aylantirish tavsiya etilmaydi.

---

## 9. Xulosa

> **Closure — ikki tomonlama qilich.** U JavaScript'ning eng kuchli xususiyatlaridan biri — private state, funksiya fabrikalari va callback'larda holatni saqlash imkonini beradi. Lekin aynan shu kuch, agar ehtiyotsizlik bilan ishlatilsa, kerak bo'lmagan katta ma'lumotlarni xotirada "unutib qo'yishga" olib kelishi mumkin.
>
> Buning oldini olish uchun:
> - Event listener'larni har doim tozalang
> - Closure ichida faqat zarur ma'lumotni saqlang
> - Obyekt-key'li kesh uchun oddiy `Map` o'rniga **`WeakMap`** ishlating
> - `WeakRef`ni faqat juda maxsus, ehtiyotkorlik talab qiladigan holatlarda qo'llang

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
