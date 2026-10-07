# Structured Clone Algorithm — `structuredClone()`, `postMessage` va nima klonlanmaydi — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.
> Asosiy fokus: Structured Clone qanday ishlaydi, `postMessage` va `structuredClone()`, klonlanmaydigan narsalar (funksiyalar, DOM node'lar, Proxy va h.k.), Transferable va SharedArrayBuffer.

---

## Mundarija

1. [Structured Clone nima va u qayerdan paydo bo'lgan?](#1-structured-clone-nima-va-u-qayerdan-paydo-bolgan)
2. [Nega alohida algoritm kerak? — "heap" izolyatsiyasi](#2-nega-alohida-algoritm-kerak--heap-izolyatsiyasi)
3. [Algoritm qanday ishlaydi: serialize → deserialize](#3-algoritm-qanday-ishlaydi-serialize--deserialize)
4. [Qaysi API'lar Structured Clone ishlatadi?](#4-qaysi-apilar-structured-clone-ishlatadi)
5. [`structuredClone()` — asoslar](#5-structuredclone--asoslar)
6. [Nima klonlanadi?](#6-nima-klonlanadi)
7. [Nima klonlanmaydi? — `DataCloneError`](#7-nima-klonlanmaydi--dataclonerror)
8. [Xato bermasdan "yo'qoladigan" narsalar](#8-xato-bermasdan-yoqoladigan-narsalar)
9. [JSON, spread, lodash `cloneDeep` bilan taqqoslash](#9-json-spread-lodash-clonedeep-bilan-taqqoslash)
10. [`postMessage` amalda](#10-postmessage-amalda)
11. [Transferable obyektlar va `SharedArrayBuffer`](#11-transferable-obyektlar-va-sharedarraybuffer)
12. [Noldan mini implementatsiya (o'quv modeli)](#12-noldan-mini-implementatsiya-oquv-modeli)
13. [Klonlanmaydigan narsalarni qanday hal qilish?](#13-klonlanmaydigan-narsalarni-qanday-hal-qilish)
14. [Loyihaning qaysi qismlarida ishlatiladi?](#14-loyihaning-qaysi-qismlarida-ishlatiladi)
15. [Performance jihatlari](#15-performance-jihatlari)
16. [Nega bu muhim — real foydalar](#16-nega-bu-muhim--real-foydalar)
17. [Eng ko'p uchraydigan xatolar](#17-eng-kop-uchraydigan-xatolar)
18. [Interview savollari va qisqa javoblar](#18-interview-savollari-va-qisqa-javoblar)
19. [Xulosa](#19-xulosa)

---

## 1. Structured Clone nima va u qayerdan paydo bo'lgan?

**Structured Clone Algorithm** — JavaScript qiymatini (ko'pincha murakkab, ichma-ich obyektni) **chuqur nusxalash** (deep copy) uchun brauzer/dvigatel ichida ishlatiladigan standart algoritm. U HTML standartida ta'riflangan va quyidagi ikki ko'rinishda ishlatiladi:

```javascript
// 1) Dasturchi to'g'ridan-to'g'ri chaqiradigan funksiya
const copy = structuredClone(original);

// 2) "Sahna orqasida" — postMessage va boshqa API'lar ichida
worker.postMessage(original);   // original nusxalanib, worker'ga yetkaziladi
```

"**Structured**" nomi shundan: algoritm oddiy "tekis" qiymatlarni emas, **tuzilmalarni** (graflarni) — ichma-ich obyektlar, **siklik havolalar**, **umumiy havolalar**, `Map`, `Set`, `Date` va h.k. ni to'g'ri nusxalaydi.

### Tarixiy kontekst

| Davr | Nima bo'lgan |
|---|---|
| **HTML5 davri (~2008–2012)** | `window.postMessage` va Web Worker'lar paydo bo'ldi. Dastlab xabarlar asosan **satr** shaklida ketardi — murakkab ma'lumotni qo'lda `JSON.stringify` qilish kerak edi |
| **Standartlashtirish** | Obyektlarni to'g'ridan-to'g'ri jo'natish uchun HTML spetsifikatsiyasida **structured clone algorithm** ta'riflandi. Keyin uni boshqa API'lar ham qabul qildi: **IndexedDB** (qiymatlarni saqlash), `history.pushState` (state) va h.k. |
| **~2021–2022** | Algoritm endi **`structuredClone()`** global funksiyasi sifatida to'g'ridan-to'g'ri dasturchiga ochildi (Firefox 94, Chrome 98, Safari 15.4, Node.js 17) |

> **Eslatma:** versiya raqamlari va qo'llab-quvvatlash holatini ishlatishdan oldin [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone) yoki caniuse'da tekshirib oling — eski brauzerlar uchun polyfill (masalan, `core-js` yoki `@ungap/structured-clone`) mavjud, lekin u algoritmning barcha qirralarini (jumladan transfer) to'liq qamrab olmasligi mumkin.

### `structuredClone()` dan oldin odamlar nima qilgan?

```javascript
// 1) Eng keng tarqalgan "hack" — JSON orqali (ko'p cheklovlari bor, 9-bo'limga qarang)
const copy = JSON.parse(JSON.stringify(obj));

// 2) Kutubxona — lodash
const copy2 = _.cloneDeep(obj);

// 3) MessageChannel "hack" — brauzerning structured clone'idan foydalanish (ASINXRON!)
function oldStructuredClone(obj) {
  return new Promise((resolve) => {
    const { port1, port2 } = new MessageChannel();
    port2.onmessage = (e) => resolve(e.data);   // e.data — algoritm yaratgan nusxa
    port1.postMessage(obj);
  });
}
```

Uchinchi usul qiziq: u **`postMessage` ichidagi algoritmning o'zidan** "bepul" foydalanadi — lekin natija faqat **keyingi macrotask'da** (async) keladi. `structuredClone()` esa **sinxron**.

---

## 2. Nega alohida algoritm kerak? — "heap" izolyatsiyasi

Bu — mavzuning **eng muhim konseptual qismi**: nima uchun oddiy havolani (reference) jo'natib bo'lmaydi?

JavaScript obyektlari **bitta realm/thread'ning heap xotirasida** yashaydi:

```
┌─────────── Main thread ───────────┐      ┌──────────── Worker thread ───────────┐
│  Heap A                            │      │  Heap B                               │
│   { name: 'Ali', fn: () => ... }   │      │   (butunlay boshqa, alohida xotira)   │
│   DOM, window, document            │      │   DOM YO'Q, window YO'Q               │
└────────────────────────────────────┘      └───────────────────────────────────────┘
          ↑  bevosita havola (pointer) berish MUMKIN EMAS  ↑
```

Sabablari:

1. **Thread'lar heap'ni bo'lishmaydi.** Agar ikkala thread bitta obyektni bir vaqtda o'zgartira olsa — **race condition**'lar paydo bo'lardi. JS single-thread modeli aynan shuni oldini oladi.
2. **Realm'lar alohida.** `iframe`, `window` yoki Worker'ning o'z `Object`, `Array`, `Date` konstruktorlari bor — boshqa realm'dan kelgan obyekt "begona" bo'lib qoladi.
3. **Xavfsizlik.** Cross-origin `iframe`ga jonli obyekt havolasi berish — uning ichki holatiga kirish yo'li bo'lardi.

**Yechim:** obyektni **neytral (xotiradan mustaqil) ko'rinishga aylantirib (serialize)**, qabul qiluvchi tomonda **yangidan qurish (deserialize)**. Natijada qabul qiluvchi **o'z heap'ida** to'liq mustaqil **nusxa** oladi.

> Funksiyalar nega klonlanmaydi degan savolning javobi ham shu yerda: funksiya — bu **kod + closure (o'zi yaratilgan scope'ga havola)**. Scope zanjirini boshqa heap/thread'ga ko'chirib bo'lmaydi (u yerda o'sha o'zgaruvchilar, DOM, global'lar yo'q).

---

## 3. Algoritm qanday ishlaydi: serialize → deserialize

Spetsifikatsiya algoritmni **ikki bosqichga** bo'ladi:

```
 Jo'natuvchi                                                   Qabul qiluvchi
┌────────────────┐  StructuredSerialize   ┌─────────────────┐  StructuredDeserialize  ┌────────────────┐
│ JS obyekt grafi │ ─────────────────────▶ │ "serialized      │ ───────────────────────▶ │ YANGI obyekt    │
│ (asl)           │   (SINXRON, chaqirilgan │  record" (neytral,│   (qabul qiluvchining    │ grafi (nusxa)   │
│                 │    paytda!)             │  dvigatelga xos)  │    realm'ida)            │                 │
└────────────────┘                         └─────────────────┘                           └────────────────┘
```

### Muhim nuqtalar

**1. Serialize — chaqirilgan paytning o'zida (sinxron).** `postMessage` yetkazib berish **asinxron**, lekin ma'lumotning "surati" **darhol** olinadi:

```javascript
const data = { n: 1 };
worker.postMessage(data);
data.n = 2;                    // worker baribir { n: 1 } oladi — surat allaqachon olingan
```

**2. "Xotira xaritasi" (memory map).** Algoritm har bir ko'rilgan obyektni xaritada eslab qoladi. Shu tufayli:

```javascript
const shared = { tag: 'x' };
const original = { a: shared, b: shared };
original.self = original;                    // sikl

const copy = structuredClone(original);

console.log(copy.a === copy.b);     // true  — umumiy havola SAQLANDI (bitta obyekt)
console.log(copy.self === copy);    // true  — sikl SAQLANDI
console.log(copy.a === shared);     // false — asl obyektdan butunlay mustaqil

JSON.stringify(original);           // TypeError: Converting circular structure to JSON
```

**3. Faqat "own enumerable string-key" property'lar.** Oddiy obyektlar uchun algoritm obyektning o'z, sanaladigan (enumerable), **satr** kalitli property'larini oladi (qiymatni `[[Get]]` orqali o'qiydi).

**4. Serializable bo'lmagan narsa uchraganda — `DataCloneError`.** Grafning ixtiyoriy joyida (chuqurda ham) funksiya, symbol-qiymat, DOM node va h.k. uchrasa — **butun operatsiya** `DOMException` (`name === 'DataCloneError'`) bilan to'xtaydi; **qisman nusxa qaytarilmaydi**.

---

## 4. Qaysi API'lar Structured Clone ishlatadi?

| API | Izoh |
|---|---|
| `structuredClone(value, options)` | To'g'ridan-to'g'ri chuqur nusxa (sinxron) |
| `window.postMessage()` | Oyna ↔ `iframe` ↔ boshqa oyna |
| `Worker.postMessage()`, `self.postMessage()` | Main thread ↔ Dedicated/Shared Worker |
| `MessagePort.postMessage()` / `MessageChannel` | Ikki tomonlama maxsus kanal |
| `BroadcastChannel.postMessage()` | Bir origin'dagi tab/oyna/worker'lar o'rtasida |
| `ServiceWorker.postMessage()`, `client.postMessage()` | Service Worker ↔ sahifalar |
| **IndexedDB** (`add`, `put`, `cursor.update`) | Qiymatlar saqlanganda (shuning uchun `Blob`, `Date`, `Map` saqlash mumkin, funksiya esa — yo'q) |
| `history.pushState(state, ...)` / `replaceState` | `state` obyekti |
| **Node.js** `worker_threads` (`postMessage`, `workerData`) va global `structuredClone` (v17+) | Server tomonida ham xuddi shunday |

> **Eslatma:** Node.js'ning `v8.serialize()/deserialize()` funksiyalari ham shunga yaqin, lekin **bir xil emas** — ular V8'ning o'z serializatsiya formatidan foydalanadi.

---

## 5. `structuredClone()` — asoslar

```javascript
const original = {
  name: 'Ali',
  tags: ['js', 'ts'],
  address: { city: 'Toshkent' },
};

const copy = structuredClone(original);

copy.address.city = 'Samarqand';
console.log(original.address.city);  // 'Toshkent' — asl o'zgarmadi (chuqur nusxa)
console.log(copy.tags === original.tags); // false
```

**Imzo:**

```javascript
structuredClone(value)
structuredClone(value, { transfer: [arrayBuffer, ...] })
```

- **Sinxron** ishlaydi (natijani darhol qaytaradi).
- `transfer` opsiyasi — ba'zi obyektlarni nusxalash o'rniga **ko'chirish** uchun (11-bo'lim).
- Asl qiymatni **o'zgartirmaydi** (transfer ro'yxatidagilar bundan mustasno).

---

## 6. Nima klonlanadi?

Algoritm quyidagi turlarni **to'g'ri** nusxalaydi:

| Tur | Izoh |
|---|---|
| **Primitive'lar** | `number` (jumladan `NaN`, `Infinity`, `-0`), `string`, `boolean`, `bigint`, `undefined`, `null` (**`symbol` dan tashqari**) |
| **Primitive wrapper obyektlari** | `new Number(1)`, `new String('a')`, `new Boolean(true)`, `Object(1n)` |
| **`Date`** | Vaqt qiymati saqlanadi, yangi `Date` yaratiladi |
| **`RegExp`** | `source` va `flags` saqlanadi (**`lastIndex` saqlanmaydi** — 0 bo'ladi) |
| **`Array`**, oddiy **`Object`** | Chuqur, ichma-ich, **sikllar va umumiy havolalar bilan** |
| **`Map`, `Set`** | Kalit va qiymatlar ham chuqur nusxalanadi, tartib saqlanadi |
| **`ArrayBuffer`**, **TypedArray'lar** (`Uint8Array` va h.k.), **`DataView`** | Binary ma'lumot nusxalanadi |
| **`Error`** (`Error`, `TypeError`, `RangeError`, ...) | `name` va `message` saqlanadi (`stack`, `cause` — brauzerga bog'liq) |
| **`Blob`, `File`, `FileList`** | Brauzerda |
| **`ImageData`, `ImageBitmap`, `DOMRect`, `DOMMatrix`, `DOMPoint`, `CryptoKey`** | "Serializable" deb belgilangan platforma obyektlari |

### Misol — turli tur'lar

```javascript
const src = {
  date: new Date('2026-10-07'),
  re: /abc/gi,
  map: new Map([['a', { n: 1 }]]),
  set: new Set([1, 2, 3]),
  bytes: new Uint8Array([1, 2, 3]),
  big: 123n,
  nan: NaN,
  negZero: -0,
  undef: undefined,
  nested: { deep: [1, [2, 3]] },
};

const c = structuredClone(src);

console.log(c.date instanceof Date);              // true
console.log(c.map.get('a') !== src.map.get('a')); // true — Map ichidagi obyekt ham nusxalangan
console.log(Object.is(c.negZero, -0));            // true — JSON bu yerda 0 qilib yuborardi
console.log('undef' in c);                        // true — kalit saqlanadi (JSON'da tushib qolardi)
console.log(c.big);                               // 123n — JSON.stringify bigint'da xato beradi
```

---

## 7. Nima klonlanmaydi? — `DataCloneError`

Quyidagilar uchraganda **`DOMException`** (`name: 'DataCloneError'`, `code: 25`) tashlanadi:

| Narsa | Nega klonlanmaydi |
|---|---|
| **Funksiyalar** (oddiy, arrow, `async`, class konstruktorlari, metodlar) | Closure/scope zanjirini boshqa heap'ga ko'chirib bo'lmaydi |
| **`Symbol`** qiymatlari | Symbol — noyob identifikator; boshqa realm'da "o'sha" symbol degan tushuncha yo'q |
| **DOM node'lar** (`Node`, `Element`, `Document`, `Window`) | Ular faqat main thread'dagi hujjatga bog'langan, "serializable" emas |
| **`Proxy` obyektlari** | Dvigatel Proxy'ni aniqlab, rad etadi (oldingi mavzudagi Vue/Immer holati!) |
| **`WeakMap`, `WeakSet`, `WeakRef`** | Ichki holatini (GC'ga bog'liq) ko'rib/ko'chirib bo'lmaydi |
| **`Promise`** | Kutilayotgan natijani serialize qilib bo'lmaydi (avval `await` qiling) |
| **Generator/iterator obyektlari** | Ichki bajarilish holatiga ega |
| **Serializable deb belgilanmagan platforma obyektlari** (masalan, `XMLHttpRequest`, `Event`, `Request`, `Response`, `console`) | Ularning ichki holati/resurslari ko'chirilmaydi |

```javascript
function tryClone(value, label) {
  try {
    structuredClone(value);
    console.log(`${label}: ✅`);
  } catch (e) {
    console.log(`${label}: ❌ ${e.name} (code ${e.code})`);
  }
}

tryClone({ fn() {} },                 'metod');          // ❌ DataCloneError (code 25)
tryClone({ cb: () => 1 },             'arrow function'); // ❌
tryClone({ s: Symbol('x') },          'symbol qiymat');  // ❌
tryClone(document.body,               'DOM node');       // ❌
tryClone(new Proxy({}, {}),           'Proxy');          // ❌
tryClone(new WeakMap(),               'WeakMap');        // ❌
tryClone(Promise.resolve(1),          'Promise');        // ❌
tryClone({ a: { b: { c: () => {} } } }, 'chuqur ichida funksiya'); // ❌ — bitta funksiya BUTUN operatsiyani buzadi
tryClone({ ok: 1, date: new Date() }, 'oddiy obyekt');   // ✅
```

> **Muhim:** xato **chuqurlikdan qat'i nazar** butun operatsiyani to'xtatadi. Katta obyektning bitta chuqur joyida yashiringan funksiya `postMessage`ni sindirishi mumkin — shuning uchun xato xabarida ko'pincha aynan qaysi qiymat klonlanmagani ko'rsatiladi (matni brauzerga qarab farq qiladi).

### Xatoni qanday ushlash

```javascript
try {
  worker.postMessage(payload);
} catch (e) {
  if (e instanceof DOMException && e.name === 'DataCloneError') {
    console.error('Payload serializatsiya qilinmadi:', e.message);
  } else {
    throw e;
  }
}
```

---

## 8. Xato bermasdan "yo'qoladigan" narsalar

Eng xavfli holatlar — **xato bo'lmaydi**, lekin natija siz kutgandan farq qiladi.

### 8.1. Prototip va class — **yo'qoladi**

```javascript
class User {
  #secret = 1;                       // private maydon
  constructor(name) { this.name = name; }
  greet() { return `Salom, ${this.name}`; }
  get upper() { return this.name.toUpperCase(); }
}

const u = new User('Ali');
const c = structuredClone(u);

console.log(c.name);                              // 'Ali'        — own property saqlandi
console.log(c.greet);                             // undefined    — metod prototipda edi
console.log(c.upper);                             // undefined    — getter prototipda edi
console.log(c instanceof User);                   // false
console.log(Object.getPrototypeOf(c) === Object.prototype); // true — oddiy obyektga aylandi
// #secret — umuman ko'chmaydi
```

**Qoida:** nusxa — **oddiy obyekt** (`Object.prototype`). Faqat **own, enumerable** ma'lumot qoladi; xatti-harakat (metodlar) va private maydonlar yo'qoladi.

### 8.2. Getter/setter — qiymatga aylanadi

```javascript
const o = { get now() { return Date.now(); } };
const c = structuredClone(o);

console.log(Object.getOwnPropertyDescriptor(c, 'now'));
// { value: 1790000000000, writable: true, enumerable: true, configurable: true }
// Getter klonlash paytida CHAQIRILDI va natija oddiy data-property bo'ldi
```

### 8.3. Property deskriptorlari va non-enumerable / symbol kalitlar

```javascript
const o = { a: 1 };
Object.defineProperty(o, 'hidden', { value: 2, enumerable: false });
o[Symbol('s')] = 3;
Object.freeze(o);

const c = structuredClone(o);
console.log(c);                    // { a: 1 } — hidden va symbol-kalit tushib qoldi
console.log(Object.isFrozen(c));   // false   — "muzlatilgan" holat saqlanmaydi
```

### 8.4. Maxsus `Error`lar

```javascript
class AppError extends Error {
  constructor(msg, code) { super(msg); this.code = code; }
}

const c = structuredClone(new AppError('Xato!', 42));
console.log(c instanceof AppError); // false
console.log(c.code);                // undefined — qo'shimcha maydon yo'qoldi
console.log(c.message);             // 'Xato!'
```

Standart `Error` turlari (`TypeError` va h.k.) turi bilan saqlanadi, lekin **o'zingiz yozgan subklass** va uning qo'shimcha maydonlari — yo'q.

### 8.5. `RegExp.lastIndex` va `Buffer`

```javascript
const re = /a/g;
re.test('aaa');
console.log(re.lastIndex);                  // 1
console.log(structuredClone(re).lastIndex); // 0 — qayta tiklanadi
```

Node.js'da `Buffer` (u `Uint8Array` subklassi) klonlanganda **oddiy `Uint8Array`**ga aylanadi — `Buffer` metodlari yo'qoladi.

### 8.6. Massivning "bo'sh joylari" va qo'shimcha property'lar

Massivning `length`i va o'z enumerable property'lari (qo'shimcha kalitlar ham) saqlanadi. Lekin umuman olganda **massiv + extra property** pattern'idan qochish tavsiya etiladi.

### Qisqa jadval — "nima saqlanadi, nima yo'qoladi"

| Saqlanadi ✅ | Yo'qoladi / o'zgaradi ⚠️ |
|---|---|
| Own enumerable string-kalitli qiymatlar | Prototip zanjiri (class → oddiy obyekt) |
| Sikllar va umumiy havolalar | Metodlar, getter/setter (getter → qiymat) |
| `Date`, `Map`, `Set`, `RegExp`, binary turlar | Symbol kalitlar, non-enumerable property'lar |
| `undefined`-qiymatli kalitlar, `NaN`, `-0`, `BigInt` | Property deskriptorlari, `freeze/seal` holati |
| Standart `Error` turlari (name, message) | Custom `Error` subklass va uning maydonlari |
| | `RegExp.lastIndex`, private `#maydon`lar |

---

## 9. JSON, spread, lodash `cloneDeep` bilan taqqoslash

### Usullar jadvali

| Imkoniyat | `JSON.parse(JSON.stringify())` | Spread / `Object.assign` | `_.cloneDeep` (lodash) | `structuredClone` |
|---|---|---|---|---|
| Chuqurligi | Chuqur | **Sayoz** (faqat 1-daraja) | Chuqur | Chuqur |
| `Date` | ❌ Satrga aylanadi | Havola nusxalanadi | ✅ | ✅ |
| `Map`, `Set` | ❌ `{}` bo'lib qoladi | Havola | ✅ | ✅ |
| `undefined` qiymat | ❌ Kalit tushib qoladi (massivda `null`) | ✅ | ✅ | ✅ |
| `NaN`, `Infinity`, `-0` | ❌ `null` / `null` / `0` | ✅ | ✅ | ✅ |
| `BigInt` | ❌ `TypeError` | ✅ | ✅ | ✅ |
| Siklik havola | ❌ `TypeError` | Havola | ✅ | ✅ |
| Funksiya | Jimgina tushib qoladi | Havola | Ichki funksiya **havola sifatida** o'tadi | ❌ **`DataCloneError`** |
| Class prototipi | ❌ Yo'qoladi | Yo'q (sayoz) | ✅ Saqlanadi | ❌ Yo'qoladi |
| Binary (`ArrayBuffer`, TypedArray) | ❌ Obyektga aylanadi | Havola | ✅ | ✅ |
| Transfer imkoniyati | ❌ | ❌ | ❌ | ✅ |
| Qo'shimcha paket kerakmi | Yo'q | Yo'q | Ha | Yo'q (o'rnatilgan) |

### Misol — JSON'ning "jimgina" buzishi

```javascript
const data = {
  date: new Date('2026-10-07'),
  map: new Map([['a', 1]]),
  nan: NaN,
  undef: undefined,
  fn: () => 1,
};

console.log(JSON.parse(JSON.stringify(data)));
// { date: '2026-10-07T00:00:00.000Z', map: {}, nan: null }
//   ↑ Date → satr,  Map → {},  NaN → null,  undef va fn — tushib qoldi (XATO HAM YO'Q!)

console.log(structuredClone({ ...data, fn: undefined }));
// { date: Date, map: Map(1), nan: NaN, undef: undefined, fn: undefined } — to'g'ri
```

### Qachon nimani tanlash?

| Vaziyat | Tavsiya |
|---|---|
| Oddiy ma'lumot (JSON-mos), kichik hajm | `structuredClone` yoki JSON — ikkalasi ham bo'ladi |
| `Date`/`Map`/`Set`/`BigInt`/sikl/binary bor | **`structuredClone`** |
| Class instance'lari — **prototip saqlanishi kerak** | `_.cloneDeep` yoki o'zingizning `clone()` metodingiz |
| Faqat 1-darajani o'zgartirish kerak | Spread (`{ ...obj, key: v }`) — tez va yetarli |
| React/Redux state'ni immutable yangilash | **Butun state'ni deep clone qilmang** — nuqtali spread yoki Immer (17-bo'limga qarang) |
| Worker'ga katta binary yuborish | `postMessage` + **transfer** |

---

## 10. `postMessage` amalda

`postMessage` — turli kontekstlar (oyna, iframe, worker, tab) o'rtasida **xabar almashish** usuli; yuklanayotgan ma'lumot **Structured Clone** bilan nusxalanadi.

### 10.1. Main thread ↔ Web Worker

```javascript
// main.js
const worker = new Worker('worker.js');

worker.postMessage({ type: 'SORT', numbers: [5, 3, 9, 1] });

worker.onmessage = (e) => {
  console.log('Natija:', e.data); // { type: 'DONE', sorted: [1, 3, 5, 9] }
};

worker.onmessageerror = (e) => {
  console.error('Xabarni deserialize qilib bo\'lmadi', e); // kamdan-kam
};
```

```javascript
// worker.js
self.onmessage = (e) => {
  const { type, numbers } = e.data;           // e.data — NUSXA (asl obyekt emas)
  if (type === 'SORT') {
    self.postMessage({ type: 'DONE', sorted: [...numbers].sort((a, b) => a - b) });
  }
};
```

> Bu — "Virtualization va Web Worker" hujjatida ko'rgan sxemaning aynan "kabeli": worker'ga yuborilgan katta massiv **nusxalanadi** (sekin), shuning uchun binary ma'lumot uchun **transfer** ishlatiladi (11-bo'lim).

### 10.2. Oyna ↔ iframe (cross-origin) — va xavfsizlik

```javascript
// Ota sahifa (https://app.example.com)
const iframe = document.querySelector('iframe');

iframe.contentWindow.postMessage(
  { type: 'INIT', token: 'abc' },
  'https://widget.example.com'       // targetOrigin — ANIQ ko'rsating!
);

window.addEventListener('message', (event) => {
  if (event.origin !== 'https://widget.example.com') return;   // ❗ MANBANI TEKSHIRING
  if (event.source !== iframe.contentWindow) return;           // ixtiyoriy, qo'shimcha himoya

  console.log('Iframe\'dan:', event.data);
});
```

**Xavfsizlik qoidalari (intervyuda ko'p so'raladi):**

1. **`targetOrigin` sifatida `'*'` ishlatmang** — xabar (maxfiy ma'lumot) kutilmagan origin'ga tushib qolishi mumkin.
2. **`message` handler ichida doim `event.origin`ni tekshiring** — aks holda istalgan sayt sizga xabar yubora oladi.
3. **`event.data`ni ishonchsiz (untrusted) ma'lumot** deb hisoblang: `innerHTML`ga, `eval`ga, URL'ga bevosita qo'ymang (XSS xavfi — keyingi "XSS/CSRF/CORS" mavzusi bilan bog'liq).
4. Kutilgan shaklni (`type` maydoni, sxema) validatsiya qiling.

### 10.3. `MessageChannel` — ikki tomonlama maxsus kanal

```javascript
const { port1, port2 } = new MessageChannel();

port1.onmessage = (e) => console.log('port1 oldi:', e.data);

// port2'ni worker'ga TRANSFER qilamiz (MessagePort — transferable)
worker.postMessage({ port: port2 }, [port2]);

port1.postMessage('Salom!');   // worker'dagi port orqali oladi
```

Bu pattern — **Comlink** va shunga o'xshash RPC kutubxonalarining asosi: worker bilan alohida "xususiy yo'lak" ochiladi.

### 10.4. `BroadcastChannel` — tab'lar o'rtasida

```javascript
// 1-tab
const bc = new BroadcastChannel('auth');
bc.postMessage({ type: 'LOGOUT' });

// 2-tab (bir origin)
const bc2 = new BroadcastChannel('auth');
bc2.onmessage = (e) => { if (e.data.type === 'LOGOUT') redirectToLogin(); };
```

### 10.5. `postMessage` imzolari

```javascript
window.postMessage(message, targetOrigin, transferList);
window.postMessage(message, { targetOrigin, transfer });

worker.postMessage(message, transferList);
worker.postMessage(message, { transfer });

port.postMessage(message, transferList);
```

---

## 11. Transferable obyektlar va `SharedArrayBuffer`

Nusxalash katta ma'lumot uchun qimmat: 100 MB `ArrayBuffer` — 100 MB **qo'shimcha** xotira va nusxalash vaqti. Buning o'rniga **uchta strategiya** bor:

| Strategiya | Nima bo'ladi | Tezlik | Jo'natuvchi kirishi |
|---|---|---|---|
| **Clone** (standart) | Ma'lumot **nusxalanadi** | O(n) — hajmga proporsional | Saqlanadi |
| **Transfer** | Egalik **ko'chiriladi**, nusxa yo'q | Deyarli O(1) | ❌ **Yo'qoladi** (detached) |
| **Share** (`SharedArrayBuffer`) | **Bitta xotira** ikki thread'da | Nusxa yo'q | ✅ Ikkalasida ham |

### 11.1. Transfer

```javascript
const buf = new ArrayBuffer(100 * 1024 * 1024); // 100 MB
console.log(buf.byteLength);                    // 104857600

worker.postMessage({ buf }, [buf]);             // 2-argument — transfer ro'yxati

console.log(buf.byteLength);                    // 0 — buffer "detached" bo'ldi (jo'natuvchida bo'sh)
```

`structuredClone` bilan ham:

```javascript
const buf = new ArrayBuffer(8);
const moved = structuredClone(buf, { transfer: [buf] });

console.log(buf.byteLength);    // 0
console.log(moved.byteLength);  // 8
```

**Transfer qilinadigan asosiy turlar:** `ArrayBuffer`, `MessagePort`, `ReadableStream`, `WritableStream`, `TransformStream`, `ImageBitmap`, `OffscreenCanvas`, `VideoFrame`, `AudioData`, `RTCDataChannel` va boshqalar.

**Muhim qoidalar:**

- Transfer ro'yxatiga **aniq** yozmasangiz, obyekt oddiy **nusxalanadi** (ichma-ich bo'lsa ham).
- TypedArray'ni transfer qilish uchun uning **`.buffer`**ini ro'yxatga yozasiz; jo'natuvchidagi view uzunligi 0 bo'lib qoladi.
- `ArrayBuffer.prototype.detached` (yangi brauzerlarda) — buffer "detached" ekanini tekshirishga imkon beradi; ishonchli umumiy usul: `byteLength === 0`.

### 11.2. `OffscreenCanvas` — DOM'siz render

```javascript
// DOM'dagi <canvas>ni worker'ga "topshirish"
const canvas = document.querySelector('canvas');
const offscreen = canvas.transferControlToOffscreen();

worker.postMessage({ canvas: offscreen }, [offscreen]);
// Endi worker bu canvas'ga chizadi, main thread band bo'lmaydi
```

Bu — "DOM node'lar klonlanmaydi" qoidasining **to'g'ri aylanma yo'li**: DOM element o'zi emas, uning chizish "huquqi" transfer qilinadi.

### 11.3. `SharedArrayBuffer`

```javascript
const shared = new SharedArrayBuffer(1024);
const view = new Int32Array(shared);

worker.postMessage({ shared });          // NUSXALANMAYDI — bir xil xotiraga ishora

// Ikkala thread ham o'qiy/yoza oladi — sinxronizatsiya uchun Atomics
Atomics.add(view, 0, 1);
```

**Cheklovlar:**

- Brauzerda **cross-origin isolation** talab qilinadi (`Cross-Origin-Opener-Policy: same-origin` va `Cross-Origin-Embedder-Policy: require-corp` sarlavhalari).
- **Race condition** xavfi qaytadi — `Atomics` yoki boshqa sinxronizatsiya shart.
- Oddiy ilovalar uchun kamdan-kam kerak; yuqori darajadagi parallel hisob-kitob (WASM, o'yin, video) uchun.

---

## 12. Noldan mini implementatsiya (o'quv modeli)

Algoritm g'oyasini tushunish uchun soddalashtirilgan versiya. **Haqiqiy dvigatel** ko'proq tur (TypedArray, Blob, Error va h.k.), transfer va Proxy aniqlashni qo'llab-quvvatlaydi.

```javascript
function myStructuredClone(value, memory = new Map()) {
  // 1) Klonlanmaydigan narsalar → DataCloneError
  if (typeof value === 'function' || typeof value === 'symbol') {
    throw new DOMException(`${typeof value} could not be cloned.`, 'DataCloneError');
  }

  // 2) Primitive'lar — o'zi qaytariladi (qiymat bo'yicha nusxa)
  if (value === null || typeof value !== 'object') return value;

  // 3) Xotira xaritasi: avval ko'rilgan obyekt bo'lsa — o'sha NUSXANI qaytaramiz
  //    (siklik va umumiy havolalarni saqlaydigan asosiy mexanizm)
  if (memory.has(value)) return memory.get(value);

  // 4) Rad etiladigan tur'lar
  if (
    value instanceof WeakMap || value instanceof WeakSet ||
    value instanceof Promise || value instanceof WeakRef ||
    (typeof Node !== 'undefined' && value instanceof Node)
  ) {
    throw new DOMException('Object could not be cloned.', 'DataCloneError');
  }

  let copy;

  if (value instanceof Date) {
    copy = new Date(value.getTime());
    memory.set(value, copy);

  } else if (value instanceof RegExp) {
    copy = new RegExp(value.source, value.flags);           // lastIndex — 0
    memory.set(value, copy);

  } else if (value instanceof Map) {
    copy = new Map();
    memory.set(value, copy);                                // AVVAL xotiraga — sikl uchun muhim!
    for (const [k, v] of value) {
      copy.set(myStructuredClone(k, memory), myStructuredClone(v, memory));
    }

  } else if (value instanceof Set) {
    copy = new Set();
    memory.set(value, copy);
    for (const v of value) copy.add(myStructuredClone(v, memory));

  } else if (value instanceof ArrayBuffer) {
    copy = value.slice(0);
    memory.set(value, copy);

  } else if (Array.isArray(value)) {
    copy = new Array(value.length);                          // uzunlik saqlanadi
    memory.set(value, copy);
    for (const key of Object.keys(value)) {
      copy[key] = myStructuredClone(value[key], memory);
    }

  } else {
    // Oddiy obyekt YOKI class instance → HAR IKKALASI oddiy obyektga aylanadi
    copy = {};                                               // prototip: Object.prototype
    memory.set(value, copy);
    for (const key of Object.keys(value)) {                  // own, enumerable, string kalitlar
      copy[key] = myStructuredClone(value[key], memory);     // getter bo'lsa — shu yerda chaqiriladi
    }
  }

  return copy;
}
```

### Sinov

```javascript
const a = { name: 'a' };
const input = { x: a, y: a, when: new Date(), list: new Set([1, 2]) };
input.self = input;

const out = myStructuredClone(input);
console.log(out.x === out.y);      // true  — umumiy havola saqlandi
console.log(out.self === out);     // true  — sikl saqlandi
console.log(out.x !== a);          // true  — asldan mustaqil

myStructuredClone({ f() {} });     // DOMException: DataCloneError
```

### Bu modelda ko'rinadigan 3 ta g'oya

1. **`memory` Map** — sikl va umumiy havolalarning kaliti. Konteynerni **rekursiyadan oldin** xaritaga yozish shart, aks holda siklda cheksiz rekursiya bo'ladi.
2. **Tur bo'yicha tarmoqlanish** — har bir "ichki slot"li tur (`Date`, `Map`, `Set`, `ArrayBuffer`) alohida ishlanadi (ularni oddiy obyekt kabi `Object.keys` bilan nusxalab bo'lmaydi).
3. **Oddiy yo'l — `copy = {}`** — shuning uchun class prototipi yo'qoladi; faqat `Object.keys` (own, enumerable, string) o'tadi.

> JavaScript'da obyekt **Proxy** ekanini aniqlashning yo'li yo'q — shuning uchun mini-implementatsiya Proxy'ni rad eta olmaydi. Haqiqiy `structuredClone` buni **dvigatel darajasida** qiladi.

---

## 13. Klonlanmaydigan narsalarni qanday hal qilish?

| Muammo | Yechim |
|---|---|
| **Funksiya** yuborish kerak | Funksiyaning o'zini emas, **buyruq (command) nomi + argumentlar**ni yuboring: `{ type: 'RESIZE', w: 100 }`. Worker tomonida `switch (type)` bilan mos funksiya chaqiriladi. RPC kerak bo'lsa — **Comlink** |
| **Class instance** (prototip kerak) | Ma'lumotni yuboring, qabul qiluvchi tomonda qayta yarating: `class User { toJSON(){…}  static from(data){ return Object.assign(new User(), data) } }` |
| **Proxy** (Vue `reactive`, Immer draft) | Asl obyektni oling: Vue — `toRaw(state)`; Immer — `current(draft)`. Keyin `structuredClone(raw)` |
| **DOM node** | Node'ning ID'sini, atributlarini, o'lchamlarini (`getBoundingClientRect()` natijasi — `DOMRect` serializable) yuboring. Canvas uchun — `transferControlToOffscreen()` |
| **Custom `Error`** | Qo'lda: `{ name: e.name, message: e.message, stack: e.stack, code: e.code }` |
| **`Symbol`** kalitlar/qiymatlar | Satr/son identifikatorga aylantiring |
| **`Promise`** | Avval `await` qiling, natijani yuboring |
| **Katta binary** | **Transfer** (yoki `SharedArrayBuffer`) |
| **Aralash ma'lumotdan faqat kerakli qismni** | Qo'lda "DTO" (data transfer object) yarating: `const { id, name } = user; post({ id, name })` |

### Misol — "funksiya o'rniga buyruq" (command pattern)

```javascript
// ❌ Ishlamaydi
worker.postMessage({ task: (x) => x * 2, value: 21 });   // DataCloneError

// ✅ Ishlaydi — kod worker ichida, main faqat NOM va ARGUMENT yuboradi
worker.postMessage({ task: 'double', value: 21 });

// worker.js
const tasks = { double: (x) => x * 2, square: (x) => x * x };
self.onmessage = (e) => {
  const { task, value } = e.data;
  self.postMessage(tasks[task](value));
};
```

### Misol — Vue reaktiv state'ni worker'ga yuborish

```javascript
import { reactive, toRaw } from 'vue';

const state = reactive({ items: [1, 2, 3] });

worker.postMessage(state);                    // ❌ DataCloneError (Proxy)
worker.postMessage(toRaw(state));             // ✅ asl obyekt
worker.postMessage(structuredClone(toRaw(state)));  // ✅ (ortiqcha — postMessage o'zi nusxalaydi)
```

### Misol — Immer draft

```javascript
import { produce, current } from 'immer';

produce(state, (draft) => {
  draft.count++;
  const snapshot = current(draft);    // draft'ning oddiy (Proxy bo'lmagan) surati
  worker.postMessage(snapshot);       // ✅
});
```

---

## 14. Loyihaning qaysi qismlarida ishlatiladi?

| Soha | Misol |
|---|---|
| **Web Worker'lar bilan ishlash** | Og'ir hisob-kitoblar (saralash, parsing, filtrlash) uchun ma'lumotni worker'ga yuborish va natijani qaytarib olish |
| **Micro-frontend / iframe integratsiyasi** | Ota sahifa ↔ iframe (to'lov formasi, widget, embed) o'rtasida xabar almashish |
| **Multi-tab sinxronizatsiya** | `BroadcastChannel` bilan logout, tema, savatcha o'zgarishini barcha tab'larga yetkazish |
| **Service Worker** | Sahifa ↔ SW (cache boshqaruvi, push, offline holat xabarlari) |
| **IndexedDB / offline-first** | Murakkab obyektlarni (Date, Blob, Map) bevosita saqlash — qo'lda serializatsiyasiz |
| **`history.pushState` state** | Sahifa holatini tarixga saqlash (funksiya/DOM bo'lmasligi shart) |
| **Immutable snapshot'lar** | Undo/redo, "draft" tahrirlash (forma bekor qilinsa — asl nusxaga qaytish), default sozlamalarni reset qilish |
| **Defensive copy** | Funksiyaga kelgan `options` yoki kutubxona ichki holatini tashqi mutatsiyadan himoyalash |
| **Test fixture'lar** | Har bir testda toza nusxa: `const data = structuredClone(baseFixture)` |
| **Node.js `worker_threads`** | Server tomonida CPU-og'ir ishlarni worker'ga topshirish (`workerData`, `postMessage`) |
| **Video/rasm/audio ishlov berish** | `ImageBitmap`, `OffscreenCanvas`, `VideoFrame` transfer qilish |

### Amaliy misol — "bekor qilish" bilan forma tahriri

```javascript
const saved = { name: 'Ali', prefs: { theme: 'dark', langs: ['uz', 'en'] } };

let draft = structuredClone(saved);       // tahrirlash uchun ishchi nusxa
draft.prefs.langs.push('ru');

function cancel() { draft = structuredClone(saved); }   // asl hech qachon o'zgarmagan
function save()   { Object.assign(saved, structuredClone(draft)); }
```

### Amaliy misol — IndexedDB'da obyektni to'g'ridan-to'g'ri saqlash

```javascript
const tx = db.transaction('notes', 'readwrite');
tx.objectStore('notes').put({
  id: 1,
  createdAt: new Date(),              // Date — to'g'ridan-to'g'ri
  tags: new Set(['js', 'idb']),       // Set — to'g'ridan-to'g'ri
  attachment: someBlob,               // Blob — to'g'ridan-to'g'ri
  // onSave: () => {}                 // ❌ funksiya bo'lsa — DataCloneError
});
```

> **Eslatma:** IndexedDB'da ma'lumot **saqlash** uchun ishlatilganda, ba'zi tur'lar (masalan, faqat tirik sessiyada ma'noga ega bo'lgan `SharedArrayBuffer`, `MessagePort` kabilar) ruxsat etilmaydi — spetsifikatsiyada bu "forStorage" rejimi deb ataladi.

---

## 15. Performance jihatlari

### 15.1. Nusxalash narxi hajmga proporsional

- `postMessage`/`structuredClone` — **O(n)**: ma'lumot qancha katta bo'lsa, shuncha **vaqt va xotira** ketadi.
- **Serialize jo'natuvchi thread'da, sinxron** bajariladi — katta obyektni main thread'dan yuborish **UI'ni bloklashi** mumkin (xabar yetkazish asinxron, lekin "surat olish" emas).
- **Deserialize** esa qabul qiluvchi thread'da bajariladi.

### 15.2. Kichik hajmda JSON ba'zan tezroq

Oddiy, JSON-mos va kichik obyektlarda `JSON.parse(JSON.stringify(x))` ba'zan `structuredClone`dan tezroq bo'lishi mumkin (JSON dvigatelda juda optimallashtirilgan). Binary ma'lumot, `Map`/`Set`/`Date`, sikllar bor bo'lsa — to'g'riligi va ko'pincha tezligi bo'yicha `structuredClone` ustun. **Aniq qaror uchun o'z ma'lumotingizda o'lchang** (`performance.now()`, DevTools Performance paneli).

### 15.3. Katta ma'lumot uchun qoidalar

| Holat | Tavsiya |
|---|---|
| Katta `ArrayBuffer`/TypedArray | **Transfer** (`postMessage(msg, [buf])`) |
| Doimiy, tez almashinadigan katta ma'lumot | `SharedArrayBuffer` + `Atomics` (cross-origin isolation kerak) |
| Juda katta JS obyekt grafi | Bo'laklarga (chunk) bo'lib yuboring; yoki worker ichida to'g'ridan-to'g'ri yuklang (masalan, `fetch` worker'ning o'zida) |
| Tez-tez kichik xabarlar | Xabarlarni guruhlang (batch), debounce/throttle bilan siyraklashtiring |

### 15.4. Yashirin tuzoq: TypedArray "view" butun buffer'ni nusxalaydi

```javascript
const big = new Uint8Array(500 * 1024 * 1024);   // 500 MB
const tiny = big.subarray(0, 10);                // 10 baytlik "oyna" (view)

worker.postMessage(tiny);
// ⚠️ view uchun uning BUTUN ArrayBuffer'i (500 MB) serialize qilinadi!

// ✅ To'g'ri: avval kichik mustaqil nusxa oling
worker.postMessage(big.slice(0, 10));            // .slice() — yangi, 10 baytlik buffer
```

---

## 16. Nega bu muhim — real foydalar

1. **Thread'lar va frame'lar o'rtasida xavfsiz aloqa** — bo'lishilgan o'zgaruvchan holat (shared mutable state) bo'lmagani uchun race condition'lar yo'q.
2. **To'g'ri chuqur nusxalash** — `Date`, `Map`, `Set`, `BigInt`, sikllar, binary — JSON'dan farqli, jimgina buzilmaydi.
3. **Kutubxonasiz** — `lodash.cloneDeep` uchun bundle hajmini oshirish shart emas.
4. **Katta ma'lumotlarni tejamkor ko'chirish** — Transferable orqali nusxalash narxisiz.
5. **Debug qilish qobiliyati** — "nega worker'ga yuborganimda `TypeError: x.method is not a function`?" (prototip yo'qolgan), "nega `DataCloneError`?" (funksiya/Proxy/DOM) kabi amaliy xatolarni tez topish.
6. **Xavfsizlik** — `postMessage`'da origin tekshiruvi, ishonchsiz ma'lumotni validatsiya qilish — real zaifliklar manbai.
7. **Intervyuda chuqur bilim ko'rsatkichi** — "nega funksiyalar klonlanmaydi?", "class instance nusxalansa nima bo'ladi?", "clone vs transfer vs share" — tipik senior savollar.

---

## 17. Eng ko'p uchraydigan xatolar

### Xato 1: Class instance nusxalansa, metodlari saqlanadi deb o'ylash

```javascript
const c = structuredClone(new User('Ali'));
c.greet();   // TypeError: c.greet is not a function
```
**Yechim:** ma'lumotni yuboring, qabul qiluvchi tomonda `User.from(data)` bilan qayta yarating.

### Xato 2: Reaktiv (Proxy) state'ni to'g'ridan-to'g'ri yuborish

```javascript
worker.postMessage(reactiveState);     // DataCloneError
```
**Yechim:** `toRaw()` (Vue) / `current()` (Immer) / oddiy ma'lumotga aylantiring.

### Xato 3: Yuborilgan obyektni keyin o'zgartirib, worker ham o'zgaradi deb kutish

```javascript
worker.postMessage(data);
data.x = 99;      // worker'dagi nusxaga TA'SIR QILMAYDI — nusxa allaqachon olingan
```
**Yechim:** o'zgarish kerak bo'lsa, yangi xabar yuboring. Haqiqiy "bo'lishish" kerak bo'lsa — `SharedArrayBuffer`.

### Xato 4: Transfer qilingan buffer'ni keyin ishlatish

```javascript
worker.postMessage(buf, [buf]);
console.log(buf.byteLength);          // 0
new Uint8Array(buf)[0];               // undefined / bo'sh — buffer "detached"
```
**Yechim:** transfer'dan keyin jo'natuvchida buffer'ga tegmang; kerak bo'lsa avval `.slice()` bilan nusxa saqlang.

### Xato 5: `postMessage(…, '*')` va `event.origin`ni tekshirmaslik

```javascript
window.addEventListener('message', (e) => {
  container.innerHTML = e.data.html;   // ❌ XSS + origin tekshiruvi yo'q
});
```
**Yechim:** `event.origin`ni ruxsat etilgan ro'yxatga qarshi tekshiring; ma'lumotni `textContent` bilan qo'ying yoki sanitizatsiya qiling.

### Xato 6: Butun state'ni `structuredClone` qilib, React'da immutable yangilash

```javascript
setState((s) => {
  const next = structuredClone(s);
  next.user.name = 'Vali';
  return next;               // ❌ HAR BIR ichki obyekt yangi havola → memo/useMemo/React.memo foydasiz
});
```
Hamma havola yangilangani uchun `React.memo`, `useMemo` (oldingi "Memoization" hujjati) **reference tengligiga** tayanadi va keraksiz qayta render'lar ko'payadi.
**Yechim:** faqat o'zgargan yo'lni yangilang: `{ ...s, user: { ...s.user, name: 'Vali' } }` yoki Immer.

### Xato 7: `Error` obyektini worker'dan qaytarib, `instanceof AppError` tekshirish

**Yechim:** xatoni `{ name, message, code, stack }` ko'rinishida qo'lda serialize qiling.

### Xato 8: Katta ma'lumotni main thread'da tez-tez `postMessage` qilish

**Yechim:** transfer, batch, chunk yoki ma'lumotni worker ichida yuklash.

### Xato 9: `structuredClone` brauzer/Node versiyasida mavjudligini tekshirmaslik

```javascript
const clone = typeof structuredClone === 'function'
  ? structuredClone(obj)
  : fallbackClone(obj);       // polyfill / JSON / lodash
```

### Xato 10: `JSON.parse(JSON.stringify())` ni "xavfsiz deep clone" deb hisoblash

Jimgina `Date → string`, `Map → {}`, `undefined`/funksiya tushib qolishi, `NaN → null` — bu xatolar **hech qanday ogohlantirishsiz** ketadi (9-bo'lim).

---

## 18. Interview savollari va qisqa javoblar

**S: Structured Clone Algorithm nima?**
> J: Bu — JS qiymatlarini (obyekt graflarini) chuqur nusxalash uchun HTML standartida ta'riflangan algoritm. U `structuredClone()`, `postMessage`, IndexedDB, `history.pushState` ichida ishlatiladi: qiymat avval neytral ko'rinishga serialize qilinadi, so'ng qabul qiluvchi realm'da yangidan quriladi.

**S: Nega `postMessage` obyektni havola bilan emas, nusxalab yuboradi?**
> J: Chunki thread'lar va realm'lar o'z heap xotirasiga ega va uni bo'lishmaydi — bu race condition'lar va xavfsizlik muammolarini oldini oladi. Shuning uchun ma'lumot serialize → deserialize qilinib, qabul qiluvchida mustaqil nusxa hosil bo'ladi.

**S: Funksiyalar nega klonlanmaydi?**
> J: Funksiya kod bilan birga o'zi yaratilgan scope'ga (closure) bog'langan. Scope zanjirini, o'zgaruvchilarni va global'larni boshqa heap/thread'ga ko'chirib bo'lmaydi; ko'chirish xavfsizlik jihatidan ham xavfli bo'lardi (boshqa kontekstda kod bajarish). Shuning uchun funksiya uchrasa `DataCloneError` tashlanadi.

**S: Nima klonlanmaydi?**
> J: Funksiyalar, `Symbol` qiymatlari, DOM node'lar, `Proxy`, `WeakMap`/`WeakSet`/`WeakRef`, `Promise` va "serializable" deb belgilanmagan platforma obyektlari. Xato chuqurlikdan qat'i nazar butun operatsiyani to'xtatadi (`DOMException`, `name: 'DataCloneError'`).

**S: Class instance'ni `structuredClone` qilsak nima bo'ladi?**
> J: Xato bermaydi, lekin natija oddiy obyekt bo'ladi: prototip (metodlar, getter'lar) va private `#maydon`lar yo'qoladi, faqat own enumerable ma'lumot qoladi; `instanceof` `false` qaytaradi.

**S: `structuredClone` va `JSON.parse(JSON.stringify())` farqi nima?**
> J: JSON `Date`ni satrga, `Map`/`Set`ni `{}`ga aylantiradi, `undefined`/funksiyani tashlab yuboradi, `NaN`/`Infinity`ni `null` qiladi, `BigInt`da va sikllarda xato beradi. `structuredClone` bularni to'g'ri nusxalaydi, binary turlarni va transferni qo'llaydi; lekin funksiya uchraganda xato beradi (JSON jimgina tashlaydi).

**S: Siklik havolalar va umumiy havolalar qanday saqlanadi?**
> J: Algoritm "xotira xaritasi" (memory map) yuritadi — har bir ko'rilgan obyekt va uning nusxasi yoziladi. Obyekt qayta uchraganda yangi nusxa yaratilmaydi, xaritadagi mavjudi qaytariladi. Shu tufayli sikl va umumiy havolalar nusxada ham saqlanadi.

**S: Transferable obyektlar nima?**
> J: Nusxalash o'rniga **egaligi ko'chiriladigan** obyektlar (`ArrayBuffer`, `MessagePort`, `OffscreenCanvas`, `ReadableStream` va h.k.). Transfer deyarli O(1) tezlikda bo'ladi, lekin jo'natuvchida obyekt "detached" bo'lib qoladi (masalan, `ArrayBuffer.byteLength` 0 bo'ladi).

**S: Transfer va `SharedArrayBuffer` farqi nima?**
> J: Transfer'da xotira bir thread'dan ikkinchisiga **ko'chadi** (bir vaqtda faqat bitta egasi bor). `SharedArrayBuffer`'da esa **bitta xotira** ikki thread'da bir vaqtda ko'rinadi, shu sababli sinxronizatsiya (`Atomics`) va cross-origin isolation (COOP/COEP) talab qilinadi.

**S: Reaktiv (Vue/Immer) obyektni worker'ga yuborsam nima uchun xato beradi?**
> J: Ular `Proxy`, Structured Clone esa Proxy'ni rad etadi (`DataCloneError`). Yechim: asl obyektni olish — Vue'da `toRaw()`, Immer'da `current(draft)`.

**S: `postMessage` bilan ishlaganda xavfsizlik bo'yicha nimalarga e'tibor berasiz?**
> J: `targetOrigin`ni aniq ko'rsatish (`'*'` emas), qabul qilishda `event.origin` (va kerak bo'lsa `event.source`)ni tekshirish, `event.data`ni ishonchsiz ma'lumot deb validatsiya qilish va uni `innerHTML`/`eval`ga qo'ymaslik.

**S: `structuredClone()` sinxronmi yoki asinxronmi?**
> J: Sinxron. (`postMessage`da ham serialize sinxron bajariladi, faqat xabarni yetkazish asinxron.) `structuredClone`dan oldin odamlar `MessageChannel` orqali asinxron "hack" ishlatishgan.

**S: Katta ma'lumotni worker'ga yuborishni qanday optimallashtirasiz?**
> J: Binary ma'lumotni transfer qilaman; doimiy ulashiladigan holat bo'lsa `SharedArrayBuffer` + `Atomics`; juda katta grafni bo'laklab yuboraman yoki worker'ning o'zida yuklayman; TypedArray `subarray` view'ini emas, `.slice()` nusxasini yuboraman (view butun buffer'ni nusxalab yuboradi).

---

## 19. Xulosa

> **Structured Clone Algorithm** — JS qiymatlarini heap'lar, thread'lar va realm'lar chegarasidan **xavfsiz o'tkazish** mexanizmi: qiymat neytral ko'rinishga **serialize** qilinadi (chaqirilgan paytning o'zida, sinxron) va qabul qiluvchi tomonda **yangi obyekt grafi** sifatida **deserialize** qilinadi. U `structuredClone()`, `postMessage`, IndexedDB va `history.pushState` ning umumiy asosi.
>
> **Kuchli tomonlari:** siklik va umumiy havolalarni saqlaydi, `Date`, `Map`, `Set`, `RegExp`, `BigInt`, binary turlarni to'g'ri nusxalaydi, kutubxona talab qilmaydi va **Transferable** orqali katta ma'lumotni nusxalamasdan ko'chira oladi.
>
> **Cheklovlari:** funksiyalar, `Symbol`, DOM node'lar, `Proxy`, `Weak*`, `Promise` — **`DataCloneError`**; class'larning prototipi, getter/setter'lari, private maydonlari, deskriptorlari va custom `Error` subklasslari esa **jimgina yo'qoladi**. Shuning uchun nusxa — doim "oddiy ma'lumot" bo'ladi, xatti-harakat emas.
>
> **Amaliy qoida:** thread'lar orasida **ma'lumot** yuboring, **kod/obyekt identifikatsiyasi**ni emas; xatti-harakatni qabul qiluvchi tomonda qayta quring (command pattern, `Class.from(data)`); katta binary uchun **transfer**, doimiy ulashuv uchun **SharedArrayBuffer**; `postMessage`da **origin**ni har doim tekshiring.
>
> Bu mavzuni chuqur tushunish — interview'da "Web Worker'ga nima yuborib bo'ladi va nega?", "deep clone usullari farqi", "`DataCloneError` sababi" kabi savollarga ishonchli javob berish hamda real loyihalarda Worker, iframe va multi-tab arxitekturalarni xatosiz qurish uchun muhim asos.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
