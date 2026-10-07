# `this` Kalit So'zi — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [`this` nima?](#1-this-nima)
2. [Asosiy qoida: `this` chaqirilish usuliga bog'liq](#2-asosiy-qoida-this-chaqirilish-usuliga-bogliq)
3. [`this`ni aniqlashning 4 ta qoidasi](#3-thisni-aniqlashning-4-ta-qoidasi)
   - [3.1. Default Binding (Oddiy chaqiruv)](#31-default-binding-oddiy-chaqiruv)
   - [3.2. Implicit Binding (Obyekt orqali chaqiruv)](#32-implicit-binding-obyekt-orqali-chaqiruv)
   - [3.3. Explicit Binding (`call`, `apply`, `bind`)](#33-explicit-binding-call-apply-bind)
   - [3.4. `new` Binding (Konstruktor orqali chaqiruv)](#34-new-binding-konstruktor-orqali-chaqiruv)
4. [Qoidalar ustuvorligi (Priority)](#4-qoidalar-ustuvorligi-priority)
5. [Arrow function va `this`](#5-arrow-function-va-this)
6. [Class'larda `this`](#6-classlarda-this)
7. [Global kontekstda `this`](#7-global-kontekstda-this)
8. [Eng ko'p uchraydigan xatolar](#8-eng-kop-uchraydigan-xatolar)
9. [`this` qayerlarda muhim ahamiyatga ega?](#9-this-qayerlarda-muhim-ahamiyatga-ega)
10. [Interview savollari va qisqa javoblar](#10-interview-savollari-va-qisqa-javoblar)
11. [Xulosa](#11-xulosa)

---

## 1. `this` nima?

`this` — JavaScript'dagi maxsus kalit so'z bo'lib, u **funksiya bajarilayotgan vaqtdagi "kontekst obyekti"**ga ishora qiladi. Boshqacha aytganda, `this` — "hozir bu kodni kim chaqiryapti?" degan savolga javob beruvchi havola.

```javascript
const user = {
  name: 'Ali',
  sayHi() {
    console.log(`Salom, men ${this.name}man`);
  },
};

user.sayHi(); // "Salom, men Aliman"
```

Bu yerda `this` — `user` obyektiga ishora qiladi, chunki `sayHi()` metodi aynan `user` orqali chaqirildi.

### Nega `this` kerak?

`this` bo'lmaganda, har bir metod ichida qaysi obyektga tegishli ekanligini bildirish uchun obyekt nomini qattiq yozib qo'yishga to'g'ri kelardi:

```javascript
// this'siz - noqulay va qayta ishlatib bo'lmaydigan kod
const user = {
  name: 'Ali',
  sayHi() {
    console.log(`Salom, men ${user.name}man`); // "user" nomini qattiq yozib qo'ydik
  },
};
```

`this` orqali esa bitta funksiyani **turli obyektlar uchun qayta ishlatish** mumkin bo'ladi — bu quyida yaqqol ko'rinadi.

---

## 2. Asosiy qoida: `this` chaqirilish usuliga bog'liq

Bu — `this`ni tushunishdagi **eng muhim tushuncha**:

> **`this`ning qiymati funksiya qayerda YOZILGANIGA emas, balki u qanday CHAQIRILGANIGA (call-site) bog'liq.**

Bitta funksiyaning o'zi, uni turlicha chaqirish orqali, turli `this` qiymatiga ega bo'lishi mumkin:

```javascript
function whoAmI() {
  console.log(this.name);
}

const person1 = { name: 'Ali', whoAmI };
const person2 = { name: 'Vali', whoAmI };

person1.whoAmI(); // "Ali"
person2.whoAmI(); // "Vali"
whoAmI();          // xatolik yoki undefined — hech qanday obyekt orqali chaqirilmadi
```

Diqqat qiling: `whoAmI` funksiyasining o'zi **bir marta** yozilgan, lekin uni **qanday chaqirishimizga qarab**, `this` turlicha bo'lib qoladi. Bu — arrow function'dan tashqari **barcha oddiy funksiyalar** uchun to'g'ri.

---

## 3. `this`ni aniqlashning 4 ta qoidasi

JavaScript'da `this`ning qiymati quyidagi 4 ta qoidadan biriga qarab belgilanadi.

### 3.1. Default Binding (Oddiy chaqiruv)

Funksiya hech qanday obyektga bog'lanmasdan, "yalang'och" chaqirilganda ishlaydi.

```javascript
function show() {
  console.log(this);
}

show();
// Strict mode'da: undefined
// Non-strict (sloppy) mode'da: global obyekt (brauzerda - window, Node.js'da - global)
```

```javascript
'use strict';
function show() {
  console.log(this); // undefined
}
show();
```

> **Muhim:** ES6 module'lar va class'lar avtomatik ravishda **strict mode**da ishlaydi, shu sababli zamonaviy JS kodda bu holatda `this` ko'pincha `undefined` bo'ladi.

---

### 3.2. Implicit Binding (Obyekt orqali chaqiruv)

Funksiya biror obyektning metodi sifatida, `obyekt.metod()` shaklida chaqirilganda ishlaydi. Bu — eng ko'p uchraydigan holat.

```javascript
const user = {
  name: 'Ali',
  greet() {
    console.log(`Salom, ${this.name}`);
  },
};

user.greet(); // "Salom, Ali" — this = user
```

**Qoida:** `this` — metod chaqirilgan **eng yaqin (immediate) obyekt**ga bog'lanadi.

```javascript
const company = {
  name: 'TechCorp',
  department: {
    name: 'IT bo\'limi',
    show() {
      console.log(this.name); // "IT bo'limi" — this eng yaqin obyekt (department)ga bog'lanadi
    },
  },
};

company.department.show(); // "IT bo'limi", "TechCorp" emas!
```

⚠️ **Muhim xato holat — "implicit binding yo'qolishi" (implicit binding loss):**

```javascript
const user = {
  name: 'Ali',
  greet() {
    console.log(`Salom, ${this.name}`);
  },
};

const greetFn = user.greet; // metodni obyektdan "ajratib" oldik
greetFn(); // "Salom, undefined" — endi this obyektga bog'lanmagan!
```

Bu holat — closure va event handler'larda `this`ning "yo'qolishi" muammosining asosiy sababi (buni 6-bo'limda batafsil ko'ramiz).

---

### 3.3. Explicit Binding (`call`, `apply`, `bind`)

`this`ni **qo'lda, aniq belgilash** uchun ishlatiladi.

#### `call()`
Funksiyani **darhol** chaqiradi, `this`ni birinchi argument sifatida, qolganlarini **vergul bilan** qabul qiladi.

```javascript
function greet(greeting) {
  console.log(`${greeting}, ${this.name}`);
}

const person = { name: 'Ali' };
greet.call(person, 'Salom'); // "Salom, Ali"
```

#### `apply()`
`call()` bilan bir xil, farqi — qolgan argumentlarni **massiv** shaklida qabul qiladi.

```javascript
greet.apply(person, ['Salom']); // "Salom, Ali"
```
> Eslab qolish: **apply → Array** (ikkalasi ham "A" harfi bilan boshlanadi)

#### `bind()`
Funksiyani darhol chaqirmaydi — `this`ni **doimiy bog'lab**, yangi funksiya qaytaradi.

```javascript
const boundGreet = greet.bind(person);
boundGreet('Salom'); // "Salom, Ali" — keyinroq chaqirildi
```

### Qisqa taqqoslash jadvali

| | Darhol chaqiradimi? | Argumentlar | Natija |
|---|---|---|---|
| `call(thisArg, a, b)` | Ha | Vergul bilan | Funksiya natijasi |
| `apply(thisArg, [a, b])` | Ha | Massiv | Funksiya natijasi |
| `bind(thisArg, a, b)` | Yo'q | Vergul bilan | Yangi (bog'langan) funksiya |

---

### 3.4. `new` Binding (Konstruktor orqali chaqiruv)

Funksiya `new` kalit so'zi bilan chaqirilganda, JavaScript avtomatik ravishda:

1. Yangi, bo'sh obyekt yaratadi
2. Bu obyektni funksiya ichidagi `this`ga bog'laydi
3. Funksiya ishini tugatgandan so'ng, agar boshqa obyekt qaytarilmagan bo'lsa, shu yangi obyektni qaytaradi

```javascript
function User(name) {
  this.name = name; // this = yangi yaratilayotgan obyekt
}

const user1 = new User('Ali');
console.log(user1.name); // "Ali"
```

Bu xuddi quyidagicha ishlaydi (soddalashtirilgan):

```javascript
function User(name) {
  const newObj = {};        // 1. Yangi obyekt yaratiladi
  newObj.__proto__ = User.prototype;
  newObj.name = name;       // 2. this = newObj, uning property'lari to'ldiriladi
  return newObj;            // 3. Obyekt qaytariladi
}
```

---

## 4. Qoidalar ustuvorligi (Priority)

Agar bir nechta qoida bir vaqtda "da'vogar" bo'lsa, quyidagi ustuvorlik tartibi qo'llaniladi (yuqoridan pastga — eng kuchlisidan eng kuchsizigacha):

```
1. new Binding           (eng kuchli)
2. Explicit Binding       (call / apply / bind)
3. Implicit Binding       (obyekt.metod())
4. Default Binding        (eng kuchsiz — undefined yoki global)
```

**Misol:**

```javascript
function greet() {
  console.log(this.name);
}

const obj = { name: 'Obyekt', greet };
const boundGreet = greet.bind({ name: 'Bind orqali' });

obj.greet();       // "Obyekt" — implicit binding
boundGreet();       // "Bind orqali" — explicit binding, implicit'dan kuchliroq

const boundObj = { name: 'Obyekt2', greet: boundGreet };
boundObj.greet();  // "Bind orqali" — explicit (bind) implicit'dan hali ham kuchli!
```

> **Interview uchun muhim eslatma:** `bind()` orqali bog'langan `this`ni keyinchalik na obyekt orqali chaqirish, na yana bir marta `call`/`apply`/`bind` bilan o'zgartirib bo'lmaydi — bu **"hard binding"** deb ataladi.

---

## 5. Arrow function va `this`

Bu — 4 ta qoidadan **mutlaqo mustasno** holat.

> **Arrow function'lar o'zining `this`iga umuman ega emas.** Ular `this`ni yaratilgan joyidagi **tashqi (lexical) scope**dan meros qiladi — xuddi oddiy o'zgaruvchidek.

```javascript
const obj = {
  name: 'Ali',
  regularFunc: function () {
    console.log(this.name); // 'Ali' — implicit binding ishlaydi
  },
  arrowFunc: () => {
    console.log(this.name); // undefined — this global/tashqi scope'dan olinadi
  },
};

obj.regularFunc(); // 'Ali'
obj.arrowFunc();   // undefined
```

`arrowFunc` obyekt ichida yozilgan bo'lsa-da, obyekt o'zi **funksiya scope hosil qilmaydi**. Shu sababli arrow function `this`ni bir daraja tashqarida, ya'ni bu kod yozilgan modul/global scope'dan meros qiladi.

### Arrow function `this`ni o'zgartirib bo'lmaydi

```javascript
const arrow = () => console.log(this.name);

arrow.call({ name: 'Test' });   // hech qanday ta'sir yo'q
arrow.apply({ name: 'Test' });  // hech qanday ta'sir yo'q
const bound = arrow.bind({ name: 'Test' });
bound();                         // baribir tashqi scope'dagi this
```

### Qachon arrow function foydali bo'ladi — Callback'larda `this`ni saqlash

```javascript
class Timer {
  constructor() {
    this.seconds = 0;
  }

  start() {
    setInterval(() => {
      this.seconds++; // TO'G'RI — this Timer instance'idan meros olinadi
      console.log(this.seconds);
    }, 1000);
  }
}
```

Agar bu yerda oddiy `function` ishlatilganida, `setInterval` uni "yalang'och" chaqirgani sababli `this` `undefined` (yoki global obyekt) bo'lib qolardi (Default Binding qoidasi ishga tushardi).

---

## 6. Class'larda `this`

Class metodlari "qopqoq ostida" oddiy funksiyalar hisoblanadi, shu sababli ular ham Implicit Binding qoidasiga bo'ysunadi — va xuddi shu sababdan **`this` "yo'qolishi"** muammosiga duch keladi.

```javascript
class Counter {
  constructor() {
    this.count = 0;
  }

  increment() {
    this.count++;
    console.log(this.count);
  }
}

const counter = new Counter();
counter.increment(); // 1 — to'g'ri ishladi, chunki counter.increment() orqali chaqirildi

const fn = counter.increment;
fn(); // XATO! "Cannot read properties of undefined (reading 'count')"
```

### Nega bunday bo'ladi?

`counter.increment`ni alohida o'zgaruvchiga (`fn`) ajratib olganimizda, funksiyaning o'zi ko'chib o'tadi, lekin uning `counter` bilan bog'lanishi **yo'qoladi**. Keyinroq `fn()` chaqirilganda, bu — hech qanday obyekt orqali emas, "yalang'och" chaqiruv (Default Binding), va class'lar avtomatik strict mode'da ishlagani uchun `this = undefined` bo'ladi.

### Bu qачон real muammoga aylanadi?

Eng ko'p — metodni **callback** yoki **event handler** sifatida uzatganda:

```javascript
class Button {
  constructor() {
    this.label = 'Salom';
  }

  handleClick() {
    console.log(this.label);
  }
}

const btn = new Button();

// XATO — DOM element handleClick'ni "yalang'och" chaqiradi
document.querySelector('button').addEventListener('click', btn.handleClick);
// this DOM elementiga ishora qiladi, btn'ga emas!
```

### Yechimlar

**1. `bind()` — constructor ichida**
```javascript
class Button {
  constructor() {
    this.label = 'Salom';
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    console.log(this.label);
  }
}
```

**2. Class field + arrow function (zamonaviy va eng ko'p tavsiya etiladigan usul)**
```javascript
class Button {
  label = 'Salom';

  handleClick = () => {
    console.log(this.label); // arrow function - this doim instance'ga bog'langan
  };
}
```
Bu yerda `handleClick` — class **metodi emas**, balki instance property'si bo'lib, uning qiymati bitta arrow function. Arrow function `constructor` ichida yaratilgani uchun, `this` doimiy ravishda o'sha instance'ga bog'lanadi — chaqirish usulidan qat'iy nazar.

**3. Chaqirish paytida arrow function bilan o'rash**
```javascript
document.querySelector('button').addEventListener('click', () => btn.handleClick());
```

---

## 7. Global kontekstda `this`

```javascript
console.log(this); 
// Brauzerda, modul bo'lmagan oddiy script: window obyekti
// ES modul ichida: undefined
// Node.js (CommonJS modul darajasida): module.exports (bo'sh obyekt)
```

Brauzerda global kontekstdagi `this` odatda `window`ga teng, lekin **strict mode**da yoki ES modul ichida bu `undefined` bo'ladi.

---

## 8. Eng ko'p uchraydigan xatolar

### Xato 1: Metodni obyektdan ajratib olish

```javascript
const obj = { name: 'Ali', greet() { console.log(this.name); } };
const greet = obj.greet;
greet(); // undefined
```

### Xato 2: `setTimeout`/`setInterval` ichida oddiy funksiya ishlatish

```javascript
const obj = {
  name: 'Ali',
  delayedGreet() {
    setTimeout(function () {
      console.log(this.name); // undefined - this endi obj emas
    }, 1000);
  },
};
```
**Yechim:** arrow function ishlatish yoki `bind()` qo'llash.

### Xato 3: Nested (ichma-ich) oddiy funksiyalarda `this`ning o'zgarib qolishi

```javascript
const obj = {
  name: 'Ali',
  outer() {
    console.log(this.name); // "Ali" — to'g'ri
    function inner() {
      console.log(this.name); // undefined — inner "yalang'och" chaqirildi!
    }
    inner();
  },
};
```
**Yechim:** arrow function ishlatish, yoki eski usul — `const self = this;` orqali tashqi `this`ni saqlab qolish.

---

## 9. `this` qayerlarda muhim ahamiyatga ega?

| Soha | Nega muhim |
|---|---|
| **Obyekt metodlari** | `this` orqali bitta funksiyani turli obyektlar uchun qayta ishlatish mumkin |
| **Event handler'lar (DOM)** | `this` odatda hodisa (event) sodir bo'lgan elementga ishora qiladi (agar oddiy funksiya ishlatilsa) |
| **Class'lar va OOP** | Metodlar instance property'lariga `this` orqali murojaat qiladi |
| **Callback funksiyalar** | `setTimeout`, `addEventListener`, massiv metodlari (`map`, `forEach`) ichida `this`ning to'g'ri saqlanishi muhim |
| **React/Vue kabi freymvorklar** | Class komponentlarda `this.state`, `this.props`ga to'g'ri murojaat qilish uchun `bind` yoki arrow function zarur |
| **Konstruktor funksiyalar/class'lar** | `new` orqali yaratilgan har bir instance o'zining `this`iga ega bo'ladi |

---

## 10. Interview savollari va qisqa javoblar

**S: `this` nimaga bog'liq — funksiya yozilgan joygami yoki chaqirilish usuligami?**
> J: Chaqirilish usuliga (call-site). Bundan yagona istisno — arrow function, u `this`ni yaratilgan joyidagi lexical scope'dan meros oladi.

**S: `call`, `apply`, `bind` orasidagi farq nima?**
> J: `call` va `apply` funksiyani darhol chaqiradi — farqi argumentlarni qanday qabul qilishida (`call` — vergul bilan, `apply` — massiv shaklida). `bind` esa funksiyani darhol chaqirmaydi, balki `this` doimiy bog'langan yangi funksiya qaytaradi.

**S: Arrow function'da `this` qanday ishlaydi?**
> J: Arrow function o'zining `this`iga ega emas — u `this`ni yaratilgan joyidagi tashqi (lexical) scope'dan meros oladi va bu qiymatni `call`/`apply`/`bind` orqali o'zgartirib bo'lmaydi.

**S: Class metodida `this` nega "yo'qoladi"?**
> J: Metod obyektdan ajratilib, boshqa joyga (masalan, callback sifatida) uzatilganda, uning obyekt bilan bog'lanishi yo'qoladi. Keyinroq u "yalang'och" chaqirilganda, Default Binding qoidasi ishga tushadi va `this = undefined` (strict mode tufayli, class'lar avtomatik strict mode'da ishlaydi).

**S: 4 ta binding qoidasining ustuvorlik tartibi qanday?**
> J: `new` binding eng kuchli, keyin explicit binding (`call`/`apply`/`bind`), keyin implicit binding (`obyekt.metod()`), va eng oxirida (eng kuchsizi) — default binding.

**S: `bind()` orqali bog'langan `this`ni yana o'zgartirish mumkinmi?**
> J: Yo'q. Bu — "hard binding" deb ataladi: `bind()` orqali bog'langan funksiyaning `this`ini keyinchalik boshqa obyekt orqali chaqirish yoki yana `call`/`apply`/`bind` bilan almashtirish mumkin emas.

---

## 11. Xulosa

> **`this` — JavaScript'da eng ko'p chalkashtiradigan, lekin aslida oddiy mantiqqa asoslangan tushuncha: uning qiymati funksiya QANDAY chaqirilishiga bog'liq, QAYERDA yozilganiga emas.** 4 ta asosiy qoida bor — default, implicit, explicit va `new` binding — va ular orasida aniq ustuvorlik tartibi mavjud.
>
> **Arrow function'lar bundan mustasno**: ular `this`ni chaqirilish usulidan emas, balki yaratilgan joyidagi lexical scope'dan meros oladi — bu ularni callback'larda `this`ni "yo'qotmasdan" ishlatish uchun juda qulay qiladi.
>
> **Class metodlarida `this` yo'qolishi** — bu aslida implicit binding'ning yo'qolishi, va uni `bind()` yoki class field + arrow function orqali oldini olish mumkin.
>
> Bu mavzuni chuqur tushunish — nafaqat interview'larda, balki React/Vue kabi freymvorklarda ishlashda, event handler'lar yozishda va umuman kod arxitekturasini to'g'ri qurishda muhim asos bo'lib xizmat qiladi.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
