# Currying va Partial Application — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [Bu mavzu qayerdan paydo bo'lgan?](#1-bu-mavzu-qayerdan-paydo-bolgan)
2. [Currying nima?](#2-currying-nima)
3. [Currying'ni noldan yozish](#3-curryingni-noldan-yozish)
4. [Universal (avtomatik) curry funksiyasi](#4-universal-avtomatik-curry-funksiyasi)
5. [Partial Application nima?](#5-partial-application-nima)
6. [Currying vs Partial Application — aniq farqi](#6-currying-vs-partial-application--aniq-farqi)
7. [Nega bu ishlaydi — closure mexanizmi](#7-nega-bu-ishlaydi--closure-mexanizmi)
8. [Loyihaning qaysi qismlarida ishlatiladi?](#8-loyihaning-qaysi-qismlarida-ishlatiladi)
9. [Function Composition bilan bog'liqligi](#9-function-composition-bilan-bogliqligi)
10. [Qanday to'g'ri ishlatiladi — amaliy qoidalar](#10-qanday-togri-ishlatiladi--amaliy-qoidalar)
11. [Nega bu muhim — real foydalar](#11-nega-bu-muhim--real-foydalar)
12. [Eng ko'p uchraydigan xatolar](#12-eng-kop-uchraydigan-xatolar)
13. [Interview savollari va qisqa javoblar](#13-interview-savollari-va-qisqa-javoblar)
14. [Xulosa](#14-xulosa)

---

## 1. Bu mavzu qayerdan paydo bo'lgan?

**Currying** va **Partial Application** — **funksional dasturlash** (functional programming) paradigmasidan kelib chiqqan tushunchalar bo'lib, ular JavaScript'ga xos emas — bu tushunchalar matematik **mantiq nazariyasi** (logic) va **lambda hisobi** (lambda calculus)dan kelib chiqqan.

### Nomi qayerdan kelib chiqqan?

"Currying" atamasi matematik **Haskell Curry** (1900–1982) nomidan olingan — u bu tushunchani rivojlantirishga katta hissa qo'shgan, garchi g'oyaning o'zi undan oldin, **Moses Schönfinkel** va **Gottlob Frege** tomonidan ishlab chiqilgan bo'lsa ham (shu sababli ba'zan "Schönfinkelization" deb ham ataladi, lekin "currying" nomi keng qabul qilingan).

Bu tushunchalar **Haskell**, **Lisp**, **ML** kabi funksional dasturlash tillarida (1970–1980-yillar) markaziy o'rin egallagan, va JavaScript **birinchi darajali funksiyalar** (first-class functions) va **closure**ga ega bo'lgani sababli, bu paradigmalarni **to'liq qo'llab-quvvatlaydi** — garchi JS asosan **multi-paradigm** til bo'lsa ham (funksional + OOP + imperative).

---

## 2. Currying nima?

**Currying** — bu **bir nechta argument qabul qiluvchi funksiyani**, **bittadan argument qabul qiluvchi, funksiyalar zanjiriga** aylantirish jarayoni.

```javascript
// Oddiy funksiya - 3 ta argumentni BIRDANIGA qabul qiladi
function add(a, b, c) {
  return a + b + c;
}
add(1, 2, 3); // 6

// Curried versiya - har safar BITTA argument qabul qiladi
function curriedAdd(a) {
  return function (b) {
    return function (c) {
      return a + b + c;
    };
  };
}
curriedAdd(1)(2)(3); // 6
```

### Bu nimani anglatadi?

`curriedAdd(1)` chaqirilganda, u **darhol yakuniy natijani hisoblamaydi** — u `b`ni kutayotgan **yangi funksiya** qaytaradi. `curriedAdd(1)(2)` chaqirilganda, u `c`ni kutayotgan **yana bir yangi funksiya** qaytaradi. Faqat `curriedAdd(1)(2)(3)` chaqirilganda, **barcha argumentlar to'plangach**, yakuniy hisob-kitob bajariladi.

### Zamonaviy sintaksis — arrow function bilan

```javascript
const curriedAdd = (a) => (b) => (c) => a + b + c;

curriedAdd(1)(2)(3); // 6

// Har bir qadamni alohida o'zgaruvchiga saqlash HAM mumkin
const addOne = curriedAdd(1);      // b => c => 1 + b + c
const addOneTwo = addOne(2);        // c => 1 + 2 + c
const result = addOneTwo(3);        // 6
```

---

## 3. Currying'ni noldan yozish

### 2 argumentli oddiy curry

```javascript
function curry2(fn) {
  return function (a) {
    return function (b) {
      return fn(a, b);
    };
  };
}

function multiply(a, b) {
  return a * b;
}

const curriedMultiply = curry2(multiply);
console.log(curriedMultiply(3)(4)); // 12
```

### Amaliy misol — moslashuvchan validatsiya funksiyasi

```javascript
const checkAge = (minAge) => (age) => age >= minAge;

const isAdult = checkAge(18); // "18 yoshdan katta" tekshiruvchi funksiya YARATILDI
const isSenior = checkAge(65); // "65 yoshdan katta" tekshiruvchi funksiya

console.log(isAdult(20)); // true
console.log(isAdult(15)); // false
console.log(isSenior(70)); // true
```

Bu yerda `checkAge(18)` — **qayta ishlatiladigan, maxsuslashtirilgan funksiya** (`isAdult`) yaratadi. Bu — currying'ning eng katta amaliy foydalaridan biri: **umumiy funksiyadan, maxsus, qayta ishlatiladigan variantlar yaratish**.

---

## 4. Universal (avtomatik) curry funksiyasi

Amaliyotda, har safar qo'lda ichma-ich funksiyalar yozish o'rniga, **istalgan funksiyani avtomatik curry qiladigan** universal `curry()` yordamchi funksiyasi yaratiladi (bu — Lodash, Ramda kabi kutubxonalarda tayyor holda mavjud).

```javascript
function curry(fn) {
  return function curried(...args) {
    // Agar YETARLI argument to'plangan bo'lsa (fn.length - funksiyaning kutilgan argumentlar soni)
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    // Aks holda, YANA BIR funksiya qaytaramiz, qolgan argumentlarni kutib
    return function (...moreArgs) {
      return curried.apply(this, [...args, ...moreArgs]);
    };
  };
}
```

### Ishlatish — moslashuvchanligi

```javascript
function sum3(a, b, c) {
  return a + b + c;
}

const curriedSum = curry(sum3);

// Bir xil funksiyani turli usulda chaqirish mumkin:
console.log(curriedSum(1)(2)(3));    // 6 - har biri alohida
console.log(curriedSum(1, 2)(3));    // 6 - ikkitasi birga, biri alohida
console.log(curriedSum(1, 2, 3));    // 6 - hammasi birdaniga (oddiy chaqiruv kabi)
console.log(curriedSum(1)(2, 3));    // 6 - biri alohida, ikkitasi birga
```

Bu — **avtomatik curry**ning kuchi: u funksiyaga **istalgan tartibda va guruhda** argument berish imkonini beradi, va `fn.length` orqali (funksiya deklaratsiyasida e'lon qilingan parametrlar soni) qachon "yetarli" argument to'planganini biladi.

> **Muhim eslatma:** `fn.length` — rest parametrlar (`...args`) yoki default qiymatli parametrlarni **hisobga olmaydi**. Shuning uchun universal `curry()` funksiyasi bunday funksiyalar bilan to'g'ri ishlamasligi mumkin — bu holatlarda kerakli argumentlar sonini **qo'lda ko'rsatish** kerak bo'ladi.

---

## 5. Partial Application nima?

**Partial Application** ("qisman qo'llash") — bu funksiyaga **bir qism argumentlarni oldindan "bog'lab qo'yish"** va qolgan argumentlarni **keyinroq** berish imkonini beruvchi texnika. Natijada — **kamroq argument talab qiladigan yangi funksiya** hosil bo'ladi.

```javascript
function partial(fn, ...presetArgs) {
  return function (...laterArgs) {
    return fn(...presetArgs, ...laterArgs);
  };
}

function add3(a, b, c) {
  return a + b + c;
}

const addFivePlus = partial(add3, 5); // "a" oldindan 5 qilib "bog'landi"
console.log(addFivePlus(2, 3)); // 5 + 2 + 3 = 10

const addFiveTenPlus = partial(add3, 5, 10); // "a" va "b" oldindan bog'landi
console.log(addFiveTenPlus(3)); // 5 + 10 + 3 = 18
```

### `bind()` — JavaScript'ning o'rnatilgan Partial Application vositasi

`Function.prototype.bind()` — aslida, `this`ni bog'lashdan tashqari, **partial application**ni ham amalga oshiradi:

```javascript
function greet(greeting, name) {
  console.log(`${greeting}, ${name}!`);
}

const sayHello = greet.bind(null, 'Salom'); // "greeting" oldindan bog'landi
sayHello('Ali'); // "Salom, Ali!"
sayHello('Vali'); // "Salom, Vali!"
```

---

## 6. Currying vs Partial Application — aniq farqi

Bu ikkalasi **ko'pincha adashtiriladi**, chunki ular tashqi ko'rinishda o'xshash natija beradi (funksiyani "bo'laklab" chaqirish), lekin ular orasida **fundamental farq** bor.

| | Currying | Partial Application |
|---|---|---|
| **Natija** | Har safar **faqat bitta** argument qabul qiluvchi funksiyalar zanjiri | **Istalgan miqdordagi** argumentni oldindan "bog'laydi", qolganini birdaniga qabul qiladi |
| **Chaqirish shakli** | `f(a)(b)(c)` — qat'iy, bittadan | `f(a, b)` — moslashuvchan, guruh-guruh |
| **Yakuniy natija qachon hisoblanadi** | Faqat **barcha** argumentlar (bittalab) to'plangach | Har safar chaqirilganda, agar yetarli argument bo'lsa |
| **Maqsad** | Funksiyani **bir argumentli funksiyalar zanjiriga** aylantirish (matematik/nazariy asos) | Funksiyaning **ma'lum qismini oldindan sozlab**, qayta ishlatiladigan variant yaratish |

### Aniq misol bilan farqni ko'rish

```javascript
function add3(a, b, c) {
  return a + b + c;
}

// CURRYING - har doim BITTADAN argument
const curried = curry(add3);
curried(1)(2)(3); // 6
// curried(1, 2)(3) HAM ishlaydi (universal curry funksiyasida), lekin
// KONSEPTUAL jihatdan currying "bitta-bitta" g'oyasiga asoslanadi

// PARTIAL APPLICATION - istalgan miqdorni oldindan "bog'lash"
const partiallyApplied = partial(add3, 1, 2); // IKKALASI birdaniga bog'landi
partiallyApplied(3); // 6 - faqat BITTA chaqiruv qoldi, ko'proq emas
```

> **Interview uchun eng muhim jumla:** Currying — funksiyani **bir argumentli funksiyalar zanjiriga aylantirish** jarayoni, natijada hosil bo'lgan funksiya **doim** yangi funksiya qaytaradi, toki barcha argumentlar yig'ilmaguncha. Partial Application — funksiyaning **istalgan sondagi** argumentini oldindan "bog'lab", qolganini **bitta chaqiruvda** kutuvchi yangi funksiya yaratish. Currying — **nazariy jihatdan qattiqroq** tuzilishga ega, Partial Application — **amaliy jihatdan moslashuvchanroq**.

---

## 7. Nega bu ishlaydi — closure mexanizmi

Currying va Partial Application'ning ikkalasi ham **closure** (yopiq funksiya) mexanizmiga asoslanadi — bu JS/Interview seriyamizdagi "Closure va Xotira" mavzusi bilan bevosita bog'liq.

```javascript
function curriedAdd(a) {
  // 'a' shu yerda "yopiladi" (closure orqali saqlanadi)
  return function (b) {
    // 'a' va 'b' ikkalasi ham saqlanadi
    return function (c) {
      // 'a', 'b', 'c' - barchasi closure orqali ERISHILADIGAN
      return a + b + c;
    };
  };
}
```

Har bir ichki funksiya, tashqi funksiyaning **lexical scope**idagi o'zgaruvchilarga (`a`, keyin `b`) closure orqali **murojaat qilib turadi**, hatto tashqi funksiya allaqachon "tugagan" bo'lsa ham. Bu — currying'ning **argumentlarni "eslab qolishi"**ning aynan sababi.

---

## 8. Loyihaning qaysi qismlarida ishlatiladi?

| Soha | Misol |
|---|---|
| **Redux/State management** | `connect(mapStateToProps)(Component)` — bu klassik curry pattern, ikkita bosqichda chaqiriladigan HOC (Higher-Order Component) |
| **Event handler'lar** | `onClick={handleClick(itemId)}` — ID'ni oldindan "bog'lab", keyin event obyekti bilan chaqiriladigan funksiya |
| **Validatsiya kutubxonalari** | `minLength(5)`, `maxLength(20)` kabi, parametrlashtirilgan validator funksiyalari yaratish |
| **Funksional dasturlash kutubxonalari** | Lodash (`_.curry`), Ramda — currying ularning **yadrosi** hisoblanadi |
| **Middleware arxitekturasi** | Express.js, Redux middleware'lari — ko'pincha curry pattern orqali sozlanadi (`logger(options)(store)(next)(action)`) |
| **API so'rov funksiyalari** | `createApiRequest(baseUrl)(endpoint)(params)` — bazaviy URL'ni oldindan bog'lab, qayta ishlatiladigan so'rov funksiyalari yaratish |
| **Funksiya kompozitsiyasi (composition)** | `pipe`/`compose` funksiyalari bilan birga, murakkab data transformatsiya zanjirlarini qurish |

### Amaliy misol — Redux'dagi curry pattern

```javascript
// Redux'ning connect() funksiyasi - klassik curried HOC
const mapStateToProps = (state) => ({ user: state.user });

const ConnectedComponent = connect(mapStateToProps)(MyComponent);
// connect(mapStateToProps) - BIRINCHI chaqiruv, "sozlangan" funksiya qaytaradi
// ...  (MyComponent) - IKKINCHI chaqiruv, yakuniy komponentni yaratadi
```

### Amaliy misol — event handler'larda

```javascript
function handleDelete(itemId) {
  return function (event) {
    console.log(`${itemId} o'chirilmoqda`, event);
    deleteItem(itemId);
  };
}

items.map((item) => (
  <button onClick={handleDelete(item.id)}>O'chirish</button>
  // handleDelete(item.id) - itemId oldindan "bog'lanadi",
  // event obyekti esa faqat FOYDALANUVCHI bosganda beriladi
));
```

---

## 9. Function Composition bilan bog'liqligi

Currying — **funksiya kompozitsiyasi** (function composition) bilan juda yaqin bog'liq, chunki curried funksiyalar odatda **bittadan argument** qabul qilgani sababli, ularni oson **zanjirlash** (pipe/compose) mumkin bo'ladi.

```javascript
const pipe = (...fns) => (initialValue) =>
  fns.reduce((value, fn) => fn(value), initialValue);

const double = (x) => x * 2;
const addTen = (x) => x + 10;
const square = (x) => x * x;

const transform = pipe(double, addTen, square);
console.log(transform(3)); // ((3*2)+10)^2 = 256
```

Bu — funksional dasturlashda **"tubeless pipeline"** deb ataladigan pattern bo'lib, currying orqali yaratilgan **bir argumentli funksiyalar** bu turdagi zanjirlarga **mukammal mos keladi** — bu currying'ning nazariy asosidagi eng katta amaliy foydalaridan biri.

---

## 10. Qanday to'g'ri ishlatiladi — amaliy qoidalar

### 10.1. Tayyor kutubxonalardan foydalanish (Lodash/Ramda)

```javascript
import { curry } from 'lodash';

function sum3(a, b, c) {
  return a + b + c;
}

const curriedSum = curry(sum3);
curriedSum(1)(2)(3); // 6
curriedSum(1, 2)(3); // 6 - moslashuvchan
```

Real loyihalarda, ko'pincha **noldan curry funksiyasi yozish shart emas** — Lodash'ning `_.curry()` yoki Ramda kutubxonasi (funksional dasturlash uchun to'liq maxsuslashtirilgan) tayyor, sinovdan o'tgan yechim beradi.

### 10.2. Faqat kerak bo'lganda ishlatish

Currying — kuchli, lekin **ba'zan ortiqcha murakkablik** olib kelishi mumkin. Oddiy, faqat bir marta chaqiriladigan funksiyalar uchun currying **shart emas**.

```javascript
// KERAKSIZ murakkablik - bu funksiya faqat 1 marta, oddiy tarzda chaqiriladi
const greet = (greeting) => (name) => `${greeting}, ${name}`;
console.log(greet('Salom')('Ali'));

// Agar qayta ishlatish rejasi bo'lmasa, oddiy funksiya YETARLI
function greetSimple(greeting, name) {
  return `${greeting}, ${name}`;
}
```

**Currying eng foydali** bo'ladigan holat — funksiyaning **bir qismi oldindan ma'lum**, qolgan qismi esa **keyinroq, turli kontekstlarda** beriladigan holatlar.

---

## 11. Nega bu muhim — real foydalar

1. **Qayta ishlatiladigan, maxsuslashtirilgan funksiyalar yaratish** — umumiy funksiyadan, ma'lum parametr bilan "oldindan sozlangan" variantlar hosil qilish (masalan, `checkAge(18)` → `isAdult`)
2. **Kodni deklarativ va o'qilishi oson qilish** — funksional pipeline'lar orqali, murakkab transformatsiyalarni qadam-baqadam, tushunarli tarzda ifodalash
3. **Funksiya kompozitsiyasini osonlashtirish** — bittadan argument qabul qiluvchi funksiyalar `pipe`/`compose` bilan oson birlashtiriladi
4. **Konfiguratsiyani ma'lumotdan ajratish** — masalan, `connect(mapStateToProps)` — "qanday ulash kerak" (konfiguratsiya) va "nimani ulash kerak" (ma'lumot) alohida bosqichlarda beriladi
5. **Test yozishni osonlashtirish** — curried funksiyalar orqali, alohida "qisman qo'llangan" versiyalarni mustaqil test qilish mumkin

---

## 12. Eng ko'p uchraydigan xatolar

### Xato 1: Har bir funksiyani ortiqcha curry qilish

```javascript
// KERAKSIZ - bu shunchaki kodni murakkablashtiradi
const add = (a) => (b) => a + b;
console.log(add(2)(3)); // agar bu funksiya faqat bir marta, oddiy tarzda chaqirilsa - buning hojati yo'q
```
**Yechim:** Currying'ni faqat qayta ishlatish yoki funksional kompozitsiya rejalashtirilgan joylarda qo'llash.

### Xato 2: `this` context'ini yo'qotib qo'yish

```javascript
const obj = {
  value: 10,
  add: function (a) {
    return function (b) {
      return this.value + a + b; // XATO! this bu yerda obj emas
    };
  },
};

const addFn = obj.add(5);
addFn(3); // XATO - this undefined yoki global obyekt
```
**Yechim:** Arrow function ishlatish (agar `this`ni tashqi scope'dan meros olish kerak bo'lsa) yoki `bind()` bilan aniq bog'lash.

### Xato 3: `fn.length`ga tayanadigan universal curry'ni rest parametrli funksiyalar bilan ishlatish

```javascript
function sumAll(...nums) {
  return nums.reduce((a, b) => a + b, 0);
}

console.log(sumAll.length); // 0 - rest parametr hisobga OLINMAYDI!
const curriedSumAll = curry(sumAll); // universal curry TO'G'RI ISHLAMAYDI
```
**Yechim:** Bunday holatlarda kerakli argumentlar sonini qo'lda ko'rsatish (`curry(sumAll, 3)` kabi) yoki maxsus yechim qo'llash.

---

## 13. Interview savollari va qisqa javoblar

**S: Currying nima?**
> J: Bu — bir nechta argument qabul qiluvchi funksiyani, har safar faqat bittadan argument qabul qiluvchi funksiyalar zanjiriga aylantirish jarayoni. Masalan, `f(a, b, c)` → `f(a)(b)(c)`.

**S: Partial Application nima va u currying'dan qanday farq qiladi?**
> J: Partial Application — funksiyaga istalgan miqdordagi argumentni oldindan "bog'lab qo'yish" va qolganini keyinroq, bitta chaqiruvda berish. Currying — har doim bittadan argument qabul qiluvchi funksiyalar zanjiriga aylantiradi. Farqi: currying "qattiq" (bir argumentli) tuzilishga ega, partial application esa istalgan sondagi argumentni bir vaqtda "bog'lashi" mumkin.

**S: Currying ichida qanday mexanizm ishlaydi?**
> J: Closure. Har bir ichki funksiya, tashqi funksiyaning lexical scope'idagi argumentlarga (masalan, `a`, keyin `b`) closure orqali murojaat qilib turadi, hatto tashqi funksiya allaqachon "tugagan" bo'lsa ham — shu sababli argumentlar "eslab qolinadi".

**S: `fn.length` universal curry funksiyasida nima uchun ishlatiladi va uning cheklovi nima?**
> J: `fn.length` — funksiyada e'lon qilingan parametrlar soni bo'lib, u universal curry funksiyasiga "qachon yetarli argument to'plangan"ini bilishga yordam beradi. Cheklovi — u rest parametrlar (`...args`) yoki default qiymatli parametrlarni hisobga olmaydi, shuning uchun bunday funksiyalar bilan to'g'ri ishlamaydi.

**S: Currying qaerda amaliy foydali bo'ladi?**
> J: Qayta ishlatiladigan, maxsuslashtirilgan funksiyalar yaratishda (masalan, validatsiya funksiyalari), Redux'ning `connect()` kabi HOC pattern'larida, event handler'larga argument "bog'lashda", va funksiya kompozitsiyasi (pipe/compose) qurishda.

---

## 14. Xulosa

> **Currying va Partial Application** — funksional dasturlash paradigmasidan kelib chiqqan, funksiyalarni **moslashuvchan, qayta ishlatiladigan** qismlarga bo'lish imkonini beruvchi ikkita yaqin, lekin **konseptual jihatdan farqli** texnika.
>
> **Currying** — funksiyani **bir argumentli funksiyalar zanjiriga** aylantiradi (`f(a)(b)(c)`), **Partial Application** esa istalgan miqdordagi argumentni **oldindan "bog'lab"**, qolganini bitta chaqiruvda kutadigan yangi funksiya yaratadi.
>
> Ikkalasi ham **closure** mexanizmiga asoslanadi va JavaScript'da (funksiyalar birinchi darajali obyekt bo'lgani sababli) **tabiiy ravishda** qo'llab-quvvatlanadi. Ular Redux, event handling, validatsiya kutubxonalari va funksiya kompozitsiyasi kabi ko'plab real loyihalarda **qayta ishlatiladigan, deklarativ va toza** kod yozishga yordam beradi.
>
> Bu mavzuni chuqur tushunish — nafaqat interview'da, balki funksional dasturlash prinsiplariga asoslangan zamonaviy JS/React kod bazalarini o'qish va yozishda ham muhim asos bo'lib xizmat qiladi.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
