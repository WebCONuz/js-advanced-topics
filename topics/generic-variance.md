# TypeScript Generic va Variance — To'liq Qo'llanma

> JavaScript/TypeScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [Generic nima va u qayerdan paydo bo'lgan?](#1-generic-nima-va-u-qayerdan-paydo-bolgan)
2. [Variance (o'zgaruvchanlik) nima?](#2-variance-ozgaruvchanlik-nima)
3. [Kovariant (Covariant)](#3-kovariant-covariant)
4. [Kontravariant (Contravariant)](#4-kontravariant-contravariant)
5. [Invariant va Bivariant](#5-invariant-va-bivariant)
6. [TypeScript'da variance qanday namoyon bo'ladi?](#6-typescriptda-variance-qanday-namoyon-boladi)
7. [`unknown` va `any` — amaliy farq](#7-unknown-va-any--amaliy-farq)
8. [Loyihaning qaysi qismlarida ishlatiladi?](#8-loyihaning-qaysi-qismlarida-ishlatiladi)
9. [Generic bilan ishlashning amaliy qoidalari](#9-generic-bilan-ishlashning-amaliy-qoidalari)
10. [Nega bu muhim — real foydalar](#10-nega-bu-muhim--real-foydalar)
11. [Eng ko'p uchraydigan xatolar](#11-eng-kop-uchraydigan-xatolar)
12. [Interview savollari va qisqa javoblar](#12-interview-savollari-va-qisqa-javoblar)
13. [Xulosa](#13-xulosa)

---

## 1. Generic nima va u qayerdan paydo bo'lgan?

**Generic** — bu tur (type)ni **parametr sifatida** qabul qiluvchi funksiya, interfeys yoki class yaratish imkonini beruvchi TypeScript mexanizmi. Bu tushuncha TypeScript'ga xos emas — u Java, C#, C++ (templates) kabi **statik tur tizimiga ega tillarning** barchasida mavjud bo'lgan, umumiy dasturlash (generic programming) paradigmasidan kelib chiqqan.

```typescript
interface Box<T> {
  value: T;
}

const numberBox: Box<number> = { value: 42 };
const stringBox: Box<string> = { value: 'salom' };
```

**Nega generic kerak?** Generic'siz, har xil tur uchun **alohida-alohida** interfeys yozishga to'g'ri kelardi:

```typescript
// Generic'siz - takrorlanuvchi, saqlab bo'lmaydigan kod
interface NumberBox { value: number; }
interface StringBox { value: string; }
interface BooleanBox { value: boolean; }
// ... har yangi tur uchun yana bir interfeys kerak bo'ladi!
```

Generic orqali esa bitta `Box<T>` — istalgan tur bilan ishlaydigan, **qayta ishlatiladigan** va shu bilan birga **tur xavfsizligini saqlaydigan** yechim beradi.

```typescript
function identity<T>(value: T): T {
  return value;
}

identity<number>(5);      // T = number
identity('salom');        // T avtomatik aniqlanadi = string (type inference)
```

---

## 2. Variance (o'zgaruvchanlik) nima?

Generic turlar bilan ishlaganda, muqarrar ravishda savol tug'iladi: agar bizda ikkita tur orasida ierarxik munosabat bo'lsa (masalan, `Dog` — `Animal`ning subtype'i), bu munosabat ularning **generic konteynerlariga** (`Box<Dog>` va `Box<Animal>`ga) ham o'tadimi?

**Variance** — aynan shu savolga javob beruvchi tur nazariyasi (type theory) tushunchasi bo'lib, u **generic turlar orasidagi ierarxiya, ularning ichidagi (parametr) turlar ierarxiyasi bilan qanday bog'liqligini** tavsiflaydi.

```
Agar Dog <: Animal (Dog - Animal'ning subtype'i) bo'lsa,
Box<Dog> va Box<Animal> orasidagi munosabat QANDAY bo'ladi?
```

Bu savolga javob — **variance turiga** bog'liq, va ularning 4 turi bor: **kovariant, kontravariant, invariant, bivariant**.

---

## 3. Kovariant (Covariant)

**Qoida:** Subtype munosabati **saqlanadi**.
```
Dog <: Animal  ⟹  Box<Dog> <: Box<Animal>
```

**Qachon kovariant bo'ladi?** Agar generic parametr faqat **"chiqish" (output)** pozitsiyasida ishlatilsa — ya'ni siz undan **faqat o'qiysiz**, unga hech narsa yozmaysiz.

```typescript
class Animal {
  name: string = 'hayvon';
}

class Dog extends Animal {
  bark() {
    console.log('Vov!');
  }
}

interface ReadonlyBox<T> {
  readonly value: T; // faqat o'qish uchun
}

let dogBox: ReadonlyBox<Dog> = { value: new Dog() };
let animalBox: ReadonlyBox<Animal> = dogBox; // ruxsat etiladi!

console.log(animalBox.value.name); // xavfsiz - Dog Animal'ning barcha xususiyatlariga ega
```

Bu — mantiqan to'g'ri: `animalBox` sifatida ishlatilganda, biz undan faqat `Animal`ga tegishli narsalarni o'qiymiz, va `Dog` obyekti bu talabni **to'liq qondiradi** (chunki `Dog` — bu "Animal + qo'shimcha xususiyatlar").

### Eng ko'p uchraydigan misol — massivlar

```typescript
let dogs: Dog[] = [new Dog()];
let animals: Animal[] = dogs; // TypeScript'da massivlar KOVARIANT!
```

---

## 4. Kontravariant (Contravariant)

**Qoida:** Subtype munosabati **teskari bo'ladi**.
```
Dog <: Animal  ⟹  Handler<Animal> <: Handler<Dog>
```

**Qachon kontravariant bo'ladi?** Agar generic parametr faqat **"kirish" (input)** pozitsiyasida ishlatilsa — ya'ni funksiya uni **parametr sifatida** qabul qiladi.

```typescript
type Handler<T> = (arg: T) => void;

let animalHandler: Handler<Animal> = (animal) => {
  console.log(animal.name);
};

let dogHandler: Handler<Dog> = animalHandler; // ruxsat etiladi!

dogHandler(new Dog()); // xavfsiz
```

**Nega bu mantiqiy?** `Handler<Dog>` — "`Dog` qabul qiladigan funksiya" degani. `animalHandler` esa **kengroq** — u `Animal`ga tegishli har qanday narsani (jumladan `Dog`ni ham) qabul qila oladi. Shuning uchun kengroq (parent) tur qabul qiluvchi funksiyani, torroq (child) tur talab qilingan joyda ishlatish **xavfsiz** — bu esa subtype yo'nalishini **teskari** qiladi.

### Real hayotdagi misol — event handler'lar

```typescript
class MouseEvent extends Event {
  x: number = 0;
  y: number = 0;
}

class ClickEvent extends MouseEvent {
  button: number = 0;
}

// Umumiy Event uchun handler - istalgan konkretroq event uchun ishlatilishi mumkin
let genericHandler: (e: Event) => void = (e) => console.log(e.type);
let clickHandler: (e: ClickEvent) => void = genericHandler; // xavfsiz - kontravariantlik
```

---

## 5. Invariant va Bivariant

### Invariant
Ikki generic tur o'rtasida **hech qanday** subtype munosabati mavjud emas — hatto ichidagi turlar bog'liq bo'lsa ham.

```typescript
// Ba'zi tillarda (masalan, Java'dagi generic massivlar) invariant bo'ladi
// TypeScript'da odatda bu holat kamdan-kam, lekin generic'larni ikkala
// yo'nalishda ham (input HAM output) ishlatilganda paydo bo'ladi
interface Container<T> {
  value: T;
  set(val: T): void; // T ham input, ham output sifatida ishlatilgan
}
```

Bunday holatlarda `Container<Dog>` va `Container<Animal>` orasida **xavfsiz** ravishda na kovariant, na kontravariant munosabat o'rnatib bo'lmaydi — chunki bittasi `set()`ni (kontravariant talab qiladi), ikkinchisi `value`ni o'qishni (kovariant talab qiladi) buzishi mumkin.

### Bivariant

Ikkala yo'nalishda ham "to'g'ri" deb qaraladi — bu **matematik jihatdan mantiqsiz**, lekin ba'zi amaliy tillarda (jumladan TypeScript'da, **metod sintaksisi** uchun) ataylab qo'llaniladigan murosa.

---

## 6. TypeScript'da variance qanday namoyon bo'ladi?

### 6.1. Massivlar va `readonly` property'lar — xavfsiz kovariantlik

```typescript
let dogs: Dog[] = [new Dog()];
let animals: Animal[] = dogs; // xavfsiz, agar faqat O'QISH uchun ishlatilsa
```

### 6.2. Mutable (o'zgaruvchan) property'lar — XAVFLI kovariantlik

TypeScript **strukturaviy tur tizimi** (structural typing) ishlatadi va ko'p holatlarda, hatto bu **nazariy jihatdan xavfsiz bo'lmasa ham**, kovariantlikka yo'l qo'yadi:

```typescript
interface Box<T> {
  value: T; // o'qish HAM, yozish HAM mumkin
}

let dogBox: Box<Dog> = { value: new Dog() };
let animalBox: Box<Animal> = dogBox; // TS ruxsat beradi... lekin XAVFLI!

animalBox.value = new Animal(); // "Animal" yozildi
console.log(dogBox.value.bark); // Runtime XATO bo'lishi mumkin - bark yo'q, chunki dogBox.value endi Animal!
```

Bu holat **"unsound"** (mantiqan to'liq to'g'ri emas) deb ataladi — TypeScript buni ataylab, amaliy qulaylik va JavaScript bilan moslik uchun qabul qilgan murosa sifatida qoldirgan.

### 6.3. Funksiya parametrlari — method vs function type farqi

Rasmiy tur nazariyasiga ko'ra, funksiya parametrlari **kontravariant** bo'lishi kerak. Lekin TypeScript'da bu ikki xil sintaksisda **turlicha** tekshiriladi:

**Metod sintaksisida — bivariant (kamroq qattiq):**

```typescript
interface Processor {
  process(animal: Animal): void; // metod sintaksisi
}

class DogProcessor implements Processor {
  process(dog: Dog): void {} // Dog - Animal'dan TORROQ, lekin TS ruxsat beradi!
}
```

**Funksiya-turi (property) sintaksisida, `strictFunctionTypes` yoqilganda — to'g'ri kontravariant:**

```typescript
interface Processor2 {
  process: (animal: Animal) => void; // arrow function turi sifatida
}

class DogProcessor2 implements Processor2 {
  process = (dog: Dog): void => {}; // XATO! strictFunctionTypes buni rad etadi
}
```

> **Interview uchun muhim jihat:** Bu farq TypeScript'ning tarixiy moslamasi — OOP meros zanjirlarida (masalan, `Array<T>`ning `forEach`, `map` kabi metodlarida) ko'proq moslashuvchanlikka ruxsat berish uchun ataylab qoldirilgan. `tsconfig.json`da `"strictFunctionTypes": true` sozlamasi funksiya-turi ifodalari uchun to'g'ri (sound) kontravariant tekshiruvni yoqadi.

---

## 7. `unknown` va `any` — amaliy farq

Bu ham generic/tur xavfsizligi mavzusining bir qismi — chunki ikkalasi ham "istalgan tur" bilan ishlashga imkon beradi, lekin butunlay boshqa darajadagi xavfsizlik bilan.

### `any` — tur tekshiruvidan **butunlay voz kechish**

```typescript
let value: any = 'salom';

value.toUpperCase();       // tekshiruv yo'q
value.nonExistentMethod(); // compile vaqtida XATO YO'Q!
value();                    // funksiyadek chaqirilsa ham xato bermaydi
```

### `unknown` — "noma'lum, lekin ishlatishdan oldin tekshir"

```typescript
let value: unknown = 'salom';

value.toUpperCase(); // XATO! "Object is of type 'unknown'"

if (typeof value === 'string') {
  value.toUpperCase(); // ENDI xavfsiz - TS value string ekanligini biladi
}
```

### Nega bu farq muhim?

```typescript
function processAny(data: any) {
  return data.value.toUpperCase(); 
  // compile vaqtida xato yo'q, lekin data = 5 bo'lsa - RUNTIME xatosi!
}

function processUnknown(data: unknown) {
  return data.value.toUpperCase(); 
  // COMPILE VAQTIDA XATO - TS bizni avval tekshirishga MAJBURLAYDI
}
```

### Taqqoslash jadvali

| | `any` | `unknown` |
|---|---|---|
| Turni tekshirish | Shart emas | Majburiy (narrowing yoki assertion) |
| Xavfsizlik | Xavfli - runtime xatolarga olib kelishi mumkin | Xavfsiz - compile vaqtida majburlaydi |
| Ishlatilish o'rni | Eski JS kodni migratsiya qilishda (vaqtinchalik) | Tashqi/noaniq ma'lumotlar (API javobi, `JSON.parse()`) |
| Tavsiya | Iloji boricha ishlatmaslik | `any` o'rniga har doim afzal |

### Amaliy misol

```typescript
// TO'G'RI YONDASHUV
async function fetchData(): Promise<unknown> {
  const res = await fetch('/api/data');
  return res.json();
}

const data = await fetchData();

if (
  typeof data === 'object' &&
  data !== null &&
  'user' in data
) {
  console.log((data as { user: { name: string } }).user.name);
}
```

---

## 8. Loyihaning qaysi qismlarida ishlatiladi?

| Soha | Misol |
|---|---|
| **API client'lar** | Generic `fetchData<T>(url: string): Promise<T>` — har xil endpoint uchun qayta ishlatiladigan funksiya |
| **State management (Redux, Zustand)** | Generic store'lar: `Store<State>`, `Action<Payload>` |
| **UI komponent kutubxonalari** | `List<T>` komponenti — har qanday tur elementlarini render qila oladigan komponent |
| **Utility funksiyalar** | `Array<T>`, `Map<K, V>`, `Promise<T>` kabi standart generic turlar |
| **Form validatsiyasi** | `Field<T>`, `FormState<Values>` kabi generic tur ta'riflari |
| **Callback/event handler tizimlari** | `EventEmitter<Events>` — kontravariantlik amalda ishlaydigan joy |
| **ORM/ma'lumotlar bazasi qatlamlari** | `Repository<Entity>` — har xil model uchun umumiy CRUD metodlari |

### Amaliy misol — generic API funksiyasi

```typescript
async function fetchData<T>(url: string): Promise<T> {
  const res = await fetch(url);
  return res.json() as T;
}

interface User {
  id: number;
  name: string;
}

const user = await fetchData<User>('/api/user/1');
console.log(user.name); // TS user.name string ekanligini biladi
```

---

## 9. Generic bilan ishlashning amaliy qoidalari

### 9.1. Generic constraint (`extends`) — parametrni cheklash

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K) {
  return obj[key];
}

const user = { name: 'Ali', age: 25 };
getProperty(user, 'name'); // OK
getProperty(user, 'unknown'); // XATO - 'unknown' user'da yo'q
```

### 9.2. Default generic qiymatlar

```typescript
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
}

const response: ApiResponse = { data: 'nimadir', status: 200 }; // T = unknown (default)
const typedResponse: ApiResponse<User> = { data: user, status: 200 };
```

### 9.3. Bir nechta generic parametr

```typescript
function merge<T, U>(obj1: T, obj2: U): T & U {
  return { ...obj1, ...obj2 };
}

const merged = merge({ name: 'Ali' }, { age: 25 }); // { name: string; age: number }
```

---

## 10. Nega bu muhim — real foydalar

1. **Kod qayta ishlatilishi (reusability)** — bitta generic funksiya/komponent ko'plab turlar uchun ishlaydi, kod takrorlanmaydi
2. **Tur xavfsizligi (type safety)** — `any` ishlatmasdan, har xil turdagi ma'lumotlar bilan ishlash mumkin, xatolar **compile vaqtida** aniqlanadi
3. **IDE yordami (IntelliSense)** — generic'lar orqali IDE avtomatik to'ldirish va tur tekshiruvini aniqroq taklif qiladi
4. **Variance'ni tushunish — xavfsiz API dizayni uchun** — funksiya parametrlarini to'g'ri joylashtirish (input/output pozitsiyalarini bilish) orqali runtime xatolarning oldini olish mumkin

---

## 11. Eng ko'p uchraydigan xatolar

### Xato 1: `any`ni "vaqtinchalik yechim" sifatida qoldirib ketish

```typescript
function processData(data: any) { // "keyin to'g'rilayman" deb yozilgan, lekin hech qachon to'g'rilanmaydi
  return data.value;
}
```
**Yechim:** Iloji boricha `unknown` yoki aniq generic tur ishlatish.

### Xato 2: Mutable generic property'lardagi kovariantlik xavfini bilmaslik

```typescript
function addAnimal(box: Box<Animal>) {
  box.value = new Cat(); // agar box aslida Box<Dog> bo'lsa - muammo!
}

const dogBox: Box<Dog> = { value: new Dog() };
addAnimal(dogBox); // TS bunga ruxsat berishi mumkin - lekin xavfli!
```
**Yechim:** Mumkin bo'lsa `readonly` ishlatish, yoki funksiya imzosini aniqroq generic bilan cheklash.

### Xato 3: Generic constraint'ni unutish

```typescript
function getLength<T>(item: T) {
  return item.length; // XATO - T'da length borligi kafolatlanmagan
}
```
**Yechim:**
```typescript
function getLength<T extends { length: number }>(item: T) {
  return item.length; // endi xavfsiz
}
```

---

## 12. Interview savollari va qisqa javoblar

**S: Generic nima va u nega kerak?**
> J: Generic — funksiya, interfeys yoki class'ga tur parametr sifatida berish imkonini beruvchi mexanizm. U kodni turli turlar uchun **qayta ishlatish** imkonini beradi, shu bilan birga `any`dan farqli o'laroq, **tur xavfsizligini saqlaydi**.

**S: Kovariant va kontravariant orasidagi farq nima?**
> J: Kovariantlikda subtype munosabati saqlanadi (`Dog <: Animal` ⟹ `Box<Dog> <: Box<Animal>`) — bu odatda faqat **o'qish** (output) pozitsiyalarida xavfsiz. Kontravariantlikda esa bu munosabat **teskari** bo'ladi — funksiya parametrlarida (input pozitsiyasida) kuzatiladi: kengroq turni qabul qiluvchi funksiya, torroq tur talab qilingan joyda ishlatilishi mumkin.

**S: TypeScript'da massivlar kovariantmi?**
> J: Ha, TypeScript'da massivlar (va umuman ko'p mutable strukturalar) kovariant hisoblanadi — bu amaliy qulaylik uchun qilingan, lekin bu ba'zan **runtime xatolarga** olib kelishi mumkin bo'lgan "unsound" (mantiqan to'liq to'g'ri bo'lmagan) yechim.

**S: `strictFunctionTypes` nima uchun kerak?**
> J: U funksiya-turi (property/arrow function) ifodalarida to'g'ri (sound) kontravariant tekshiruvni yoqadi. Bu sozlama yoqilmasa, TypeScript funksiya parametrlarini bivariant deb tekshiradi — bu esa xavfsizlikni kamaytiradi. Metod sintaksisiga bu sozlama ta'sir qilmaydi (moslashuvchanlik uchun ataylab shunday qilingan).

**S: `unknown` va `any` orasidagi asosiy farq nima?**
> J: `any` — tur tekshiruvini butunlay o'chiradi, hech qanday xavfsizlik kafolati bermaydi. `unknown` esa qiymatni ishlatishdan oldin majburiy tur tekshiruvini (narrowing yoki assertion) talab qiladi, shuning uchun u har doim `any`dan xavfsizroq.

**S: Qachon `unknown` ishlatish kerak?**
> J: Tashqi manbadan keladigan (API javobi, `JSON.parse()`, foydalanuvchi kiritmasi) noaniq turdagi ma'lumotlarni ifodalashda — bu TypeScript'ni ma'lumot ishlatilishidan oldin tur tekshiruvini majburlashga undaydi, `any`dan farqli o'laroq.

---

## 13. Xulosa

> **Generic — TypeScript'ning eng kuchli xususiyatlaridan biri**, u kodni qayta ishlatilishi mumkin bo'lgan va shu bilan birga tur xavfsiz qilib yozish imkonini beradi. Uning ichida yashirin, lekin muhim tushuncha — **variance**: generic turlar orasidagi ierarxiya ularning parametr turlari ierarxiyasi bilan qanday bog'lanishi.
>
> **Kovariantlik** — faqat "chiqish" (o'qish) pozitsiyalarida xavfsiz, **kontravariantlik** — "kirish" (funksiya parametri) pozitsiyalarida namoyon bo'ladi. TypeScript, strukturaviy tur tizimi va JavaScript bilan moslik saqlash maqsadida, ba'zi joylarda (mutable property'lar, metod sintaksisi) **ataylab xavfsiz bo'lmagan (unsound) murosalarga** yo'l qo'yadi.
>
> **`unknown`** — `any`ning xavfsiz muqobili bo'lib, u tashqi va noaniq ma'lumotlar bilan ishlashda TypeScript'ning tur tekshiruvidan to'liq foydalanish imkonini beradi.
>
> Bu mavzularni chuqur tushunish — nafaqat interview'da, balki katta va murakkab TypeScript loyihalarida xavfsiz, ishonchli va oson qo'llab-quvvatlanadigan API'lar loyihalashda ham muhim asos bo'lib xizmat qiladi.

---

*Tayyorlandi: JS/TS texnik intervyu tayyorgarligi uchun*
