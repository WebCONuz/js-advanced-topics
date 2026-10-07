# Symbol va Well-known Symbols — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.
> Asosiy fokus: `Symbol`, `Symbol.iterator`, `Symbol.asyncIterator` va iterator protokollari.

---

## Mundarija

1. [Symbol nima va u qayerdan paydo bo'lgan?](#1-symbol-nima-va-u-qayerdan-paydo-bolgan)
2. [Symbol yaratish va asosiy xususiyatlari](#2-symbol-yaratish-va-asosiy-xususiyatlari)
3. [Global Symbol Registry: `Symbol.for()` va `Symbol.keyFor()`](#3-global-symbol-registry-symbolfor-va-symbolkeyfor)
4. [Symbol'lar property kaliti sifatida](#4-symbollar-property-kaliti-sifatida)
5. [Well-known Symbols — umumiy ko'rinish](#5-well-known-symbols--umumiy-korinish)
6. [`Symbol.iterator` va Iterator protokoli](#6-symboliterator-va-iterator-protokoli)
7. [Custom iterable obyektlar yaratish](#7-custom-iterable-obyektlar-yaratish)
8. [Generator'lar va `Symbol.iterator`](#8-generatorlar-va-symboliterator)
9. [`Symbol.asyncIterator` va `for await...of`](#9-symbolasynciterator-va-for-awaitof)
10. [Boshqa muhim well-known symbol'lar](#10-boshqa-muhim-well-known-symbollar)
11. [Loyihaning qaysi qismlarida ishlatiladi?](#11-loyihaning-qaysi-qismlarida-ishlatiladi)
12. [Nega bu muhim — real foydalar](#12-nega-bu-muhim--real-foydalar)
13. [Eng ko'p uchraydigan xatolar](#13-eng-kop-uchraydigan-xatolar)
14. [Interview savollari va qisqa javoblar](#14-interview-savollari-va-qisqa-javoblar)
15. [Xulosa](#15-xulosa)

---

## 1. Symbol nima va u qayerdan paydo bo'lgan?

**Symbol** — ES2015 (ES6) standartida kiritilgan **primitive (oddiy) ma'lumot turi** bo'lib, uning har bir qiymati **butunlay noyob (unique)** va **o'zgarmas (immutable)** hisoblanadi.

```javascript
const a = Symbol('id');
const b = Symbol('id');

console.log(a === b);   // false — bir xil tavsifga ega bo'lsa ham, ikkalasi NOYOB
console.log(typeof a);  // "symbol"
```

### Nega Symbol yaratilgan? — tarixiy sabab

ES2015 gacha obyekt property kalitlari **faqat string** bo'la olardi. Bu ikkita jiddiy muammo tug'dirgan:

**1. Nom to'qnashuvi (name collision):**

```javascript
// Kutubxona A obyektga "id" property qo'shadi
user.id = 'lib-a-id';

// Kutubxona B ham "id" property qo'shadi — A'ning qiymatini BUZADI!
user.id = 'lib-b-id';
```

**2. Orqaga moslik (backward compatibility) muammosi — eng muhim sabab:**

Til yaratuvchilariga yangi "sehrli" (maxsus) protokollar qo'shish kerak edi (masalan, "bu obyekt ustida sikl yurish mumkin"). Agar buni oddiy string nom bilan qilishsa (masalan, `obj.iterator()` yoki `obj.next()`), millionlab mavjud veb-saytlarda foydalanuvchilar **o'zlari yozgan** `iterator` nomli metodlar bilan **to'qnashib**, ularni buzib qo'yardi.

**Yechim:** Noyobligi **kafolatlangan** yangi kalit turi — **Symbol**. Endi til `Symbol.iterator` kabi maxsus kalitlarni qo'sha oladi, va ular hech qachon foydalanuvchi kodidagi hech qanday string kalit bilan to'qnashmaydi.

> **Interview uchun eng muhim jumla:** Symbol'lar ikki asosiy maqsadda yaratilgan: (1) to'qnashuvsiz, noyob property kalitlari yaratish; (2) tilga **mavjud kodni buzmasdan** yangi protokollarni (iterator, toPrimitive va h.k.) qo'shish — aynan shu uchun "well-known symbols" mavjud.

---

## 2. Symbol yaratish va asosiy xususiyatlari

### 2.1. Yaratish

```javascript
const sym1 = Symbol();              // tavsifsiz
const sym2 = Symbol('myKey');       // tavsif bilan (faqat debugging uchun)

console.log(sym2.toString());       // "Symbol(myKey)"
console.log(sym2.description);      // "myKey" (ES2019)
```

> **Tavsif (description)** faqat **debugging** (xatolarni topish) uchun — u symbol'ning noyobligiga **hech qanday ta'sir qilmaydi**.

### 2.2. `new` bilan chaqirib bo'lmaydi

```javascript
const s = new Symbol('x'); // TypeError: Symbol is not a constructor
```

Symbol — primitive, shuning uchun `new` ishlatilmaydi (xuddi `new Number()` dan farqli o'laroq, bu yerda wrapper yaratish umuman taqiqlangan).

### 2.3. Avtomatik string'ga aylantirilmaydi

```javascript
const sym = Symbol('test');

console.log(String(sym));      // "Symbol(test)" — aniq (explicit) aylantirish ishlaydi
console.log(sym.toString());   // "Symbol(test)"

console.log(`${sym}`);         // TypeError! Template literal — implicit konvertatsiya
console.log(sym + '');         // TypeError!
```

Bu — ataylab qilingan xavfsizlik chorasi: symbol tasodifan string'ga aylanib, oddiy property kalitiga (va shu bilan noyobligini yo'qotishga) aylanib ketmasligi uchun.

### 2.4. Qisqa xulosa jadvali

| Xususiyat | Qiymati |
|---|---|
| `typeof` | `"symbol"` |
| Noyoblik | Har bir `Symbol()` chaqiruvi — yangi, noyob qiymat |
| `new` bilan | Mumkin emas |
| Implicit string konvertatsiya | `TypeError` |
| Property kaliti sifatida | Mumkin (asosiy ishlatilish o'rni) |
| `JSON.stringify` | E'tiborga olinmaydi (o'tkazib yuboriladi) |

---

## 3. Global Symbol Registry: `Symbol.for()` va `Symbol.keyFor()`

Oddiy `Symbol()` har safar yangi, noyob qiymat yaratadi. Lekin ba'zan **turli joylardan (hatto turli fayl yoki iframe'lardan) bir xil symbol'ga murojaat qilish** kerak bo'ladi. Buning uchun **global registry** mavjud.

```javascript
const s1 = Symbol.for('app.userId');
const s2 = Symbol.for('app.userId');

console.log(s1 === s2);  // true — registry'da bir xil kalit bilan topildi

console.log(Symbol.keyFor(s1));  // "app.userId" — symbol'ning registry kalitini qaytaradi

const local = Symbol('local');
console.log(Symbol.keyFor(local)); // undefined — bu registry'da emas
```

**Qanday ishlaydi?** `Symbol.for(key)` — registry'da shu `key` bilan symbol bormi, tekshiradi. Bor bo'lsa — uni qaytaradi, yo'q bo'lsa — yangisini yaratib, registry'ga qo'yadi va qaytaradi.

> **Muhim:** Well-known symbol'lar (`Symbol.iterator` va h.k.) **global registry'da emas**: `Symbol.keyFor(Symbol.iterator)` → `undefined`. Lekin ular har bir realm (iframe, worker) uchun mos ravishda bir xil ma'noga ega.

**Amaliy misol — React:** React elementlarini belgilash uchun `$$typeof` maydonida `Symbol.for(...)` ishlatiladi. Bu xavfsizlik uchun muhim: JSON orqali serverdan kelgan zararli ma'lumot Symbol'ni **taqlid qila olmaydi** (chunki JSON symbol'larni qo'llab-quvvatlamaydi), shuning uchun XSS orqali soxta React elementi kiritib bo'lmaydi.

---

## 4. Symbol'lar property kaliti sifatida

```javascript
const id = Symbol('id');

const user = {
  name: 'Ali',
  [id]: 12345,           // computed property name — qavs shart!
};

console.log(user[id]);   // 12345
console.log(user.id);    // undefined — bu string "id", symbol emas
```

### Symbol kalitlar "yashirin" holatda

```javascript
console.log(Object.keys(user));          // ['name'] — symbol YO'Q
console.log(Object.getOwnPropertyNames(user)); // ['name']
console.log(JSON.stringify(user));       // '{"name":"Ali"}' — symbol tushib qoladi

for (const key in user) {
  console.log(key);                       // faqat "name"
}
```

### Ularni topishning yo'llari

```javascript
console.log(Object.getOwnPropertySymbols(user)); // [Symbol(id)]
console.log(Reflect.ownKeys(user));              // ['name', Symbol(id)] — hammasi
```

### Nusxalashda

```javascript
const copy = { ...user };
console.log(copy[id]); // 12345 — spread va Object.assign enumerable symbol kalitlarni NUSXALAYDI
```

### Symbol — "haqiqiy private" emas!

```javascript
// Symbol kalitlar "yashirin", lekin SIR emas
Object.getOwnPropertySymbols(user); // istalgan kishi topa oladi
```

Haqiqiy maxfiylik kerak bo'lsa, **`#private` maydonlar** (ES2022) yoki closure ishlatiladi. Symbol — faqat **nom to'qnashuvidan himoya**, xavfsizlik chorasi emas.

---

## 5. Well-known Symbols — umumiy ko'rinish

**Well-known symbols** — tilning o'zi tomonidan oldindan belgilangan, `Symbol` konstruktorining **statik property'lari** sifatida mavjud bo'lgan maxsus symbol'lar. Ular orqali dasturchi **o'z obyektlarining til ichki operatsiyalaridagi xatti-harakatini sozlay** oladi.

| Symbol | Nimani boshqaradi | Qayerda ishlatiladi |
|---|---|---|
| `Symbol.iterator` | Obyektni **iterable** qiladi | `for...of`, spread, destructuring, `Array.from` |
| `Symbol.asyncIterator` | Obyektni **async iterable** qiladi | `for await...of` |
| `Symbol.toPrimitive` | Obyektning primitive'ga aylanish usuli | `+obj`, `` `${obj}` ``, `obj + 1` |
| `Symbol.toStringTag` | `Object.prototype.toString` natijasi | `[object Nomi]` |
| `Symbol.hasInstance` | `instanceof` xatti-harakati | `x instanceof Klass` |
| `Symbol.species` | Hosila obyektlar yaratishda ishlatiladigan konstruktor | `Array.prototype.map` kabi metodlar |
| `Symbol.isConcatSpreadable` | `concat()` obyektni yoyishi yoki yo'qligi | `arr.concat(obj)` |
| `Symbol.match`, `replace`, `search`, `split` | Regex-o'xshash obyektlar | `String.prototype.match` va h.k. |
| `Symbol.unscopables` | `with` bayonoti uchun | Deyarli ishlatilmaydi |

Quyida eng muhimlarini — **iterator** juftligini — chuqur ko'ramiz.

---

## 6. `Symbol.iterator` va Iterator protokoli

JavaScript'da "sikl yurish" (iteration) ikkita **protokol** (kelishuv) asosida ishlaydi.

### 6.1. Iterator protokoli

**Iterator** — `next()` metodiga ega obyekt. Har safar `next()` chaqirilganda u quyidagi shakldagi obyekt qaytaradi:

```javascript
{ value: <qiymat>, done: false }  // yana element bor
{ value: undefined, done: true }  // tugadi
```

```javascript
// Qo'lda yozilgan iterator
const iterator = {
  current: 1,
  next() {
    if (this.current <= 3) {
      return { value: this.current++, done: false };
    }
    return { value: undefined, done: true };
  },
};

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

### 6.2. Iterable protokoli

**Iterable** — `[Symbol.iterator]()` metodiga ega obyekt, bu metod **iterator qaytaradi**.

```javascript
const iterable = {
  [Symbol.iterator]() {
    let current = 1;
    return {
      next() {
        return current <= 3
          ? { value: current++, done: false }
          : { value: undefined, done: true };
      },
    };
  },
};

for (const n of iterable) {
  console.log(n); // 1, 2, 3
}
```

### 6.3. Farqi — juda muhim

```
Iterable  →  [Symbol.iterator]() metodiga ega  (sikl yuriladigan "manba")
Iterator  →  next() metodiga ega               (joriy holatni saqlovchi "kursor")
```

```
iterable ──[Symbol.iterator]()──► iterator ──next()──► { value, done }
```

Iterable **qayta-qayta** yangi iterator yarata oladi (shuning uchun massivni ikki marta aylanib chiqish mumkin). Iterator esa o'z **holatini** saqlaydi va bir marta tugagach, qayta ishlatib bo'lmaydi.

### 6.4. `for...of` ichkarida nima qiladi?

```javascript
for (const x of iterable) { /* ... */ }

// Ichkarida taxminan shunday ishlaydi:
const it = iterable[Symbol.iterator]();
let result = it.next();
while (!result.done) {
  const x = result.value;
  /* ... sikl tanasi ... */
  result = it.next();
}
```

### 6.5. Qaysi o'rnatilgan turlar iterable?

| Iterable | Iterable EMAS |
|---|---|
| `Array`, `String`, `Map`, `Set` | Oddiy obyekt `{}` |
| `TypedArray`, `arguments` | `WeakMap`, `WeakSet` |
| `NodeList` (DOM) | `Promise` |
| Generator obyektlari | Raqamlar, `null`, `undefined` |

```javascript
const obj = { a: 1, b: 2 };
for (const x of obj) {}  // TypeError: obj is not iterable

// Yechim: Object.entries / keys / values iterable (massiv) qaytaradi
for (const [k, v] of Object.entries(obj)) {
  console.log(k, v);
}
```

### 6.6. Iterable protokoliga tayanadigan til xususiyatlari

```javascript
const set = new Set([1, 2, 3]);

[...set];                    // spread
const [first, ...rest] = set; // array destructuring
Array.from(set);             // Array.from
new Map([['a', 1], ['b', 2]]); // Map konstruktori
Promise.all(set);            // Promise.all/race/allSettled/any
for (const x of set) {}      // for...of
```

Bularning barchasi ichkarida `[Symbol.iterator]()` ni chaqiradi. Shuning uchun **bitta `Symbol.iterator` metodi qo'shsangiz, obyektingiz butun ekotizim bilan ishlaydigan bo'lib qoladi.**

---

## 7. Custom iterable obyektlar yaratish

### 7.1. Klassik misol — `Range`

```javascript
class Range {
  constructor(start, end, step = 1) {
    this.start = start;
    this.end = end;
    this.step = step;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const { end, step } = this;

    return {
      next() {
        if (current <= end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      },
    };
  }
}

const range = new Range(1, 10, 3);

console.log([...range]);                 // [1, 4, 7, 10]
console.log(Math.max(...range));         // 10
for (const n of range) console.log(n);   // 1, 4, 7, 10
for (const n of range) console.log(n);   // YANA ishlaydi — har safar yangi iterator!
```

### 7.2. Lazy (dangasa) hisoblash — cheksiz ketma-ketlik

Iterator'ning eng katta kuchi: qiymatlar **faqat so'ralganda** hisoblanadi, shuning uchun **cheksiz** ketma-ketlikni ifodalash mumkin.

```javascript
const fibonacci = {
  [Symbol.iterator]() {
    let [a, b] = [0, 1];
    return {
      next() {
        const value = a;
        [a, b] = [b, a + b];
        return { value, done: false }; // HECH QACHON tugamaydi
      },
    };
  },
};

for (const n of fibonacci) {
  if (n > 50) break;      // biz o'zimiz to'xtatamiz
  console.log(n);         // 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
}

// [...fibonacci];  // XATO! Cheksiz sikl, xotira to'ladi
```

### 7.3. `return()` metodi — erta chiqishda tozalash

Agar `for...of` `break`, `return` yoki xato bilan **erta to'xtatilsa**, JS iterator'ning **`return()`** metodini (agar mavjud bo'lsa) avtomatik chaqiradi. Bu — resurslarni (fayl, ulanish) tozalash uchun.

```javascript
const resourceIterable = {
  [Symbol.iterator]() {
    let i = 0;
    return {
      next() {
        return i < 5 ? { value: i++, done: false } : { value: undefined, done: true };
      },
      return() {
        console.log('Resurslar tozalandi!');
        return { done: true };
      },
    };
  },
};

for (const x of resourceIterable) {
  if (x === 2) break; // → "Resurslar tozalandi!" chiqadi
}
```

### 7.4. Iterator'ning o'zi ham iterable bo'lishi

```javascript
const iterator = [1, 2, 3][Symbol.iterator]();
console.log(iterator[Symbol.iterator]() === iterator); // true
```

Barcha o'rnatilgan iterator'lar **o'zini qaytaradigan** `[Symbol.iterator]()` ga ega — shuning uchun ularni to'g'ridan-to'g'ri `for...of` ga berish mumkin. **Lekin** bunday iterator'lar **bir martalik** (one-shot):

```javascript
const it = [1, 2, 3][Symbol.iterator]();
console.log([...it]); // [1, 2, 3]
console.log([...it]); // [] — bo'sh! Iterator allaqachon tugagan
```

---

## 8. Generator'lar va `Symbol.iterator`

Qo'lda `next()` va `done` yozish zerikarli. **Generator** (`function*`) bu ishni avtomatlashtiradi — generator funksiya **iterator** (va iterable) qaytaradi.

```javascript
class Range {
  constructor(start, end, step = 1) {
    this.start = start;
    this.end = end;
    this.step = step;
  }

  *[Symbol.iterator]() {           // generator metod
    for (let i = this.start; i <= this.end; i += this.step) {
      yield i;
    }
  }
}

console.log([...new Range(1, 10, 3)]); // [1, 4, 7, 10]
```

Bu 7.1-bo'limdagi uzun versiya bilan **funksional jihatdan bir xil**, lekin ancha qisqa va o'qilishi oson.

### `yield*` — boshqa iterable'ga delegatsiya

```javascript
class Playlist {
  constructor() {
    this.rock = ['Song A', 'Song B'];
    this.pop = ['Song C', 'Song D'];
  }

  *[Symbol.iterator]() {
    yield* this.rock;   // massivning barcha elementlarini birin-ketin beradi
    yield* this.pop;
  }
}

console.log([...new Playlist()]); // ['Song A', 'Song B', 'Song C', 'Song D']
```

### Generator yordamida lazy pipeline

```javascript
function* take(iterable, n) {
  let i = 0;
  for (const x of iterable) {
    if (i++ >= n) return;
    yield x;
  }
}

function* map(iterable, fn) {
  for (const x of iterable) yield fn(x);
}

// Cheksiz fibonacci'dan faqat birinchi 5 ta kvadrat — hammasi LAZY
console.log([...take(map(fibonacci, (x) => x * x), 5)]); // [0, 1, 1, 4, 9]
```

> **Eslatma:** Zamonaviy JS muhitlarida (ES2025 **Iterator Helpers**) `Iterator.prototype.map`, `.filter`, `.take` kabi tayyor metodlar mavjud. Ishlatishdan oldin maqsadli brauzer/Node versiyalarida qo'llab-quvvatlanishini tekshirish tavsiya etiladi.

> **Oldingi mavzu bilan bog'liqlik:** "Async/await ichki mexanizmi" hujjatida ko'rganimizdek, `async/await` aynan generator'lar ustiga qurilgan. Generator — bu `Symbol.iterator` protokolining "pauza qilish va davom ettirish" bilan boyitilgan versiyasi.

---

## 9. `Symbol.asyncIterator` va `for await...of`

Agar qiymatlar **asinxron** keladigan bo'lsa (tarmoq, fayl oqimi, sahifalab yuklash) — **async iterator protokoli** kerak bo'ladi (ES2018).

### 9.1. Farqi

| | Sync | Async |
|---|---|---|
| Kalit | `Symbol.iterator` | `Symbol.asyncIterator` |
| `next()` qaytaradi | `{ value, done }` | `Promise<{ value, done }>` |
| Sikl | `for...of` | `for await...of` |
| Generator | `function*` | `async function*` |

### 9.2. Qo'lda yozilgan async iterator

```javascript
const asyncIterable = {
  [Symbol.asyncIterator]() {
    let i = 0;
    return {
      async next() {
        await new Promise((r) => setTimeout(r, 500)); // asinxron kutish
        return i < 3
          ? { value: i++, done: false }
          : { value: undefined, done: true };
      },
    };
  },
};

(async () => {
  for await (const x of asyncIterable) {
    console.log(x); // 0 (500ms), 1 (1000ms), 2 (1500ms)
  }
})();
```

### 9.3. Async generator — qulayroq usul

```javascript
async function* fetchPages(baseUrl) {
  let page = 1;
  while (true) {
    const res = await fetch(`${baseUrl}?page=${page}`);
    const items = await res.json();
    if (items.length === 0) return;   // ma'lumot tugadi
    yield* items;                     // har bir elementni birin-ketin beramiz
    page++;
  }
}

(async () => {
  for await (const item of fetchPages('/api/products')) {
    console.log(item);
    // Bu yerda `break` qilsak — keyingi sahifalar UMUMAN so'ralmaydi (lazy!)
  }
})();
```

Bu — **pagination'ni** yashiruvchi juda toza pattern: iste'molchi kod "sahifa" haqida o'ylamaydi, shunchaki elementlarni oladi.

### 9.4. Real misol — Node.js'da fayl qatorlarini o'qish

```javascript
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

async function countLines(path) {
  const rl = createInterface({ input: createReadStream(path) });
  let count = 0;

  for await (const line of rl) {   // readline interface — async iterable
    count++;
  }
  return count;
}
```

Katta fayl **butunlay xotiraga yuklanmaydi** — qatorlar bittalab, kerak bo'lganda keladi. Node.js'dagi oqimlar (`Readable` stream'lar) ham async iterable.

### 9.5. Xato va `return()` holati

```javascript
async function* gen() {
  try {
    yield 1;
    yield 2;
  } finally {
    console.log('Tozalash'); // break yoki xato bo'lganda ham chaqiriladi
  }
}

for await (const x of gen()) {
  break; // → "Tozalash" chiqadi (async iterator'ning return() si chaqiriladi)
}
```

### 9.6. Muhim eslatma

```javascript
// Sync for...of async iterable bilan ISHLAMAYDI
for (const x of asyncIterable) {} // TypeError: asyncIterable is not iterable

// for await...of sync iterable bilan ISHLAYDI (har bir qiymatni await qiladi)
for await (const x of [Promise.resolve(1), Promise.resolve(2)]) {
  console.log(x); // 1, 2
}
```

`for await...of` faqat `async function` ichida (yoki ES modul'ning top-level'ida) ishlatilishi mumkin.

---

## 10. Boshqa muhim well-known symbol'lar

### 10.1. `Symbol.toPrimitive` — primitive'ga aylantirish

```javascript
const money = {
  amount: 100,
  currency: 'USD',

  [Symbol.toPrimitive](hint) {
    // hint: "number" | "string" | "default"
    if (hint === 'number') return this.amount;
    if (hint === 'string') return `${this.amount} ${this.currency}`;
    return this.amount; // "default"
  },
};

console.log(+money);          // 100        (hint = "number")
console.log(`${money}`);      // "100 USD"  (hint = "string")
console.log(money + 50);      // 150        (hint = "default")
console.log(money * 2);       // 200        (hint = "number")
```

### 10.2. `Symbol.toStringTag` — `Object.prototype.toString` natijasi

```javascript
class Database {
  get [Symbol.toStringTag]() {
    return 'Database';
  }
}

console.log(Object.prototype.toString.call(new Database())); // "[object Database]"
console.log(String(new Database()));                          // "[object Database]"
```

Shu tufayli `Promise`, `Map` kabi o'rnatilgan turlar `[object Promise]`, `[object Map]` ko'rinishida chiqadi.

### 10.3. `Symbol.hasInstance` — `instanceof` ni sozlash

```javascript
class EvenNumber {
  static [Symbol.hasInstance](value) {
    return Number.isInteger(value) && value % 2 === 0;
  }
}

console.log(2 instanceof EvenNumber);  // true
console.log(3 instanceof EvenNumber);  // false
```

### 10.4. `Symbol.isConcatSpreadable`

```javascript
const arrayLike = { length: 2, 0: 'a', 1: 'b', [Symbol.isConcatSpreadable]: true };
console.log([1, 2].concat(arrayLike)); // [1, 2, 'a', 'b']
```

### 10.5. `Symbol.species`

`Array` kabi turlar `map`, `filter` kabi metodlarda **yangi obyekt** yaratganda qaysi konstruktorni ishlatishini `Symbol.species` orqali belgilaydi. Bu asosan **`Array`/`Promise` dan meros oluvchi** kutubxona yozuvchilar uchun kerak; kundalik kodda kam uchraydi.

---

## 11. Loyihaning qaysi qismlarida ishlatiladi?

| Soha | Qanday ishlatiladi |
|---|---|
| **Custom ma'lumot tuzilmalari** | `LinkedList`, `Tree`, `Graph`, `Matrix` kabi o'zingiz yozgan struktura'larga `Symbol.iterator` qo'shib, `for...of`, spread bilan ishlashni ta'minlash |
| **Lazy hisoblash va cheksiz ketma-ketliklar** | Generator + iterator orqali faqat kerak bo'lgan qiymatlarni hisoblash |
| **Sahifalab yuklash (pagination)** | `async function*` + `for await...of` — API sahifalarini shaffof "oqim"ga aylantirish |
| **Fayl va tarmoq oqimlari (streams)** | Node.js `Readable` stream'lar, `readline` — `for await...of` bilan |
| **Kutubxona/freymvork ichki metadata'si** | Noyob symbol kalitlar bilan obyektga nom to'qnashuvisiz maxsus ma'lumot biriktirish (masalan, ichki holat, brendlash) |
| **React** | `$$typeof` maydonida `Symbol.for(...)` — element turini xavfsiz belgilash |
| **Til darajasidagi moslashtirish** | `Symbol.toPrimitive` (pul, vaqt, vektor tiplari), `Symbol.toStringTag` (debugging), `Symbol.hasInstance` (custom type-check) |
| **Polyfill va protokol qo'shish** | Mavjud turlarga yangi, to'qnashmaydigan xatti-harakat qo'shish |

### Amaliy misol — noyob metadata kaliti

```javascript
// Kutubxona ichida
const INTERNAL_STATE = Symbol('internalState');

function track(obj) {
  obj[INTERNAL_STATE] = { createdAt: Date.now() };
  return obj;
}

// Foydalanuvchining o'z property'lari bilan HECH QACHON to'qnashmaydi,
// Object.keys(), JSON.stringify() va for...in ga ham "ko'rinmaydi"
const data = track({ name: 'Ali' });
console.log(Object.keys(data));        // ['name']
console.log(JSON.stringify(data));     // '{"name":"Ali"}'
```

---

## 12. Nega bu muhim — real foydalar

1. **Til bilan "bir tilda gaplashish"** — bitta `[Symbol.iterator]` metodi orqali o'z obyektingiz `for...of`, spread, destructuring, `Array.from`, `Map`/`Set` konstruktori, `Promise.all` kabi o'nlab imkoniyat bilan ishlaydi.
2. **Xotira samaradorligi** — lazy iterator'lar katta yoki cheksiz ma'lumotlarni **butunlay xotiraga yuklamasdan** qayta ishlashga imkon beradi.
3. **Toza abstraksiya** — iste'molchi kod ma'lumot **qayerdan** va **qanday** kelayotganini (massiv, daraxt, tarmoq sahifalari) bilmaydi — faqat "sikl yuradi".
4. **Orqaga moslik** — tilga yangi imkoniyatlar mavjud kodni buzmasdan qo'shilgan (symbol'lar tufayli).
5. **To'qnashuvsiz kengaytirish** — kutubxonalar o'z metadata'sini foydalanuvchi kodi bilan to'qnashmasdan biriktira oladi.
6. **Zamonaviy asinxron dasturlash** — `for await...of` stream'lar, pagination va real-time ma'lumotlar bilan ishlashni sezilarli soddalashtiradi.

---

## 13. Eng ko'p uchraydigan xatolar

### Xato 1: Ikkita `Symbol('x')` ni teng deb o'ylash

```javascript
Symbol('x') === Symbol('x'); // false!
// Bir xil symbol kerak bo'lsa — Symbol.for('x') yoki o'zgaruvchida saqlab, qayta ishlatish
```

### Xato 2: Symbol'ni haqiqiy "private" deb hisoblash

```javascript
const secret = Symbol('secret');
const obj = { [secret]: 'parol' };
Object.getOwnPropertySymbols(obj); // [Symbol(secret)] — topib olish mumkin!
```
**Yechim:** Haqiqiy maxfiylik uchun `#privateField` yoki closure.

### Xato 3: Symbol kalitni `.` bilan o'qishga urinish

```javascript
const id = Symbol('id');
const user = { [id]: 1 };
user.id;   // undefined — bu string "id"
user[id];  // 1 — to'g'ri
```

### Xato 4: Symbol'ni string'ga implicit aylantirish

```javascript
const s = Symbol('test');
console.log('Qiymat: ' + s);  // TypeError
console.log('Qiymat: ' + String(s)); // to'g'ri
```

### Xato 5: Oddiy obyektni `for...of` bilan aylantirishga urinish

```javascript
for (const x of { a: 1 }) {}  // TypeError: not iterable
// Yechim: Object.entries(), yoki obyektga [Symbol.iterator] qo'shish
```

### Xato 6: Iterator va iterable'ni aralashtirish (bir martalik iterator)

```javascript
const it = new Set([1, 2, 3]).values(); // iterator (bir martalik!)
[...it]; // [1, 2, 3]
[...it]; // [] — qayta ishlatib bo'lmaydi
```
**Yechim:** Qayta yurish kerak bo'lsa, **iterable**'ning o'zini (masalan, `Set`) saqlang, har safar yangi iterator oling.

### Xato 7: Cheksiz iterable'ni spread qilish

```javascript
[...fibonacci];            // cheksiz sikl, xotira to'ladi
Array.from(fibonacci);     // xuddi shunday
```
**Yechim:** `take()` kabi chegaralovchi yordamchi yoki `break` bilan `for...of`.

### Xato 8: `for await...of` ni `async` kontekstdan tashqarida ishlatish

```javascript
for await (const x of source) {}  // SyntaxError (oddiy funksiya yoki script ichida)
```
**Yechim:** `async function` ichida yoki ES modul top-level'ida ishlatish.

### Xato 9: `JSON.stringify` symbol'larni saqlab qoladi deb kutish

```javascript
JSON.stringify({ [Symbol('a')]: 1, b: 2 }); // '{"b":2}'
```

---

## 14. Interview savollari va qisqa javoblar

**S: Symbol nima va nega kerak?**
> J: Symbol — ES2015'da kiritilgan, har bir qiymati noyob va o'zgarmas bo'lgan primitive tur. U ikki maqsadda yaratilgan: noyob, to'qnashmaydigan property kalitlari yaratish va tilga mavjud kodni buzmasdan yangi protokollarni (iterator, toPrimitive va h.k.) qo'shish.

**S: `Symbol('a') === Symbol('a')` nima qaytaradi va nega?**
> J: `false`. Tavsif (description) faqat debugging uchun, har bir `Symbol()` chaqiruvi yangi noyob qiymat yaratadi. Bir xil symbol kerak bo'lsa, `Symbol.for('a')` ishlatiladi — u global registry'dan foydalanadi.

**S: `Symbol()` va `Symbol.for()` farqi nima?**
> J: `Symbol()` har safar yangi, noyob symbol yaratadi. `Symbol.for(key)` esa global registry'da shu kalit bilan symbol borligini tekshiradi: bor bo'lsa uni qaytaradi, yo'q bo'lsa yaratib registry'ga qo'yadi. Shuning uchun `Symbol.for('x') === Symbol.for('x')` true bo'ladi (hatto turli fayl/iframe'larda ham).

**S: Symbol kalitlar `Object.keys()` va `JSON.stringify()` da ko'rinadimi?**
> J: Yo'q, ular e'tiborga olinmaydi. Ularni `Object.getOwnPropertySymbols()` yoki `Reflect.ownKeys()` orqali olish mumkin. Spread va `Object.assign` esa enumerable symbol kalitlarni nusxalaydi.

**S: Symbol haqiqiy private property'mi?**
> J: Yo'q. Symbol kalitlar "yashirin" (oddiy enumerasiyada ko'rinmaydi), lekin `getOwnPropertySymbols` orqali topish mumkin. Haqiqiy maxfiylik uchun `#private` maydonlar (ES2022) yoki closure ishlatiladi.

**S: Iterable va Iterator o'rtasidagi farq nima?**
> J: Iterable — `[Symbol.iterator]()` metodiga ega obyekt bo'lib, u iterator qaytaradi (masalan, massiv). Iterator — `next()` metodiga ega, har chaqiruvda `{ value, done }` qaytaradigan va joriy holatni saqlaydigan obyekt. Iterable qayta-qayta yangi iterator yarata oladi, iterator esa odatda bir martalik.

**S: `for...of` ichkarida qanday ishlaydi?**
> J: U obyektning `[Symbol.iterator]()` metodini chaqirib iterator oladi, so'ng `done: true` bo'lgunga qadar `next()` ni chaqirib, har safar `value` ni sikl o'zgaruvchisiga beradi. Agar sikl `break`/`return`/xato bilan erta to'xtasa, iterator'ning `return()` metodi (agar bor bo'lsa) chaqiriladi.

**S: Oddiy obyektni qanday iterable qilish mumkin?**
> J: Unga `[Symbol.iterator]` metodi qo'shiladi — qo'lda `next()` bilan yoki, ancha qulayi, generator metod (`*[Symbol.iterator]() { ... }`) orqali. Shundan keyin `for...of`, spread va destructuring bilan ishlaydi.

**S: `Symbol.iterator` va `Symbol.asyncIterator` farqi nima?**
> J: `Symbol.iterator` sinxron iterator uchun — `next()` to'g'ridan-to'g'ri `{ value, done }` qaytaradi va `for...of` bilan ishlatiladi. `Symbol.asyncIterator` esa asinxron uchun — `next()` `Promise<{ value, done }>` qaytaradi va `for await...of` bilan ishlatiladi. Async generator (`async function*`) async iterator'ni qulay yaratish usuli.

**S: Generator iterator bilan qanday bog'liq?**
> J: Generator funksiya chaqirilganda generator obyekti qaytaradi, u ham iterator, ham iterable. Shu sababli `*[Symbol.iterator]() { yield ... }` yozuvi qo'lda `next()` va `done` boshqarishdan ancha qisqa va xavfsizroq.

**S: `Symbol.toPrimitive` nima uchun kerak?**
> J: U obyekt primitive'ga (son yoki satrga) aylantirilganda xatti-harakatni boshqaradi. Metod `hint` argumentini (`"number"`, `"string"`, `"default"`) oladi va kontekstga qarab turli qiymat qaytara oladi (`+obj`, `` `${obj}` ``, `obj + 1`).

**S: Nega til yaratuvchilari `iterator` degan oddiy string metod o'rniga Symbol tanladi?**
> J: Orqaga moslik uchun. Oddiy string nom (`obj.iterator`) mavjud veb-saytlardagi foydalanuvchi kodi bilan to'qnashib, uni buzishi mumkin edi. Symbol esa noyobligi kafolatlangani uchun hech qanday mavjud kod bilan to'qnashmaydi.

---

## 15. Xulosa

> **Symbol** — noyob va o'zgarmas primitive tur bo'lib, ikki muhim vazifani bajaradi: **nom to'qnashuvisiz property kalitlari** yaratish va tilga **mavjud kodni buzmasdan** yangi protokollarni qo'shish.
>
> **Well-known symbol'lar** — tilning o'z protokollariga "ulanish nuqtalari". Ular orqali dasturchi o'z obyektlarining `for...of`, `instanceof`, primitive'ga aylanish, `toString` kabi til operatsiyalaridagi xatti-harakatini sozlay oladi.
>
> **`Symbol.iterator`** obyektni **iterable** qiladi: u iterator (`next()` → `{ value, done }`) qaytaradi, va shundan keyin obyekt `for...of`, spread, destructuring, `Array.from`, `Map`/`Set`, `Promise.all` bilan ishlaydi. **Generator'lar** bu protokolni qisqa va qulay yozish imkonini beradi, **lazy** hisoblash esa cheksiz va katta ma'lumotlar bilan xotirani tejab ishlashga yo'l ochadi.
>
> **`Symbol.asyncIterator`** xuddi shu g'oyani **asinxron** dunyoga olib o'tadi: `for await...of` va `async function*` yordamida stream'lar, pagination va real-time ma'lumotlar toza, ketma-ket kod ko'rinishida yoziladi.
>
> Bu mavzuni chuqur tushunish — interview'da `for...of`, spread, generator va async iteration "ichkarida qanday ishlashi" haqidagi savollarga ishonchli javob berish, hamda o'zingizning ma'lumot tuzilmalaringizni JavaScript ekotizimiga to'liq integratsiya qilish uchun muhim asos.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
