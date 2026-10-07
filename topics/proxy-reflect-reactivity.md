# Proxy va Reflect API — Reaktiv Tizimlar, Validation va Logging — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.
> Asosiy fokus: `Proxy`, `Reflect`, Vue 3 / MobX reaktivligi qanday ishlashi, validation va logging proxy'lari.

---

## Mundarija

1. [Proxy nima va u qayerdan paydo bo'lgan?](#1-proxy-nima-va-u-qayerdan-paydo-bolgan)
2. [Proxy asoslari: target, handler, trap](#2-proxy-asoslari-target-handler-trap)
3. [Barcha trap'lar jadvali](#3-barcha-traplar-jadvali)
4. [Reflect nima va nega kerak?](#4-reflect-nima-va-nega-kerak)
5. [Receiver muammosi — Reflect nega shart](#5-receiver-muammosi--reflect-nega-shart)
6. [Invariant'lar — Proxy'ning "qizil chiziqlari"](#6-invariantlar--proxyning-qizil-chiziqlari)
7. [Amaliy pattern'lar: validation, logging va boshqalar](#7-amaliy-patternlar-validation-logging-va-boshqalar)
8. [Reaktiv tizimlar qanday ishlaydi?](#8-reaktiv-tizimlar-qanday-ishlaydi)
9. [Noldan mini reaktivlik tizimi (Vue 3 uslubida)](#9-noldan-mini-reaktivlik-tizimi-vue-3-uslubida)
10. [Vue 2 vs Vue 3 — `defineProperty` vs `Proxy`](#10-vue-2-vs-vue-3--defineproperty-vs-proxy)
11. [MobX, Immer va boshqa kutubxonalar](#11-mobx-immer-va-boshqa-kutubxonalar)
12. [Cheklovlar va tuzoqlar](#12-cheklovlar-va-tuzoqlar)
13. [Loyihaning qaysi qismlarida ishlatiladi?](#13-loyihaning-qaysi-qismlarida-ishlatiladi)
14. [Nega bu muhim — real foydalar](#14-nega-bu-muhim--real-foydalar)
15. [Eng ko'p uchraydigan xatolar](#15-eng-kop-uchraydigan-xatolar)
16. [Interview savollari va qisqa javoblar](#16-interview-savollari-va-qisqa-javoblar)
17. [Xulosa](#17-xulosa)

---

## 1. Proxy nima va u qayerdan paydo bo'lgan?

**Proxy** — ES2015 (ES6) standartida kiritilgan, boshqa obyekt (**target**) atrofida "qobiq" yaratib, shu obyekt ustidagi **fundamental operatsiyalarni** (o'qish, yozish, o'chirish, `in`, funksiya chaqiruvi, `new` va h.k.) **ushlab olish va qayta aniqlash** imkonini beruvchi o'rnatilgan obyekt.

```javascript
const target = { name: 'Ali' };

const proxy = new Proxy(target, {
  get(target, key) {
    console.log(`"${String(key)}" o'qildi`);
    return target[key];
  },
});

proxy.name; // "name" o'qildi → 'Ali'
```

Oddiy so'z bilan: Proxy — obyekt oldidagi **"darvozabon"**. Obyektga kirayotgan har bir murojaat avval darvozabondan o'tadi, u esa ruxsat berishi, o'zgartirishi, yozib qo'yishi yoki rad etishi mumkin.

### Bu tushuncha qayerdan kelib chiqqan?

Proxy — **metadasturlash (metaprogramming)** vositasi: kod boshqa kodning (bu yerda — obyektning) xatti-harakatini boshqaradi.

**ES2015 gacha vaziyat:**

- JavaScript'da obyektning o'qish/yozish xatti-harakatini ushlash uchun faqat **`Object.defineProperty`** (getter/setter) bor edi. Lekin u **faqat oldindan ma'lum, mavjud property**lar bilan ishlaydi — yangi qo'shilgan yoki o'chirilgan property'ni **ko'rmaydi**.
- 2012–2015 yillarda `Object.observe` taklif qilingan edi, lekin u standartlashtirilmadi va 2015-yilda bekor qilindi — uning o'rnini aynan Proxy egalladi.
- Proxy dizayni akademik tadqiqotlar (Tom Van Cutsem va Mark Miller kabi mualliflarning "direct proxies" g'oyasi) asosida shakllangan.

**Natija:** ilgari faqat tilning ichki mexanizmlariga (masalan, massivning `length`i yoki `arguments` obyekti) tegishli bo'lgan "sehrli" xatti-harakatni endi dasturchi **o'z obyektlarida** yarata oladigan bo'ldi.

### Reaktivlik tarixi (nega Proxy ahamiyatli)

| Yondashuv | Qayerda | Mohiyati |
|---|---|---|
| **Dirty checking** | AngularJS (1.x) | Muntazam ravishda barcha qiymatlarni "oldingisi bilan" solishtirish |
| **Observable funksiyalar** | Knockout | `name()` / `name('yangi')` — qo'lda wrapper |
| **`Object.defineProperty`** | Vue 2, MobX 4 | Har bir mavjud property'ga getter/setter o'rnatish |
| **`Proxy`** | Vue 3, MobX 5+, Svelte 5, Valtio, Immer | Butun obyektni "ushlash" — qo'shish/o'chirish ham ko'rinadi |
| **Immutable + `setState`** | React | Reaktivlik yo'q; yangi qiymat berib, qayta render so'raladi |

---

## 2. Proxy asoslari: target, handler, trap

```javascript
const proxy = new Proxy(target, handler);
```

| Qism | Ma'nosi |
|---|---|
| **target** | "Asl" obyekt (massiv, funksiya, class, boshqa Proxy ham bo'lishi mumkin) |
| **handler** | Trap'lar (ushlagich metodlar) saqlanadigan obyekt |
| **trap** | Muayyan operatsiyani ushlab oladigan handler metodi (`get`, `set`, `has` va h.k.) |

Agar handler'da biror trap **yo'q** bo'lsa, shu operatsiya **to'g'ridan-to'g'ri target'ga yo'naltiriladi** (forward qilinadi):

```javascript
const target = { a: 1 };
const proxy = new Proxy(target, {}); // bo'sh handler — "shaffof" proxy

proxy.b = 2;
console.log(target.b); // 2 — yozuv target'ga o'tdi
```

### Asosiy uchta trap

```javascript
const handler = {
  // proxy.key o'qilganda
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver);
  },

  // proxy.key = value yozilganda
  set(target, key, value, receiver) {
    return Reflect.set(target, key, value, receiver); // true/false qaytarish SHART
  },

  // delete proxy.key
  deleteProperty(target, key) {
    return Reflect.deleteProperty(target, key);
  },
};
```

### `set` trap — `true` qaytarish shart

```javascript
'use strict';
const p = new Proxy({}, {
  set() {
    return false; // "yozish muvaffaqiyatsiz" degani
  },
});

p.x = 1; // TypeError: 'set' on proxy: trap returned falsish for property 'x'
```

- **Strict mode** (ES modullar, class ichi, `'use strict'`) da `false` qaytarilsa — **`TypeError`**.
- Sloppy mode'da yozuv **jimgina e'tiborsiz qoldiriladi**.
- Muvaffaqiyatli bo'lsa **`true`** qaytarish kerak. Bu — eng ko'p uchraydigan xatolardan biri (15-bo'limga qarang).

---

## 3. Barcha trap'lar jadvali

Jami **13 ta** trap bor. Ularning har biri uchun **bir xil nomli `Reflect` metodi** mavjud.

| Trap | Qaysi operatsiyani ushlaydi | Mos `Reflect` metodi |
|---|---|---|
| `get(target, key, receiver)` | `proxy.key`, `proxy[key]` | `Reflect.get` |
| `set(target, key, value, receiver)` | `proxy.key = v` | `Reflect.set` |
| `has(target, key)` | `key in proxy` | `Reflect.has` |
| `deleteProperty(target, key)` | `delete proxy.key` | `Reflect.deleteProperty` |
| `ownKeys(target)` | `Object.keys`, `for...in`, `Object.getOwnPropertyNames`, spread, `JSON.stringify` | `Reflect.ownKeys` |
| `getOwnPropertyDescriptor(target, key)` | `Object.getOwnPropertyDescriptor`, `Object.keys` (enumerable tekshiruvi) | `Reflect.getOwnPropertyDescriptor` |
| `defineProperty(target, key, desc)` | `Object.defineProperty` | `Reflect.defineProperty` |
| `getPrototypeOf(target)` | `Object.getPrototypeOf`, `instanceof` | `Reflect.getPrototypeOf` |
| `setPrototypeOf(target, proto)` | `Object.setPrototypeOf` | `Reflect.setPrototypeOf` |
| `isExtensible(target)` | `Object.isExtensible` | `Reflect.isExtensible` |
| `preventExtensions(target)` | `Object.preventExtensions` | `Reflect.preventExtensions` |
| `apply(target, thisArg, args)` | `proxy(...)`, `.call`, `.apply` (target — funksiya bo'lsa) | `Reflect.apply` |
| `construct(target, args, newTarget)` | `new proxy(...)` (target — konstruktor bo'lsa) | `Reflect.construct` |

> **Muhim:** `apply` va `construct` trap'lari faqat **target funksiya bo'lganda** ishlaydi.

### Bir operatsiya — bir nechta trap

`Object.keys(proxy)` ichkarida **ikkita** trap'ni chaqiradi: avval `ownKeys` (kalitlar ro'yxati), keyin har bir kalit uchun `getOwnPropertyDescriptor` (enumerable ekanini tekshirish uchun).

```javascript
const p = new Proxy({ a: 1, b: 2 }, {
  ownKeys(t) { console.log('ownKeys'); return Reflect.ownKeys(t); },
  getOwnPropertyDescriptor(t, k) {
    console.log('descriptor:', k);
    return Reflect.getOwnPropertyDescriptor(t, k);
  },
});

Object.keys(p);
// ownKeys
// descriptor: a
// descriptor: b
```

---

## 4. Reflect nima va nega kerak?

**Reflect** — ES2015'da Proxy bilan **birga** kiritilgan, **konstruktor emas** (`new Reflect()` ishlamaydi), statik metodlardan iborat o'rnatilgan obyekt (xuddi `Math` kabi). Uning metodlari Proxy trap'lari bilan **bir-biriga mos (1:1)**.

```javascript
Reflect.get(obj, 'name');              // obj.name
Reflect.set(obj, 'name', 'Vali');      // obj.name = 'Vali'   → true/false
Reflect.has(obj, 'name');              // 'name' in obj
Reflect.deleteProperty(obj, 'name');   // delete obj.name     → true/false
Reflect.ownKeys(obj);                  // string + symbol kalitlari (non-enumerable ham)
Reflect.apply(fn, thisArg, args);      // fn.apply(thisArg, args)
Reflect.construct(Cls, args);          // new Cls(...args)
```

### Reflect nima uchun kerak? — 4 ta sabab

**1. Operatorlarning funksiya shakli.** `in`, `delete`, `new` — operator. `Reflect.has`, `Reflect.deleteProperty`, `Reflect.construct` — ularni **funksiya** sifatida chaqirish va uzatish imkonini beradi.

**2. Xato tashlash o'rniga `boolean` qaytaradi.**

```javascript
const frozen = Object.freeze({ x: 1 });

// Object.defineProperty — XATO TASHLAYDI
try {
  Object.defineProperty(frozen, 'y', { value: 2 });
} catch (e) {
  console.log(e.message); // Cannot define property y, object is not extensible
}

// Reflect.defineProperty — shunchaki false qaytaradi
console.log(Reflect.defineProperty(frozen, 'y', { value: 2 })); // false
```

Trap'lar ichida bu qulay: natijani to'g'ridan-to'g'ri qaytarish mumkin (`return Reflect.set(...)`).

**3. `receiver` parametri** — `this` ni to'g'ri uzatish (keyingi bo'limda — eng muhim sabab).

**4. Trap'ning "standart xatti-harakati"ni chaqirish.** Trap ichida "o'z mantig'imni qo'shaman, keyin odatdagidek davom etaman" deyish uchun `Reflect.xxx(...)` ishlatiladi:

```javascript
set(target, key, value, receiver) {
  console.log('yozilmoqda:', key);
  return Reflect.set(target, key, value, receiver); // standart xatti-harakat
}
```

> **Eslatma:** `target[key] = value` ham ishlaydi, lekin `Reflect` — to'g'riroq, chunki `receiver`ni, qaytarish qiymatini va maxsus holatlarni (getter/setter, prototip zanjiri) **spetsifikatsiyaga mos** boshqaradi.

### `Object.xxx` vs `Reflect.xxx` farqi (qisqa)

| | `Object.defineProperty` | `Reflect.defineProperty` |
|---|---|---|
| Muvaffaqiyatsizlikda | `TypeError` tashlaydi | `false` qaytaradi |
| Muvaffaqiyatli bo'lsa | O'sha obyektni qaytaradi | `true` qaytaradi |
| `Reflect.ownKeys` vs `Object.keys` | — | `ownKeys` — **barcha** kalitlar (symbol va non-enumerable ham), `Object.keys` — faqat enumerable string'lar |

---

## 5. Receiver muammosi — Reflect nega shart

Bu — Proxy/Reflect bo'yicha **eng nozik va eng ko'p so'raladigan** jihat. Quyidagi misolni ko'ring:

```javascript
const user = {
  _name: 'Ali',
  get name() {
    return this._name;   // getter ichida `this` ishlatilgan
  },
};
```

### Variant A: `target[key]` bilan (noto'g'ri)

```javascript
const proxyA = new Proxy(user, {
  get(target, key) {
    console.log('o\'qildi:', key);
    return target[key];            // getter chaqirilganda this = target (asl obyekt)
  },
});

proxyA.name;
// o'qildi: name
// (this._name proxy'dan o'tmadi — "_name" o'qilgani KO'RINMAYDI)
```

### Variant B: `Reflect.get(..., receiver)` bilan (to'g'ri)

```javascript
const proxyB = new Proxy(user, {
  get(target, key, receiver) {
    console.log('o\'qildi:', key);
    return Reflect.get(target, key, receiver);  // getter'da this = receiver (proxy)
  },
});

proxyB.name;
// o'qildi: name
// o'qildi: _name     ← getter ichidagi this._name ham proxy orqali o'tdi!
```

### Receiver nima?

`receiver` — operatsiya **dastlab qaysi obyektga nisbatan** chaqirilgani (odatda **proxy'ning o'zi**, yoki proxy'dan meros olgan obyekt). `Reflect.get(target, key, receiver)` getter'ni `this = receiver` bilan chaqiradi.

### Nega bu reaktivlikda hal qiluvchi?

Vue 3'da `computed` va getter'lar reaktiv obyekt ichida ishlatiladi. Agar getter ichidagi `this._name` **proxy orqali o'tmasa**, Vue `_name` ni **kuzata olmaydi** — ya'ni `_name` o'zgarganda `name` ga bog'liq hech narsa yangilanmaydi. Shuning uchun Vue ichida **doim** `Reflect.get(target, key, receiver)` ishlatiladi.

> **Interview uchun eng muhim jumla:** Proxy trap ichida `target[key]` o'rniga `Reflect.get(target, key, receiver)` ishlatilishining sababi — getter/setter ichidagi `this` **proxy'ga bog'lanishi** kerak. Aks holda getter ichidagi ichki murojaatlar proxy'dan "aylanib o'tib", ushlanmay qoladi.

---

## 6. Invariant'lar — Proxy'ning "qizil chiziqlari"

Proxy hamma narsani o'zgartira olmaydi. JS dvigateli handler natijasini **target'ning haqiqiy holatiga qarshi** tekshiradi. Agar trap "yolg'on" gapirsa — **`TypeError`**.

```javascript
const target = {};
Object.defineProperty(target, 'fixed', {
  value: 1,
  writable: false,
  configurable: false,   // o'zgartirib ham, o'chirib ham bo'lmaydi
});

const p = new Proxy(target, {
  get() { return 2; },   // "yolg'on" gapirmoqchi
});

p.fixed;
// TypeError: 'get' on proxy: property 'fixed' is a read-only and
// non-configurable data property on the proxy target but the proxy
// did not return its actual value
```

**Asosiy invariant'lar:**

| Trap | Qoida |
|---|---|
| `get` | Target'dagi `writable: false` + `configurable: false` property uchun **aynan o'sha qiymat** qaytarilishi kerak |
| `set` | Bunday property'ga boshqa qiymat yozilgan deb `true` qaytarib bo'lmaydi |
| `deleteProperty` | `configurable: false` property'ni "o'chirdim" deb `true` qaytarib bo'lmaydi |
| `has` | Non-configurable property'ni "yo'q" deb bo'lmaydi |
| `ownKeys` | Natijada target'ning barcha non-configurable kalitlari bo'lishi, takrorlanish bo'lmasligi, faqat string/symbol bo'lishi shart |
| `getPrototypeOf` | Target kengaytirib bo'lmaydigan (non-extensible) bo'lsa — haqiqiy prototipni qaytarish shart |

Maqsad: **tilning asosiy kafolatlari buzilmasligi** (masalan, `Object.freeze` qilingan obyekt "qotgan" bo'lib qolishi).

---

## 7. Amaliy pattern'lar: validation, logging va boshqalar

### 7.1. Validation Proxy — yozishdan oldin tekshirish

```javascript
function createValidated(target, schema) {
  return new Proxy(target, {
    set(target, key, value, receiver) {
      const validator = schema[key];

      if (!validator) {
        throw new Error(`Noma'lum maydon: "${String(key)}"`);
      }
      if (!validator(value)) {
        throw new TypeError(`"${String(key)}" uchun noto'g'ri qiymat: ${value}`);
      }
      return Reflect.set(target, key, value, receiver);
    },
  });
}

const user = createValidated({}, {
  name: (v) => typeof v === 'string' && v.length >= 2,
  age: (v) => Number.isInteger(v) && v >= 0 && v <= 150,
});

user.name = 'Ali';   // OK
user.age = 25;       // OK
user.age = -5;       // TypeError: "age" uchun noto'g'ri qiymat: -5
user.email = 'x';    // Error: Noma'lum maydon: "email"
```

**Afzalligi:** validatsiya mantiqi **bitta joyda**, obyektni ishlatayotgan kod esa oddiy `user.age = 25` yozadi. Bu — **tashqi manbadan kelgan** (forma, API) ma'lumotlarni himoyalash uchun qulay.

### 7.2. Logging / Tracing Proxy

```javascript
function withLogging(target, name = 'obj') {
  return new Proxy(target, {
    get(target, key, receiver) {
      const value = Reflect.get(target, key, receiver);

      // Symbol kalitlarni (Symbol.toPrimitive, Symbol.iterator ...) logga yozmaymiz
      if (typeof key === 'symbol') return value;

      // Metod chaqiruvlarini kuzatish
      if (typeof value === 'function') {
        return function (...args) {
          console.log(`${name}.${key}(${args.join(', ')}) chaqirildi`);
          return value.apply(this, args);
        };
      }

      console.log(`GET ${name}.${key} →`, value);
      return value;
    },

    set(target, key, value, receiver) {
      console.log(`SET ${name}.${String(key)} =`, value);
      return Reflect.set(target, key, value, receiver);
    },

    deleteProperty(target, key) {
      console.log(`DELETE ${name}.${String(key)}`);
      return Reflect.deleteProperty(target, key);
    },
  });
}

const cart = withLogging({ items: [], total: 0, add(x) { this.items.push(x); } }, 'cart');

cart.total = 100;       // SET cart.total = 100
cart.total;             // GET cart.total → 100
cart.add('olma');       // cart.add(olma) chaqirildi
delete cart.total;      // DELETE cart.total
```

> **Eslatma (oldingi mavzu bilan bog'liq):** log satrida `${key}` ni to'g'ridan-to'g'ri template literal ichida **symbol** bilan ishlatish `TypeError` beradi (symbol implicit string'ga aylanmaydi). Shuning uchun `String(key)` ishlatiladi yoki symbol'lar oldindan filtrlanadi.

> **Eslatma:** `console.log(proxy)` yoki `JSON.stringify(proxy)` ham `get` trap'ni chaqiradi (`toJSON`, `Symbol.toPrimitive`, `constructor` kabi kalitlar uchun) — logging proxy'da bu "shovqin"ni hisobga oling.

### 7.3. Default qiymatlar (mavjud bo'lmagan kalit uchun)

```javascript
function withDefault(target, defaultValue) {
  return new Proxy(target, {
    get(target, key, receiver) {
      if (typeof key === 'string' && !(key in target)) return defaultValue;
      return Reflect.get(target, key, receiver);
    },
  });
}

const counts = withDefault({}, 0);
counts.olma++;          // get → 0, set → 1
counts.olma++;          // 2
console.log(counts.nok); // 0 (hech qachon o'rnatilmagan)
```

So'zlarni sanash, guruhlash (`groupBy`) kabi masalalarda `if (!obj[k]) obj[k] = 0` takrorlanishini yo'qotadi.

### 7.4. Manfiy indeksli massiv (Python uslubida)

```javascript
function negativeIndex(arr) {
  return new Proxy(arr, {
    get(target, key, receiver) {
      if (typeof key === 'string' && /^-\d+$/.test(key)) {
        key = String(target.length + Number(key));
      }
      return Reflect.get(target, key, receiver);
    },
  });
}

const list = negativeIndex([10, 20, 30, 40]);
console.log(list[-1]); // 40
console.log(list[-2]); // 30
console.log(list[0]);  // 10
```

(Zamonaviy muqobil: `arr.at(-1)` — ES2022.)

### 7.5. Read-only (o'zgarmas) proxy — chuqur

```javascript
function readonly(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      const value = Reflect.get(target, key, receiver);
      // Ichki obyektlarni ham o'rab chiqamiz (lazy deep)
      return typeof value === 'object' && value !== null ? readonly(value) : value;
    },
    set(target, key) {
      throw new Error(`"${String(key)}" ni o'zgartirib bo'lmaydi (read-only)`);
    },
    deleteProperty(target, key) {
      throw new Error(`"${String(key)}" ni o'chirib bo'lmaydi (read-only)`);
    },
  });
}

const config = readonly({ db: { host: 'localhost', port: 5432 } });

console.log(config.db.host);  // 'localhost'
config.db.port = 1;           // Error: "port" ni o'zgartirib bo'lmaydi (read-only)
```

Vue 3'dagi `readonly()` va `shallowReadonly()` xuddi shu g'oyaga asoslangan (faqat u xato o'rniga ogohlantirish chiqaradi).

### 7.6. "Private" maydonlarni yashirish (`_` prefiksi)

```javascript
const isPrivate = (key) => typeof key === 'string' && key.startsWith('_');

function hidePrivate(target) {
  return new Proxy(target, {
    get: (t, k, r) => (isPrivate(k) ? undefined : Reflect.get(t, k, r)),
    set: (t, k, v, r) => {
      if (isPrivate(k)) throw new Error(`"${k}" — private maydon`);
      return Reflect.set(t, k, v, r);
    },
    has: (t, k) => !isPrivate(k) && Reflect.has(t, k),
    ownKeys: (t) => Reflect.ownKeys(t).filter((k) => !isPrivate(k)),
    deleteProperty: (t, k) => {
      if (isPrivate(k)) throw new Error(`"${k}" — private maydon`);
      return Reflect.deleteProperty(t, k);
    },
  });
}

const account = hidePrivate({ owner: 'Ali', _pin: 1234 });

console.log(account._pin);          // undefined
console.log('_pin' in account);     // false
console.log(Object.keys(account));  // ['owner']
console.log(JSON.stringify(account)); // '{"owner":"Ali"}'
```

> **Eslatma:** bu — **qulaylik/konvensiya**, haqiqiy xavfsizlik emas: asl `target` obyektiga havola bo'lsa, undan hamma narsani o'qish mumkin. Haqiqiy maxfiylik uchun `#private` maydonlar yoki closure ishlatiladi.

### 7.7. Funksiya proxy'lari — `apply` va `construct`

**`apply` — memoization** (oldingi "Memoization" hujjati bilan bog'liq):

```javascript
function memoizeProxy(fn) {
  const cache = new Map();

  return new Proxy(fn, {
    apply(target, thisArg, args) {
      const key = JSON.stringify(args);
      if (cache.has(key)) return cache.get(key);

      const result = Reflect.apply(target, thisArg, args);
      cache.set(key, result);
      return result;
    },
  });
}

const slowSquare = (n) => { for (let i = 0; i < 1e8; i++); return n * n; };
const fastSquare = memoizeProxy(slowSquare);

fastSquare(9); // sekin (birinchi marta)
fastSquare(9); // darhol (keshdan)
```

**`construct` — Singleton:**

```javascript
function singleton(Class) {
  let instance;
  return new Proxy(Class, {
    construct(target, args, newTarget) {
      return (instance ??= Reflect.construct(target, args, newTarget));
    },
  });
}

class Database { constructor() { this.id = Math.random(); } }
const DB = singleton(Database);

console.log(new DB() === new DB()); // true — har doim bitta instance
```

### 7.8. Bekor qilinadigan proxy — `Proxy.revocable`

```javascript
const { proxy, revoke } = Proxy.revocable({ secret: 42 }, {});

console.log(proxy.secret); // 42

revoke(); // ruxsat bekor qilindi

proxy.secret; // TypeError: Cannot perform 'get' on a proxy that has been revoked
```

Vaqtinchalik, muddatli kirish (access) berish uchun foydali: kodning boshqa qismiga obyektni berasiz, vaqti tugagach `revoke()` qilib, **undan keyingi barcha murojaatlarni to'xtatasiz**.

---

## 8. Reaktiv tizimlar qanday ishlaydi?

**Reaktivlik** — ma'lumot o'zgarganda, **unga bog'liq barcha narsa avtomatik yangilanishi**. Masalan: `state.count` o'zgarsa — ekrandagi matn ham, hisoblangan qiymatlar ham yangilanadi.

Har qanday reaktiv tizim **uchta savolga** javob berishi kerak:

```
1. Kim nimani O'QIDI?       →  KUZATISH (track)
2. Nima O'ZGARDI?           →  ANIQLASH (Proxy trap: set / deleteProperty)
3. Kim yangilanishi kerak?  →  CHAQIRISH (trigger)
```

### Asosiy g'oya — qadamlar

```
┌─────────────────────────────────────────────────────────────┐
│ 1. effect(() => render(state.count))  ← effect ishga tushadi │
│                                                               │
│ 2. Effect ichida `state.count` O'QILADI                       │
│       → Proxy `get` trap ishga tushadi                        │
│       → track(): "joriy effect `count` ga bog'liq" deb       │
│                  ro'yxatga yozib qo'yiladi                    │
│                                                               │
│ 3. Keyin `state.count++` yoziladi                             │
│       → Proxy `set` trap ishga tushadi                        │
│       → trigger(): `count` ga bog'liq effect'lar topiladi     │
│                    va QAYTA ISHGA TUSHIRILADI                 │
└─────────────────────────────────────────────────────────────┘
```

### Ma'lumotlar tuzilmasi — bog'liqliklar xaritasi

```
targetMap : WeakMap
   └── target obyekt  →  depsMap : Map
                            └── key (masalan 'count')  →  dep : Set
                                                              └── effect1, effect2, ...
```

- **`WeakMap`** — kalit sifatida **obyekt** (target) olinadi va unga **zaif havola** saqlanadi: target obyekt boshqa hech qayerda ishlatilmay qolsa, uning bog'liqliklari ham **avtomatik tozalanadi** (memory leak yo'q). Bu — "Closure va Xotira" hujjatida ko'rgan `WeakMap` afzalligining **real, katta miqyosdagi** qo'llanilishi.
- **`Map`** — property nomi → effect'lar to'plami.
- **`Set`** — bir xil effect ikki marta yozilmasligi uchun.

---

## 9. Noldan mini reaktivlik tizimi (Vue 3 uslubida)

Quyida Vue 3'ning asosiy g'oyasini aks ettiruvchi, ~60 qatorli **o'quv (soddalashtirilgan)** implementatsiya.

### 9.1. `track`, `trigger`, `effect`

```javascript
const targetMap = new WeakMap();          // target → Map(key → Set(effect))
const ITERATE_KEY = Symbol('iterate');    // Object.keys / for...in uchun maxsus kalit
let activeEffect = null;

function track(target, key) {
  if (!activeEffect) return;              // effect tashqarisida o'qilsa — kuzatmaymiz

  let depsMap = targetMap.get(target);
  if (!depsMap) targetMap.set(target, (depsMap = new Map()));

  let dep = depsMap.get(key);
  if (!dep) depsMap.set(key, (dep = new Set()));

  dep.add(activeEffect);
}

function trigger(target, key) {
  const dep = targetMap.get(target)?.get(key);
  if (!dep) return;

  [...dep].forEach((eff) => {             // nusxa olamiz — iteratsiya paytida Set o'zgarishi mumkin
    if (eff === activeEffect) return;     // effect o'zini o'zi cheksiz chaqirmasin
    eff.scheduler ? eff.scheduler(eff) : eff();
  });
}

function effect(fn, options = {}) {
  const run = () => {
    const prev = activeEffect;
    activeEffect = run;                   // "hozir shu effect ishlayapti"
    try {
      return fn();
    } finally {
      activeEffect = prev;                // ichma-ich effect'lar uchun tiklash
    }
  };
  run.scheduler = options.scheduler;
  run();                                  // darhol bir marta ishga tushadi
  return run;
}
```

### 9.2. `reactive` — Proxy bilan o'rash

```javascript
const reactiveCache = new WeakMap();      // target → proxy (bir obyektga bitta proxy)
const hasOwn = (obj, key) => Object.prototype.hasOwnProperty.call(obj, key);

function reactive(target) {
  if (typeof target !== 'object' || target === null) return target;
  if (reactiveCache.has(target)) return reactiveCache.get(target);

  const proxy = new Proxy(target, {
    get(target, key, receiver) {
      const result = Reflect.get(target, key, receiver);
      track(target, key);                              // kim o'qidi — yozib qo'yamiz
      return reactive(result);                         // ichki obyekt ham reaktiv (LAZY chuqur)
    },

    set(target, key, value, receiver) {
      const hadKey = hasOwn(target, key);
      const oldValue = target[key];
      const result = Reflect.set(target, key, value, receiver);

      if (!hadKey) {
        trigger(target, key);                          // yangi property qo'shildi
        trigger(target, ITERATE_KEY);                  // kalitlar ro'yxati o'zgardi
      } else if (!Object.is(oldValue, value)) {
        trigger(target, key);                          // faqat haqiqatan o'zgarsa
      }
      return result;
    },

    deleteProperty(target, key) {
      const had = hasOwn(target, key);
      const result = Reflect.deleteProperty(target, key);
      if (had && result) {
        trigger(target, key);
        trigger(target, ITERATE_KEY);
      }
      return result;
    },

    has(target, key) {
      track(target, key);                              // `'x' in state` ham kuzatiladi
      return Reflect.has(target, key);
    },

    ownKeys(target) {
      track(target, ITERATE_KEY);                      // Object.keys / for...in
      return Reflect.ownKeys(target);
    },
  });

  reactiveCache.set(target, proxy);
  return proxy;
}
```

### 9.3. Ishlatish

```javascript
const state = reactive({ count: 0, user: { name: 'Ali' } });

effect(() => {
  console.log(`count: ${state.count}, name: ${state.user.name}`);
});
// count: 0, name: Ali           (darhol bir marta)

state.count++;
// count: 1, name: Ali

state.user.name = 'Vali';
// count: 1, name: Vali          (ichki obyekt ham reaktiv!)

state.count = 1;
// (hech narsa) — qiymat o'zgarmadi, trigger ham bo'lmadi

state.newProp = 123;
// (hech narsa) — effect `newProp` ni ham, kalitlar ro'yxatini ham o'qimagan
```

```javascript
// Kalitlar ro'yxatiga bog'liq effect
const list = reactive({ a: 1 });
effect(() => console.log('kalitlar:', Object.keys(list)));
// kalitlar: ['a']

list.b = 2;     // kalitlar: ['a', 'b']   ← Vue 2 da buni ko'rib bo'lmasdi!
delete list.a;  // kalitlar: ['b']
```

### 9.4. Batching (guruhlash) — scheduler va microtask

Agar har bir o'zgarishda effect **darhol** ishlasa, 3 ta ketma-ket o'zgarish = 3 ta keraksiz render. Haqiqiy tizimlar (Vue) o'zgarishlarni **to'playdi** va **bir marta**, **microtask**'da bajaradi — bu "Event Loop" hujjatidagi microtask navbati bilan bevosita bog'liq.

```javascript
const queue = new Set();
let flushPending = false;

function queueJob(job) {
  queue.add(job);                         // Set — bir xil job ikki marta turmaydi
  if (!flushPending) {
    flushPending = true;
    queueMicrotask(() => {                // sinxron kod tugagach, microtask sifatida
      const jobs = [...queue];
      queue.clear();
      flushPending = false;
      jobs.forEach((job) => job());
    });
  }
}

const s = reactive({ count: 0 });

effect(() => console.log('render:', s.count), { scheduler: queueJob });
// render: 0

s.count++;
s.count++;
s.count++;
console.log('sinxron kod tugadi');
// sinxron kod tugadi
// render: 3          ← 3 ta o'zgarish, lekin render FAQAT 1 marta
```

Vue'da bu mexanizm **`nextTick`** deb ataladi: DOM yangilanishi o'zgarish yuz bergan **zahoti** emas, **sinxron kod tugagach** (microtask'da) bajariladi.

### 9.5. `ref` — nega kerak?

`Proxy` faqat **obyektlar** bilan ishlaydi. Primitive (`number`, `string`, `boolean`) ni o'rab bo'lmaydi:

```javascript
const count = reactive(0);   // 0 — primitive, Proxy yaratib bo'lmaydi
```

Yechim — primitive'ni **obyekt ichiga solib**, `.value` orqali getter/setter bilan kuzatish:

```javascript
function ref(initial) {
  const r = {
    get value() {
      track(r, 'value');
      return initial;
    },
    set value(newVal) {
      if (Object.is(newVal, initial)) return;
      initial = newVal;
      trigger(r, 'value');
    },
  };
  return r;
}

const count = ref(0);
effect(() => console.log('count:', count.value));
count.value++;   // count: 1
```

> **Interview uchun muhim:** Vue 3'dagi `ref()` — **Proxy emas**, getter/setter'li oddiy obyekt (class). `.value` shuning uchun kerak: JS'da primitive'ga "havola" berish va uning o'zgarishini ushlashning boshqa yo'li yo'q. `ref` ichiga **obyekt** berilsa, ichki qiymat `reactive()` (ya'ni Proxy) bilan o'raladi.

### 9.6. `computed` va `watchEffect` — g'oya

| Vue API | Mohiyati |
|---|---|
| `reactive()` | Obyektni **Proxy**ga o'raydi (`get` → track, `set` → trigger) |
| `ref()` | Primitive uchun `.value` getter/setter (track/trigger) |
| `watchEffect(fn)` | Bevosita **`effect`** — avtomatik bog'liqliklarni kuzatadi |
| `computed(getter)` | **Dangasa (lazy) va keshlanadigan** effect: bog'liqlik o'zgarsa "iflos" (dirty) deb belgilanadi, qiymat faqat **kimdir o'qiganda** qayta hisoblanadi → "Memoization" hujjatidagi g'oyaning reaktiv varianti |
| Komponent `render` | Har bir komponentning render funksiyasi — **scheduler**'li effect (scheduler = `queueJob`) |
| `nextTick()` | Navbatdagi microtask flush tugashini kutadi |

**To'liq zanjir (Vue 3'da `state.count++` bosilganda):**

```
state.count++  →  Proxy set trap  →  trigger('count')
              →  render effect'ning scheduler'i  →  queueJob (microtask'ga qo'yadi)
              →  sinxron kod tugaydi  →  flush: komponent qayta render
              →  virtual DOM diff  →  haqiqiy DOM yangilanadi
```

> **Eslatma:** Yuqoridagi kod — **tushunish uchun soddalashtirilgan model**. Haqiqiy Vue'da qo'shimcha murakkabliklar bor: massiv metodlari (`push`, `includes`) uchun maxsus ishlov, `Map`/`Set` uchun alohida handler'lar, effect'lar qaramliklarini har ishga tushganda tozalash (conditional tarmoqlar uchun), effect scope'lar va Vue 3.5'dagi yangi ichki optimizatsiyalar. Lekin **asosiy g'oya** — aynan shu: `Proxy get → track`, `Proxy set → trigger`.

---

## 10. Vue 2 vs Vue 3 — `defineProperty` vs `Proxy`

Vue 2 (`Object.defineProperty`) quyidagi muammolarga ega edi:

```javascript
// Vue 2
const vm = new Vue({ data: { user: { name: 'Ali' }, list: [1, 2, 3] } });

vm.user.age = 25;          // ❌ reaktiv EMAS — yangi property qo'shildi, kuzatilmaydi
delete vm.user.name;       // ❌ o'chirish ham ko'rinmaydi
vm.list[0] = 99;           // ❌ indeks bo'yicha yozuv ko'rinmaydi
vm.list.length = 0;        // ❌ length o'zgarishi ham

// Maxsus yechimlar kerak edi:
Vue.set(vm.user, 'age', 25);
Vue.delete(vm.user, 'name');
vm.list.splice(0, 1, 99);
```

**Sabab:** `defineProperty` getter/setter'ni **mavjud, oldindan ma'lum** property'ga o'rnatadi. Yangi kalit paydo bo'lganda — unga getter/setter **yo'q**.

### Taqqoslash jadvali

| | Vue 2 (`defineProperty`) | Vue 3 (`Proxy`) |
|---|---|---|
| Yangi property qo'shish | ❌ Ko'rinmaydi (`Vue.set` kerak) | ✅ Ko'rinadi |
| Property o'chirish | ❌ Ko'rinmaydi (`Vue.delete`) | ✅ Ko'rinadi |
| Massiv indeksi / `length` | ❌ Ko'rinmaydi | ✅ Ko'rinadi |
| `Map`, `Set`, `WeakMap` | ❌ Qo'llab-quvvatlanmaydi | ✅ Qo'llab-quvvatlanadi |
| Boshlang'ich ishga tushirish | **Eager** — butun obyekt daraxti darhol aylanib chiqiladi | **Lazy** — ichki obyekt faqat **o'qilganda** o'raladi |
| Brauzer qo'llab-quvvatlashi | IE11 gacha | IE11 **yo'q** (Proxy polyfill qilib bo'lmaydi) |
| `in` operatori, `Object.keys` | ❌ Kuzatilmaydi | ✅ `has`, `ownKeys` trap'lari bilan |

> **Nega Proxy polyfill qilib bo'lmaydi?** Proxy — **til darajasidagi fundamental mexanizm**: u har bir `obj.x` o'qishni ushlaydi. Buni oddiy JS kodi bilan "taqlid" qilib bo'lmaydi (nom oldindan ma'lum bo'lmagan property'larni ushlash uchun til o'zi yordam berishi kerak). Shu sababli Vue 3 **IE11 qo'llab-quvvatlashini to'xtatdi**.

---

## 11. MobX, Immer va boshqa kutubxonalar

### MobX

- **MobX 4 va undan eski** — `Object.defineProperty` (eski brauzerlar uchun).
- **MobX 5 (2018)** dan boshlab — `observable()` oddiy obyekt, massiv, `Map`, `Set` uchun **Proxy** ishlatadi (shu tufayli yangi property qo'shish, massiv indeksi kuzatiladi).
- MobX 6'da `makeObservable` / `makeAutoObservable` **class instance**ning property'larini o'sha instance'ning o'zida observable qilib belgilaydi, `observable({...})` esa Proxy qaytaradi. Kerak bo'lsa, `configure({ useProxies: 'never' })` bilan Proxy'ni o'chirish mumkin.

```javascript
import { observable, autorun } from 'mobx';

const store = observable({ count: 0, todos: [] });

autorun(() => console.log('count:', store.count)); // = effect()
store.count++;                                      // autorun qayta ishlaydi
store.todos.push('yangi');                          // massiv ham kuzatiladi
```

MobX falsafasi Vue bilan bir xil: **"o'qigan narsangizga avtomatik obuna bo'lasiz"** (transparent reactive programming).

### Immer — Proxy bilan "immutable'ni mutable kabi yozish"

```javascript
import { produce } from 'immer';

const state = { user: { name: 'Ali' }, tags: ['a', 'b'] };

const next = produce(state, (draft) => {
  draft.user.name = 'Vali';   // oddiy mutatsiya kabi yozamiz
  draft.tags.push('c');
});

console.log(state.user.name); // 'Ali'   — asl obyekt O'ZGARMADI
console.log(next.user.name);  // 'Vali'  — yangi nusxa
console.log(state.tags === next.tags); // false (o'zgargan)
```

**Qanday ishlaydi:** `draft` — **Proxy**. Immer yozuvlarni (`set`) yozib boradi va oxirida **faqat o'zgargan qismlarni** nusxalaydi (structural sharing), o'zgarmagan qismlar esa **bir xil havola** bilan qoladi. Redux Toolkit'ning `createSlice` ichidagi "mutatsiya"li reducer'lar aynan Immer orqali ishlaydi.

### Boshqalar

| Kutubxona | Proxy qo'llanilishi |
|---|---|
| **Valtio** | Proxy-asosidagi state; `proxy()` + `useSnapshot()` |
| **Svelte 5** | `$state` rune obyekt va massivlarni chuqur reaktiv Proxy'ga aylantiradi |
| **SolidJS** | `createStore` — Proxy asosida |
| **Immer** | `produce` draft'lari |
| **Comlink** | Web Worker funksiyalarini "lokal" kabi chaqirish uchun Proxy |
| **Test kutubxonalari** | Mock/spy obyektlar (kutilmagan metod chaqiruvlarini ushlash) |

---

## 12. Cheklovlar va tuzoqlar

### 12.1. Proxy ≠ target (identifikatsiya)

```javascript
const target = { a: 1 };
const proxy = new Proxy(target, {});

console.log(proxy === target);   // false!
console.log(typeof proxy);       // 'object' (target bilan bir xil)
console.log(Array.isArray(new Proxy([], {}))); // true
```

Muammo: obyektni `Set`/`Map` kalitida yoki `===` bilan solishtirganda, **asl obyekt** va **uning proxy'si** — turli narsa. Vue'da `toRaw(proxy)` — asl obyektni qaytaradi.

### 12.2. Ichki slot'li obyektlar (`Map`, `Set`, `Date`, `#private`)

```javascript
const p = new Proxy(new Map(), {});
p.set('a', 1);
// TypeError: Method Map.prototype.set called on incompatible receiver #<Map>
```

**Sabab:** `Map`, `Set`, `Date`, `Promise` kabi o'rnatilgan turlar **ichki slot**larga ega va ularning metodlari `this` **haqiqiy `Map`** bo'lishini talab qiladi; `this = proxy` bo'lsa — xato. Xuddi shu muammo `#private` class maydonlarida ham bor:

```javascript
class Counter {
  #count = 0;
  inc() { return ++this.#count; }
}

const c = new Proxy(new Counter(), {});
c.inc();
// TypeError: Cannot read private member #count from an object whose class did not declare it
```

**Yechim:** metodlarni `target`ga bog'lash:

```javascript
const safe = (target) =>
  new Proxy(target, {
    get(t, k) {
      const v = Reflect.get(t, k, t);               // receiver = target
      return typeof v === 'function' ? v.bind(t) : v;
    },
  });

const m = safe(new Map());
m.set('a', 1);      // ishlaydi
```

(Vue 3 bu muammoni `Map`/`Set` uchun **maxsus collection handler'lar** bilan hal qiladi.)

### 12.3. Klonlash va `postMessage`

```javascript
const state = reactive({ a: 1 });
structuredClone(state);
// DataCloneError: #<Object> could not be cloned.

worker.postMessage(state);   // xuddi shu xato
```

Proxy **structured clone algoritmi** bilan klonlanmaydi (bu mavzu "Web Worker" qismida ham tegilgan edi). Yechim: avval asl obyektga qaytish (`toRaw(state)`) yoki oddiy nusxa olish (`JSON.parse(JSON.stringify(...))`, `{ ...state }`).

### 12.4. Samaradorlik

Har bir murojaat trap orqali o'tadi — to'g'ridan-to'g'ri murojaatdan **sekinroq** (dvigatel optimizatsiyalari kamroq qo'llanadi). Odatiy UI ilovalarida bu sezilmaydi, lekin **juda issiq sikllarda** (millionlab o'qish) Proxy'ni ishlatmaslik yoki asl obyekt bilan ishlash ma'qul.

### 12.5. Asl obyektni to'g'ridan-to'g'ri o'zgartirish — reaktivlikni aylanib o'tadi

```javascript
const raw = { count: 0 };
const state = reactive(raw);

raw.count++;      // ❌ Proxy'dan o'tmadi — trigger bo'lmaydi, UI yangilanmaydi
state.count++;    // ✅ to'g'ri
```

### 12.6. Destructuring reaktivlikni yo'qotadi

```javascript
const state = reactive({ count: 0 });

let { count } = state;    // count — oddiy son (0), Proxy bilan aloqasi uzildi
count++;                  // state.count o'zgarmadi

// Vue'da yechim:
const { count: countRef } = toRefs(state);  // har bir property uchun ref
```

---

## 13. Loyihaning qaysi qismlarida ishlatiladi?

| Soha | Misol |
|---|---|
| **Frontend freymvorklar** | Vue 3 `reactive`, Svelte 5 `$state`, SolidJS stores, Valtio |
| **State management** | MobX (`observable`), Valtio, Immer asosidagi Redux Toolkit reducer'lari |
| **Validatsiya va "xavfsiz" modellar** | Forma/ORM modellari: yozishdan oldin tur va qoida tekshirish |
| **Debugging va observability** | Logging proxy'lari, "kim bu property'ni o'zgartirdi?" izlash, audit log |
| **API klient'lar (dynamic)** | `api.users.get(1)` → `GET /users/1` — noma'lum metod nomlarini `get` trap orqali URL'ga aylantirish |
| **Mocking va testing** | Spy/stub'lar, "kutilmagan metod chaqirilsa — xato" turidagi qat'iy mock'lar |
| **Web Worker bilan RPC** | Comlink: worker'dagi funksiyani oddiy funksiya kabi chaqirish |
| **Immutable yangilanishlar** | Immer `produce` |
| **ORM / query builder** | Dinamik property'lar orqali zanjirli so'rovlar (`db.users.where(...)`) |
| **Xavfsizlik** | `Proxy.revocable` — muddatli kirish; "sandbox" obyektlar |

### Amaliy misol — dinamik API klient

```javascript
function createApi(baseUrl) {
  const make = (path) =>
    new Proxy(() => {}, {
      get(_, key) {
        return make(`${path}/${String(key)}`);         // api.users.profile → /users/profile
      },
      apply(_, __, args) {
        const id = args[0] !== undefined ? `/${args[0]}` : '';
        return fetch(`${baseUrl}${path}${id}`).then((r) => r.json());
      },
    });
  return make('');
}

const api = createApi('https://api.example.com');
api.users(5);          // GET https://api.example.com/users/5
api.posts.comments(9); // GET https://api.example.com/posts/comments/9
```

Bu — **`get` + `apply`** trap'larini birlashtirib, "yo'q" metodlarni **dinamik yaratish** namunasi.

---

## 14. Nega bu muhim — real foydalar

1. **Avtomatik reaktivlik** — dasturchi `state.count++` yozadi, qolganini (kuzatish, yangilash, guruhlash) tizim o'zi bajaradi. `setState` yoki qo'lda `notify()` shart emas.
2. **To'liq qamrov** — qo'shish, o'chirish, indeks, `in`, `Object.keys` — hammasi ushlanadi (Vue 2'ning `Vue.set` muammolari yo'q).
3. **Lazy (dangasa) samaradorlik** — katta obyekt daraxti boshida to'liq aylanib chiqilmaydi; ichki obyekt faqat kerak bo'lganda o'raladi.
4. **Toza API** — validatsiya, logging, default qiymatlar kabi **kesishuvchi mas'uliyatlar** (cross-cutting concerns) asl obyekt kodiga tegmasdan, alohida qatlamda yoziladi.
5. **Immutable'ni oson yozish** — Immer orqali murakkab ichma-ich yangilanishlar oddiy mutatsiya sintaksisida.
6. **Metadasturlash imkoniyati** — DSL'lar, dinamik API'lar, ORM'lar, mock'lar uchun kuchli asos.
7. **Intervyuda "chuqur bilim" ko'rsatkichi** — Proxy/Reflect, `receiver`, invariant'lar, track/trigger — senior darajadagi savollarning doimiy mavzusi.

---

## 15. Eng ko'p uchraydigan xatolar

### Xato 1: `set` trap'da `true` qaytarmaslik

```javascript
const p = new Proxy({}, {
  set(target, key, value) {
    target[key] = value;
    // return true; ← UNUTILDI
  },
});

'use strict';
p.x = 1; // strict mode'da TypeError: 'set' on proxy: trap returned falsish
```
**Yechim:** `return Reflect.set(target, key, value, receiver);` yoki kamida `return true;`.

### Xato 2: Trap ichida proxy'ning o'ziga murojaat qilish (cheksiz rekursiya)

```javascript
const p = new Proxy({}, {
  get(target, key) {
    return p[key];    // ❌ p[key] yana get trap'ni chaqiradi → cheksiz sikl
  },
});

p.x; // RangeError: Maximum call stack size exceeded
```
**Yechim:** `target[key]` yoki `Reflect.get(target, key, receiver)` ishlating.

### Xato 3: `receiver` ni uzatmaslik

```javascript
get(target, key) {
  return Reflect.get(target, key);       // ❌ receiver yo'q — getter'dagi this = target
}
get(target, key, receiver) {
  return Reflect.get(target, key, receiver); // ✅
}
```

### Xato 4: Symbol kalitlarni hisobga olmaslik

```javascript
get(target, key) {
  console.log(`o'qildi: ${key}`);   // ❌ key symbol bo'lsa — TypeError (implicit string konvertatsiya)
  // ...
}
// ✅ console.log(`o'qildi: ${String(key)}`) yoki typeof key === 'symbol' tekshiruvi
```
`console.log(proxy)`, spread, `for...of`, `${proxy}` kabi amallar `Symbol.iterator`, `Symbol.toPrimitive` kabi kalitlarni ham so'raydi.

### Xato 5: Ichki obyektlarning reaktiv/kuzatilmasligini unutish

```javascript
function shallowObserve(target) {
  return new Proxy(target, {
    set(t, k, v, r) { console.log('o\'zgardi:', k); return Reflect.set(t, k, v, r); },
  });
}

const s = shallowObserve({ user: { name: 'Ali' } });
s.user.name = 'Vali';   // ❌ hech narsa chiqmadi — s.user — asl (o'ralmagan) obyekt
```
**Yechim:** `get` trap'da ichki obyektni ham `Proxy` bilan o'rash (lazy deep), natijani `WeakMap` bilan keshlash.

### Xato 6: Asl (raw) obyektni to'g'ridan-to'g'ri o'zgartirish

```javascript
const raw = { n: 0 };
const state = reactive(raw);
raw.n = 5;   // ❌ reaktivlik aylanib o'tildi
```
**Qoida:** reaktiv obyektga faqat **proxy orqali** murojaat qiling; asl obyektga havolani yashiring.

### Xato 7: Destructuring bilan reaktivlikni yo'qotish

```javascript
const { count } = reactive({ count: 0 });  // count endi oddiy son
```
**Yechim:** `toRefs`, `toRef` yoki `state.count` shaklida ishlatish.

### Xato 8: Proxy'ni `structuredClone`/`postMessage` ga berish

**Yechim:** avval asl ma'lumotni oling (`toRaw`) yoki oddiy nusxa yarating.

### Xato 9: `Map`/`Set`/`Date`/`#private` li obyektni oddiy Proxy bilan o'rash

**Yechim:** metodlarni `target`ga `bind` qilish yoki maxsus handler yozish (12.2-bo'lim).

### Xato 10: Proxy'ni "xavfsizlik chorasi" deb hisoblash

Agar asl `target`ga havola oshkor bo'lsa, proxy'ni **aylanib o'tish** mumkin. Proxy — **xatti-harakatni boshqarish** vositasi, kirishni to'liq yopish uchun emas.

---

## 16. Interview savollari va qisqa javoblar

**S: Proxy nima?**
> J: Proxy — boshqa obyekt (target) atrofidagi qobiq bo'lib, shu obyekt ustidagi fundamental operatsiyalarni (o'qish, yozish, o'chirish, `in`, funksiya chaqiruvi, `new` va h.k.) handler'dagi trap'lar orqali ushlab olish va qayta aniqlash imkonini beradi. ES2015'da kiritilgan metadasturlash vositasi.

**S: Reflect nima va u nima uchun Proxy bilan birga ishlatiladi?**
> J: Reflect — Proxy trap'lariga 1:1 mos statik metodlar to'plami (`Reflect.get`, `set`, `has`, ...). U (1) operatsiyaning standart xatti-harakatini chaqirish imkonini beradi, (2) xato tashlash o'rniga `boolean` qaytaradi, (3) eng muhimi — `receiver` parametri orqali getter/setter ichidagi `this` ni to'g'ri uzatadi.

**S: `Reflect.get(target, key, receiver)` dagi `receiver` nima uchun kerak?**
> J: `receiver` — operatsiya boshlangan obyekt (odatda proxy'ning o'zi). Uni uzatmasak, getter ichidagi `this` asl `target` bo'lib qoladi va getter ichidagi `this.xxx` murojaatlari proxy'dan "aylanib" o'tadi — natijada reaktiv tizim ularni kuzata olmaydi.

**S: Vue 3 reaktivligi qanday ishlaydi?**
> J: `reactive()` obyektni Proxy'ga o'raydi. `get` trap ichida `track()` joriy ishlayotgan effect'ni (`activeEffect`) shu property'ga bog'laydi (`WeakMap → Map → Set` tuzilmasida). `set` trap'da `trigger()` shu property'ga bog'liq effect'larni qayta ishga tushiradi. Komponent render'i ham scheduler'li effect bo'lib, yangilanishlar microtask'da guruhlanadi (`nextTick`).

**S: Vue 2 va Vue 3 reaktivligi orasidagi asosiy farq nima?**
> J: Vue 2 `Object.defineProperty` ishlatgan: faqat mavjud property'lar kuzatilgan, yangi qo'shish/o'chirish va massiv indeksi ko'rinmagan (`Vue.set`/`Vue.delete` kerak bo'lgan), butun obyekt eager aylanib chiqilgan. Vue 3 Proxy ishlatadi: qo'shish, o'chirish, indeks, `in`, `Object.keys`, `Map`/`Set` hammasi kuzatiladi, ichki obyektlar esa lazy o'raladi. Evaziga IE11 qo'llab-quvvatlanmaydi.

**S: `ref` nima uchun kerak, `reactive` yetmaydimi?**
> J: Proxy faqat obyektlarni o'ray oladi, primitive'larni (son, satr) emas. `ref` primitive'ni `{ value }` ko'rinishidagi obyekt ichiga solib, `.value` getter/setter orqali track/trigger qiladi. `ref` Proxy emas, getter/setter'li obyekt; ichiga obyekt berilsa, u `reactive` bilan o'raladi.

**S: Nega reaktiv tizimda bog'liqliklar `WeakMap`da saqlanadi?**
> J: Kalit sifatida obyekt (target) kerak, va unga zaif havola saqlanishi lozim: target boshqa joyda ishlatilmay qolsa, GC uni va unga bog'liq barcha dependency yozuvlarini avtomatik tozalaydi. Oddiy `Map` ishlatilsa, bunday obyektlar xotirada abadiy qolib, memory leak bo'lar edi.

**S: `set` trap'dan nima qaytarish kerak?**
> J: Muvaffaqiyatli bo'lsa `true` (odatda `Reflect.set(...)` natijasi). Strict mode'da `false` yoki `undefined` qaytarilsa `TypeError` tashlanadi, sloppy mode'da yozuv jimgina e'tiborsiz qoldiriladi.

**S: Proxy invariant'lari nima?**
> J: JS dvigateli trap natijasini target'ning haqiqiy holatiga qarshi tekshiradi. Masalan, `writable: false` va `configurable: false` property uchun `get` trap boshqa qiymat qaytara olmaydi — aks holda `TypeError`. Bu tilning asosiy kafolatlarini (masalan, `Object.freeze`) Proxy buzishining oldini oladi.

**S: Nega `new Proxy(new Map(), {})` bilan `set()` chaqirsak xato chiqadi?**
> J: `Map` ichki slot'larga ega va uning metodlari `this` haqiqiy `Map` bo'lishini talab qiladi. Proxy orqali chaqirilganda `this = proxy` bo'ladi → "incompatible receiver" xatosi. Yechim: metodlarni `target`ga `bind` qilish yoki maxsus handler (Vue shunday qiladi). Xuddi shu sabab `#private` maydonlarda ham.

**S: Proxy'ni polyfill qilib bo'ladimi?**
> J: Yo'q. U har bir (oldindan noma'lum nomli) property murojaatini ushlashi kerak — bu til dvigatelining o'zi yordamisiz imkonsiz. Shuning uchun Proxy'ga tayangan kutubxonalar (Vue 3) eski brauzerlarni (IE11) qo'llab-quvvatlamaydi.

**S: Immer qanday ishlaydi?**
> J: `produce(base, recipe)` `base`ning "draft" Proxy'sini yaratadi. Recipe ichidagi mutatsiyalar draft orqali yozib olinadi; oxirida faqat o'zgargan qismlar nusxalanib yangi immutable obyekt hosil qilinadi (structural sharing), o'zgarmaganlari bir xil havola bilan qoladi. Asl obyekt o'zgarmaydi.

**S: Proxy va `Object.defineProperty` ning asosiy farqi nima?**
> J: `defineProperty` — **bitta, aniq property**ga getter/setter beradi (oldindan nomi ma'lum bo'lishi kerak). Proxy — **butun obyekt** ustidagi barcha operatsiyalarni, jumladan hali mavjud bo'lmagan property'larga murojaat, `delete`, `in`, `Object.keys`ni ham ushlaydi.

**S: Validation proxy'ning afzalligi nima?**
> J: Validatsiya mantig'i `set` trap'da bitta joyda jamlanadi, obyektdan foydalanuvchi kod esa oddiy `obj.age = 25` yozadi. Noto'g'ri qiymat yozilishi **manbada** (yozish paytida) to'xtatiladi, keyinroq noaniq joyda xato chiqmaydi.

---

## 17. Xulosa

> **Proxy** — obyekt oldidagi "darvozabon": u o'qish, yozish, o'chirish, `in`, `Object.keys`, funksiya chaqiruvi, `new` kabi **13 ta fundamental operatsiyani** ushlab, o'zgartira oladi. **Reflect** — shu operatsiyalarning "standart" variantlari to'plami bo'lib, trap ichida asl xatti-harakatni to'g'ri (ayniqsa `receiver` bilan) chaqirishga xizmat qiladi.
>
> **Reaktiv tizimlar** (Vue 3, MobX 5+, Svelte 5, Valtio) bir xil g'oyaga tayanadi: Proxy'ning `get` trap'i **kim nimani o'qiganini yozib qo'yadi** (`track`), `set`/`deleteProperty` trap'i **nima o'zgarganini aniqlab, bog'liqlarni qayta ishga tushiradi** (`trigger`). Bog'liqliklar `WeakMap → Map → Set` tuzilmasida saqlanadi (memory leak'siz), yangilanishlar esa microtask'da guruhlanadi.
>
> **Vue 2 → Vue 3** o'tishi (`defineProperty` → `Proxy`) `Vue.set`/`Vue.delete` kabi to'siqlarni yo'qotdi va lazy chuqur reaktivlikni berdi — evaziga eski brauzer (IE11) qo'llab-quvvatlashi yo'qoldi, chunki Proxy polyfill qilinmaydi.
>
> **Amaliy pattern'lar** — validation, logging, default qiymatlar, read-only, dinamik API klient, memoization, singleton, revocable proxy — kesishuvchi mas'uliyatlarni asl obyektga tegmasdan, alohida qatlamda hal qiladi.
>
> **Tuzoqlar:** `set`dan `true` qaytarmaslik, trap ichida proxy'ning o'ziga murojaat qilish, `receiver`ni unutish, symbol kalitlar, `Map`/`Set`/`#private` bilan `this` muammosi, destructuring va raw obyektni o'zgartirish bilan reaktivlikni yo'qotish, proxy'ni `structuredClone`/`postMessage`ga berish.
>
> Bu mavzuni chuqur tushunish — interview'da "Vue 3 reaktivligi ichkarida qanday ishlaydi?" degan klassik savolga ishonchli javob berish, va zamonaviy state-management kutubxonalarining "sehri" aslida oddiy `get`/`set` trap'lari ekanini anglash uchun muhim asos.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
