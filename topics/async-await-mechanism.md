# Async/Await Ichki Mexanizmi — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [`async/await` nima va u qayerdan paydo bo'lgan?](#1-asyncawait-nima-va-u-qayerdan-paydo-bolgan)
2. [Zaruriy asos: Generator funksiyalar](#2-zaruriy-asos-generator-funksiyalar)
3. [`async/await` = Generator + avtomatik "runner"](#3-asyncawait--generator--avtomatik-runner)
4. [`async function` har doim Promise qaytaradi](#4-async-function-har-doim-promise-qaytaradi)
5. [Xato qanday tarqaladi (error propagation)](#5-xato-qanday-tarqaladi-error-propagation)
6. [`Promise.then/catch` bilan solishtirish](#6-promisethencatch-bilan-solishtirish)
7. [Parallel va ketma-ket bajarilish](#7-parallel-va-ketma-ket-bajarilish)
8. [Loyihaning qaysi qismlarida ishlatiladi?](#8-loyihaning-qaysi-qismlarida-ishlatiladi)
9. [Qanday to'g'ri ishlatiladi — amaliy qoidalar](#9-qanday-togri-ishlatiladi--amaliy-qoidalar)
10. [Nega bu muhim — real foydalar](#10-nega-bu-muhim--real-foydalar)
11. [Eng ko'p uchraydigan xatolar](#11-eng-kop-uchraydigan-xatolar)
12. [Interview savollari va qisqa javoblar](#12-interview-savollari-va-qisqa-javoblar)
13. [Xulosa](#13-xulosa)

---

## 1. `async/await` nima va u qayerdan paydo bo'lgan?

`async/await` — JavaScript'da asinxron kodni **sinxron kod kabi ko'rinishda** yozish imkonini beruvchi sintaksis. U **ES2017 (ES8)** standartida rasman kiritilgan.

```javascript
async function getUser() {
  const response = await fetch('/api/user');
  const data = await response.json();
  return data;
}
```

**Qayerdan paydo bo'lgan?** `async/await` — bu **yangi, mustaqil mexanizm emas**. U allaqachon mavjud bo'lgan ikkita texnologiya ustiga qurilgan:

1. **Promise'lar** (ES2015/ES6) — asinxron natijani ifodalash uchun obyekt
2. **Generator funksiyalar** (`function*`, ES2015/ES6) — funksiya bajarilishini "pauza qilish va davom ettirish" qobiliyati

`async/await` paydo bo'lishidan oldin, dasturchilar xuddi shunday "sinxrondek ko'rinuvchi" kodni **generator + maxsus "runner" kutubxonalar** (masalan, `co` kutubxonasi) yordamida qo'lda yozishgan. `async/await` — bu shu g'oyani **tilning o'zi darajasida, rasmiy ravishda standartlashtirilgan** varianti.

> **Qisqa formula:** `async/await` = Generator + Promise + til darajasida avtomatlashtirilgan "runner" (ijrochi mexanizm)

---

## 2. Zaruriy asos: Generator funksiyalar

`async/await`ning "sahna orqasida" qanday ishlashini tushunish uchun, avval generator funksiyalarni tushunish kerak.

```javascript
function* genFunc() {
  console.log('1-qism boshlandi');
  const x = yield 'birinchi pauza';
  console.log('2-qism, x =', x);
  const y = yield 'ikkinchi pauza';
  console.log('3-qism, y =', y);
  return 'tugadi';
}

const gen = genFunc();

console.log(gen.next());
// "1-qism boshlandi" konsolga chiqadi
// { value: 'birinchi pauza', done: false }

console.log(gen.next('A'));
// "2-qism, x = A" konsolga chiqadi
// { value: 'ikkinchi pauza', done: false }

console.log(gen.next('B'));
// "3-qism, y = B" konsolga chiqadi
// { value: 'tugadi', done: true }
```

### Bu yerda nima sodir bo'lmoqda?

- `yield` — funksiya bajarilishini **shu joyda to'xtatadi** va boshqaruvni (hamda `yield`dan keyingi qiymatni) tashqariga, chaqiruvchiga qaytaradi
- `gen.next(qiymat)` chaqirilganda, funksiya **aynan to'xtagan joyidan**, `yield`ning "natijasi" sifatida berilgan `qiymat`ni qabul qilib, davom etadi

Bu — **pauza qilish va davom ettirish** qobiliyati, va aynan shu qobiliyat `async/await`ning yuragi hisoblanadi.

---

## 3. `async/await` = Generator + avtomatik "runner"

Endi tasavvur qiling: generator'ni **qo'lda emas**, balki **avtomatik tarzda** — har safar `yield` qilingan Promise hal bo'lishini kutib, keyin o'zi `.next()`ni chaqiradigan qilib ishlatsak-chi? Aynan shu — `async/await`ning ichki mexanizmi!

### `async/await` kodi:

```javascript
async function fetchUser() {
  console.log('Boshlandi');
  const response = await fetch('/api/user');
  console.log('Response keldi');
  const data = await response.json();
  console.log('Data keldi', data);
  return data;
}
```

### Uning "generator + runner" ekvivalenti:

```javascript
function* fetchUserGenerator() {
  console.log('Boshlandi');
  const response = yield fetch('/api/user');
  console.log('Response keldi');
  const data = yield response.json();
  console.log('Data keldi', data);
  return data;
}

// Generator'ni AVTOMATIK "yuritadigan" (drive qiluvchi) funksiya:
function asyncRunner(generatorFunc) {
  return new Promise((resolve, reject) => {
    const gen = generatorFunc();

    function step(action) {
      let result;
      try {
        result = action(); // gen.next() yoki gen.throw() chaqiriladi
      } catch (err) {
        return reject(err); // generator xato tashlasa - Promise reject bo'ladi
      }

      const { value, done } = result;

      if (done) {
        return resolve(value); // generator tugadi - Promise resolve bo'ladi
      }

      // value - bu yield qilingan narsa (Promise bo'lishi mumkin)
      Promise.resolve(value).then(
        (val) => step(() => gen.next(val)),   // muvaffaqiyat - keyingi qadamga o'tish
        (err) => step(() => gen.throw(err))   // xato - generator ICHIGA "tashlanadi"
      );
    }

    step(() => gen.next());
  });
}

// Ishlatish:
asyncRunner(fetchUserGenerator()).then((data) => console.log('Yakuniy natija', data));
```

### Qadam-baqadam nima sodir bo'lmoqda?

1. `step()` funksiyasi generator'ni bir qadam oldinga suradi (`gen.next()` chaqiradi)
2. Generator `yield promise` qaytarganda, `step()` shu Promise'ning hal bo'lishini **kutadi**
3. Promise **resolve** bo'lsa → natija generator ichiga `gen.next(val)` orqali "qaytariladi" — bu xuddi `await`dan keyingi kodning davom etishiga teng
4. Promise **reject** bo'lsa → xato generator ichiga `gen.throw(err)` orqali "tashlanadi" — bu esa `try/catch` orqali ushlanishi mumkin bo'lgan xatoni yaratadi

> **Asosiy formula:** `await promise` ≈ `yield promise` + "runner Promise natijasini avtomatik kutib, natijani (yoki xatoni) generator ichiga qaytaradi".

Bu — nazariy model, JS dvigateli (V8 kabi) buni aynan shunday, `yield` orqali emas, balki ichki, past darajadagi mexanizmlar (masalan, "continuation" — davom ettirish nuqtalari) orqali amalga oshiradi, lekin **konseptual va xulq-atvor jihatidan** natija bir xil.

---

## 4. `async function` har doim Promise qaytaradi

Bu — juda muhim, ko'p interview'da so'raladigan qoida.

```javascript
async function foo() {
  return 42;
}

console.log(foo()); // Promise { 42 } — oddiy qiymat qaytarilsa ham, avtomatik Promise'ga o'raladi

foo().then(val => console.log(val)); // 42
```

Bu — yuqoridagi `asyncRunner`dagi `resolve(value)` qismiga mos keladi: generator `done: true` bilan tugaganda (`return` bajarilganda), uning qiymati orqali runner **tashqi Promise'ni resolve qiladi**.

```javascript
async function bar() {
  return Promise.resolve('ichma-ich Promise');
}

bar().then(val => console.log(val)); // "ichma-ich Promise" — ichki Promise avtomatik "flatten" qilinadi
```

Agar `async` funksiya ichida boshqa Promise `return` qilinsa, tashqi Promise avval ichkisining hal bo'lishini kutadi, keyin uning natijasi bilan resolve bo'ladi (Promise'lar avtomatik "tekislanadi" — flatten qilinadi, ichma-ich Promise hosil bo'lmaydi).

---

## 5. Xato qanday tarqaladi (error propagation)

### 5.1. `try/catch` bilan — xato ICHKARIDA ushlanadi

```javascript
async function fetchData() {
  try {
    const response = await fetch('/api/data');
    const data = await response.json();
    return data;
  } catch (err) {
    console.log('Xato ushlandi:', err.message);
    return null; // xato "yutildi", funksiya normal (resolved) Promise qaytaradi
  }
}

fetchData().then(result => console.log('Natija:', result));
// Agar xato bo'lsa: "Xato ushlandi: ..." keyin "Natija: null"
```

Bu — yuqoridagi generator modelida ko'rganimizdek, `await`langan Promise **reject** bo'lganda, runner xatoni `gen.throw(err)` orqali generator ichiga "tashlaydi". Agar shu joy `try/catch` bilan o'ralgan bo'lsa, xato **oddiy sinxron `throw` kabi** `catch` blokida ushlanadi.

### 5.2. `try/catch` bo'lmasa — funksiyaning qaytargan Promise'i **reject** bo'ladi

```javascript
async function fetchData() {
  const response = await fetch('/api/data'); // agar bu yerda xato bo'lsa...
  const data = await response.json();
  return data;
}

fetchData()
  .then(data => console.log('Natija:', data))
  .catch(err => console.log('Tashqarida ushlandi:', err.message)); // shu yerda ushlanadi
```

Agar `async function` ichida `try/catch` bo'lmasa va biror `await` reject bo'lsa (yoki oddiy `throw` sodir bo'lsa), bu xato **funksiyaning o'zi qaytargan Promise'ni reject qiladi**. Bu — sinxron funksiyada `throw` qilingan xato uni chaqirgan joyga "otilishi" bilan konseptual jihatdan bir xil, faqat natija **rad etilgan (rejected) Promise** ko'rinishida bo'ladi.

```javascript
async function foo() {
  throw new Error('Xatolik!'); // await shart emas, oddiy throw ham xuddi shunday ishlaydi
}

foo().catch(err => console.log(err.message)); // "Xatolik!"
```

### 5.3. Agar xatoni umuman hech kim ushlamasa

```javascript
async function foo() {
  throw new Error('Ushlanmagan xato');
}

foo(); // na try/catch, na .catch() yo'q
```

Bu holatda konsolda **"Uncaught (in promise) Error"** ogohlantirishi chiqadi va `unhandledrejection` global hodisasi (Node.js'da — `process.on('unhandledRejection', ...)`) ishga tushadi. Dastur "qulab tushmaydi", lekin bu — jiddiy xato belgisi, chunki xato "yo'qolib ketgan" hisoblanadi.

```javascript
window.addEventListener('unhandledrejection', (event) => {
  console.log('Ushlanmagan Promise rad etildi:', event.reason);
});
```

### 5.4. Zanjirlangan `await`larda xato qanday "yuqoriga ko'tariladi"

```javascript
async function level3() {
  throw new Error('level3da xato');
}

async function level2() {
  await level3(); // xato bu yerga "ko'tariladi"
  console.log('Bu qator hech qachon bajarilmaydi');
}

async function level1() {
  try {
    await level2(); // xato bu yerga ham "ko'tariladi"
  } catch (err) {
    console.log('level1da ushlandi:', err.message); // "level1da ushlandi: level3da xato"
  }
}

level1();
```

`level3()` xato tashlaydi → uning Promise'i reject bo'ladi → `level2()` ichidagi `await level3()` shu reject'ni "qabul qiladi" va, `level2`da `try/catch` bo'lmagani sababli, **`level2()`ning ham Promise'ini reject qiladi** → `level1()` ichidagi `await level2()` buni ushlaydi, va u yerda `try/catch` **bor**ligi sababli, xato aynan shu yerda to'xtaydi.

> **Muhim tushuncha:** Bu — sinxron kodda funksiyalar ichida `throw` qilingan xato, uni ushlaydigan `try/catch` topilguncha, chaqiruv zanjiri (call stack) bo'ylab "yuqoriga ko'tarilishi" bilan bir xil mantiq — faqat bu yerda "ko'tarilish" `await` zanjiri orqali, asinxron tarzda sodir bo'ladi.

---

## 6. `Promise.then/catch` bilan solishtirish

```javascript
// async/await bilan
async function getData() {
  try {
    const res = await fetch('/api');
    return await res.json();
  } catch (err) {
    console.log('Xato:', err);
  }
}

// Xuddi shu narsa .then/.catch bilan (mohiyatan bir xil)
function getDataPromise() {
  return fetch('/api')
    .then(res => res.json())
    .catch(err => console.log('Xato:', err));
}
```

Bu ikkalasi **funksional jihatdan bir xil natija** beradi, chunki `async/await` "sahna orqasida" aynan `.then()`/`.catch()` zanjiriga aylantiriladi. `try/catch` — bu `.catch()`ning o'qish uchun qulayroq, sinxron kodga o'xshash ko'rinishi, xolos.

---

## 7. Parallel va ketma-ket bajarilish

`await`ni to'g'ri joyda ishlatmaslik — eng ko'p uchraydigan **performance xatosi**.

### Xato: keraksiz ketma-ketlik (sequential)

```javascript
async function loadData() {
  const user = await fetchUser();       // 1 soniya kutadi
  const posts = await fetchPosts();     // yana 1 soniya kutadi (garchi user'ga bog'liq bo'lmasa ham!)
  return { user, posts };
}
// Jami: ~2 soniya
```

### To'g'ri: parallel bajarish

```javascript
async function loadData() {
  const [user, posts] = await Promise.all([
    fetchUser(),
    fetchPosts(),
  ]);
  return { user, posts };
}
// Jami: ~1 soniya (ikkalasi bir vaqtda boshlanadi)
```

> **Qoida:** Agar asinxron amallar bir-biriga **bog'liq bo'lmasa**, ularni `await` bilan ketma-ket emas, `Promise.all()` bilan **parallel** boshlash kerak.

---

## 8. Loyihaning qaysi qismlarida ishlatiladi?

| Soha | Misol |
|---|---|
| **API so'rovlari** | `fetch`, `axios` orqali serverdan ma'lumot olish |
| **Ma'lumotlar bazasi bilan ishlash** | Node.js backend'da `await db.query(...)` |
| **Fayl operatsiyalari** | Node.js'da `fs.promises.readFile()`, `writeFile()` |
| **Autentifikatsiya oqimlari** | Login, token yangilash, sessiya tekshirish kabi ketma-ket asinxron qadamlar |
| **Testlash (testing)** | Jest, Mocha kabi kutubxonalarda asinxron test funksiyalari (`async () => {...}`) |
| **Build vositalari va CLI skriptlar** | Fayllarni o'qish/yozish, tashqi buyruqlarni bajarish kabi ketma-ket amallar |
| **React/Vue kabi freymvorklarda** | `useEffect` ichida ma'lumot yuklash, forma yuborish (`onSubmit`) funksiyalari |

---

## 9. Qanday to'g'ri ishlatiladi — amaliy qoidalar

### 9.1. Har doim xatoni boshqaring

```javascript
async function safeFetch(url) {
  try {
    const res = await fetch(url);
    if (!res.ok) throw new Error(`HTTP xato: ${res.status}`);
    return await res.json();
  } catch (err) {
    console.error('So\'rov muvaffaqiyatsiz:', err.message);
    throw err; // yoki fallback qiymat qaytarish
  }
}
```

### 9.2. Massiv ustida `await` — `forEach` emas, `for...of` yoki `Promise.all`

```javascript
// XATO — forEach ichida await ishlamaydi (kutilgan tartibda)
async function processAll(items) {
  items.forEach(async (item) => {
    await processItem(item); // forEach bu Promise'larni "kutmaydi"!
  });
  console.log('Hammasi tugadi'); // bu HAR DOIM birinchi chiqadi, kutilmagan holda
}

// TO'G'RI — ketma-ket kerak bo'lsa
async function processAll(items) {
  for (const item of items) {
    await processItem(item); // har biri navbat bilan kutiladi
  }
  console.log('Hammasi tugadi'); // to'g'ri joyda chiqadi
}

// TO'G'RI — parallel kerak bo'lsa
async function processAll(items) {
  await Promise.all(items.map(item => processItem(item)));
  console.log('Hammasi tugadi');
}
```

### 9.3. Top-level `await` (zamonaviy ES modullarida)

```javascript
// modul faylining eng yuqori darajasida (top-level), async function shart emas
const data = await fetch('/api/config').then(r => r.json());
console.log(data);
```
Bu — ES2022'da kiritilgan imkoniyat, faqat **ES module** fayllarida ishlaydi (`<script type="module">` yoki `.mjs`).

---

## 10. Nega bu muhim — real foydalar

1. **O'qilishi osonroq kod** — "Promise pyramid of doom" (chuqur ichma-ich `.then()` zanjirlari) o'rniga tekis, sinxrondek kod
2. **Xatolarni boshqarish qulayligi** — bitta `try/catch` bilan bir nechta asinxron qadamdagi xatolarni ushlash mumkin, har bir `.then()`ga alohida `.catch()` yozish shart emas
3. **Debugging qulayligi** — stack trace'lar sinxron kodga yaqinroq va tushunarliroq bo'ladi
4. **Shartli mantiqni yozish osonligi** — `if/else`, sikllar (`for`, `while`) bilan asinxron kodni tabiiy tarzda birlashtirish mumkin

```javascript
// async/await bilan - shartli mantiq tabiiy
async function getUserData(id) {
  const user = await fetchUser(id);
  if (user.isActive) {
    const details = await fetchDetails(user.id);
    return { ...user, details };
  }
  return user;
}

// Faqat Promise bilan - xuddi shu narsa ancha chalkash bo'lardi
function getUserData(id) {
  return fetchUser(id).then(user => {
    if (user.isActive) {
      return fetchDetails(user.id).then(details => ({ ...user, details }));
    }
    return user;
  });
}
```

---

## 11. Eng ko'p uchraydigan xatolar

### Xato 1: `async` funksiyani `await`siz chaqirish va natijani kutmaslik

```javascript
async function save() {
  await db.save(data);
}

function handleSubmit() {
  save(); // "fire and forget" — xato bo'lsa umuman bilinmaydi!
  console.log('Saqlandi'); // bu save() tugashini kutmasdan chiqadi
}
```

### Xato 2: `Promise.all` ichida bitta xato hammasini "buzadi"

```javascript
const results = await Promise.all([fetchA(), fetchB(), fetchC()]);
// Agar fetchB() reject bo'lsa, BUTUN Promise.all darhol reject bo'ladi,
// fetchA() va fetchC() natijalari muvaffaqiyatli bo'lsa ham yo'qoladi!
```
**Yechim:** Agar har bir natija muhim bo'lsa (ba'zilari xato bo'lsa ham), `Promise.allSettled()` ishlatish kerak.

```javascript
const results = await Promise.allSettled([fetchA(), fetchB(), fetchC()]);
// har biri {status: 'fulfilled', value} yoki {status: 'rejected', reason} qaytaradi
```

### Xato 3: Keraksiz ketma-ket `await` (parallel qilish mumkin bo'lgan joyda)

Yuqorida 7-bo'limda ko'rsatilgan — mustaqil so'rovlarni ketma-ket `await` qilish performance'ni yomonlashtiradi.

---

## 12. Interview savollari va qisqa javoblar

**S: `async/await` qayerdan kelib chiqqan, u qanday ishlaydi?**
> J: `async/await` — ES2017'da kiritilgan sintaktik shakar bo'lib, u generator funksiyalar va Promise'lar ustiga qurilgan. Har bir `await promise` — bu, konseptual jihatdan, `yield promise`ga teng: "runner" mexanizmi Promise natijasini kutib, uni generator ichiga qaytarib beradi (yoki xato bo'lsa, tashlaydi).

**S: `async function` nima qaytaradi?**
> J: Har doim Promise. Agar funksiya oddiy qiymat `return` qilsa, u avtomatik `Promise.resolve(qiymat)`ga o'raladi. Agar boshqa Promise `return` qilinsa, u "tekislanadi" (flatten qilinadi).

**S: `await`langan Promise reject bo'lsa nima bo'ladi?**
> J: Agar `try/catch` bo'lsa — xato o'sha yerda ushlanadi. Bo'lmasa — bu xato funksiyaning o'zi qaytargan Promise'ni reject qiladi, va bu holat chaqiruvchi tomonda `.catch()` yoki tashqi `try/catch` orqali ushlanishi kerak.

**S: Xato hech qayerda ushlanmasa nima bo'ladi?**
> J: `unhandledrejection` (brauzerda) yoki `unhandledRejection` (Node.js) global hodisasi ishga tushadi. Dastur qulab tushmaydi, lekin bu jiddiy xato holati hisoblanadi va logging/monitoring orqali kuzatilishi kerak.

**S: `forEach` ichida `await` ishlaydimi?**
> J: Yo'q, kutilganidek ishlamaydi — `forEach` async callback'larni "kutmaydi", ular parallel ishga tushadi va `forEach`ning o'zi ularni kutmasdan darhol tugaydi. Ketma-ket bajarish uchun `for...of`, parallel uchun `Promise.all` bilan `map` ishlatish kerak.

**S: `Promise.all` va `Promise.allSettled` farqi nima?**
> J: `Promise.all` — agar ichidagi Promise'lardan **bittasi ham** reject bo'lsa, butun natija darhol reject bo'ladi. `Promise.allSettled` esa har bir Promise natijasini (muvaffaqiyatli yoki xato bo'lishidan qat'iy nazar) kutadi va har biri uchun status/natija qaytaradi.

---

## 13. Xulosa

> **`async/await` — Promise'lar va generator funksiyalar ustiga qurilgan sintaktik shakar bo'lib**, asinxron kodni sinxron kod kabi o'qish va yozish imkonini beradi. Uning ichki mexanizmi: `await` — generator'dagi `yield`ga o'xshaydi, va til darajasidagi "runner" har bir Promise'ning hal bo'lishini avtomatik kutib, natijani (yoki xatoni) funksiya ichiga qaytaradi.
>
> **Xato tarqalishi sinxron `throw` mantig'iga o'xshaydi**: `try/catch` bo'lsa — ichkarida ushlanadi; bo'lmasa — funksiyaning qaytargan Promise'i reject bo'ladi va yuqoriga, chaqiruv zanjiri bo'ylab "ko'tariladi", toki uni ushlaydigan joy topilguncha yoki `unhandledrejection` sifatida "yo'qolib ketguncha".
>
> Bu mavzuni chuqur tushunish — nafaqat interview'da, balki real loyihalarda performance muammolarini (keraksiz ketma-ketlik), xato boshqaruvidagi bo'shliqlarni va race condition'larni to'g'ri aniqlash hamda tuzatishda ham asosiy poydevor bo'lib xizmat qiladi.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
