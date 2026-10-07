# Performance: Reflow, Repaint va Layout Thrashing — To'liq Qo'llanma

> JavaScript/Frontend texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [Bu mavzu qayerdan paydo bo'lgan — brauzer rendering pipeline'i](#1-bu-mavzu-qayerdan-paydo-bolgan--brauzer-rendering-pipelinei)
2. [Reflow (Layout) va Repaint (Paint) — aniq farqi](#2-reflow-layout-va-repaint-paint--aniq-farqi)
3. [Layout Thrashing nima?](#3-layout-thrashing-nima)
4. [Layout Thrashing'ning oldini olish](#4-layout-thrashingning-oldini-olish)
5. [`requestAnimationFrame` (rAF)](#5-requestanimationframe-raf)
6. [`requestIdleCallback` (rIC)](#6-requestidlecallback-ric)
7. [rAF vs rIC — asosiy farqi](#7-raf-vs-ric--asosiy-farqi)
8. [Loyihaning qaysi qismlarida ishlatiladi?](#8-loyihaning-qaysi-qismlarida-ishlatiladi)
9. [Qanday aniqlash mumkin — DevTools bilan ishlash](#9-qanday-aniqlash-mumkin--devtools-bilan-ishlash)
10. [Nega bu muhim — real foydalar](#10-nega-bu-muhim--real-foydalar)
11. [Eng ko'p uchraydigan xatolar](#11-eng-kop-uchraydigan-xatolar)
12. [Interview savollari va qisqa javoblar](#12-interview-savollari-va-qisqa-javoblar)
13. [Xulosa](#13-xulosa)

---

## 1. Bu mavzu qayerdan paydo bo'lgan — brauzer rendering pipeline'i

Bu mavzuni tushunish uchun, avval brauzer sahifani ekranga **qanday chiqarishini** bilish kerak. Brauzer HTML/CSS/JS'ni oddiy matndan **piksellarga** aylantirish uchun bir necha bosqichdan iborat jarayondan (**rendering pipeline**) o'tadi:

```
JavaScript → Style → Layout (Reflow) → Paint (Repaint) → Composite
```

| Bosqich | Nima qiladi |
|---|---|
| **JavaScript** | DOM/CSSOM'ni o'zgartiradi (masalan, `element.style.width = '200px'`) |
| **Style** | Qaysi CSS qoidalari qaysi elementga tegishli ekanligini hisoblaydi |
| **Layout (Reflow)** | Har bir elementning **aniq o'lchami va pozitsiyasini** (geometriyasini) hisoblaydi |
| **Paint (Repaint)** | Elementlarning **vizual ko'rinishini** (rang, soya, matn) piksel darajasida chizadi |
| **Composite** | Turli "qatlamlar"ni (layer) birlashtirib, yakuniy tasvirni ekranga chiqaradi |

Bu pipeline **1990-yillarning oxiri — 2000-yillarning boshida**, brauzerlar murakkab veb-sahifalarni samarali render qilish uchun ishlab chiqqan arxitekturadan kelib chiqadi, va u hozirgi barcha zamonaviy brauzerlarda (Chrome, Firefox, Safari) — turli ichki optimizatsiyalar bilan — asosiy prinsip sifatida saqlanib qolgan.

**Muhim tushuncha:** Har bir bosqich — **oldingi bosqichga bog'liq**. Agar siz elementning **o'lchamini** o'zgartirsangiz, brauzer **Layout**dan boshlab, **butun zanjir bo'ylab qayta ishlashga** majbur bo'ladi. Agar faqat **rangini** o'zgartirsangiz, Layout kerak emas — jarayon **Paint**dan boshlanadi (arzonroq). Agar faqat `transform`/`opacity` o'zgarsa, hatto Paint ham kerak emas — faqat **Composite** ishlaydi (eng arzon).

```
width, height, top, left, margin  →  Layout → Paint → Composite   (ENG QIMMAT)
color, background, box-shadow      →           Paint → Composite   (O'RTACHA)
transform, opacity                 →                    Composite  (ENG ARZON)
```

---

## 2. Reflow (Layout) va Repaint (Paint) — aniq farqi

### Reflow (Layout)

Brauzer sahifadagi elementlarning **geometriyasini** (kengligi, balandligi, pozitsiyasi) qayta hisoblaydigan jarayon. Bu — **eng "qimmat"** operatsiya, chunki bitta elementning o'lchami o'zgarsa, bu boshqa **ko'plab** elementlarning joylashuviga ham ta'sir qilishi mumkin (kaskadli ta'sir).

```javascript
element.style.width = '200px';      // Reflow
element.style.padding = '10px';     // Reflow
element.style.fontSize = '20px';    // Reflow (matn hajmi o'zgaradi)
element.classList.add('new-size');  // agar klassda o'lcham o'zgarishi bo'lsa - Reflow
```

### Repaint (Paint)

Elementlarning **vizual xususiyatlarini** (geometriyaga ta'sir qilmaydigan) qayta chizish. Bu Layout'dan **arzonroq**, chunki elementlar joylashuvi o'zgarmaydi — faqat ularning "ko'rinishi" yangilanadi.

```javascript
element.style.backgroundColor = 'red';  // faqat Repaint
element.style.color = 'blue';           // faqat Repaint
element.style.visibility = 'hidden';    // faqat Repaint (display: none'dan farqli)
```

### Composite

Faqat `transform` va `opacity` kabi maxsus xususiyatlar bilan ishlaydi, ular **alohida GPU qatlamlarida** qayta ishlanadi va Layout HAM, Paint HAM talab qilmaydi.

```javascript
element.style.transform = 'translateX(100px)'; // faqat Composite - eng tez
element.style.opacity = '0.5';                   // faqat Composite - eng tez
```

---

## 3. Layout Thrashing nima?

**Layout Thrashing** (yoki "forced synchronous layout") — JavaScript kodi **DOM'ni o'qish va yozishni ketma-ket, aralashtirib** bajarganda yuz beradigan **jiddiy performance muammosi**. Bu brauzerni bir necha marta, **majburiy ravishda, sinxron tarzda** Layout'ni qayta hisoblashga majbur qiladi — holbuki bu hisoblashlar odatda **bitta marta**, frame oxiriga qoldirilishi mumkin edi.

### Nega bu yuz beradi?

Brauzer, samaradorlik uchun, DOM'ga **yozish** amallarini (`style.width = ...` kabi) darhol bajarmaydi — ularni "navbatga qo'yadi" va **frame oxirida, bitta marta** Layout'ni qayta hisoblaydi. Bu jarayon **"batching"** (guruhlash) deb ataladi.

**Lekin** — agar siz DOM'ga yozgandan **darhol keyin** biror **"o'qish" xususiyatiga** (`offsetHeight`, `offsetWidth`, `getBoundingClientRect()`, `scrollTop` va h.k.) murojaat qilsangiz, brauzer bu qiymatni **aniq va yangilangan holda** qaytarishi shart. Shuning uchun u navbatdagi barcha "yozish"larni **majburan, darhol** bajaradi ("flush" qiladi) — bu esa **"forced synchronous layout"** deb ataladi.

### Klassik misol

```javascript
const boxes = document.querySelectorAll('.box');

boxes.forEach((box) => {
  const height = box.offsetHeight;              // O'QISH - Layout talab qiladi
  box.style.height = (height * 2) + 'px';        // YOZISH - Layout'ni "iflos" qiladi
});
```

**Bu kodda har bir iteratsiyada:**
1. `box.offsetHeight` — o'qish. Brauzer navbatdagi barcha "yozish"larni (agar bo'lsa) darhol bajaradi va Layout'ni qayta hisoblaydi
2. `box.style.height = ...` — yozish. Layout "iflos" (dirty) holatga o'tadi

Keyingi iteratsiyada yana `offsetHeight` o'qilganda, brauzer **yana** majburan Layout'ni qayta hisoblaydi — chunki oldingi qadamda u "iflos" bo'lgan edi.

**Natija:** 1000 ta element bo'lsa — bu **1000 marta majburiy, sinxron Layout hisoblash** degani, garchi to'g'ri yozilganda bu **bitta marta** bajarilishi mumkin bo'lsa ham!

```
Iteratsiya 1: O'QISH → YOZISH (Layout "iflos" bo'ldi)
Iteratsiya 2: O'QISH (Layout MAJBURAN qayta hisoblanadi!) → YOZISH
Iteratsiya 3: O'QISH (Yana MAJBURAN!) → YOZISH
... va hokazo, N marta
```

Nomi **"thrashing"** (chayqalish) bo'lishining sababi — kod doimiy ravishda "o'qish ↔ yozish" orasida chayqalib, brauzerni keraksiz, takrorlanuvchi hisob-kitoblarga majburlaydi.

### Layout Thrashing'ga sabab bo'luvchi xususiyatlar (o'qishda "trigger" bo'luvchi)

```
offsetTop, offsetLeft, offsetWidth, offsetHeight, offsetParent
scrollTop, scrollLeft, scrollWidth, scrollHeight
clientTop, clientLeft, clientWidth, clientHeight
getComputedStyle()
getBoundingClientRect()
```

Bu xususiyatlarning har biri, agar Layout "iflos" holatda bo'lsa, uni **majburan** qayta hisoblashga sabab bo'ladi.

---

## 4. Layout Thrashing'ning oldini olish

### Yechim 1: O'qish va yozishni **ajratish** (batching)

```javascript
const boxes = document.querySelectorAll('.box');

// AVVAL - barcha O'QISH amallarini bajaramiz
const heights = Array.from(boxes).map((box) => box.offsetHeight);

// KEYIN - barcha YOZISH amallarini bajaramiz
boxes.forEach((box, i) => {
  box.style.height = (heights[i] * 2) + 'px';
});
```

Endi brauzer **faqat bitta marta** Layout'ni hisoblaydi (barcha o'qishlar tugagandan keyin), va **bitta marta** barcha yozishlarni "batch" qilib bajaradi.

### Yechim 2: `DocumentFragment` — bir nechta elementni birgalikda qo'shish

```javascript
const fragment = document.createDocumentFragment();

for (let i = 0; i < 1000; i++) {
  const div = document.createElement('div');
  div.textContent = `Element ${i}`;
  fragment.appendChild(div); // hali DOM'ga emas, xotiradagi fragment'ga
}

document.body.appendChild(fragment); // BITTA marta, haqiqiy DOM'ga qo'shiladi
```

### Yechim 3: O'qish natijalarini keshlash

```javascript
// XATO - har safar qayta o'qiladi
elements.forEach((el) => {
  console.log(el.getBoundingClientRect().top); // har safar Layout trigger qilishi mumkin
  el.style.top = '10px';
});

// TO'G'RI - avval hammasi o'qiladi, keshlanadi
const rects = elements.map((el) => el.getBoundingClientRect());
elements.forEach((el, i) => {
  el.style.top = (rects[i].top + 10) + 'px';
});
```

### Yechim 4: CSS klass orqali o'zgartirish

```javascript
// Har bir style'ni alohida o'zgartirish o'rniga
el.style.width = '100px';
el.style.height = '200px';
el.style.margin = '10px';

// Bitta CSS klass qo'shish - brauzer buni ko'proq optimallashtirishi mumkin
el.classList.add('updated-box');
```

### Yechim 5: `transform` va `opacity` ishlatish

```javascript
// SEKIN - Layout'ni ishga tushiradi
el.style.left = '100px';

// TEZ - faqat Composite bosqichida ishlaydi
el.style.transform = 'translateX(100px)';
```

### Yechim 6: `display: none` bilan elementni vaqtincha "chiqarib qo'yish"

Agar bir elementga **ko'plab** o'zgarishlar ketma-ket qilinishi kerak bo'lsa, uni vaqtincha `display: none` qilib (bu Layout'dan chiqaradi), barcha o'zgarishlarni bajarib, keyin qaytadan ko'rsatish mumkin:

```javascript
el.style.display = 'none'; // Layout'dan olib tashlanadi
// ... ko'plab o'zgarishlar, DOM'dan o'qishlarsiz ...
el.style.width = '100px';
el.style.height = '200px';
el.style.display = 'block'; // qaytadan Layout'ga qo'shiladi, BITTA marta hisoblanadi
```

---

## 5. `requestAnimationFrame` (rAF)

`requestAnimationFrame` — brauzerga "keyingi **repaint**dan oldin shu funksiyani bajar" deb aytadigan API. U brauzerning **rendering sikliga sinxronlashtirilgan** holda ishlaydi (odatda soniyasiga ~60 marta, ya'ni har ~16.6ms'da bir marta, ekran yangilanish chastotasiga qarab).

```javascript
let x = 0;

function animate() {
  element.style.transform = `translateX(${x}px)`;
  x += 1;

  if (x < 300) {
    requestAnimationFrame(animate); // keyingi frame'da yana chaqiriladi
  }
}

requestAnimationFrame(animate);
```

### Nega `setTimeout`/`setInterval` o'rniga `requestAnimationFrame`?

```javascript
// YOMON - brauzer render sikli bilan sinxronlashmagan
setInterval(() => {
  element.style.left = x + 'px';
}, 16); // taxminiy, aniq emas, va tab fonga o'tsa ham ishlashda davom etadi!

// YAXSHI - brauzer render sikliga to'liq sinxronlashtirilgan
function loop() {
  element.style.left = x + 'px';
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);
```

**Afzalliklari:**
- Brauzerning haqiqiy render sikliga **aniq sinxronlashtirilgan** — silliq, "jank"siz (uzilishsiz) animatsiya
- Agar brauzer tab'i **fonda** bo'lsa, `requestAnimationFrame` **avtomatik to'xtaydi** — bu batareya va protsessor resurslarini tejaydi
- Bir nechta `requestAnimationFrame` chaqiruvlari **bitta frame ichida guruhlanadi** — bu keraksiz qo'shimcha reflow'larning oldini oladi

---

## 6. `requestIdleCallback` (rIC)

`requestIdleCallback` — brauzerga "foydalanuvchi uchun **muhim** ishlar (rendering, input'ga javob berish) tugagandan keyin, agar **bo'sh vaqt** qolsa, shu funksiyani bajar" deb aytadigan API.

```javascript
const tasks = [/* juda ko'p, ustuvor bo'lmagan vazifalar */];

function processTasks(deadline) {
  while (deadline.timeRemaining() > 0 && tasks.length > 0) {
    const task = tasks.shift();
    processTask(task);
  }

  if (tasks.length > 0) {
    requestIdleCallback(processTasks); // qolganlarni keyingi bo'sh vaqtga qoldirish
  }
}

requestIdleCallback(processTasks);
```

`deadline.timeRemaining()` — hozirgi "bo'sh vaqt" oynasida qancha millisekund vaqt qolganini bildiradi (odatda ~50ms yoki undan kam).

### `timeout` opsiyasi bilan kafolatlash

```javascript
requestIdleCallback(processTasks, { timeout: 2000 });
// Agar brauzer 2000ms ichida hech qachon "bo'sh" bo'lmasa,
// funksiya baribir majburan chaqiriladi
```

---

## 7. rAF vs rIC — asosiy farqi

| | `requestAnimationFrame` | `requestIdleCallback` |
|---|---|---|
| **Ishga tushish vaqti** | Har bir frame'dan **oldin** (repaint'dan oldin), ~16.6ms'da bir marta | Brauzer **bo'sh** bo'lganda, frame'lar orasidagi "bo'sh vaqt"da |
| **Kafolat** | Har frame'da chaqirilishi deyarli kafolatlangan | Chaqirilishi **kafolatlanmagan** (`timeout` bilan qisman hal qilinadi) |
| **Maqsad** | Vizual yangilanishlar, animatsiyalar — **darhol ko'rinishi kerak** | Fon ishlari — **kechiktirilishi mumkin**, past ustuvorlik |
| **Brauzer qo'llab-quvvatlashi** | Keng qo'llab-quvvatlanadi | Ba'zi brauzerlarda cheklangan qo'llab-quvvatlash bo'lishi mumkin, tekshirish tavsiya etiladi |

### Vizual taqqoslash

```
Frame 1                    Frame 2                    Frame 3
|--rAF--|---bo'sh vaqt---| |--rAF--|---bo'sh vaqt---| |--rAF--|
         ▲                          ▲
    rIC shu yerda ishlashi     rIC shu yerda ishlashi
    mumkin (agar vaqt bo'lsa)  mumkin (agar vaqt bo'lsa)
```

> **Interview uchun eng muhim jumla:** `requestAnimationFrame` — **vizual, darhol ko'rinishi kerak bo'lgan** o'zgarishlar uchun, brauzer render siklining bir qismi sifatida ishlaydi. `requestIdleCallback` — **foydalanuvchi tajribasiga ta'sir qilmaydigan**, orqa fon ishlari uchun, brauzer bo'sh vaqtida ishlaydi va chaqirilishi kafolatlanmagan.

---

## 8. Loyihaning qaysi qismlarida ishlatiladi?

| Soha | Qaysi texnika | Sabab |
|---|---|---|
| **CSS animatsiyalar (JS orqali)** | `requestAnimationFrame` | Silliq, brauzer render sikliga sinxronlashtirilgan harakat |
| **Scroll-bog'liq effektlar (parallax)** | `requestAnimationFrame` | Scroll pozitsiyasini o'qib, vizual elementlarni yangilash |
| **Katta ro'yxatlarni render qilish (virtualizatsiya)** | Batching + `DocumentFragment` | Minglab elementni bir vaqtda DOM'ga qo'shishdan qochish |
| **Drag & Drop** | `requestAnimationFrame` | Sichqoncha harakati bilan bir vaqtda elementni silliq surish |
| **Analytics/logging yuborish** | `requestIdleCallback` | Foydalanuvchi tajribasiga ta'sir qilmasdan, fon vazifasi sifatida |
| **Katta ma'lumotlarni oldindan qayta ishlash (pre-processing)** | `requestIdleCallback` | Ustuvor bo'lmagan hisob-kitoblarni keyinga qoldirish |
| **React/Vue kabi freymvorklar** | Ichki DOM batching mexanizmlari | Virtual DOM orqali ko'plab o'zgarishlarni bitta real DOM yangilanishiga birlashtirish |

---

## 9. Qanday aniqlash mumkin — DevTools bilan ishlash

Chrome DevTools'ning **Performance** panelida "record" qilib, sahifa bilan ishlaganda, agar quyidagilarni ko'rsangiz — bu Layout Thrashing belgisi:

1. **"Recalculate Style"** va **"Layout"** yozuvlari **juda ko'p va tez-tez** takrorlanadi
2. Konsolda **"Forced reflow"** yoki **"Forced synchronous layout"** ogohlantirishi chiqadi
3. **"Purple" (binafsha rang)** bloklar — Layout operatsiyalarini bildiradi, va ular ko'p bo'lsa, bu muammo belgisi

```
Performance panelida "Bottom-Up" yoki "Call Tree" ko'rinishida
"Layout" operatsiyasining necha marta chaqirilganini kuzatish orqali
Layout Thrashing bor-yo'qligini aniqlash mumkin.
```

---

## 10. Nega bu muhim — real foydalar

1. **Foydalanuvchi tajribasi (UX)** — silliq, "jank"siz animatsiyalar va tezkor interaktivlik foydalanuvchi qoniqishini oshiradi
2. **Batareya va resurs tejash** — `requestAnimationFrame` fon rejimida avtomatik to'xtaydi, bu mobil qurilmalarda batareyani tejaydi
3. **Katta ma'lumotlar bilan ishlashda barqarorlik** — minglab elementli jadval yoki ro'yxatlarni to'g'ri render qilish, sahifani "muzlatib qo'ymaslik" uchun zarur
4. **60 FPS maqsadi** — silliq animatsiya uchun har bir frame **16.6ms**dan kam vaqtda tugashi kerak; Layout Thrashing bu byudjetni osongina buzadi
5. **SEO va Core Web Vitals** — Google'ning **CLS (Cumulative Layout Shift)** va boshqa performance ko'rsatkichlari, to'g'ri Layout boshqaruviga bevosita bog'liq

---

## 11. Eng ko'p uchraydigan xatolar

### Xato 1: Sikl ichida o'qish va yozishni aralashtirish

```javascript
// XATO
items.forEach((item) => {
  const width = item.offsetWidth; // O'QISH
  item.style.width = (width + 10) + 'px'; // YOZISH
});
```
**Yechim:** O'qish va yozishni alohida bosqichlarga ajratish (4-bo'limga qarang).

### Xato 2: Animatsiya uchun `setTimeout` ishlatish

```javascript
// XATO - sinxronlashmagan, notekis animatsiya
setInterval(() => {
  x += 1;
  el.style.left = x + 'px';
}, 16);
```
**Yechim:** `requestAnimationFrame` ishlatish.

### Xato 3: `requestIdleCallback`ni muhim ishlar uchun ishlatish

```javascript
// XATO - foydalanuvchi ko'rishi kerak bo'lgan narsa uchun rIC ishlatilgan
requestIdleCallback(() => {
  el.style.display = 'block'; // foydalanuvchi buni DARHOL ko'rishi kerak edi!
});
```
**Yechim:** Vizual, darhol ko'rinishi kerak bo'lgan ishlar uchun `requestAnimationFrame` yoki oddiy sinxron kod, faqat **kechiktirilishi mumkin bo'lgan** ishlar uchun `requestIdleCallback`.

### Xato 4: `getComputedStyle()`ni sikl ichida ko'p marta chaqirish

```javascript
// XATO
elements.forEach((el) => {
  const style = getComputedStyle(el); // har safar Layout trigger qilishi mumkin
  console.log(style.width);
});
```
**Yechim:** Iloji bo'lsa, natijalarni oldindan hisoblab, keshlab qo'yish.

---

## 12. Interview savollari va qisqa javoblar

**S: Layout Thrashing nima va u nega yuz beradi?**
> J: Layout Thrashing — JavaScript kodi DOM'dan o'qish va DOM'ga yozish amallarini ketma-ket, aralashtirib bajarganda yuz beradi. Brauzer odatda yozish amallarini "navbatga qo'yib", frame oxirida bitta marta Layout'ni hisoblaydi, lekin agar yozishdan keyin darhol biror "o'qish" xususiyati (`offsetHeight` kabi) chaqirilsa, brauzer navbatdagi yozishlarni majburan, sinxron tarzda bajarishga majbur bo'ladi — bu sikl ichida takrorlansa, ko'plab keraksiz Layout hisoblashlariga olib keladi.

**S: Layout Thrashing'ning oldini qanday olish mumkin?**
> J: Asosiy usul — o'qish va yozish amallarini **ajratish**: avval barcha kerakli qiymatlarni o'qib, keshlab olish, keyin barcha yozishlarni bajarish. Shuningdek, `DocumentFragment` ishlatish, CSS klasslar orqali o'zgartirish, va imkon qadar `transform`/`opacity` kabi faqat Composite bosqichida ishlaydigan xususiyatlardan foydalanish yordam beradi.

**S: Reflow va Repaint orasidagi farq nima?**
> J: Reflow (Layout) — elementlarning geometriyasini (o'lcham, pozitsiya) qayta hisoblash, bu qimmat operatsiya, chunki bir elementning o'zgarishi boshqalarga ham ta'sir qilishi mumkin. Repaint (Paint) — faqat vizual xususiyatlarni (rang, soya) qayta chizish, geometriya o'zgarmaydi, shuning uchun Reflow'dan arzonroq.

**S: `requestAnimationFrame` va `setTimeout` farqi nima?**
> J: `requestAnimationFrame` brauzerning render sikliga aniq sinxronlashtirilgan (har repaint'dan oldin chaqiriladi) va tab fonga o'tganda avtomatik to'xtaydi, bu esa uni animatsiyalar uchun ancha samarali va barqaror qiladi. `setTimeout`/`setInterval` esa render sikli bilan sinxronlashmagan va fon rejimida ham ishlab, resurslarni behuda sarflashi mumkin.

**S: `requestAnimationFrame` va `requestIdleCallback` farqi nima?**
> J: `requestAnimationFrame` — vizual, darhol ko'rinishi kerak bo'lgan o'zgarishlar uchun, har frame'dan oldin (kafolatli) chaqiriladi. `requestIdleCallback` — brauzer bo'sh bo'lgan vaqtda, kechiktirilishi mumkin bo'lgan fon ishlari uchun ishlatiladi, va uning chaqirilishi kafolatlanmagan.

**S: Nega `transform` va `opacity` boshqa CSS xususiyatlaridan tezroq?**
> J: Chunki ular faqat brauzerning **Composite** bosqichida, alohida GPU qatlamida qayta ishlanadi — Layout va Paint bosqichlarini butunlay chetlab o'tadi. `width`, `left` kabi xususiyatlar esa Layout'dan boshlab, butun pipeline'ni qayta ishga tushiradi.

---

## 13. Xulosa

> **Brauzer rendering pipeline'i** — JavaScript/CSS o'zgarishlarini ekrandagi piksellarga aylantirish jarayoni bo'lib, u **Style → Layout (Reflow) → Paint (Repaint) → Composite** bosqichlaridan iborat. Har bir bosqich oldingisiga bog'liq, va qimmatligi bo'yicha farqlanadi — Layout eng qimmat, Composite eng arzon.
>
> **Layout Thrashing** — DOM'dan o'qish va DOM'ga yozishni aralashtirib bajarish natijasida yuzaga keladigan, brauzerni ko'plab keraksiz, majburiy sinxron Layout hisoblashlariga majburlaydigan jiddiy performance muammosi. Uning oldini olish uchun o'qish va yozish amallarini alohida bosqichlarga ajratish — eng muhim qoida.
>
> **`requestAnimationFrame`** — vizual, render siklga sinxronlashtirilgan yangilanishlar uchun, **`requestIdleCallback`** esa foydalanuvchi tajribasiga ta'sir qilmaydigan, kechiktirilishi mumkin bo'lgan fon ishlari uchun mo'ljallangan ikkita muhim brauzer API'si.
>
> Bu mavzuni chuqur tushunish — nafaqat interview'da, balki katta, murakkab veb-ilovalarda **silliq, tez va samarali** foydalanuvchi tajribasini ta'minlashda muhim, amaliy ahamiyatga ega bilim hisoblanadi.

---

*Tayyorlandi: JS/Frontend texnik intervyu tayyorgarligi uchun*
