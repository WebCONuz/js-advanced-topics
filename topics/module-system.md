# Modul Tizimi — CommonJS vs ES Modules — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [Modul tizimi nima va u qayerdan paydo bo'lgan?](#1-modul-tizimi-nima-va-u-qayerdan-paydo-bolgan)
2. [CommonJS (CJS) — asoslari](#2-commonjs-cjs--asoslari)
3. [ES Modules (ESM) — asoslari](#3-es-modules-esm--asoslari)
4. [`require` vs `import` — runtime xatti-harakati](#4-require-vs-import--runtime-xatti-harakati)
5. [Tree-shaking — nega faqat ESM'da samarali ishlaydi](#5-tree-shaking--nega-faqat-esmda-samarali-ishlaydi)
6. [Circular Dependency — ikkalasida qanday hal qilinadi](#6-circular-dependency--ikkalasida-qanday-hal-qilinadi)
7. [Qo'shimcha muhim farqlar](#7-qoshimcha-muhim-farqlar)
8. [Loyihaning qaysi qismlarida ishlatiladi?](#8-loyihaning-qaysi-qismlarida-ishlatiladi)
9. [Qanday ishlatiladi — amaliy sozlash](#9-qanday-ishlatiladi--amaliy-sozlash)
10. [Nega bu muhim — real foydalar](#10-nega-bu-muhim--real-foydalar)
11. [Eng ko'p uchraydigan xatolar](#11-eng-kop-uchraydigan-xatolar)
12. [Interview savollari va qisqa javoblar](#12-interview-savollari-va-qisqa-javoblar)
13. [Xulosa](#13-xulosa)

---

## 1. Modul tizimi nima va u qayerdan paydo bo'lgan?

**Modul tizimi** — dasturni mustaqil, qayta ishlatiladigan fayllarga (modullarga) bo'lish va ular orasida kod (funksiya, o'zgaruvchi, class) almashish mexanizmi.

### Nega bu kerak bo'lgan — tarixiy kontekst

JavaScript dastlab (1995-yillarda) faqat **brauzer skriptlari** uchun mo'ljallangan edi, va unda **hech qanday rasmiy modul tizimi yo'q edi**. Barcha `<script>` teglar **bitta umumiy global scope**da ishlar edi:

```html
<script src="a.js"></script>
<script src="b.js"></script>
<!-- a.js va b.js bir xil global o'zgaruvchilar maydonini "bo'lishadi" - nom to'qnashuvi xavfi katta! -->
```

Bu — kod kattalashgan sari **nom to'qnashuvlari**, **bog'liqliklarni boshqarish qiyinligi** kabi jiddiy muammolarni keltirib chiqargan.

### Yechim sifatida paydo bo'lgan ikkita asosiy standart:

1. **CommonJS (CJS)** — 2009-yilda, **Node.js** yaratilishi bilan birga qabul qilingan norasmiy standart. Node.js server tomonida ishlaydigan JS uchun mo'ljallangan, va u yerda fayllarni **sinxron** o'qish tabiiy edi (disk operatsiyalari tez).

2. **ES Modules (ESM)** — **ECMAScript 2015 (ES6)** standartida rasman kiritilgan, JavaScript tilining **o'zига** tegishli modul tizimi. U brauzer va server muhitlarining ikkalasida ham ishlashi uchun, va tarmoq orqali **asinxron** yuklashga mos qilib loyihalangan.

```javascript
// CommonJS (Node.js'ning dastlabki va hozirgача standart usuli)
const fs = require('fs');
module.exports = { myFunc };

// ES Modules (zamonaviy, rasmiy JS standarti)
import fs from 'fs';
export { myFunc };
```

---

## 2. CommonJS (CJS) — asoslari

```javascript
// math.js
function add(a, b) {
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

module.exports = { add, subtract };
```

```javascript
// main.js
const { add, subtract } = require('./math.js');
console.log(add(2, 3)); // 5
```

**Asosiy xususiyatlar:**
- `require()` — modulni yuklovchi **oddiy funksiya**
- `module.exports` (yoki qisqacha `exports`) — modul nimani "tashqariga chiqarishi"ni belgilaydigan obyekt
- Har bir modul **birinchi marta** `require()` qilinganda bajariladi va natija **keshlanadi** (keyingi `require()` chaqiruvlari saqlangan natijani qaytaradi)

---

## 3. ES Modules (ESM) — asoslari

```javascript
// math.mjs
export function add(a, b) {
  return a + b;
}

export function subtract(a, b) {
  return a - b;
}

export default function multiply(a, b) {
  return a * b;
}
```

```javascript
// main.mjs
import multiply, { add, subtract } from './math.mjs';
console.log(add(2, 3)); // 5
```

**Asosiy xususiyatlar:**
- `import`/`export` — tilning **rasmiy, deklarativ** sintaksisi (funksiya chaqiruvi emas)
- `export default` — modulning "asosiy" eksporti (har modulda faqat bitta bo'lishi mumkin)
- Named export'lar (`export const`, `export function`) — bir nechta bo'lishi mumkin
- Fayl kengaytmasi odatda `.mjs`, yoki `package.json`da `"type": "module"` ko'rsatilgan bo'lsa `.js`

---

## 4. `require` vs `import` — runtime xatti-harakati

### CommonJS — **sinxron**, runtime'da ishlaydi

`require()` — bu oddiy **funksiya chaqiruvi**, shuning uchun uni dastur bajarilishining istalgan nuqtasida, hatto **shartli holatda** chaqirish mumkin:

```javascript
if (process.env.NODE_ENV === 'development') {
  const devTools = require('./dev-tools'); // CJS'da bunday shartli require mutlaqo normal
}

function loadModule(name) {
  return require(`./${name}`); // dinamik nom bilan ham chaqirish mumkin
}
```

`require()` chaqirilganda, Node.js modul faylini **darhol, sinxron tarzda** o'qiydi, bajaradi va natijasini qaytaradi — bu dastur oqimini modul to'liq yuklanguncha **bloklaydi**.

### ES Modules — **statik tahlil qilinadi**, hoisted bo'ladi

`import` — funksiya chaqiruvi emas, balki **deklarativ sintaksis**. Uni shartli holatda yoki funksiya ichida ishlatib bo'lmaydi:

```javascript
if (condition) {
  import fs from 'fs'; // SINTAKTIK XATO! import faqat modul darajasida (top-level) bo'lishi mumkin
}
```

**Hoisting xususiyati:** Barcha `import` bayonotlari, faylning qayerida yozilganidan qat'iy nazar, kodning eng yuqorisiga "ko'tariladi" va bajarilishdan **oldin** JS dvigateli tomonidan qayta ishlanadi:

```javascript
console.log(myFunc); // ISHLAYDI! import "pastda" yozilgan bo'lsa ham
import { myFunc } from './utils.js';
```

Bu — ES Modules **statik tahlil qilinganligi** uchun: dvigatel avval butun modul grafini (qaysi fayl qaysi faylni import qilishini) tahlil qiladi, keyingina haqiqiy bajarishga o'tadi.

### Dinamik import — ESM'da ham mavjud

```javascript
if (condition) {
  const module = await import('./module.js'); // dynamic import() - Promise qaytaradi
}
```

Bu — alohida, `Promise` qaytaruvchi funksiya bo'lib, u ESM'ning **statik** `import` bayonotidan farqli, **asinxron va shartli** yuklashga ruxsat beradi.

---

## 5. Tree-shaking — nega faqat ESM'da samarali ishlaydi

**Tree-shaking** — bu bundler (Webpack, Rollup, esbuild, Vite) tomonidan, **haqiqatda ishlatilmagan kod**ni yakuniy bundle fayldan olib tashlash jarayoni. Bu — bundle hajmini kamaytirish va ilova yuklanish tezligini oshirish uchun juda muhim optimizatsiya.

### ESM — statik struktura tree-shaking'ni osonlashtiradi

```javascript
// utils.js
export function usedFunction() { console.log('ishlatiladi'); }
export function unusedFunction() { console.log('ishlatilmaydi'); }

// main.js
import { usedFunction } from './utils.js';
usedFunction();
```

ESM'da `import`/`export` **statik** bo'lgani sababli (ular kod bajarilishidan oldin, "compile-time"da to'liq aniq), bundler build vaqtida, **hech qanday kodni bajarmasdan**, aniq bilishi mumkin: `unusedFunction` hech qayerda import qilinmagan, demak uni bundle'dan olib tashlash mumkin.

```
main.js → utils.js
           ├── usedFunction    ✅ bundle'ga kiradi
           └── unusedFunction  ❌ olib tashlanadi (tree-shaken)
```

### CommonJS — dinamik tabiat tree-shaking'ni qiyinlashtiradi

```javascript
// Bu CJS'da mutlaqo to'g'ri kod, lekin statik tahlil qilib bo'lmaydi:
module.exports = condition ? { a: 1 } : { b: 2 };

for (const key in someObject) {
  exports[key] = someObject[key]; // exports runtime'da DINAMIK to'ldirilmoqda
}
```

`require()` va `module.exports` — bular oddiy funksiya chaqiruvi va oddiy obyekt bo'lib, ular **dinamik** bo'lishi mumkin. Bundler bu holatlarda modulni haqiqatda bajarmasdan, qaysi export'lar kerakli ekanligini **ishonchli aniqlay olmaydi**. Shu sababli CommonJS modullarida tree-shaking **juda cheklangan yoki umuman ishlamaydi**.

> **Interview uchun eng muhim jumla:** Tree-shaking ESM'da samarali ishlashining sababi — `import`/`export`ning **statik** (dinamik bo'lmagan, shartsiz, doim modul darajasida) bo'lishi. Bu bundler'ga kodni bajarmasdan, faqat matnni tahlil qilib, butun bog'liqlik grafini (dependency graph) qurish imkonini beradi.

---

## 6. Circular Dependency — ikkalasida qanday hal qilinadi

**Circular dependency** — `A` moduli `B`ni, `B` esa `A`ni import qilishi kabi "aylanma" bog'liqlik holati.

### CommonJS'da circular dependency

```javascript
// a.js
console.log('a.js boshlandi');
exports.done = false;
const b = require('./b.js');
console.log('a.js ichida b.done =', b.done);
exports.done = true;

// b.js
console.log('b.js boshlandi');
exports.done = false;
const a = require('./a.js'); // a.js hali TO'LIQ tugallanmagan!
console.log('b.js ichida a.done =', a.done);
exports.done = true;

// main.js
require('./a.js');
```

**Natija:**
```
a.js boshlandi
b.js boshlandi
b.js ichida a.done = false   ← a.js hali tugamagan, uning exports'i TO'LIQ emas!
a.js ichida b.done = true
```

**Nima uchun bunday?** CommonJS'da modul birinchi marta `require()` qilinganda, u **keshga** qo'yiladi va bajarilish boshlanadi. Agar bajarilish jarayonida (hali tugamasdan) shu modul **yana** `require()` qilinsa, Node.js **hozirgi, hali to'liq bo'lmagan `module.exports` holatini** qaytaradi.

```
Jarayon:
1. main.js → require('a.js') → a.js bajarila boshlaydi
2. a.js → require('b.js') → b.js bajarila boshlaydi
3. b.js → require('a.js') → LEKIN a.js HALI TUGAMAGAN!
   Node.js keshdan a.js'ning HOZIRGI (to'liq bo'lmagan) exports'ini qaytaradi
4. b.js davom etadi, tugaydi
5. a.js davom etadi, tugaydi
```

### ES Modules'da circular dependency — "live bindings" orqali yaxshiroq hal qilinadi

```javascript
// a.mjs
export let done = false;
import { done as bDone } from './b.mjs';
console.log('a.mjs ichida bDone =', bDone);
done = true;

// b.mjs
export let done = false;
import { done as aDone } from './a.mjs';
console.log('b.mjs ichida aDone =', aDone);
done = true;
```

ESM'da import qilingan qiymatlar — bu statik nusxalar emas, balki **"live bindings"** (jonli havolalar). Import qilingan o'zgaruvchi, asl modul o'sha qiymatni keyinroq yangilasa, import qiluvchi tomonda **avtomatik yangilanadi** — chunki bu aslida bitta xotira joyiga ishora, nusxa emas.

```javascript
// counter.mjs
export let count = 0;
export function increment() { count++; }

// main.mjs
import { count, increment } from './counter.mjs';
console.log(count); // 0
increment();
console.log(count); // 1 - AVTOMATIK yangilandi! (CJS'da bu bo'lmasdi)
```

Bu xususiyat circular dependency'larni yanada yaxshiroq hal qilishga yordam beradi: hatto modul hali to'liq bajarilmagan bo'lsa ham, ESM import qilingan bog'lanishni "reserve" qiladi, va modul keyinroq qiymatni yangilasa, bu o'zgarish avtomatik ravishda ko'rinadi.

> **Muhim farq:** CJS `require()` — qiymatning **statik nusxasini** qaytaradi (aniqrog'i, `require()` chaqirilgan paytdagi holatni "suratga oladi"). ESM `import` esa **live binding** — asl o'zgaruvchiga to'g'ridan-to'g'ri, doimiy ishora.

---

## 7. Qo'shimcha muhim farqlar

### 7.1. `this` qiymati modul darajasida

```javascript
// CommonJS
console.log(this); // module.exports (bo'sh obyekt)

// ESM
console.log(this); // undefined
```

### 7.2. `__dirname`, `__filename`

```javascript
// CommonJS - mavjud
console.log(__dirname, __filename);

// ESM - mavjud EMAS, o'rniga import.meta.url ishlatiladi
import { fileURLToPath } from 'url';
import path from 'path';
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
```

### 7.3. Named va default export'larni aralashtirish

```javascript
// ESM - ikkalasi ham mavjud, aniq farqlangan
export default function main() {}
export const helper = () => {};

import main, { helper } from './module.js';
```

```javascript
// CJS - hammasi bitta module.exports obyektiga tushadi
module.exports = function main() {};
module.exports.helper = () => {};
```

### 7.4. Yuklash tartibi va asinxronlik

CommonJS — har doim **sinxron**. ESM esa (ayniqsa brauzerda, tarmoq orqali import qilinganda) **asinxron** yuklanishi mumkin — bu ESM'ga parallel yuklash va HTTP/2 orqali optimallashtirish kabi imkoniyatlar beradi.

### Qisqa taqqoslash jadvali

| Xususiyat | CommonJS | ES Modules |
|---|---|---|
| Yuklash | Sinxron, runtime'da | Statik tahlil, asinxron yuklanishi mumkin |
| Sintaksis joylashuvi | Istalgan joyda (shartli, funksiya ichida) | Faqat modul darajasida (top-level) |
| Tree-shaking | Amalda ishlamaydi | Yaxshi ishlaydi |
| Circular dependency | "Yarim tayyor" exports qaytarilishi mumkin | Live bindings orqali yanada barqaror |
| Qiymat bog'lanishi | Nusxa (statik "surat") | Live binding (doimiy ishora) |
| Dinamik import | `require()` — har doim dinamik bo'lishi mumkin | `import()` — alohida, Promise qaytaruvchi funksiya |
| `this` | `module.exports` | `undefined` |

---

## 8. Loyihaning qaysi qismlarida ishlatiladi?

| Soha | Odatda qaysi tizim | Sabab |
|---|---|---|
| **Node.js backend (eski/o'rta loyihalar)** | CommonJS | Node.js'ning tarixiy default'i, ko'p eski kutubxonalar shu bilan yozilgan |
| **Zamonaviy Node.js loyihalar** | ES Modules | `package.json`da `"type": "module"` orqali, zamonaviy standart |
| **Frontend (React, Vue, va h.k.)** | ES Modules | Webpack/Vite kabi bundler'lar ESM'ni afzal ko'radi (tree-shaking uchun) |
| **NPM kutubxonalar** | Ikkalasi ham (dual package) | Ko'p kutubxonalar CJS va ESM formatlarini **ikkalasini ham** taqdim etadi (moslik uchun) |
| **Browser'da to'g'ridan-to'g'ri skript** | ES Modules | `<script type="module">` orqali, `require` brauzerda umuman mavjud emas |
| **Build vositalari, CLI skriptlar** | Odatda CommonJS yoki ESM (loyihaga bog'liq) | Node.js muhitiga qarab tanlanadi |

---

## 9. Qanday ishlatiladi — amaliy sozlash

### 9.1. Node.js'da ESM'ni yoqish

```json
// package.json
{
  "type": "module"
}
```
Bundan keyin barcha `.js` fayllar ESM sifatida talqin qilinadi. CommonJS kerak bo'lgan fayllar uchun `.cjs` kengaytmasi ishlatiladi.

### 9.2. Ikkala tizimni ham qo'llab-quvvatlash (dual package)

```json
// package.json
{
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "exports": {
    "require": "./dist/index.cjs",
    "import": "./dist/index.mjs"
  }
}
```
Bu — kutubxona yaratuvchilar uchun keng tarqalgan amaliyot: bir xil kod ikkala modul tizimi foydalanuvchilari uchun ham ishlashini ta'minlaydi.

### 9.3. CommonJS'dan ESM'ga import qilish

```javascript
// ESM ichida CJS modulni import qilish odatda ishlaydi (default import sifatida)
import express from 'express'; // express CJS bilan yozilgan, lekin bu ishlaydi
```

### 9.4. ESM'dan CommonJS'ga import qilish — cheklov

```javascript
// CJS ichida ESM modulni oddiy require() bilan olib bo'lmaydi!
const esmModule = require('./esm-module.mjs'); // XATO!

// Buning o'rniga dinamik import() ishlatiladi (Promise qaytaradi)
const esmModule = await import('./esm-module.mjs');
```

---

## 10. Nega bu muhim — real foydalar

1. **Bundle hajmini kamaytirish** — tree-shaking orqali faqat kerakli kod yakuniy fayl(lar)ga qo'shiladi, bu ilovaning yuklanish tezligini sezilarli oshiradi
2. **Kodni tashkil qilish va qayta ishlatish** — modullar orqali kod mantiqiy qismlarga bo'linadi, nom to'qnashuvlari oldini olinadi
3. **Circular dependency'larni to'g'ri boshqarish** — ESM'ning live binding xususiyati, murakkab loyihalarda "yarim tayyor" modul holatlari sabab bo'ladigan xatolarni kamaytiradi
4. **Zamonaviy tooling bilan moslik** — Webpack, Vite, Rollup kabi zamonaviy bundler'lar ESM'ga optimallashtirilgan, shuning uchun ESM ishlatish build jarayonini samaraliroq qiladi
5. **Standart va kelajakka yo'naltirilganlik** — ESM — rasmiy JS standarti, brauzer va Node.js'ning ikkalasida ham **bir xil sintaksis** bilan ishlaydi

---

## 11. Eng ko'p uchraydigan xatolar

### Xato 1: CJS va ESM sintaksisini aralashtirib yuborish

```javascript
// XATO - bitta faylda ikkala sintaksisni aralashtirish
import fs from 'fs';
module.exports = { fs }; // SyntaxError yoki noaniq xatti-harakat
```
**Yechim:** Bitta fayl faqat bitta modul tizimiga tegishli bo'lishi kerak (fayl kengaytmasi yoki `package.json` orqali aniqlanadi).

### Xato 2: ESM'da `__dirname`ni to'g'ridan-to'g'ri ishlatishga urinish

```javascript
// ESM ichida - XATO, __dirname mavjud emas
console.log(__dirname); // ReferenceError
```
**Yechim:** `import.meta.url` orqali qayta hosil qilish (7.2-bo'limga qarang).

### Xato 3: CJS'da ESM-only kutubxonani `require()` bilan chaqirish

Ko'p zamonaviy kutubxonalar (masalan, `chalk@5+`, `node-fetch@3+`) faqat ESM formatida chiqarilgan. Ularni eski CJS loyihada `require()` bilan chaqirish xato beradi.
**Yechim:** Dinamik `import()` ishlatish yoki loyihani ESM'ga o'tkazish.

### Xato 4: Tree-shaking ishlamayotganini tushunmaslik

```javascript
// Agar shu kabi "barrel export" fayli ishlatilsa:
export * from './module1';
export * from './module2';
// ba'zi bundler sozlamalarida bu tree-shaking'ni qiyinlashtirishi mumkin
```
**Yechim:** Kerakli narsalarni aniq (`named`) import qilish, "side effect"ga ega modullarni `package.json`da `sideEffects: false` bilan belgilash.

---

## 12. Interview savollari va qisqa javoblar

**S: CommonJS va ES Modules orasidagi eng asosiy farq nima?**
> J: CommonJS `require()`/`module.exports` orqali, **runtime'da, sinxron** tarzda ishlaydi va dinamik xususiyatga ega. ES Modules `import`/`export` orqali, **compile-time'da statik tahlil qilinadigan**, deklarativ sintaksisga ega — bu esa tree-shaking va live binding kabi imkoniyatlarni beradi.

**S: Nega tree-shaking faqat ESM'da yaxshi ishlaydi?**
> J: ESM'da `import`/`export` statik bo'lgani uchun (shartsiz, faqat modul darajasida), bundler kodni bajarmasdan, faqat tahlil qilib, qaysi export'lar ishlatilishini aniq bila oladi. CommonJS'da `require()`/`module.exports` dinamik (shartli, runtime'da o'zgaruvchan) bo'lishi mumkin, shuning uchun bundler bunday ishonchli tahlil qila olmaydi.

**S: Circular dependency CommonJS'da qanday muammoga olib kelishi mumkin?**
> J: Agar `A` moduli `B`ni, `B` esa hali tugallanmagan `A`ni `require()` qilsa, `B` `A`ning **to'liq bo'lmagan (hali barcha exportlari tayyor bo'lmagan)** holatini oladi — bu `undefined` qiymatlar yoki kutilmagan xatolarga olib kelishi mumkin.

**S: ESM circular dependency'ni qanday yaxshiroq hal qiladi?**
> J: ESM import qilingan qiymatlarni **"live binding"** sifatida ishlatadi — bu statik nusxa emas, balki asl o'zgaruvchiga to'g'ridan-to'g'ri ishora. Shu sababli, hatto modul hali to'liq bajarilmagan bo'lsa ham, keyinroq yangilangan qiymat import qiluvchi tomonda avtomatik ko'rinadi.

**S: `import`ni nima uchun funksiya ichida yoki shart ichida ishlatib bo'lmaydi?**
> J: Chunki `import` — statik, deklarativ sintaksis bo'lib, u dvigatel tomonidan kod bajarilishidan oldin, modul darajasida tahlil qilinishi kerak. Buning o'rniga, dinamik yuklash kerak bo'lsa, alohida `import()` funksiyasi (Promise qaytaruvchi) ishlatiladi.

---

## 13. Xulosa

> **CommonJS va ES Modules — JavaScript'dagi ikkita asosiy modul tizimi**, ular kodni fayllarga bo'lish va ular orasida bog'liqlikni boshqarish uchun ishlatiladi. CommonJS — Node.js'ning tarixiy, **runtime'da, sinxron** ishlaydigan yechimi; ES Modules — JavaScript tilining rasmiy, **statik tahlil qilinadigan** standarti.
>
> Bu ikkisi orasidagi **statiklik farqi** — ko'plab amaliy oqibatlarga olib keladi: **tree-shaking** ESM'da yaxshi ishlaydi, chunki bog'liqlik grafini kodni bajarmasdan qurish mumkin; **circular dependency**lar ESM'da **live binding** tufayli yanada barqaror hal qilinadi, CJS'da esa modulning "yarim tayyor" holati bilan bog'liq muammolar yuzaga kelishi mumkin.
>
> Zamonaviy JavaScript ekotizimi asta-sekin **ES Modules**ga o'tmoqda — bu brauzer va server muhitlarida bir xil standartni ta'minlaydi, hamda zamonaviy build vositalari (Webpack, Vite, Rollup) bilan yaxshiroq integratsiyalashadi. Ammo CommonJS hali ham ko'plab Node.js loyihalari va kutubxonalarda keng qo'llanilmoqda, shu sababli ikkalasini ham chuqur tushunish zamonaviy JS dasturchisi uchun muhim ko'nikma hisoblanadi.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
