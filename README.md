# JavaScript / TypeScript — Texnik Intervyu Tayyorgarligi

> Middle va Senior darajadagi frontend/fullstack texnik suhbatlarga tayyorgarlik uchun **chuqur, misollarga boy va o'zbek tilida** yozilgan mavzular to'plami.

Bu repozitoriy — alohida-alohida **mavzu hujjatlari**dan iborat. Har bir hujjat bitta mavzuni "noldan tushunish"dan boshlab, **intervyuda shu mavzu bo'yicha savol berilganda ishonch bilan javob bera olish** darajasigacha olib boradi.

---

## Mundarija haqida eslatma

Bu README'da ataylab **mundarija (fayllar ro'yxati) berilmagan**: mavzular ro'yxati hali to'liq shakllanmagan va to'plam muntazam kengayib bormoqda. Barcha hujjatlar [`topics/`](./topics) papkasida joylashgan — ularni shu yerdan topishingiz mumkin.

---

## Loyihaning maqsadi

Texnik suhbatlarda ko'pincha ikki xil muammo uchraydi:

1. **Yuzaki bilim** — tushuncha nomini bilasiz (`closure`, `event loop`, `Proxy`), lekin "ichkarida qanday ishlaydi?" va "nega aynan shunday?" degan savolga javob berish qiyin.
2. **Tarqoq manbalar** — bitta mavzuni tushunish uchun o'nlab maqola, hujjat va video ko'rishga to'g'ri keladi.

Bu loyiha shu muammolarni hal qiladi: har bir mavzu **bitta joyda**, **izchil tuzilishda**, **ishlaydigan kod misollari** va **interview savol-javoblari** bilan jamlangan.

### Asosiy tamoyillar

| Tamoyil                             | Ma'nosi                                                                                                   |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------- |
| **"Nima" emas, "nega" va "qanday"** | Faqat ta'rif emas — mexanizm, kelib chiqish sababi va ichki ishlash tartibi tushuntiriladi                |
| **Misol bilan**                     | Har bir tushuncha qisqa, ishga tushirib ko'rsa bo'ladigan kod bilan ko'rsatiladi                          |
| **Intervyuga yo'naltirilgan**       | Har bir hujjat oxirida tipik savollar va qisqa, o'zlashtirish oson javoblar bor                           |
| **Amaliy ahamiyat**                 | Mavzu real loyihaning qayerida uchrashi va qanday xatolarga olib kelishi ko'rsatiladi                     |
| **Mavzular o'rtasidagi bog'liqlik** | Tushunchalar bir-biriga bog'lab tushuntiriladi (masalan, closure ↔ memory leak ↔ `WeakMap` ↔ memoization) |

---

## Nima uchun tayyorlangan?

- **Texnik intervyuga tizimli tayyorlanish** — tasodifiy savollarni yodlash o'rniga, asosiy mexanizmlarni chuqur tushunish orqali istalgan variatsiyadagi savolga javob berish.
- **Bilimdagi bo'shliqlarni aniqlash** — har bir hujjat oxiridagi savollar orqali o'zini tekshirish.
- **Tezkor qaytarish (revision) materiali** — intervyu arafasida kerakli mavzuni 15–20 daqiqada qayta ko'zdan kechirish.
- **Kundalik ish uchun ma'lumotnoma** — real loyihadagi xatolarni (memory leak, layout thrashing, `this` yo'qolishi, `DataCloneError` va h.k.) tushunish va tuzatish.
- **O'zbek tilidagi sifatli texnik kontent** — chuqur frontend mavzularini ona tilida o'qish imkoniyati.

---

## Kimlar uchun?

- Texnik intervyuga tayyorlanayotgan **Junior+ / Middle / Senior** JavaScript/TypeScript dasturchilari
- Frontend (React, Vue va h.k.) va Node.js backend dasturchilari
- Bilimini "ichki mexanizm" darajasida mustahkamlamoqchi bo'lganlar
- Jamoada intervyu o'tkazuvchilar (savollar uchun g'oya manbai sifatida)

**Tavsiya etilgan boshlang'ich bilim:** JavaScript sintaksisi, funksiyalar, obyektlar, massivlar, Promise'larning asosiy tushunchasi.

---

## Qamrab olingan yo'nalishlar

