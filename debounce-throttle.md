# Debounce va Throttle — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.

---

## Mundarija

1. [Nega bu mavzu kerak — qayerdan paydo bo'lgan?](#1-nega-bu-mavzu-kerak--qayerdan-paydo-bolgan)
2. [Debounce nima?](#2-debounce-nima)
3. [Throttle nima?](#3-throttle-nima)
4. [Debounce vs Throttle — vizual taqqoslash](#4-debounce-vs-throttle--vizual-taqqoslash)
5. [Debounce'ni noldan yozish](#5-debounceni-noldan-yozish)
6. [Throttle'ni noldan yozish](#6-throttleni-noldan-yozish)
7. [Loyihaning qaysi qismlarida ishlatiladi?](#7-loyihaning-qaysi-qismlarida-ishlatiladi)
8. [React kabi freymvorklarda ishlatish](#8-react-kabi-freymvorklarda-ishlatish)
9. [Nima uchun muhim — real muammolar](#9-nima-uchun-muhim--real-muammolar)
10. [Eng ko'p uchraydigan xatolar](#10-eng-kop-uchraydigan-xatolar)
11. [Interview savollari va qisqa javoblar](#11-interview-savollari-va-qisqa-javoblar)
12. [Xulosa](#12-xulosa)

---

## 1. Nega bu mavzu kerak — qayerdan paydo bo'lgan?

Zamonaviy veb-ilovalarda ko'plab hodisalar (event'lar) **juda tez-tez, ba'zan soniyasiga o'nlab yoki yuzlab marta** ishga tushadi:

- `scroll` — sahifa aylantirilganda
- `resize` — oyna o'lchami o'zgartirilganda
- `input`/`keyup` — foydalanuvchi yozayotganda
- `mousemove` — sichqoncha harakatlanganda

Agar har bir hodisa uchun **og'ir** amal (masalan, API so'rovi, DOM'ni qayta chizish, murakkab hisob-kitob) bajarilsa, brauzer sekinlashadi, tarmoq resurslari behuda sarflanadi, ba'zan esa UI butunlay "osilib qoladi".

**Debounce** va **throttle** — bu muammoni hal qilish uchun frontend dasturlashda paydo bo'lgan ikkita **funksional dizayn pattern**i (naqshi). Ular tilning o'zida (`JavaScript` spec'ida) mavjud emas — bu dasturchilar tomonidan ishlab chiqilgan, keyinchalik keng tarqalgan yechimlar bo'lib, hozirda ko'plab kutubxonalarda (Lodash, Underscore.js) tayyor holda mavjud, shuningdek ularni **noldan yozishni bilish** — juda ko'p uchraydigan interview savoli.

---

## 2. Debounce nima?

**Debounce** — funksiyaning bajarilishini, **oxirgi chaqiruvdan keyin** ma'lum vaqt (`delay`) o'tguncha kechiktiradigan texnika. Agar shu kutish vaqti ichida funksiya **yana chaqirilsa**, avvalgi "kutish" bekor qilinadi va hisoblash **qaytadan boshlanadi**.

**Oddiy so'z bilan:** "Foydalanuvchi harakatni **to'xtatgandan** keyingina ishla."

### Vizual misol

Foydalanuvchi qidiruv maydoniga "salom" so'zini yozmoqda, har bir harf orasida 100ms:

```
s -> a -> l -> o -> m
|    |    |    |    |
0ms 100 200 300 400ms  (harflar yozildi)

delay = 300ms bo'lsa:

Har bir harf debounce timer'ni QAYTA boshlaydi.
Faqat "m" harfidan keyin 300ms hech narsa yozilmasa,
FAQAT O'SHANDA funksiya bir marta chaqiriladi (700ms atrofida).
```

### Klassik misol

```javascript
function debounce(fn, delay) {
  let timeoutId;
  return function (...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
  };
}

const search = debounce((query) => {
  console.log('Qidirilmoqda:', query);
}, 300);

input.addEventListener('input', (e) => search(e.target.value));
```

Foydalanuvchi qanchalik tez yozmasin, `search` funksiyasi (va demak, API so'rovi) faqat foydalanuvchi **300ms davomida yozishni to'xtatganda**, bir marta chaqiriladi.

---

## 3. Throttle nima?

**Throttle** — funksiyani belgilangan vaqt oralig'ida **eng ko'pi bilan bir marta** ishga tushirishni kafolatlaydi, chaqiruvlar qanchalik tez-tez sodir bo'lishidan qat'iy nazar.

**Oddiy so'z bilan:** "Harakat davom etsa ham, muntazam oraliqlarda, lekin **cheklab** ishla."

### Vizual misol

Foydalanuvchi sahifani doimiy aylantirmoqda (`scroll`), `throttle delay = 200ms`:

```
scroll hodisalari: |||||||||||||||||||||||  (juda tez-tez, masalan har 10ms'da)

throttle bilan:    |----200ms----|----200ms----|----200ms----|
                    ▲              ▲              ▲
                 bajariladi     bajariladi     bajariladi

Boshqa barcha oraliqdagi chaqiruvlar E'TIBORGA OLINMAYDI (yoki oxirgisi keyinroq bajariladi).
```

### Klassik misol

```javascript
function throttle(fn, delay) {
  let lastCall = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastCall >= delay) {
      lastCall = now;
      fn.apply(this, args);
    }
  };
}

const handleScroll = throttle(() => {
  console.log('Scroll pozitsiyasi:', window.scrollY);
}, 200);

window.addEventListener('scroll', handleScroll);
```

Foydalanuvchi qanchalik tez aylantirmasin, `handleScroll` **har 200ms'da eng ko'pi bilan bir marta** ishlaydi.

---

## 4. Debounce vs Throttle — vizual taqqoslash

```
Hodisalar (masalan, har 50ms'da bitta click):
X---X---X---X---X---X---X---X---X-------------(bo'sh, harakat to'xtadi)

DEBOUNCE (delay=300ms):
Faqat oxirgi X'dan 300ms o'tgach BITTA marta ishlaydi:
-------------------------------------Y
                                      ▲
                              (faqat 1 marta, oxirida)

THROTTLE (delay=300ms):
Muntazam oraliqlarda ishlaydi, harakat davom etsa ham:
Y-------Y-------Y-------Y
▲       ▲       ▲       ▲
(bir necha marta, muntazam oraliqda)
```

### Taqqoslash jadvali

| | Debounce | Throttle |
|---|---|---|
| **Asosiy mantiq** | Har yangi chaqiruvda "hisoblagich qaytadan boshlanadi" | Belgilangan oraliqda eng ko'pi bilan bir marta ishlaydi |
| **Qachon ishga tushadi** | Faqat harakat **to'xtagandan** keyin | Harakat davomida, **muntazam oraliqlarda** |
| **Tez-tez chaqiruvlar natijasi** | Faqat **oxirgisi** bajariladi | Bir nechta marta, tekis taqsimlangan holda bajariladi |
| **Tipik ishlatilish** | Qidiruv input'i, forma validatsiyasi, `resize` | `scroll`, `mousemove`, tugmani ketma-ket bosishdan himoya |

---

## 5. Debounce'ni noldan yozish

### 5.1. Asosiy versiya

```javascript
function debounce(fn, delay) {
  let timeoutId;

  return function (...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => {
      fn.apply(this, args);
    }, delay);
  };
}
```

### 5.2. Edge case: `this` va argumentlarni saqlash

Agar `debounce` biror obyekt metodini o'rasa, `this` to'g'ri saqlanishi kerak:

```javascript
const obj = {
  value: 42,
  log: debounce(function () {
    console.log(this.value); // this = obj bo'lishi kerak
  }, 300),
};

obj.log(); // 300ms'dan keyin "42" chiqadi
```

Bu — wrapper funksiya **oddiy `function`** (arrow emas) qilib yozilgani, va `setTimeout` ichida `fn.apply(this, args)` orqali asl `this` va argumentlar asl funksiyaga to'g'ri uzatilgani tufayli ishlaydi.

### 5.3. Edge case: `cancel()` metodi

Komponent yo'q qilinganda yoki shart o'zgarganda, kutilayotgan chaqiruvni bekor qilish kerak bo'lishi mumkin:

```javascript
function debounce(fn, delay) {
  let timeoutId;

  function debounced(...args) {
    const context = this;
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => {
      fn.apply(context, args);
    }, delay);
  }

  debounced.cancel = function () {
    clearTimeout(timeoutId);
    timeoutId = undefined;
  };

  return debounced;
}
```

```javascript
const debouncedSave = debounce(saveData, 500);
debouncedSave();
debouncedSave.cancel(); // saveData hech qachon chaqirilmaydi
```

### 5.4. To'liq, production-ready versiya (`immediate`/`leading` va `flush` bilan)

```javascript
function debounce(fn, delay, options = {}) {
  const { immediate = false } = options;
  let timeoutId = null;
  let lastArgs = null;
  let lastContext = null;

  function invoke() {
    fn.apply(lastContext, lastArgs);
    lastArgs = lastContext = null;
  }

  function debounced(...args) {
    lastArgs = args;
    lastContext = this;

    const callNow = immediate && !timeoutId;

    if (timeoutId) clearTimeout(timeoutId);

    timeoutId = setTimeout(() => {
      timeoutId = null;
      if (!immediate) invoke();
    }, delay);

    if (callNow) invoke();
  }

  debounced.cancel = function () {
    clearTimeout(timeoutId);
    timeoutId = null;
    lastArgs = lastContext = null;
  };

  debounced.flush = function () {
    if (timeoutId) {
      clearTimeout(timeoutId);
      timeoutId = null;
      invoke();
    }
  };

  return debounced;
}
```

**`immediate: true`** — funksiya **birinchi** chaqiruvda darhol ishga tushadi, keyingi tez-tez chaqiruvlar esa debounce qilinadi (masalan, tugmani bir marta bosishda darhol reaksiya, lekin ketma-ket bosishlardan himoya).

**`flush()`** — navbatda turgan chaqiruvni, kutmasdan, darhol bajarish (masalan, forma yuborilayotganda, oxirgi debounce qilingan validatsiyani majburan tugatish).

---

## 6. Throttle'ni noldan yozish

### 6.1. Asosiy versiya (timestamp asosida)

```javascript
function throttle(fn, delay) {
  let lastCall = 0;

  return function (...args) {
    const now = Date.now();
    if (now - lastCall >= delay) {
      lastCall = now;
      fn.apply(this, args);
    }
  };
}
```

**Muammo:** Bu versiyada faqat **"leading edge"** (oraliq boshida) ishlaydi — agar oraliq oxirida yana chaqiruv bo'lsa-yu, keyin harakat to'xtasa, oxirgi chaqiruv **butunlay yo'qolib ketishi** mumkin.

### 6.2. To'liq versiya — "trailing call" kafolati bilan

```javascript
function throttle(fn, delay) {
  let lastCall = 0;
  let timeoutId = null;
  let lastArgs = null;
  let lastContext = null;

  function invoke(time) {
    lastCall = time;
    fn.apply(lastContext, lastArgs);
    lastArgs = lastContext = null;
  }

  function throttled(...args) {
    const now = Date.now();
    const remaining = delay - (now - lastCall);

    lastArgs = args;
    lastContext = this;

    if (remaining <= 0) {
      // Yetarlicha vaqt o'tgan - darhol bajaramiz
      if (timeoutId) {
        clearTimeout(timeoutId);
        timeoutId = null;
      }
      invoke(now);
    } else if (!timeoutId) {
      // Oxirgi (trailing) chaqiruvni kafolatlash uchun
      timeoutId = setTimeout(() => {
        timeoutId = null;
        invoke(Date.now());
      }, remaining);
    }
  }

  throttled.cancel = function () {
    clearTimeout(timeoutId);
    timeoutId = null;
    lastCall = 0;
    lastArgs = lastContext = null;
  };

  return throttled;
}
```

Bu versiya **ham "leading" (boshida), ham "trailing" (oxirida)** chaqiruvni kafolatlaydi — ya'ni birinchi harakat darhol ishlaydi, va agar harakat to'xtaganda navbatda kutilayotgan chaqiruv bo'lsa, u ham albatta bajariladi.

---

## 7. Loyihaning qaysi qismlarida ishlatiladi?

| Holat | Qaysi texnika | Nega |
|---|---|---|
| **Qidiruv input'i (autocomplete)** | Debounce | Har harf uchun emas, foydalanuvchi yozishni tugatgach so'rov yuborish |
| **Forma validatsiyasi (real-time)** | Debounce | Har bosilgan tugma uchun emas, foydalanuvchi to'xtaganda tekshirish |
| **Oyna o'lchamini o'zgartirish (`resize`)** | Debounce | Layout hisob-kitobini faqat o'lcham o'zgarishi tugagandan keyin bajarish |
| **`scroll` hodisasi (infinite scroll, parallax)** | Throttle | Aylantirish davomida muntazam, lekin cheklangan yangilanish kerak |
| **`mousemove` (drag & drop, chizish)** | Throttle | Harakat davomida silliq, lekin ortiqcha yuk bermaydigan yangilanish |
| **Tugmani ketma-ket bosishdan himoya** | Debounce (immediate) yoki Throttle | Bir marta yuborish (submit) so'rovini takrorlanishdan saqlash |
| **Analytics/log yuborish** | Debounce yoki Throttle | Foydalanuvchi harakatlarini ortiqcha so'rovlarsiz kuzatish |
| **Auto-save (qoralama saqlash)** | Debounce | Foydalanuvchi yozishni davom ettirsa, saqlashni kechiktirish |

---

## 8. React kabi freymvorklarda ishlatish

```jsx
import { useMemo, useEffect, useRef } from 'react';

function SearchInput() {
  const inputRef = useRef();

  const debouncedSearch = useMemo(
    () =>
      debounce((query) => {
        console.log('Qidirilmoqda:', query);
        // API so'rovi shu yerda
      }, 300),
    []
  );

  // Komponent yo'q qilinganda debounce'ni bekor qilish - MUHIM!
  useEffect(() => {
    return () => {
      debouncedSearch.cancel();
    };
  }, [debouncedSearch]);

  return (
    <input
      ref={inputRef}
      onChange={(e) => debouncedSearch(e.target.value)}
    />
  );
}
```

> **Muhim eslatma:** React'da `debounce`/`throttle` funksiyasini har render'da **qayta yaratmaslik** kerak (`useMemo` yoki `useCallback` bilan "eslab qolish"), aks holda har safar yangi timer boshlanadi va debounce/throttle mantiqi ishlamay qoladi. Shuningdek, komponent `unmount` bo'lganda `cancel()` chaqirish orqali xotira sizishining (memory leak) oldini olish kerak.

---

## 9. Nima uchun muhim — real muammolar

### 9.1. Performance — ortiqcha ishlov berishning oldini olish

```javascript
// Debounce'siz - har harfda API so'rovi (juda ko'p keraksiz so'rov!)
input.addEventListener('input', (e) => {
  fetch(`/api/search?q=${e.target.value}`); // "salom" - 5 ta so'rov!
});

// Debounce bilan - faqat 1 ta so'rov (yozish tugagach)
input.addEventListener('input', debounce((e) => {
  fetch(`/api/search?q=${e.target.value}`);
}, 300));
```

### 9.2. Server yukini kamaytirish

Agar minglab foydalanuvchi qidiruv input'iga yozayotgan bo'lsa-yu, debounce qo'llanilmasa, server **keraksiz ravishda o'n barobar ko'proq** so'rov qabul qilishi mumkin — bu esa xarajat va tezlikka bevosita ta'sir qiladi.

### 9.3. UI silliqligini saqlash

`scroll` yoki `mousemove` kabi hodisalarda throttle'siz og'ir hisob-kitob bajarish, brauzerning **frame rate**ini pasaytirib, foydalanuvchi tajribasini yomonlashtiradi (UI "burishib" qolgandek ko'rinadi).

---

## 10. Eng ko'p uchraydigan xatolar

### Xato 1: Debounce va Throttle'ni aralashtirib yuborish

Qidiruv uchun throttle ishlatish — har 300ms'da so'rov yuboraveradi, garchi foydalanuvchi hali yozishni davom ettirayotgan bo'lsa ham. Bu keraksiz so'rovlarni keltirib chiqaradi. **To'g'ri tanlov: debounce.**

### Xato 2: `this`ni yo'qotib qo'yish

```javascript
function debounce(fn, delay) {
  let timeoutId;
  return (...args) => { // XATO - arrow function ishlatilgan
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn.apply(this, args), delay);
    // bu yerdagi 'this' - debounce() chaqirilgan joyning this'i, wrapper chaqirilgan joyники emas!
  };
}
```
**Yechim:** Tashqi wrapper funksiyani **oddiy `function`** qilib yozish kerak, arrow emas.

### Xato 3: React'da funksiyani har render'da qayta yaratish

```javascript
// XATO - har render'da yangi debounce yaratiladi, timer hech qachon to'g'ri ishlamaydi
function SearchInput() {
  const debouncedSearch = debounce((q) => search(q), 300); // har safar YANGI funksiya!
  return <input onChange={(e) => debouncedSearch(e.target.value)} />;
}
```
**Yechim:** `useMemo` yoki komponent tashqarisida yaratish.

### Xato 4: `cleanup` (tozalashni) unutish

Komponent yo'q qilinganda `cancel()` chaqirmaslik — bu debounce qilingan funksiya, komponent allaqachon ekrandan olib tashlangandan keyin ham ishga tushib, xatolik yoki xotira muammosiga olib kelishi mumkin.

---

## 11. Interview savollari va qisqa javoblar

**S: Debounce va throttle orasidagi asosiy farq nima?**
> J: Debounce funksiya bajarilishini kechiktiradi va har yangi chaqiruvda "hisoblagichni" qaytadan boshlaydi — faqat harakat to'xtagandan keyin bir marta ishlaydi. Throttle esa funksiyani belgilangan vaqt oralig'ida eng ko'pi bilan bir marta ishga tushiradi, chaqiruvlar davom etayotgan bo'lsa ham.

**S: Qachon debounce, qachon throttle ishlatiladi?**
> J: Debounce — foydalanuvchi "to'xtaganda" bir marta ishlash kerak bo'lgan holatlarda (qidiruv input'i, forma validatsiyasi, `resize`). Throttle — harakat davomida muntazam, lekin cheklangan yangilanish kerak bo'lgan holatlarda (`scroll`, `mousemove`, drag & drop).

**S: Debounce implementatsiyasida qanday edge case'lar bor?**
> J: `this` context'ini to'g'ri saqlash (`apply`/`call` orqali), argumentlarni to'g'ri uzatish, `cancel()` metodi orqali kutilayotgan chaqiruvni bekor qilish imkoniyati, va `immediate`/`leading` rejimi — funksiyani birinchi chaqiruvda darhol ishga tushirish imkoniyati.

**S: Throttle'ning "leading" va "trailing" rejimi nima?**
> J: "Leading" — funksiya birinchi chaqiruvda darhol ishlaydi. "Trailing" — agar oraliq oxirida kutilayotgan chaqiruv bo'lsa, u ham albatta bajarilishi kafolatlanadi (aks holda oxirgi chaqiruv "yo'qolib ketishi" mumkin).

**S: React'da debounce ishlatishda nimaga e'tibor berish kerak?**
> J: Debounce funksiyasini har render'da qayta yaratmaslik kerak — `useMemo` yoki `useCallback` bilan "eslab qolish" lozim. Shuningdek, komponent `unmount` bo'lganda `cancel()` chaqirib, xotira sizishi va keraksiz ishga tushishlarning oldini olish kerak.

---

## 12. Xulosa

> **Debounce va throttle — performance optimizatsiyasi uchun ishlatiladigan ikkita fundamental funksional pattern**, ular frontend dasturlashda tez-tez ishga tushuvchi hodisalar (scroll, input, resize, mousemove) sabab bo'ladigan ortiqcha ishlov berishning oldini olish uchun ishlab chiqilgan.
>
> **Debounce** — "harakat to'xtagandan keyin, bir marta ishla" mantig'iga, **throttle** — "harakat davomida, lekin cheklab, muntazam ishla" mantig'iga asoslanadi.
>
> Ularni **noldan to'g'ri yozish** uchun `this` context'ini saqlash, argumentlarni to'g'ri uzatish, `cancel()` orqali bekor qilish imkoniyati va kerak bo'lganda `immediate`/`leading`-`trailing` rejimlarini hisobga olish talab etiladi — bu jihatlar aynan interview'larda ko'p e'tibor qaratiladigan "edge case"lardir.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
