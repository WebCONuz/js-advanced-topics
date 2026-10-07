# Event Loop, Microtask va Macrotask — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [Nega Event Loop kerak?](#1-nega-event-loop-kerak)
2. [Asosiy komponentlar](#2-asosiy-komponentlar)
3. [Event Loop qanday ishlaydi — qadam-baqadam](#3-event-loop-qanday-ishlaydi--qadam-baqadam)
4. [Misol 1: Oddiy tartib](#4-misol-1-oddiy-tartib)
5. [Misol 2: Ichma-ich (nested) Promise'lar](#5-misol-2-ichma-ich-nested-promiselar)
6. [async/await va Event Loop](#6-asyncawait-va-event-loop)
7. [Node.js'dagi farqlar: process.nextTick va setImmediate](#7-nodejsdagi-farqlar-processnexttick-va-setimmediate)
8. [Microtask Starvation — xavfli holat](#8-microtask-starvation--xavfli-holat)
9. [Taqqoslash jadvali](#9-taqqoslash-jadvali)
10. [Interview savollari va qisqa javoblar](#10-interview-savollari-va-qisqa-javoblar)
11. [Xulosa](#11-xulosa)

---

## 1. Nega Event Loop kerak?

JavaScript — **single-threaded** til, ya'ni bir vaqtning o'zida faqat **bitta** amalni bajara oladi (bitta Call Stack bor). Agar shunday bo'lsa, `setTimeout`, `fetch`, fayl o'qish kabi uzoq davom etadigan amallar butun dasturni "muzlatib" qo'yishi kerak edi-ku?

Aynan shu muammoni **Event Loop** hal qiladi — u asinxron (asynchronous) amallarni brauzer yoki Node.js muhitiga topshiradi, ular tugagach, natijani to'g'ri vaqtda qaytadan Call Stack'ga qaytaradi, va bularning barchasi **asosiy oqimni bloklamasdan** amalga oshadi.

---

## 2. Asosiy komponentlar

Event Loop tushunish uchun quyidagi 4 ta qismni bilish shart:

### 2.1. Call Stack
Kodning **synchronous** qismi shu yerda bajariladi. LIFO (Last In, First Out) tamoyilida ishlaydi.

### 2.2. Web APIs (brauzerda) / C++ APIs (Node.js'da)
`setTimeout`, `fetch`, DOM event'lar, fayl operatsiyalari kabi asinxron amallar shu yerga "topshiriladi" va fonda bajariladi.

### 2.3. Microtask Queue (Job Queue)
Yuqori ustuvorlikka ega navbat. Bu yerga quyidagilar tushadi:
- `Promise.then/catch/finally`
- `queueMicrotask()`
- `MutationObserver`
- `async/await` (ichki qismda Promise orqali ishlaydi)

### 2.4. Macrotask Queue (Callback / Task Queue)
Past ustuvorlikka ega navbat. Bu yerga quyidagilar tushadi:
- `setTimeout`, `setInterval`
- `setImmediate` (faqat Node.js)
- I/O operatsiyalari
- UI rendering, event handler'lar (`click`, `scroll` va h.k.)

```
┌─────────────────┐
│   Call Stack     │ ← sync kod shu yerda bajariladi
└─────────────────┘
        │
        │ bo'shagach
        ▼
┌─────────────────────────┐
│   Microtask Queue        │ ← TO'LIQ bo'shatiladi (har safar)
│   (Promise.then, va h.k.)│
└─────────────────────────┘
        │
        │ bo'shagach
        ▼
┌─────────────────────────┐
│   Macrotask Queue         │ ← FAQAT BITTA task olinadi
│   (setTimeout, va h.k.)   │
└─────────────────────────┘
        │
        └──── Event Loop qaytadan Microtask Queue'ni tekshiradi
```

---

## 3. Event Loop qanday ishlaydi — qadam-baqadam

Event Loop'ning asosiy qoidasi juda muhim va deyarli har bir interview'da so'raladi:

> **Call Stack bo'shagandan so'ng, avval Microtask Queue TO'LIQ tozalanadi (yangi qo'shilgan microtasklar bilan birga), va faqat shundan keyin Macrotask Queue'dan BITTA task olinadi. Shu tsikl takrorlanaveradi.**

Umumiy algoritm:

```
1. Call Stack'dagi barcha sync kodni bajar
2. Microtask Queue'ni TO'LIQ bo'shat
   (agar bajarish jarayonida yangi microtask qo'shilsa — ularni ham shu yerda bajar)
3. Bitta Macrotask'ni Call Stack'ga olib kelib bajar
4. 2-qadamga qaytish (Microtask Queue'ni yana tekshir)
5. Takrorlash...
```

---

## 4. Misol 1: Oddiy tartib

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
queueMicrotask(() => console.log('4'));
console.log('5');
```

### Natija:
```
1
5
3
4
2
```

### Tushuntirish

| Qadam | Kod | Nima bo'ladi |
|---|---|---|
| 1 | `console.log('1')` | Sync — darhol bajariladi |
| 2 | `setTimeout(...)` | Web API'ga jo'natiladi, callback **Macrotask Queue**ga qo'yiladi |
| 3 | `Promise.resolve().then(...)` | Callback **Microtask Queue**ga qo'yiladi |
| 4 | `queueMicrotask(...)` | Callback **Microtask Queue**ga qo'yiladi |
| 5 | `console.log('5')` | Sync — darhol bajariladi |

Call Stack bo'shagandan keyin:
- **Microtask Queue** to'liq tozalanadi → `3`, keyin `4` chiqadi
- Faqat shundan keyin **Macrotask Queue**dan `setTimeout` callback'i olinadi → `2` chiqadi

---

## 5. Misol 2: Ichma-ich (nested) Promise'lar

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => {
  console.log('3');
  Promise.resolve().then(() => console.log('4'));
});

Promise.resolve().then(() => console.log('5'));

console.log('6');
```

### Natija:
```
1
6
3
5
4
2
```

### Tushuntirish

**Sync qism:**
```
1
6
```

**Microtask Queue boshlanishida:**
```
[ Promise1.then → "3", Promise2.then → "5" ]
```

**Birinchi microtask bajarilganda** (`console.log('3')`), uning ICHIDA yangi `Promise.resolve().then(() => console.log('4'))` yaratiladi — bu ham **shu tsikldagi Microtask Queue**ning oxiriga qo'shiladi:

```
[ Promise2.then → "5", yangi Promise → "4" ]
```

**Keyingi microtask:** `5` chiqadi
**Keyingi microtask (yangi qo'shilgan):** `4` chiqadi

Microtask Queue endi bo'sh — faqat shundan keyin Macrotask Queue'ga o'tiladi:

**Macrotask:** `2` chiqadi

> **Eng muhim xulosa:** Agar microtask ichida yangi microtask yaratilsa, u ham **shu siklda**, keyingi macrotask'ga navbat bermasdan oldin bajariladi. Bu Microtask Queue "to'liq bo'shaguncha" degani.

---

## 6. async/await va Event Loop

`async/await` — bu Promise ustidan qurilgan **syntactic sugar** (sintaktik shakar), ya'ni "ichida" xuddi Promise kabi ishlaydi, faqat kodni yozish qulayroq bo'ladi.

```javascript
console.log('1');

async function foo() {
  console.log('2');
  await null; // shu yerdan keyingi qism MICROTASK sifatida navbatga qo'yiladi
  console.log('3');
}

foo();

console.log('4');
```

### Natija:
```
1
2
4
3
```

### Tushuntirish

- `foo()` chaqirilganda, funksiya ichidagi kod **darhol, sinxron tarzda** boshlanadi → `2` chiqadi
- `await` ga yetganda, funksiya **"pauza"** qiladi va qolgan qismi (`console.log('3')`) **microtask** sifatida navbatga qo'yiladi — xuddi `.then()` kabi
- `foo()` chaqiruvi shu yerda "to'xtaydi", boshqaruv qaytadan asosiy kodga o'tadi → `4` chiqadi
- Call Stack bo'shagach, Microtask Queue ishga tushadi → `3` chiqadi

> **Interview uchun eslatma:** `await someValue` aslida `Promise.resolve(someValue).then(davomEtish)` bilan bir xil samara beradi. Shuning uchun `await`dan keyingi barcha kod **microtask** sifatida bajariladi, hatto `await`ilgan qiymat oddiy son yoki `null` bo'lsa ham.

---

## 7. Node.js'dagi farqlar: process.nextTick va setImmediate

Brauzerdan farqli o'laroq, **Node.js**da yana ikkita maxsus queue bor:

### `process.nextTick()`
Bu **eng yuqori ustuvorlikka** ega — hatto oddiy microtasklardan ham oldin bajariladi. Har bir operatsiyadan keyin, navbatdagi ish boshlanishidan oldin `nextTick` queue **to'liq** tozalanadi.

### `setImmediate()`
Bu Macrotask Queue'ning maxsus turi — I/O operatsiyalaridan keyin bajarilishi mo'ljallangan.

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);
setImmediate(() => console.log('3'));

process.nextTick(() => console.log('4'));

Promise.resolve().then(() => console.log('5'));

console.log('6');
```

### Natija (Node.js'da):
```
1
6
4
5
2 yoki 3 (tartib holatga qarab farq qilishi mumkin)
```

**Ustuvorlik tartibi (Node.js'da):**
```
Sync kod → process.nextTick queue → Promise (microtask) queue → Macrotask (Timers, setImmediate, I/O)
```

> **Interview uchun eslatma:** `setTimeout(fn, 0)` va `setImmediate(fn)`ning qaysi biri oldin bajarilishi ba'zan **kafolatlanmagan** — bu Node.js ichki holatiga (event loop'ning qaysi fazasida turganiga) bog'liq. Faqat I/O callback ichida chaqirilganda, `setImmediate` har doim `setTimeout(fn, 0)`dan oldin bajarilishi kafolatlanadi.

---

## 8. Microtask Starvation — xavfli holat

Agar microtask ichida cheksiz ravishda yangi microtask yaratilaversa, Macrotask Queue (jumladan `setTimeout`, render, I/O, hatto user interaction'lar) **umuman ishga tushmay qoladi** — bu **"microtask starvation"** deb ataladi va brauzerni "osilib qolishiga" (UI freeze) olib keladi.

```javascript
// XATO MISOL — bu barcha macrotasklarni abadiy bloklaydi
function loop() {
  Promise.resolve().then(loop);
}
loop();

setTimeout(() => console.log('Bu hech qachon chiqmaydi!'), 0);
```

**Nega bu xavfli?** Chunki Event Loop qoidasiga ko'ra, Macrotask Queue'ga o'tishdan oldin Microtask Queue **to'liq** bo'shashi kerak. Agar u hech qachon to'liq bo'shamasa (chunki har safar yangi microtask qo'shilib turadi), Macrotask'lar abadiy navbatda qolib ketadi.

---

## 9. Taqqoslash jadvali

| | Microtask Queue | Macrotask Queue |
|---|---|---|
| **Misollar** | `Promise.then/catch/finally`, `queueMicrotask`, `async/await`, `MutationObserver` | `setTimeout`, `setInterval`, `setImmediate` (Node), I/O, DOM event'lar, rendering |
| **Ustuvorlik** | Yuqori | Past |
| **Tozalash tartibi** | Har safar **to'liq** bo'shatiladi (yangi qo'shilganlari bilan birga) | Har safar navbatdan faqat **bitta** task olinadi |
| **Xavfi** | Microtask starvation (cheksiz rekursiya bilan Macrotask'larni bloklashi mumkin) | Nisbatan xavfsizroq, lekin sekinroq bajariladi |

---

## 10. Interview savollari va qisqa javoblar

**S: Event Loop nima va u nima uchun kerak?**
> J: Event Loop — JavaScript'ning single-threaded bo'lishiga qaramay, asinxron operatsiyalarni (masalan, `setTimeout`, tarmoq so'rovlari) asosiy oqimni bloklamasdan bajarish imkonini beruvchi mexanizm. U Call Stack, Web API, Microtask va Macrotask Queue'lar orasida muvofiqlashtirib turadi.

**S: Microtask va Macrotask o'rtasidagi asosiy farq nima?**
> J: Microtask'lar (Promise, queueMicrotask) doim Macrotask'lardan (setTimeout, setInterval) OLDIN bajariladi. Har safar Call Stack bo'shaganda, Event Loop avval Microtask Queue'ni TO'LIQ tozalaydi, keyin Macrotask Queue'dan bitta task oladi.

**S: Nega `setTimeout(fn, 0)` darhol bajarilmaydi?**
> J: Chunki `setTimeout` — bu Macrotask, va u bajarilishidan oldin Call Stack bo'shashi hamda Microtask Queue'dagi BARCHA vazifalar (jumladan bajarilish jarayonida yangi qo'shilganlari ham) to'liq tugatilishi shart.

**S: `await`dan keyingi kod qanday tartibda bajariladi?**
> J: `await`dan keyingi kod avtomatik ravishda **microtask** sifatida navbatga qo'yiladi — xuddi `.then()` callback'i kabi. Shuning uchun u sinxron koddan keyin, lekin `setTimeout` kabi macrotasklardan oldin bajariladi.

**S: Microtask starvation nima?**
> J: Bu — microtask ichida doimiy ravishda yangi microtask yaratilishi natijasida Macrotask Queue (va demak, `setTimeout`, render, foydalanuvchi interaktivligi) umuman ishga tushmay qolishi holati. Bu brauzerda UI'ning "osilib qolishiga" olib kelishi mumkin.

**S: Node.js'da `process.nextTick` va oddiy Promise microtask'ning farqi bormi?**
> J: Ha. `process.nextTick` Node.js'ga xos, alohida navbat bo'lib, u har doim oddiy Promise microtasklaridan HAM oldin bajariladi. Ikkalasi ham sinxron koddan keyin, lekin Macrotasklardan oldin ishlaydi.

---

## 11. Xulosa

> **Event Loop — JavaScript'ning "yuragi".** U single-thread bo'la turib, minglab asinxron amallarni bir vaqtda "boshqarayotgandek" ko'rinishga imkon beradi. Buning siri — Call Stack, Web API, Microtask Queue va Macrotask Queue o'rtasidagi aniq va qat'iy tartibda: har doim avval sync kod, keyin **to'liq** Microtask Queue, va faqat shundan keyin Macrotask Queue'dan **bitta** vazifa.
>
> Bu mexanizmni chuqur tushunish — nafaqat interview'larda, balki real loyihalarda performance muammolarini (masalan, UI freeze yoki noto'g'ri tartibda bajariluvchi kod) aniqlash va tuzatishda ham juda muhim.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
