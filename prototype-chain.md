# Prototip Zanjiri (Prototype Chain) — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [Prototip nima va u qayerdan paydo bo'ladi?](#1-prototip-nima-va-u-qayerdan-paydo-boladi)
2. [Prototip zanjiri qanday ishlaydi?](#2-prototip-zanjiri-qanday-ishlaydi)
3. [`prototype` va `__proto__` — aniq farqi](#3-prototype-va-__proto__--aniq-farqi)
4. [`Object.create(null)` va `{}` orasidagi farq](#4-objectcreatenull-va--orasidagi-farq)
5. [Konstruktor funksiyalar va prototip](#5-konstruktor-funksiyalar-va-prototip)
6. [Class sintaksisi — prototip ustidagi "shakar"](#6-class-sintaksisi--prototip-ustidagi-shakar)
7. [Property Shadowing (Soyalash)](#7-property-shadowing-soyalash)
8. [Prototip zanjirini tekshirish usullari](#8-prototip-zanjirini-tekshirish-usullari)
9. [Qayerlarda ishlatiladi va nega muhim?](#9-qayerlarda-ishlatiladi-va-nega-muhim)
10. [Eng ko'p uchraydigan xatolar](#10-eng-kop-uchraydigan-xatolar)
11. [Interview savollari va qisqa javoblar](#11-interview-savollari-va-qisqa-javoblar)
12. [Xulosa](#12-xulosa)

---

## 1. Prototip nima va u qayerdan paydo bo'ladi?

JavaScript boshqa ko'plab tillardan (Java, C++ kabi) farqli o'laroq, **class-based** emas, balki **prototype-based** (prototipga asoslangan) meros tizimiga ega. Bu shuni anglatadiki, obyektlar boshqa class'dan emas, balki bevosita **boshqa obyektlardan** meros oladi.

Har bir JavaScript obyekti o'zida `[[Prototype]]` degan **yashirin ichki xususiyat**ga ega (bu ikki qavs bilan yozilgan nom — spec'dagi rasmiy belgilash, dasturchi to'g'ridan-to'g'ri murojaat qila olmaydigan ichki slot). Bu xususiyat boshqa bir obyektga (yoki `null`ga) ishora qiladi.

```javascript
const obj = { a: 1 };
```

Siz `{ a: 1 }` deb yozganingizda, tashqi ko'rinishda bu faqat bitta `a` property'siga ega obyektdek tuyuladi. Lekin "sahna orqasida" JavaScript avtomatik ravishda bu obyektning `[[Prototype]]`ini **`Object.prototype`**ga bog'laydi — bu esa `obj`ga o'nlab tayyor metodlarni (`toString`, `hasOwnProperty`, `valueOf` va h.k.) **bepul** taqdim etadi.

```javascript
console.log(obj.toString); // function toString() { [native code] }
// Biz bu funksiyani yozmadik — u prototip orqali "meros" bo'lib keldi!
```

**Qayerdan paydo bo'ladi?** — Prototip obyekt yaratilgan **paytda** avtomatik belgilanadi:
- `{}` yozilsa → `Object.prototype`ga bog'lanadi
- `[]` yozilsa → `Array.prototype`ga bog'lanadi (u esa o'z navbatida `Object.prototype`ga bog'langan)
- `function(){}` yozilsa → `Function.prototype`ga bog'lanadi
- `new Konstruktor()` orqali yaratilsa → `Konstruktor.prototype`ga bog'lanadi
- `Object.create(nimadir)` orqali yaratilsa → aynan siz ko'rsatgan obyektga bog'lanadi

---

## 2. Prototip zanjiri qanday ishlaydi?

**Asosiy qoida:** Agar siz obyektdan biror property yoki metodni **o'qimoqchi** bo'lsangiz-u, bu obyektning o'zida (own property sifatida) topilmasa, JavaScript avtomatik ravishda uning `[[Prototype]]`iga qaraydi. Agar u yerda ham topilmasa — yana bir daraja yuqoriga, uning ham prototipiga qaraydi. Bu jarayon **prototip zanjiri** (prototype chain) deb ataladi va u `null`ga yetguncha davom etadi.

```javascript
const arr = [1, 2, 3];

arr.push(4);          // Array.prototype'dan
arr.toString();        // Array.prototype → Object.prototype'dan (Array'da override qilingan)
arr.hasOwnProperty(0); // Object.prototype'dan
```

**Vizual ko'rinishda:**

```
arr (o'zida: 0,1,2,3 elementlar)
  │
  ▼ [[Prototype]]
Array.prototype (o'zida: push, pop, map, filter, forEach ...)
  │
  ▼ [[Prototype]]
Object.prototype (o'zida: toString, hasOwnProperty, valueOf ...)
  │
  ▼ [[Prototype]]
null  ← zanjir shu yerda TUGAYDI
```

**Qidiruv jarayoni qadam-baqadam** (`arr.toString()` chaqirilganda):

1. `arr`ning o'zida `toString` bormi? → Yo'q (lekin `Array.prototype`da bor, chunki Array uni override qilgan)
2. `Array.prototype`da `toString` bormi? → **Ha, topildi!** Qidiruv shu yerda to'xtaydi.

Agar `Array.prototype`da `toString` bo'lmaganida edi, qidiruv yana bir bosqich yuqoriga — `Object.prototype`ga o'tar edi.

> **Muhim eslatma:** Agar property hech qayerda topilmasa (zanjirning oxirigacha, ya'ni `null`gacha yetib borilsa), natija `undefined` bo'ladi — xatolik chiqmaydi.

```javascript
console.log(arr.nonExistentMethod); // undefined, xato emas
```

---

## 3. `prototype` va `__proto__` — aniq farqi

Bu ikkalasi **nomi o'xshash, lekin vazifasi tamomila boshqacha** — interview'da eng ko'p adashtiriladigan juftlik.

### `prototype` — faqat funksiyalarda bo'ladi

`prototype` — bu **funksiya obyektining oddiy property'si**, faqat funksiyalarda (aniqrog'i, konstruktor sifatida ishlatiladigan funksiya va class'larda) mavjud. Bu — `new` orqali yaratiladigan barcha instance'lar uchun **"qolip" (blueprint)** vazifasini o'taydi.

```javascript
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  console.log(`Salom, men ${this.name}man`);
};

console.log(typeof Person.prototype); // "object"
```

### `__proto__` — har bir obyektda bo'ladi

`__proto__` — har bir **obyekt instance**sida mavjud bo'lgan, obyektning yashirin `[[Prototype]]`iga kirish imkonini beruvchi accessor property (getter/setter). Bu — obyektning **haqiqiy, ishlaydigan meros havolasi**.

```javascript
const ali = new Person('Ali');
console.log(ali.__proto__ === Person.prototype); // true!
```

### Ular qanday bog'lanadi?

`new Person('Ali')` chaqirilganda ichkarida shunday bo'ladi:

1. Yangi bo'sh obyekt yaratiladi
2. Bu obyektning `__proto__`si `Person.prototype`ga **tenglashtiriladi**
3. `this` shu yangi obyektga bog'lanadi, konstruktor tanasi bajariladi

```
                    Person.prototype
                    { greet: fn, constructor: Person }
                          ▲
                          │ (__proto__ orqali bog'lanadi)
                        ali
                    { name: 'Ali' }
```

### Aniq farq jadvali

| | `prototype` | `__proto__` |
|---|---|---|
| **Kimda bo'ladi?** | Faqat funksiyalarda | Har qanday obyektda |
| **Vazifasi** | `new` orqali yaratiladigan instance'lar uchun "qolip" belgilaydi | Obyektning haqiqiy meros zanjiridagi havolasini ko'rsatadi |
| **Qachon yaratiladi?** | Funksiya e'lon qilinganda avtomatik | Obyekt yaratilganda avtomatik belgilanadi |
| **Tavsiya etilgan muqobili** | — | `Object.getPrototypeOf()` / `Object.setPrototypeOf()` |

### Yodda qolishi kerak bo'lgan formula

```javascript
instance.__proto__ === Constructor.prototype // har doim true
```

> **Interview uchun eslatma:** `__proto__` — tarixiy sabablarga ko'ra qo'llab-quvvatlanadigan, lekin rasman **deprecated** (eskirgan) hisoblanadi. Zamonaviy kodda uning o'rniga `Object.getPrototypeOf(obj)` va `Object.setPrototypeOf(obj, proto)` ishlatish tavsiya etiladi.

---

## 4. `Object.create(null)` va `{}` orasidagi farq

### `{}` — oddiy obyekt literal

```javascript
const obj1 = {};
console.log(Object.getPrototypeOf(obj1)); // Object.prototype
console.log(obj1.toString);                // function — mavjud
```

`{}` yozilganda, JS avtomatik ravishda uni `Object.prototype`ga bog'laydi — shu sababli har qanday oddiy obyekt `toString()`, `hasOwnProperty()`, `valueOf()` kabi meros metodlarga ega bo'ladi.

### `Object.create(null)` — "sof" (bare) obyekt

```javascript
const obj2 = Object.create(null);
console.log(Object.getPrototypeOf(obj2)); // null
console.log(obj2.toString);                // undefined — umuman yo'q!
```

`Object.create(null)` — bu obyektning `[[Prototype]]`ini bevosita `null` qilib yaratadi. Bu obyekt **hech qanday** meros zanjiriga ega emas — hatto `Object.prototype`dan ham hech narsa olmaydi.

### Nega bu farq amaliy jihatdan muhim?

**1. Prototype pollution xavfidan himoya**

```javascript
// XAVFLI holat oddiy {} bilan
const userInput = {};
userInput['__proto__'] = { isAdmin: true }; // potentsial xavfli holat

// Object.create(null) bilan bunday muammo yo'q,
// chunki __proto__ Object.prototype orqali ishlaydigan mexanizm, u yerda esa umuman yo'q
```

**2. "Sof lug'at" (dictionary) sifatida foydalanish**

Agar obyekt faqat **key-value** saqlash uchun ishlatilsa (masalan, foydalanuvchidan kelgan ma'lumotlarni key sifatida saqlash), `Object.create(null)` xavfsizroq, chunki tasodifiy nom to'qnashuvi (masalan, kimdir `"toString"` degan key qo'shsa) bo'lmaydi.

```javascript
const safeMap = Object.create(null);
safeMap['toString'] = 'salom'; // hech qanday muammo yo'q, chunki bu yerda oldindan toString metodi yo'q edi
```

> **Zamonaviy tavsiya:** Ko'pchilik holatlarda buning o'rniga `Map` ishlatish afzalroq, chunki u yanada tushunarli API va kafolatlangan xavfsizlikka ega. `Object.create(null)` ko'proq performance-critical yoki juda maxsus holatlarda qo'llaniladi.

### Qisqa taqqoslash jadvali

| | `{}` | `Object.create(null)` |
|---|---|---|
| Prototip | `Object.prototype` | `null` |
| `toString()`, `hasOwnProperty()` | Mavjud | Yo'q |
| Konsolda ko'rinishi | `{ a: 1 }` | `[Object: null prototype] { a: 1 }` |
| Prototype pollution xavfi | Bor | Yo'q |
| Ishlatilish o'rni | Kundalik obyektlar | "Sof" lug'at, xavfsizlik talab qilinganda |

---

## 5. Konstruktor funksiyalar va prototip

Class sintaksisi (ES6) paydo bo'lishidan oldin, OOP JavaScript'da **konstruktor funksiyalar** orqali amalga oshirilgan — va bu hamon prototip mexanizmining "chin" (haqiqiy) ishlash tarzini ko'rsatadi.

```javascript
function Animal(name) {
  this.name = name;
}

// Metodlar prototype'ga qo'shiladi - bu barcha instance'lar UCHUN BITTA marta xotirada saqlanadi
Animal.prototype.speak = function () {
  console.log(`${this.name} ovoz chiqarmoqda`);
};

const dog = new Animal('It');
const cat = new Animal('Mushuk');

dog.speak(); // "It ovoz chiqarmoqda"
cat.speak(); // "Mushuk ovoz chiqarmoqda"

console.log(dog.speak === cat.speak); // true — ikkalasi HAM BITTA funksiyani ishlatadi!
```

### Nega metodlar `prototype`ga qo'yiladi, `this` ichiga emas?

```javascript
// SAMARASIZ usul — har bir instance uchun ALOHIDA nusxa yaratiladi
function AnimalBad(name) {
  this.name = name;
  this.speak = function () {          // har safar YANGI funksiya yaratiladi!
    console.log(`${this.name} ovoz chiqarmoqda`);
  };
}

// SAMARALI usul — barcha instance'lar BITTA funksiyani ULASHADI
function AnimalGood(name) {
  this.name = name;
}
AnimalGood.prototype.speak = function () {
  console.log(`${this.name} ovoz chiqarmoqda`);
};
```

Agar 1000 ta `Animal` instance yaratilsa, birinchi usulda **1000 ta alohida** `speak` funksiyasi xotirada saqlanadi. Ikkinchi usulda esa faqat **bitta** `speak` funksiyasi bor — barcha instance'lar uni prototip zanjiri orqali "ulashadi" (share qiladi). Bu — **xotirani sezilarli darajada tejaydi**.

---

## 6. Class sintaksisi — prototip ustidagi "shakar"

ES6'da kiritilgan `class` — bu yangi meros tizimi emas, balki konstruktor funksiya + prototip mexanizmi ustiga qurilgan **qulayroq sintaksis** (syntactic sugar).

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    console.log(`${this.name} ovoz chiqarmoqda`);
  }
}

const dog = new Animal('It');

console.log(typeof Animal); // "function" — class aslida funksiya!
console.log(dog.__proto__ === Animal.prototype); // true
console.log(Animal.prototype.speak); // function — speak METOD emas, prototype'ga qo'shilgan
```

Bu ikki kod bo'lagi **funksional jihatdan bir xil**:

```javascript
// Class bilan
class Animal {
  constructor(name) { this.name = name; }
  speak() { console.log(`${this.name} ovoz chiqarmoqda`); }
}

// Konstruktor funksiya bilan (bir xil natija)
function Animal(name) { this.name = name; }
Animal.prototype.speak = function () { console.log(`${this.name} ovoz chiqarmoqda`); };
```

### `extends` — prototip zanjirini uzaytirish

```javascript
class Dog extends Animal {
  bark() {
    console.log(`${this.name} vovullamoqda`);
  }
}

const rex = new Dog('Rex');
rex.speak(); // Animal.prototype'dan meros
rex.bark();  // Dog.prototype'ning o'zidan

console.log(rex.__proto__ === Dog.prototype);            // true
console.log(rex.__proto__.__proto__ === Animal.prototype); // true — extends orqali zanjir uzaytiriladi
```

```
rex → Dog.prototype → Animal.prototype → Object.prototype → null
```

---

## 7. Property Shadowing (Soyalash)

Agar obyektning o'zida biror property mavjud bo'lsa, bu property **prototipdagi shu nomli property'ni "soyalab" qo'yadi** (shadowing) — ya'ni qidiruv obyektning o'zida to'xtab qoladi, prototipga umuman borilmaydi.

```javascript
function Animal(name) {
  this.name = name;
}

Animal.prototype.sound = 'Noma\'lum ovoz';

const cat = new Animal('Mushuk');
console.log(cat.sound); // "Noma'lum ovoz" — prototipdan olindi

cat.sound = 'Miyov'; // endi cat'ning O'ZIDA sound property'si paydo bo'ladi
console.log(cat.sound); // "Miyov" — endi obyektning o'zidan olinadi, prototipga borilmaydi

console.log(Animal.prototype.sound); // "Noma'lum ovoz" — prototip o'zgarmadi!
```

> **Muhim tushuncha:** `obj.property = value` yozish **hech qachon** prototipdagi qiymatni o'zgartirmaydi — u faqat obyektning o'zida yangi (yoki mavjud) "own property" yaratadi/yangilaydi.

---

## 8. Prototip zanjirini tekshirish usullari

```javascript
const dog = new Animal('It');

// 1. Prototipni olish (tavsiya etilgan zamonaviy usul)
Object.getPrototypeOf(dog); // Animal.prototype

// 2. Prototipni o'rnatish
Object.setPrototypeOf(dog, SomeOtherPrototype);

// 3. Obyekt biror prototip zanjirida bormi - tekshirish
Animal.prototype.isPrototypeOf(dog); // true

// 4. instanceof - obyekt shu konstruktor zanjirida ekanligini tekshiradi
dog instanceof Animal; // true

// 5. Faqat OWN (obyektning o'zidagi, meros bo'lmagan) property'larni tekshirish
dog.hasOwnProperty('name');  // true - o'zida bor
dog.hasOwnProperty('speak'); // false - bu prototipdan meros

// 6. Barcha (o'z + meros) property'lar bo'yicha iteratsiya
for (const key in dog) {
  console.log(key); // name, speak - ikkalasi ham chiqadi (for...in prototip zanjirini ham aylanadi!)
}

// 7. Faqat o'z (own) property'lar
Object.keys(dog); // ['name'] - faqat o'zida borlar
```

---

## 9. Qayerlarda ishlatiladi va nega muhim?

| Soha | Nega muhim |
|---|---|
| **Metodlarni ulashish (memory efficiency)** | Barcha instance'lar bitta metodni "ulashadi", har biriga alohida nusxa yaratilmaydi |
| **Built-in obyektlar (`Array`, `String`, `Object`)** | `.map()`, `.filter()`, `.slice()` kabi barcha tayyor metodlar aynan shu mexanizm orqali ishlaydi |
| **OOP va meros (inheritance)** | `class ... extends` prototip zanjirini uzaytirish orqali ishlaydi |
| **Polyfill yaratish** | Eski brauzerlarda yo'q metodlarni `SomeType.prototype.method = ...` orqali qo'shish mumkin |
| **Xavfsizlik (prototype pollution)** | Zararli kod `__proto__` orqali `Object.prototype`ga zarar yetkazishi mumkinligini tushunish |
| **Performance optimizatsiyasi** | Chuqur prototip zanjirlari property qidiruvini sekinlashtirishi mumkinligini bilish |

### Real hayotdagi misol — polyfill

```javascript
// Agar eski brauzerda Array.prototype.includes mavjud bo'lmasa:
if (!Array.prototype.includes) {
  Array.prototype.includes = function (item) {
    return this.indexOf(item) !== -1;
  };
}

[1, 2, 3].includes(2); // endi barcha massivlarda ishlaydi
```

---

## 10. Eng ko'p uchraydigan xatolar

### Xato 1: `for...in` bilan faqat "o'z" property'larni kutish

```javascript
Array.prototype.customMethod = function () {};
const arr = [1, 2, 3];

for (const key in arr) {
  console.log(key); // 0, 1, 2, customMethod ← kutilmagan natija!
}
```
**Yechim:** `for...in` o'rniga `Object.keys()`, `for...of`, yoki `hasOwnProperty` bilan filtrlash ishlatish.

### Xato 2: Prototipga to'g'ridan-to'g'ri `__proto__` orqali murojaat qilish (performance)

```javascript
obj.__proto__ = anotherProto; // sekinlashtiradi va deprecated
```
**Yechim:** `Object.setPrototypeOf()` yoki, yaxshisi, `Object.create()` orqali boshidan to'g'ri prototip bilan yaratish.

### Xato 3: Prototype pollution xavfini e'tiborsiz qoldirish

```javascript
JSON.parse('{"__proto__": {"isAdmin": true}}');
// Agar bu ma'lumot ehtiyotsizlik bilan obyektga merge qilinsa,
// butun Object.prototype'ga zararli property qo'shilib ketishi mumkin
```
**Yechim:** Foydalanuvchi kiritgan ma'lumotlarni obyektga merge qilishda ehtiyot bo'lish, kerak bo'lsa `Object.create(null)` yoki `Map` ishlatish.

---

## 11. Interview savollari va qisqa javoblar

**S: Prototip zanjiri (prototype chain) nima?**
> J: Bu — JavaScript obyektlarining bir-biriga `[[Prototype]]` orqali bog'langan zanjiri. Agar obyektda biror property topilmasa, JS avtomatik ravishda uning prototipiga, keyin prototipning prototipiga qarab boradi — toki property topilguncha yoki zanjir `null`ga yetguncha.

**S: `prototype` va `__proto__` farqi nima?**
> J: `prototype` — faqat funksiyalarda mavjud bo'lgan property, `new` orqali yaratiluvchi instance'lar uchun "qolip" vazifasini bajaradi. `__proto__` esa har bir obyektda mavjud bo'lgan, uning haqiqiy meros zanjiridagi havolasi. Formula: `instance.__proto__ === Constructor.prototype`.

**S: `Object.create(null)` va `{}` orasidagi farq nima?**
> J: `{}` avtomatik ravishda `Object.prototype`ga bog'lanadi va shu sababli `toString`, `hasOwnProperty` kabi meros metodlarga ega bo'ladi. `Object.create(null)` esa hech qanday prototipga ega bo'lmagan (`__proto__ = null`) "sof" obyekt yaratadi — bunda hech qanday meros metod yo'q, bu esa uni "sof lug'at" sifatida yoki prototype pollution'dan himoyalanish uchun foydali qiladi.

**S: Nega metodlarni `this.method = function(){}` emas, `Prototype.method = function(){}` orqali qo'shish tavsiya etiladi?**
> J: Chunki `this` ichida yozilgan metod har bir instance uchun **alohida nusxada** yaratiladi (xotira isrofi), `prototype`ga qo'yilgan metod esa barcha instance'lar tomonidan **bitta marta yaratilib, ulashiladi** — bu xotirani tejaydi.

**S: `class` sintaksisi prototipdan tubdan farq qiladimi?**
> J: Yo'q. `class` — bu konstruktor funksiya + prototip mexanizmi ustiga qurilgan sintaktik shakar (syntactic sugar). `class` ichidagi metodlar "sahna orqasida" baribir `Konstruktor.prototype`ga qo'shiladi.

**S: `for...in` sikli bilan `Object.keys()` orasidagi farq nima?**
> J: `for...in` — obyektning **butun prototip zanjiri** bo'ylab barcha sanaladigan (enumerable) property'larni aylanadi (meros bo'lganlarini ham). `Object.keys()` esa faqat obyektning **o'zidagi** (own) property'larni qaytaradi.

---

## 12. Xulosa

> **Prototip zanjiri — JavaScript'ning meros (inheritance) tizimining yuragi.** Har bir obyekt boshqa bir obyektga (yoki `null`ga) ishora qiluvchi yashirin `[[Prototype]]`ga ega, va property qidiruvi shu zanjir bo'ylab yuqoriga qarab davom etadi.
>
> **`prototype`** — funksiyalarga tegishli "qolip" property, **`__proto__`** — har bir obyektning haqiqiy meros havolasi. Ikkalasi bog'liq, lekin bir xil narsa emas.
>
> **`Object.create(null)`** hech qanday prototipga ega bo'lmagan "sof" obyektlar yaratish imkonini beradi — bu ba'zi xavfsizlik va performance stsenariylarida foydali.
>
> **`class` sintaksisi** — bu yangi paradigma emas, balki tanish va tushunarli konstruktor funksiya + prototip mexanizmi ustiga qurilgan qulay qobiq.
>
> Bu mavzuni chuqur tushunish — nafaqat interview'da, balki JavaScript'ning ichki ishlash mantiqini, xotira samaradorligini va xavfsizlik masalalarini (prototype pollution kabi) to'g'ri anglash uchun ham muhim asos hisoblanadi.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