Hujjatlar quyidagi umumiy yo'nalishlar bo'yicha tuzilgan (bu **yakuniy ro'yxat emas**):

- **JavaScript yadrosi** — ish vaqti (runtime) modeli, scope va closure, `this`, prototip zanjiri, xotira boshqaruvi
- **Asinxronlik va parallellik** — event loop, microtask/macrotask, `async/await` ichki mexanizmi, Worker'lar va ular o'rtasida ma'lumot almashish
- **Funksional pattern'lar va til imkoniyatlari** — memoization, currying, `Symbol` va iterator protokollari, `Proxy`/`Reflect` va reaktivlik
- **TypeScript turlar tizimi** — generic'lar, variance, `unknown` va `any`
- **Modul tizimi va build** — CommonJS va ES Modules, tree-shaking
- **Brauzer va performance** — rendering pipeline, reflow/repaint, `requestAnimationFrame`
- **Frontend arxitektura** — katta ma'lumotlar bilan ishlash, virtualization, debounce/throttle

### Kengaytirish rejasidagi yo'nalishlar

Keyingi bosqichlarda quyidagi sohalar qo'shilishi mumkin: xavfsizlik (XSS, CSRF, CORS, CSP), race condition va so'rovlarni bekor qilish, state management arxitekturasi, test strategiyalari, build vositalarining ichki tuzilishi, HTTP va brauzer ichki ishlashi, PWA va Service Worker.

---

## Hujjatlar tuzilishi

Barcha mavzu hujjatlari **bir xil tuzilishda** yozilgan, shuning uchun istalgan hujjatni o'qishda nimani qayerdan topishni bilasiz:

1. **Mavzu nima va qayerdan paydo bo'lgan** — tarixiy kontekst, hal qiladigan muammo
2. **Qanday ishlaydi** — asosiy mexanizm, bosqichma-bosqich, ichki model
3. **Amaliy misollar** — ishga tushiriladigan kod, "noto'g'ri" va "to'g'ri" variantlar
4. **Loyihaning qaysi qismlarida ishlatiladi** — real stsenariylar
5. **Nega muhim** — amaliy foyda va oqibatlar
6. **Eng ko'p uchraydigan xatolar** — tuzoqlar va ularning yechimlari
7. **Interview savollari va qisqa javoblar** — o'zingizni tekshirish uchun
8. **Xulosa** — mavzuning qisqa, eslab qolish oson mag'zi

---

## Repozitoriy tuzilmasi

```
.
├── README.md        ← siz o'qiyotgan fayl
└── topics/          ← barcha mavzu hujjatlari (.md)
```

---

## Qanday foydalanish kerak?

**Chuqur o'rganish uchun (1–2 hafta oldin):**

1. Mavzuni boshidan oxirigacha o'qing.
2. Kod misollarini brauzer konsoli yoki Node.js'da **o'zingiz yozib ishga tushiring** — natijani bashorat qilib, keyin tekshiring.
3. "Eng ko'p uchraydigan xatolar" bo'limini alohida diqqat bilan o'qing — intervyuda aynan shu joylar so'raladi.
4. Interview savollariga **avval o'zingiz javob bering** (ovoz chiqarib), keyin hujjatdagi javob bilan solishtiring.

**Tezkor qaytarish uchun (intervyu arafasida):**

- Faqat "Interview savollari" va "Xulosa" bo'limlarini ko'rib chiqing.

**Samarali o'rganish maslahatlari:**

- Bitta mavzuni boshqa mavzular bilan **bog'lab** o'ylang (masalan, `this` ↔ closure ↔ prototip).
- Javob berganda **"nima" → "nega" → "misol"** tartibidan foydalaning.
- Savolga javob berishdan oldin talabni **aniqlashtiruvchi savol** bering (ayniqsa arxitektura savollarida).

---

## Kod misollarini ishga tushirish

- Misollar zamonaviy **JavaScript (ES2022+)** va **TypeScript**da yozilgan.
- Brauzer misollari uchun zamonaviy brauzer (Chrome, Firefox, Safari, Edge) konsoli yetarli.
- Node.js misollari uchun **joriy LTS versiya** tavsiya etiladi.
- Ayrim API'lar (masalan, `structuredClone`, `requestIdleCallback`) eski muhitlarda mavjud bo'lmasligi mumkin — kerak bo'lsa hujjatdagi eslatmalarga qarang.

---

## Muhim eslatmalar

- **Aniqlik va dolzarblik.** Materiallar tayyorlanish vaqtidagi bilimlarga asoslangan. Brauzer/Node.js versiyalari, API'larning qo'llab-quvvatlanishi va freymvork ichki tafsilotlari (masalan, Vue, React, MobX) vaqt o'tishi bilan o'zgarishi mumkin. **Muhim qarorlar va aniq tafsilotlar uchun** [MDN Web Docs](https://developer.mozilla.org), [ECMAScript](https://tc39.es/ecma262/) va [HTML Standard](https://html.spec.whatwg.org/) kabi rasmiy manbalarni tekshiring.
- **Soddalashtirilgan modellar.** Ba'zi hujjatlardagi "noldan yozilgan" implementatsiyalar (masalan, reaktivlik tizimi, deep clone) **o'quv maqsadida soddalashtirilgan** — ular g'oyani tushuntiradi, production kutubxonalarning o'rnini bosmaydi.
- **AI yordami.** Hujjatlar sun'iy intellekt (Claude) yordamida tayyorlangan va muallif tomonidan tuzilgan. Shuning uchun ularni **tanqidiy o'qish** va noaniq joylarni rasmiy manbalar bilan solishtirish tavsiya etiladi.
- **Intervyu — faqat bilim emas.** Hujjatlar texnik asosni beradi; suhbatda muloqot, muammoni tahlil qilish va fikrlash jarayonini ko'rsata olish ham baholanadi.

---

## Hissa qo'shish

Xato topdingizmi, aniqlashtirish yoki yangi mavzu taklifingiz bormi? Quyidagicha yordam berishingiz mumkin:

1. **Issue** oching — xato, noaniqlik yoki yangi mavzu g'oyasini yozing.
2. **Pull Request** yuboring — kichik tuzatishlar (imlo, kod xatosi, aniqlashtirish) ayniqsa qadrlanadi.
3. Yangi mavzu qo'shsangiz, mavjud hujjatlardagi **bir xil tuzilishga** amal qiling (yuqoridagi "Hujjatlar tuzilishi" bo'limi).

**Hissa qo'shishda:**

- Kod misollari **ishlaydigan** va qisqa bo'lsin.
- Da'volar uchun iloji bo'lsa manba (MDN, spetsifikatsiya) ko'rsating.
- Terminlarni izchil ishlating (masalan, inglizcha termin birinchi marta o'zbekcha izoh bilan keladi).

---

_Omad tilayman! Chuqur tushunish — eng yaxshi tayyorgarlik._
