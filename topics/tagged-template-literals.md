# Tagged Template Literals — styled-components va boshqa kutubxonalar qanday ishlatadi — To'liq Qo'llanma

> JavaScript texnik intervyularga tayyorgarlik uchun tushunarli va misollar bilan boyitilgan material.
> Asosiy fokus: tag funksiyasi qanday chaqiriladi, `strings` / `raw` / `values`, `String.raw`, `styled-components` ichki mexanizmi, `lit-html`, `gql`, `sql` kabi real qo'llanilishlar.

---

## Mundarija

1. [Tagged template nima va u qayerdan paydo bo'lgan?](#1-tagged-template-nima-va-u-qayerdan-paydo-bolgan)
2. [Oddiy template literal va tagged template — farqi](#2-oddiy-template-literal-va-tagged-template--farqi)
3. [Tag funksiyaga nima uzatiladi?](#3-tag-funksiyaga-nima-uzatiladi)
4. [`strings` massivining maxsus xususiyatlari](#4-strings-massivining-maxsus-xususiyatlari)
5. [Cooked va raw, `String.raw`](#5-cooked-va-raw-stringraw)
6. [Tag nima qaytarishi mumkin?](#6-tag-nima-qaytarishi-mumkin)
7. [Amaliy tag'lar (recipes)](#7-amaliy-taglar-recipes)
8. [styled-components qanday ishlaydi?](#8-styled-components-qanday-ishlaydi)
9. [Mini styled-components noldan (o'quv modeli)](#9-mini-styled-components-noldan-oquv-modeli)
10. [Boshqa kutubxonalar: lit-html, gql, sql, htm, zx, Jest](#10-boshqa-kutubxonalar-lit-html-gql-sql-htm-zx-jest)
11. [TypeScript bilan ishlash](#11-typescript-bilan-ishlash)
12. [Loyihaning qaysi qismlarida ishlatiladi?](#12-loyihaning-qaysi-qismlarida-ishlatiladi)
13. [Nega bu muhim — real foydalar](#13-nega-bu-muhim--real-foydalar)
14. [Cheklovlar va zamonaviy tendensiyalar](#14-cheklovlar-va-zamonaviy-tendensiyalar)
15. [Eng ko'p uchraydigan xatolar](#15-eng-kop-uchraydigan-xatolar)
16. [Interview savollari va qisqa javoblar](#16-interview-savollari-va-qisqa-javoblar)
17. [Xulosa](#17-xulosa)

---

## 1. Tagged template nima va u qayerdan paydo bo'lgan?

**Tagged template** (teglangan shablon) — template literal'ning oldiga **funksiya nomi (tag)** qo'yib yozilgan maxsus chaqiruv shakli:

```javascript
tag`Salom, ${ism}!`
//  ↑ backtick'dan oldin turgan funksiya — "tag"
```

Bu — **oddiy funksiya chaqiruvining** (`tag(...)`) boshqacha yozilishi. Farqi: funksiyaga argumentlar **maxsus shaklda**, shablonning **statik matn bo'laklari** va **dinamik qiymatlari** alohida-alohida uzatiladi. Natijada funksiya shablonni **o'zi xohlagancha** qayta ishlay oladi (matn yig'ishi shart ham emas).

### Qayerdan kelib chiqqan?

| Davr | Nima bo'lgan |
|---|---|
| **ES5 va undan oldin** | Satr yig'ish faqat `+` bilan: `'Salom, ' + ism + '!'`. Ko'p qatorli satr yo'q, xavfsiz interpolatsiya vositasi yo'q |
| **~2011–2014** | TC39'da "quasi-literals" nomi bilan taklif (strawman) ishlab chiqildi — g'oya *safe string interpolation* va *DSL'larni tilga ko'mish* edi |
| **ES2015 (ES6)** | **Template literal** (backtick) va **tagged template** rasmiy tilga kirdi |
| **ES2016** | Tag'ga uzatiladigan `strings` obyekti **har bir chaqiruv joyi (call site) uchun keshlanadigan** bo'ldi |
| **ES2018** | "Template Literal Revision": tagged template ichida **noto'g'ri escape** ketma-ketliklariga (`\u`, `\x`) ruxsat berildi (LaTeX, Windows yo'llari kabi DSL'lar uchun) |
| **2016-yildan** | `styled-components` (Glen Maddern va Max Stoiber) tagged template'larni **CSS-in-JS** uchun ommalashtirdi |

> **Qiziq izoh:** "quasi" so'zi hozir ham yashab kelmoqda — Babel/ESTree AST'da `TaggedTemplateExpression` tugunining maydonlari `tag` va **`quasi`**, `TemplateLiteral` esa **`quasis`** (matn bo'laklari) va `expressions` (qiymatlar) dan iborat. `babel-plugin-styled-components` aynan shu tugunlar ustida ishlaydi.

### Hal qiladigan asosiy muammolar

1. **Xavfsizlik** — trusted (dasturchi yozgan) matn va untrusted (foydalanuvchi bergan) qiymatni **aralashtirib yubormaslik** (SQL injection, XSS).
2. **Ichki DSL** — JS ichida boshqa "til"ni (CSS, HTML, SQL, GraphQL) tabiiy sintaksisda yozish.
3. **Statik va dinamik qismni ajratish** — keshlash, optimallashtirish va kompilyatsiya uchun poydevor.

---

## 2. Oddiy template literal va tagged template — farqi

### Oddiy template literal (tag'siz)

```javascript
const ism = 'Ali';
const yosh = 25;

const s = `Salom, ${ism}! Yoshingiz ${yosh}.`;
// "Salom, Ali! Yoshingiz 25."
```

Bu — **standart tag**: har bir qiymat `ToString` bilan satrga aylantirib, bo'laklar bilan ketma-ket ulanadi. Xuddi shu mantiq qo'lda:

```javascript
function defaultTag(strings, ...values) {
  return strings.reduce((acc, str, i) => acc + `${values[i - 1]}` + str);
}
```

Bu yerda `${values[i - 1]}` ataylab template ichida yozilgan: oddiy template literal bilan **aynan bir xil** konvertatsiya (`ToString`) ishlatilishi uchun. (Eslatma: "Symbol" hujjatida ko'rganimizdek, `Symbol` qiymatini template ichiga qo'ysangiz — `TypeError`.)

### Tagged template

```javascript
function tag(strings, ...values) {
  console.log(strings); // ['Salom, ', '! Yoshingiz ', '.']
  console.log(values);  // ['Ali', 25]
  return 'istalgan narsa'; // matn bo'lishi SHART EMAS
}

const natija = tag`Salom, ${ism}! Yoshingiz ${yosh}.`;
```

### Taqqoslash jadvali

| | Oddiy template literal | Tagged template |
|---|---|---|
| Sintaksis | `` `Salom ${x}` `` | `` tag`Salom ${x}` `` |
| Kim qayta ishlaydi | Dvigatel (standart yig'ish) | **Sizning funksiyangiz** |
| Natija turi | Doim `string` | **Istalgan tur** (obyekt, funksiya, DOM, Promise ...) |
| Statik va dinamik qism | Aralashib ketadi | **Alohida** uzatiladi |
| Xom (`raw`) matnga kirish | Yo'q | `strings.raw` |
| Noto'g'ri escape (`\u`) | `SyntaxError` | Ruxsat (ES2018) |

---

## 3. Tag funksiyaga nima uzatiladi?

Tagged template chaqirilganda dvigatel **desugar** (oddiy chaqiruvga aylantirish) qiladi:

```javascript
tag`A ${x} B ${y} C`

// Taxminan quyidagiga teng:
tag(['A ', ' B ', ' C'], x, y)
//  ↑ strings (maxsus massiv)   ↑ ...values
```

### Qoidalar

1. **Birinchi argument** — `strings`: matn bo'laklari massivi (maxsus `raw` property'li).
2. **Qolgan argumentlar** — interpolatsiya qiymatlari (`${...}` ichidagi ifodalar natijasi), **o'zgartirilmagan holda** (satrga aylantirilmaydi!).
3. **Doimiy munosabat:** `strings.length === values.length + 1`.

```javascript
const t = (s, ...v) => `${s.length}:${v.length}`;

t``                    // '1:0'  — strings = ['']
t`faqat matn`          // '1:0'  — strings = ['faqat matn']
t`${1}`                // '2:1'  — strings = ['', '']
t`${1}${2}`            // '3:2'  — strings = ['', '', '']
t`a${1}b${2}c`         // '3:2'  — strings = ['a', 'b', 'c']
```

Qiymat bo'lmagan joyda ham bo'sh satr (`''`) bo'laklari saqlanadi — shu sababli `strings` va `values` ni har doim **juftlab** (zip) yurish mumkin:

```javascript
function zip(strings, ...values) {
  let out = '';
  for (let i = 0; i < strings.length; i++) {
    out += strings[i];
    if (i < values.length) out += values[i]; // oxirgi bo'lakdan keyin qiymat yo'q
  }
  return out;
}
```

### Qiymatlar satrga aylantirilmaydi — "xom" holda keladi

```javascript
const fn = (s, ...v) => v;

fn`${42} ${{ a: 1 }} ${() => 'salom'} ${[1, 2]} ${null}`;
// [42, { a: 1 }, [Function (anonymous)], [1, 2], null]
```

Shu xususiyat `styled-components`ning **kaliti**: interpolatsiya sifatida **funksiya** uzatish mumkin (`${props => props.color}`), tag esa uni keyinroq (render paytida) chaqiradi.

### Qiymatlar *oldindan* (eager) hisoblanadi

Ifodalar chapdan o'ngga, tag chaqirilishidan **oldin** hisoblanadi. Kechiktirilgan (lazy) hisoblash kerak bo'lsa — **funksiya** uzating:

```javascript
tag`${qimmatHisob()}`;        // qimmatHisob() HAR DOIM, darhol chaqiriladi
tag`${() => qimmatHisob()}`;  // funksiya uzatildi — tag xohlasa chaqiradi
```

### Tag — har qanday ifoda bo'lishi mumkin

```javascript
tag`...`                      // oddiy funksiya
obj.method`...`               // metod (this = obj)
styled.div`...`               // property
styled(Button)`...`           // chaqiruv natijasi
styled.input.attrs({...})`...` // zanjir
getTag()`...`                 // funksiya qaytaradigan funksiya
tag`a``b`                     // zanjirli: tag`a` natijasi yana tag bo'ladi
```

```javascript
const obj = {
  nom: 'obj',
  tag(s) { return this.nom + s[0]; },
};
obj.tag`!`; // 'obj!' — this to'g'ri bog'lanadi (oldingi "this" hujjatidagi implicit binding)
```

---

## 4. `strings` massivining maxsus xususiyatlari

### 4.1. `raw` property

```javascript
function f(strings) {
  console.log(strings);     // ['Qator1\nQator2']   (cooked)
  console.log(strings.raw); // ['Qator1\\nQator2']  (raw)
}
f`Qator1\nQator2`;
```

### 4.2. Frozen (muzlatilgan)

```javascript
function f(strings) {
  console.log(Object.isFrozen(strings));     // true
  console.log(Object.isFrozen(strings.raw)); // true
  strings[0] = 'x';  // strict mode'da TypeError, sloppy'da jimgina e'tiborsiz
}
f`abc`;
```

### 4.3. Identity — har bir *chaqiruv joyi* uchun bitta obyekt (juda muhim)

Spetsifikatsiya `strings` obyektini (**template object**) har bir chaqiruv joyi (**call site**) uchun **bir marta yaratib, keshlaydi**:

```javascript
const id = (strings) => strings;

function ayni() { return id`salom`; }
console.log(ayni() === ayni());       // true  — bir xil joy, bir xil obyekt

console.log(id`salom` === id`salom`); // false — turli joy (matn bir xil bo'lsa ham!)

const seen = new Set();
for (let i = 0; i < 5; i++) seen.add(id`x`);
console.log(seen.size);               // 1 — sikl ichida ham bitta joy
```

**Bu nima uchun juda foydali?** `strings` obyektini **kalit** sifatida ishlatib, qimmat ishni (parse, kompilyatsiya) **faqat bir marta** bajarish mumkin:

```javascript
const templateCache = new WeakMap();   // kalit — obyekt → WeakMap aynan o'rinli

function html(strings, ...values) {
  let compiled = templateCache.get(strings);
  if (!compiled) {
    compiled = qimmatKompilyatsiya(strings);   // faqat BIRINCHI marta
    templateCache.set(strings, compiled);
  }
  return compiled(values);                      // keyingi chaqiruvlar — arzon
}
```

`lit-html` va `htm` aynan shu mexanizmga tayanadi (10-bo'lim). `WeakMap` ishlatilishining sababi — "Closure va Xotira" hujjatidagi bilan bir xil: template obyekti ishlatilmay qolsa, kesh yozuvi ham GC tomonidan avtomatik tozalanadi.

> **Eslatma:** Transpilyatorlar (Babel, TypeScript → ES5) bu xatti-harakatni **qo'lda taqlid qiladi**: template obyektini bir marta yaratib, o'zgaruvchida (`_templateObject`) saqlaydi. `eval`/`new Function` ichidagi shablon esa har safar yangi parse qilinadi — shuning uchun u yerda identity saqlanmaydi.

---

## 5. Cooked va raw, `String.raw`

Spetsifikatsiya har bir bo'lak uchun ikki xil qiymat yuritadi:

| Atama | Qayerda | Ma'nosi |
|---|---|---|
| **Cooked** (pishirilgan) | `strings[i]` | Escape ketma-ketliklari **qayta ishlangan** (`\n` → haqiqiy yangi qator) |
| **Raw** (xom) | `strings.raw[i]` | Manba kodda yozilgan **aynan o'sha belgilar** (`\n` → `\` va `n`) |

```javascript
function show(strings) {
  return { cooked: strings[0], raw: strings.raw[0] };
}

show`A\tB\nC`;
// cooked: "A	B
//          C"          ← tab va yangi qator belgilari
// raw:    'A\\tB\\nC'  ← backslash + t, backslash + n (9 belgi)
```

### `String.raw` — o'rnatilgan raw tag

```javascript
String.raw`C:\new\table`;   // 'C:\\new\\table'  (konsolda: C:\new\table)
`C:\new\table`;             // "C:" + yangi qator + "ew" + tab + "able"  ← buzilgan!

String.raw`a${1 + 1}b\n`;   // 'a2b\\n'  — qiymatlar o'rniga qo'yiladi, escape'lar qolmaydi
```

Qo'lda yozilsa:

```javascript
function myRaw(strings, ...values) {
  return strings.raw.reduce((acc, s, i) => acc + `${values[i - 1]}` + s);
}
```

**Qachon kerak:** Windows yo'llari, regex manbalari (`\d`, `\.` kabi backslash'lar ko'p joyda), LaTeX.

```javascript
const re = new RegExp(String.raw`^\d{3}-\d{2}$`);   // "\\d" deb ikki marta yozish shart emas
```

### ES2018: noto'g'ri escape — faqat tagged template'da ruxsat

```javascript
`\unicode`;                       // ❌ SyntaxError (oddiy template'da)

function latex(strings) { return strings.raw[0]; }
latex`\unicode va \xerxes`;       // ✅ ishlaydi

function cooked(strings) { return strings[0]; }
cooked`\unicode`;                 // undefined — cooked qiymat "yaroqsiz" bo'lsa undefined
```

Ya'ni tagged template'da `strings[i]` **`undefined`** bo'lishi mumkin (faqat noto'g'ri escape bo'lsa), `strings.raw[i]` esa **doim** satr. Shuning uchun DSL yozayotgan tag'lar `raw` bilan ishlashi xavfsizroq.

---

## 6. Tag nima qaytarishi mumkin?

Tag — oddiy funksiya, shuning uchun **istalgan** qiymatni qaytaradi:

```javascript
// 1) Satr
const upper = (s, ...v) => String.raw({ raw: s }, ...v).toUpperCase();

// 2) Obyekt (ma'lumot tuzilmasi)
const query = (s, ...v) => ({ text: s.join('?'), params: v });
query`SELECT * FROM t WHERE a = ${1}`;
// { text: 'SELECT * FROM t WHERE a = ?', params: [1] }

// 3) Funksiya / komponent (styled-components shunday!)
const comp = (s, ...v) => (props) => /* ... */;

// 4) DOM tuguni
const dom = (s, ...v) => document.createRange().createContextualFragment(/* ... */);

// 5) Promise
const run = (s, ...v) => fetch(/* ... */);

// 6) Boshqa tag (zanjirli chaqiruv)
const curried = (s) => (s2) => s[0] + s2[0];
curried`a``b`; // 'ab'
```

> **Muhim g'oya:** Tagged template — **"matn" haqida emas, "ma'lumot + qolip" haqida**. Tag shablonning *tuzilishini* (strings) va *qiymatlarini* (values) oladi va ulardan nimani xohlasa shuni yaratadi.

---

## 7. Amaliy tag'lar (recipes)

### 7.1. Qiymatlarni ajratib ko'rsatish

```javascript
function highlight(strings, ...values) {
  return strings.reduce(
    (out, str, i) => out + str + (i < values.length ? `<mark>${values[i]}</mark>` : ''),
    ''
  );
}

highlight`Mahsulot: ${'Olma'}, narxi ${5000} so'm`;
// "Mahsulot: <mark>Olma</mark>, narxi <mark>5000</mark> so'm"
```

### 7.2. Xavfsiz HTML (XSS'dan himoya)

```javascript
const escapeHtml = (v) =>
  String(v).replace(/[&<>"']/g, (c) => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;',
  }[c]));

function safeHtml(strings, ...values) {
  return strings.reduce(
    (out, str, i) => out + str + (i < values.length ? escapeHtml(values[i]) : ''),
    ''
  );
}

const userInput = '<img src=x onerror=alert(1)>';
safeHtml`<p>${userInput}</p>`;
// '<p>&lt;img src=x onerror=alert(1)&gt;</p>'
```

**Asosiy g'oya:** `strings` — **dasturchi yozgan, ishonchli** matn (to'g'ridan-to'g'ri manba kodda), `values` — **ishonchsiz** ma'lumot bo'lishi mumkin. Tag ikkalasini *ajrata oladi* va faqat qiymatlarni tozalaydi. Oddiy `+` yoki oddiy template literal buni qila olmaydi.

> **Ogohlantirish:** bu misol faqat **matn kontekstini** (`<p>...</p>`) himoyalaydi. Atribut ichi (`href="${url}"` — `javascript:` URL), `<script>`/`style` ichi yoki event handler kontekstlari uchun boshqa qoidalar kerak. Production'da tayyor, sinovdan o'tgan sanitizer/framework escaping'idan foydalaning.

### 7.3. SQL — parametrlashtirilgan so'rov (SQL injection'dan himoya)

```javascript
function sql(strings, ...values) {
  return {
    text: strings.reduce((acc, str, i) => acc + `$${i}` + str),  // $1, $2, ...
    values,
  };
}

const id = 5;
const status = 'active';
const q = sql`SELECT * FROM users WHERE id = ${id} AND status = ${status}`;
// {
//   text: 'SELECT * FROM users WHERE id = $1 AND status = $2',
//   values: [5, 'active']
// }

await pool.query(q);   // node-postgres {text, values} obyektini qabul qiladi
```

Qiymatlar so'rov matniga **hech qachon qo'shilmaydi** — alohida (`values`) drayverga uzatiladi, shuning uchun `'; DROP TABLE users; --` kabi kiritma shunchaki oddiy qiymat bo'lib qoladi.

```javascript
// ❌ XAVFLI — oddiy template literal
pool.query(`SELECT * FROM users WHERE name = '${name}'`);
// name = "x' OR '1'='1"  →  butun jadval qaytadi

// ✅ XAVFSIZ — tagged template (parametrlar alohida)
pool.query(sql`SELECT * FROM users WHERE name = ${name}`);
```

> **Cheklov:** parametr sifatida faqat **qiymatlar** uzatiladi. Jadval/ustun **nomlari** (identifier) parametr bo'la olmaydi — ular uchun alohida mexanizm (whitelist yoki kutubxonaning `sql.identifier` kabi yordamchisi) kerak. Production uchun `postgres`, `slonik`, `sql-template-strings` kabi kutubxonalarni ishlating.

### 7.4. Sonlarni formatlash

```javascript
function money(strings, ...values) {
  return strings.reduce((out, str, i) => {
    const v = values[i - 1];
    return out + (typeof v === 'number' ? v.toLocaleString('en-US') : v) + str;
  });
}

money`Jami: ${1234567} so'm`;  // "Jami: 1,234,567 so'm"
```

### 7.5. Sarflangan vaqtni o'lchovchi tag (oddiy)

```javascript
function debug(strings, ...values) {
  console.log('Statik qismlar:', strings);
  console.log('Dinamik qiymatlar:', values);
  return String.raw({ raw: strings }, ...values);
}
```

`String.raw({ raw: strings }, ...values)` — "raw massivdan satr yig'ish"ning qisqa yo'li (obyekt `raw` property'ga ega bo'lishi kifoya).

---

## 8. styled-components qanday ishlaydi?

### 8.1. Foydalanuvchi nuqtai nazaridan

```jsx
import styled from 'styled-components';

const Button = styled.button`
  padding: 8px 16px;
  border-radius: 6px;
  color: ${(props) => (props.$primary ? '#fff' : '#222')};
  background: ${(props) => (props.$primary ? '#0070f3' : '#eee')};

  &:hover {
    opacity: 0.85;
  }
`;

function App() {
  return (
    <>
      <Button $primary>Saqlash</Button>
      <Button>Bekor qilish</Button>
    </>
  );
}
```

### 8.2. "Sehr"ning asosi — bu shunchaki funksiya chaqiruvi

Yuqoridagi `Button` ta'rifi desugar qilinsa:

```javascript
const Button = styled.button(
  [
    '\n  padding: 8px 16px;\n  border-radius: 6px;\n  color: ',   // strings[0]
    ';\n  background: ',                                         // strings[1]
    ';\n\n  &:hover {\n    opacity: 0.85;\n  }\n',               // strings[2]
  ],
  (props) => (props.$primary ? '#fff' : '#222'),                // interpolations[0]
  (props) => (props.$primary ? '#0070f3' : '#eee'),             // interpolations[1]
);
```

Ya'ni:
- **`styled.button`** — funksiya (aslida `styled('button')` ning natijasi); `styled.div`, `styled.span` va h.k. kutubxonada tayyor teg nomlari ro'yxati bo'yicha yaratiladi.
- **Tag** chaqiriladi → **React komponenti** qaytaradi.
- **`${props => ...}`** — funksiya, **hali chaqirilmaydi**; keyinroq, *render paytida* props bilan chaqiriladi.

### 8.3. Ikki bosqichli hayot sikli

```
┌───────────────────────────────────────────────────────────────────┐
│ 1) ANIQLASH VAQTI (modul yuklanganda — BIR MARTA)                  │
│                                                                     │
│    styled.button`...${fn}...`                                       │
│       → tag(strings, ...interpolations)                             │
│       → qoidalarni saqlaydi: [matn, fn, matn, fn, matn]             │
│       → componentId yaratadi (masalan "sc-bdfBwQ")                  │
│       → React komponentini qaytaradi                                │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────┐
│ 2) RENDER VAQTI (har render'da)                                     │
│                                                                     │
│   a) props (va theme) bilan barcha interpolatsiyalarni hisoblaydi   │
│        - funksiya → chaqiriladi (natija ham yana ochiladi)          │
│        - css`` natijasi / boshqa komponent → "yoyiladi" (flatten)   │
│        - false / null / undefined → bo'sh satr                      │
│   b) tayyor CSS matnidan HASH oladi → sinf nomi (generated class)   │
│   c) shu sinf oldin kiritilmaganmi? Kiritilgan bo'lsa → o'tkazib    │
│        yuboradi. Yo'q bo'lsa:                                        │
│          CSS'ni preprocessor'dan (stylis) o'tkazadi                 │
│          (ichma-ich `&`, vendor prefix) → <style> ga yozadi         │
│   d) DOM elementni `className="sc-bdfBwQ hBKMXz"` bilan render qiladi │
└───────────────────────────────────────────────────────────────────┘
```

### 8.4. Muhim tafsilotlar

| Mavzu | Qanday ishlaydi |
|---|---|
| **Ikki sinf nomi** | Elementda odatda ikkita sinf bo'ladi: barqaror **componentId** (`sc-...`, komponentni aniqlash va selektor uchun) va **hash'dan hosil bo'lgan** sinf (aynan shu props kombinatsiyasi uchun CSS) |
| **Bir xil CSS → bir xil sinf** | Hash CSS matnidan olinadi, shuning uchun bir xil natija beradigan props'lar **bitta** sinfdan foydalanadi, qayta kiritilmaydi |
| **Har xil props → yangi sinf** | Har bir *noyob* CSS matni uchun alohida sinf va qoida yaratiladi |
| **`<style>` ga yozish** | Development'da odatda matn tugunlari sifatida (DevTools'da qoidalar ko'rinadi); production'da tezroq **CSSOM `insertRule`** ("speedy" rejim) ishlatiladi — sozlama/versiyaga qarab farq qilishi mumkin |
| **Statik optimizatsiya** | Agar komponentda **funksiya interpolatsiyalari yo'q** bo'lsa, CSS bir marta hisoblab olinadi (har renderda qayta hisoblanmaydi) |
| **Komponent selektori** | `${Button}:hover &` ishlaydi, chunki styled komponentning `toString()` metodi `.sc-xxxx` selektorini qaytaradi — template literal komponentni satrga aylantirganda aynan shu chaqiriladi |
| **Kengaytirish** | `styled(Button)` mavjud komponent ustiga qo'shimcha qoidalar qo'yadi |
| **`.attrs()`** | `styled.input.attrs(props => ({ type: 'text' }))` `` `...` `` — tag'dan oldin qo'shimcha prop'lar belgilanadi (zanjirli tag) |
| **`css` yordamchi tag** | Qayta ishlatiladigan CSS bo'lagi; ichidagi **funksiya interpolatsiyalarini kechiktirib** saqlaydi (quyida batafsil) |
| **Transient props (`$`)** | `$primary` kabi `$` bilan boshlangan prop'lar DOM'ga **uzatilmaydi** (aks holda React "noma'lum atribut" ogohlantirishi beradi) |

> **Versiyalarga e'tibor:** v5 → v6 orasida ichki tafsilotlar (prop filtrlash, tashqi paketlar, `shouldForwardProp` standart xatti-harakati) o'zgargan. Aniq xatti-harakat uchun o'zingiz ishlatayotgan versiyaning rasmiy hujjatiga qarang.

### 8.5. Nega aynan tagged template?

1. **Haqiqiy CSS sintaksisi** — DevTools'dan ko'chirib qo'yish, CSS bilimidan foydalanish; camelCase obyekt (`backgroundColor`) o'rniga `background-color`.
2. **Dinamik qiymat = funksiya** — tag qiymatlarni "xom" holda oladi, shuning uchun `${props => ...}` ni *kechiktirib*, render paytida chaqira oladi. Oddiy template literal buni qila olmasdi (funksiya satrga aylanib ketardi).
3. **Statik va dinamik qismning ajralishi** — `strings` doimiy, faqat `values` o'zgaradi → hash, kesh va statik optimizatsiya osonlashadi.
4. **Kompilyatsiya vaqtida tahlil qilish mumkin** — `TaggedTemplateExpression` AST'da aniq tanilgani uchun Babel/SWC plugin'lari: CSS'ni **minify** qiladi, komponentlarga `displayName` qo'shadi, SSR uchun ID beradi; **Linaria** kabi "zero-runtime" yechimlar esa CSS'ni build vaqtida `.css` faylga **chiqarib oladi**.
5. **Tooling** — editor plugin'lari tag nomiga qarab shablon ichini CSS sifatida bo'yaydi va tekshiradi.

### 8.6. `css` yordamchisi nega kerak?

```javascript
// ❌ XATO — oddiy template literal: funksiya SATRGA aylanib, CSS ichiga matn bo'lib tushadi
const truncateBad = `
  max-width: ${(p) => p.$width}px;
`;
// "max-width: (p) => p.$width px;"  ← yaroqsiz CSS

// ✅ TO'G'RI — css tag: funksiyani saqlab turadi, render paytida chaqiriladi
import { css } from 'styled-components';

const truncate = css`
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: ${(p) => p.$width}px;
`;

const Title = styled.h2`
  ${truncate}
  font-size: 20px;
`;
```

`css` tag qiymatlarni **hisoblamaydi**, balki `[matn, funksiya, matn, ...]` massivini qaytaradi; tashqi `styled` komponent render paytida uni "yoyib" (flatten) hisoblaydi.

---

## 9. Mini styled-components noldan (o'quv modeli)

Quyida g'oyani ko'rsatuvchi ~60 qatorli **soddalashtirilgan** versiya. Haqiqiy kutubxona ancha murakkab (stylis, SSR, theme, ref forwarding, ichma-ich selektorlar, kengaytirish, dev ogohlantirishlari va h.k.).

```jsx
import { createElement } from 'react';

// ── 1) Yordamchilar ────────────────────────────────────────────────
const injected = new Set();   // allaqachon <style>'ga yozilgan sinflar
let styleEl = null;

function getStyleEl() {
  if (!styleEl) {
    styleEl = document.createElement('style');
    styleEl.setAttribute('data-mini-styled', '');
    document.head.appendChild(styleEl);
  }
  return styleEl;
}

function hash(str) {                       // sodda, deterministik hash (djb2 turi)
  let h = 5381;
  for (let i = 0; i < str.length; i++) {
    h = ((h << 5) + h + str.charCodeAt(i)) | 0;
  }
  return (h >>> 0).toString(36);
}

function inject(className, css) {
  if (injected.has(className)) return;     // bir xil CSS — qayta yozilmaydi
  injected.add(className);
  getStyleEl().appendChild(document.createTextNode(`.${className}{${css}}\n`));
}

// ── 2) Interpolatsiyani hisoblash ──────────────────────────────────
function resolve(value, props) {
  if (typeof value === 'function') return resolve(value(props), props); // funksiya → natijasi
  if (Array.isArray(value)) return value.map((v) => resolve(v, props)).join(''); // css`` natijasi
  return value == null || value === false ? '' : String(value);
}

// ── 3) css tag: hisoblamaydi, faqat bo'laklarni saqlaydi ───────────
export function css(strings, ...interpolations) {
  return strings.reduce(
    (acc, str, i) =>
      acc.concat(str, i < interpolations.length ? [interpolations[i]] : []),
    []
  );
}

// ── 4) styled: tag yasovchi ────────────────────────────────────────
function createStyled(tagName) {
  return function tag(strings, ...interpolations) {     // ← TAG: modul yuklanganda, BIR MARTA

    function Styled({ children, className, ...props }) { // ← komponent: HAR RENDER'da
      let cssText = '';
      strings.forEach((str, i) => {
        cssText += str;                                   // statik bo'lak
        if (i < interpolations.length) {
          cssText += resolve(interpolations[i], props);   // dinamik bo'lak (props bilan)
        }
      });

      const generated = 'ms-' + hash(cssText);
      inject(generated, cssText);

      // `$` bilan boshlangan (transient) prop'lar DOM'ga o'tmaydi
      const domProps = Object.fromEntries(
        Object.entries(props).filter(([key]) => !key.startsWith('$'))
      );

      return createElement(
        tagName,
        { ...domProps, className: [className, generated].filter(Boolean).join(' ') },
        children
      );
    }

    return Styled;
  };
}

// ── 5) styled.div, styled.button ... — Proxy orqali ────────────────
export const styled = new Proxy(createStyled, {
  get: (_, tagName) => createStyled(tagName),   // styled.div → createStyled('div')
});
```

> `Proxy` bu yerda faqat ixchamlik uchun. Haqiqiy kutubxona teg nomlari ro'yxatini aylanib, `styled.div = styled('div')` shaklida **oldindan** biriktiradi — bu "Proxy va Reflect" hujjatidagi `get` trap g'oyasining yana bir amaliy ko'rinishi.

### Ishlatish

```jsx
const Button = styled.button`
  padding: 8px 16px;
  border: 0;
  border-radius: 6px;
  color: ${(p) => (p.$primary ? '#fff' : '#222')};
  background: ${(p) => (p.$primary ? '#0070f3' : '#eee')};
`;

const truncate = css`
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: ${(p) => p.$width}px;
`;

const Title = styled.h2`
  ${truncate}
  font-size: 20px;
`;

<Button $primary>Saqlash</Button>        // → class="ms-1a2b3c"  (ko'k fon)
<Button>Bekor qilish</Button>             // → class="ms-9z8y7x"  (kulrang fon)
<Title $width={240}>Uzun sarlavha ...</Title>
```

`<head>` ichidagi `<style>`:

```css
.ms-1a2b3c{ padding: 8px 16px; ... color: #fff; background: #0070f3; }
.ms-9z8y7x{ padding: 8px 16px; ... color: #222; background: #eee; }
```

### Bu modeldan ko'rinadigan 4 ta g'oya

1. **Tag — "fabrika":** u faqat `strings` va `interpolations`ni *eslab qoladi* va komponent qaytaradi. Haqiqiy ish *render paytida*.
2. **Funksiya interpolatsiyasi = kechiktirilgan hisoblash:** tag ularni chaqirmaydi, `resolve` render paytida props bilan chaqiradi.
3. **Hash → sinf → bir marta kiritish:** bir xil CSS qayta yozilmaydi (`Set` bilan). Bu — "Memoization" hujjatidagi *kalit → natija* g'oyasining CSS varianti.
4. **`css` tag ham xuddi shu tamoyil:** hisoblamaydi, bo'laklarni saqlaydi (lazy).

### Bu model nima qila olmaydi (haqiqiy kutubxona qiladi)

- Ichma-ich selektorlar (`&:hover`, `& > span`), media query'lar — **stylis** preprocessor kerak.
- `ref` uzatish, `as` prop, `styled(Component)` kengaytirish, `.attrs`, `ThemeProvider`.
- SSR (server tomonida stilni yig'ib, HTML'ga qo'shish) va hydration.
- Render paytida DOM'ga yozish — yon ta'sir (side effect); production kutubxonalar buni ehtiyotkorlik bilan boshqaradi.
- Statik qoidalarni oldindan hisoblash, dev ogohlantirishlari, komponent selektorlari (`toString`).

---

## 10. Boshqa kutubxonalar: lit-html, gql, sql, htm, zx, Jest

### 10.1. `lit-html` / Lit — samarali DOM yangilash

```javascript
import { html, render } from 'lit-html';

const view = (name) => html`<h1>Salom, ${name}!</h1>`;

render(view('Ali'), document.body);    // 1-marta: <template> yaratiladi, DOM quriladi
render(view('Vali'), document.body);   // faqat matn tuguni yangilanadi!
```

**Qanday ishlaydi:** `html` tag `{ strings, values }` ga o'xshash yengil obyekt qaytaradi (`TemplateResult`). Render'da `strings` obyekti **kalit** sifatida ishlatilib (4.3-bo'lim!), shu shablon uchun `<template>` element **bir marta** yaratiladi. Keyingi render'larda `strings` o'sha (identity bir xil) bo'lgani uchun faqat **`values` yangilanadi** — virtual DOM yoki diff shart emas.

Shu sabab "bir xil call site → bir xil `strings`" kafolati Lit'ning samaradorligining poydevori.

### 10.2. `htm` — JSX o'rniga build'siz

```javascript
import htm from 'htm';
import { h } from 'preact';

const html = htm.bind(h);
const el = html`<div class="card">${title}</div>`;
// → h('div', { class: 'card' }, title)
```

`htm` ham shablonni `strings` identity bo'yicha **bir marta parse qiladi**, keyin faqat qiymatlarni almashtiradi.

### 10.3. `gql` (graphql-tag) — GraphQL hujjati

```javascript
import { gql } from '@apollo/client';

const USER_QUERY = gql`
  query GetUser($id: ID!) {
    user(id: $id) {
      id
      name
      ...UserAvatar
    }
  }
  ${USER_AVATAR_FRAGMENT}
`;
// → DocumentNode (AST obyekti), matn emas
```

Tag GraphQL matnini **AST**ga parse qiladi (natijalar kesh qilinadi). Build vaqtida `babel-plugin-graphql-tag` / codegen yordamida bu parse'ni **oldindan** bajarib, runtime ish hajmini kamaytirish mumkin.

### 10.4. SQL kutubxonalari

```javascript
// postgres (porsager)
const users = await sql`SELECT * FROM users WHERE id = ${id}`;
// ${id} avtomatik parametr ($1) sifatida jo'natiladi
```

### 10.5. `zx` — shell buyruqlari

```javascript
import { $ } from 'zx';

const dir = 'my folder; rm -rf /';
await $`ls ${dir}`;   // qiymat avtomatik QUOTE qilinadi — shell injection oldi olinadi
```

### 10.6. Jest / Vitest — `test.each` jadvali

```javascript
test.each`
  a    | b    | kutilgan
  ${1} | ${1} | ${2}
  ${1} | ${2} | ${3}
  ${2} | ${1} | ${3}
`('$a + $b = $kutilgan', ({ a, b, kutilgan }) => {
  expect(a + b).toBe(kutilgan);
});
```

Bu — tag **parser** sifatida ishlashining go'zal namunasi: `strings[0]` ichidagi ustun nomlari (`a | b | kutilgan`) o'qiladi, `values` esa qatorlar bo'yicha guruhlanadi.

### 10.7. Emotion, Linaria va boshqalar

| Kutubxona | Tagged template qanday ishlatiladi |
|---|---|
| **Emotion** | `` css`...` ``, `` styled.div`...` `` — `styled-components`ga o'xshash; obyekt sintaksisi ham bor |
| **Linaria** | `` css`...` `` / `` styled.div`...` `` — CSS'ni **build vaqtida** `.css` fayllarga chiqaradi (zero-runtime) |
| **styled-jsx / goober / twin.macro** | Shu g'oyaning variantlari |
| **`dedent`, `outdent`** | Ko'p qatorli shablonlardan umumiy chekinishni olib tashlaydi |
| **i18n kutubxonalari** | `` t`Salom, ${name}` `` — tarjima kalitiga aylantirish |
| **Regex helper'lar** | `` re`\d+${suffix}` `` — regex'ni xavfsiz yig'ish |

---

## 11. TypeScript bilan ishlash

### 11.1. `TemplateStringsArray`

```typescript
// lib.d.ts'da:
interface TemplateStringsArray extends ReadonlyArray<string> {
  readonly raw: readonly string[];
}

function tag(strings: TemplateStringsArray, ...values: unknown[]): string {
  return strings.raw.join('|');
}
```

`ReadonlyArray` — chunki `strings` **frozen** (4.2-bo'lim).

### 11.2. Qiymat turlarini cheklash

```typescript
type SqlValue = string | number | boolean | null | Date;

function sql(strings: TemplateStringsArray, ...values: SqlValue[]) {
  return { text: strings.reduce((a, s, i) => a + `$${i}` + s), values };
}

sql`SELECT * FROM t WHERE a = ${5}`;           // ✅
sql`SELECT * FROM t WHERE a = ${{ x: 1 }}`;    // ❌ obyekt SqlValue emas — kompilyatsiya xatosi
```

Interpolatsiyalar turini tekshirish tagged template'ning TypeScript'dagi katta afzalligi.

### 11.3. Generic tag

```typescript
function gql<TData = unknown>(strings: TemplateStringsArray, ...values: unknown[]): TypedDocument<TData> {
  /* ... */
}

const q = gql<{ user: { id: string } }>`query { user { id } }`;
```

### 11.4. styled-components + TypeScript

```tsx
import styled from 'styled-components';

interface ButtonProps {
  $primary?: boolean;   // transient prop
}

const Button = styled.button<ButtonProps>`
  background: ${(p) => (p.$primary ? '#0070f3' : '#eee')};
`;

<Button $primary>OK</Button>;
<Button $primary="ha">OK</Button>;   // ❌ tur xatosi
```

> **Eslatma:** ES5'ga kompilyatsiya qilinganda TypeScript tagged template'larni `__makeTemplateObject` yordamchisi orqali pasaytiradi (`strings` + `raw` massivlarini qo'lda yaratadi).

---

## 12. Loyihaning qaysi qismlarida ishlatiladi?

| Soha | Misol |
|---|---|
| **CSS-in-JS** | `styled-components`, Emotion, Linaria — komponent darajasidagi stillar |
| **HTML shablonlash** | `lit-html`/Lit, `htm`, web component'lar |
| **Ma'lumotlar bazasi** | `postgres`, `slonik`, `sql-template-strings` — parametrlashtirilgan so'rovlar |
| **GraphQL** | Apollo/urql `gql`, Relay — so'rov hujjatlari |
| **Testlash** | Jest/Vitest `test.each` jadvallari, snapshot yordamchilari |
| **CLI va skriptlar** | `zx`, `execa` — xavfsiz shell buyruqlari |
| **Lokalizatsiya (i18n)** | Tarjima kalitlari, ko'plik (plural) shakllari |
| **Xavfsizlik** | Avtomatik escaping (HTML, URL, shell), trusted/untrusted ajratish |
| **Logging va debugging** | Strukturalangan log: `` log`user ${id} kirdi` `` → `{msg, fields}` |
| **Ichki DSL** | Regex quruvchilar, matematik/LaTeX ifodalar, konfiguratsiya qolipi |

---

## 13. Nega bu muhim — real foydalar

1. **Xavfsizlik arxitekturasi** — statik (ishonchli) va dinamik (ishonchsiz) qismlarning tilning o'zida ajralishi SQL injection / XSS / shell injection'ga qarshi eng toza himoya usullaridan biri.
2. **Tabiiy DSL'lar** — CSS, HTML, SQL, GraphQL'ni JS ichida o'z sintaksisida yozish: o'qilishi va qo'llab-quvvatlanishi oson.
3. **Samaradorlik** — `strings` identity keshi (`lit-html`, `htm`) qimmat parse/kompilyatsiyani **bir marta** bajarishga imkon beradi.
4. **Kompilyatsiya vaqtida optimallashtirish** — statik tahlil qilinadigan sintaksis tufayli minify, `displayName`, CSS extraction mumkin.
5. **Kechiktirilgan hisoblash** — qiymat sifatida funksiya uzatish (`${props => ...}`) dinamik, props'ga bog'liq stillarni osongina ifodalaydi.
6. **Intervyuda chuqur bilim ko'rsatkichi** — "`styled.div` ichkarida nima?", "`strings` nega frozen?", "tagged template oddiy `+`dan nimasi bilan xavfsizroq?" kabi savollar senior darajada tez-tez uchraydi.

---

## 14. Cheklovlar va zamonaviy tendensiyalar

### 14.1. Runtime CSS-in-JS narxi

`styled-components` kabi **runtime** yechimlar har renderda interpolatsiyalarni hisoblaydi, CSS'ni hash qiladi, preprocessor'dan o'tkazadi va `<style>`ga yozadi. Bu:

- JS bundle hajmini oshiradi (kutubxona + stylis);
- juda dinamik stillarda (masalan, sichqoncha koordinatasi bo'yicha) **minglab sinf** yaratishi mumkin (kutubxona bu haqda ogohlantirish ham beradi) — bunday holatda `style` prop yoki CSS o'zgaruvchilari (`--x`) ishlating;
- SSR da qo'shimcha sozlash (`ServerStyleSheet`) talab qiladi.

### 14.2. React Server Components

Runtime CSS-in-JS React kontekst va hook'larga tayanadi, shuning uchun Next.js App Router kabi muhitlarda asosan **Client Component**larda ishlaydi va qo'shimcha "registry" sozlamasi kerak bo'ladi. Shu sabab jamoalar ko'proq **zero-runtime** yondashuvlarga (CSS Modules, Tailwind, Linaria, Panda CSS, vanilla-extract) o'tmoqda.

### 14.3. Kutubxonaning holati

2025-yil davomida `styled-components` mualliflari loyiha **maintenance rejimiga** o'tgani haqida e'lon qilgan. Bu tendensiya va loyiha holatini **rasmiy repozitoriy/blog orqali tekshiring** — yangi loyiha boshlashda bu muhim omil.

> **Intervyu uchun muhim:** tagged template mexanizmini bilish kutubxona omadidan mustaqil — `lit-html`, `gql`, `sql`, `zx` va boshqalar uchun ham o'sha tamoyil ishlaydi. Shuning uchun intervyuda *mexanizmni* tushuntira olish, *"styled-components eng yaxshi"* deb talqin qilishdan muhimroq.

---

## 15. Eng ko'p uchraydigan xatolar

### Xato 1: Styled komponentni render ichida e'lon qilish

```jsx
function Card() {
  const Box = styled.div`padding: 8px;`;   // ❌ HAR RENDER'da YANGI komponent turi
  return <Box>...</Box>;
}
```

React har renderda boshqa *tur*ni ko'radi → komponent daraxti **qayta mount** qilinadi (holat yo'qoladi, sekinlashadi).
**Yechim:** styled komponentlarni **modul darajasida**, funksiyadan tashqarida e'lon qiling.

### Xato 2: Funksiyani oddiy template literal ichida ishlatish

```javascript
const mixin = `color: ${(p) => p.color};`;     // ❌ funksiya matnga aylanadi
const mixin = css`color: ${(p) => p.color};`;  // ✅
```

### Xato 3: `$`siz prop'lar DOM'ga o'tib ketishi

```jsx
<Button primary>…</Button>      // ❌ <button primary="true"> — React ogohlantirishi
<Button $primary>…</Button>     // ✅ transient prop
```

### Xato 4: Oddiy `+` yoki template literal bilan SQL/HTML yig'ish

```javascript
db.query(`SELECT * FROM users WHERE id = ${id}`);   // ❌ SQL injection
db.query(sql`SELECT * FROM users WHERE id = ${id}`);// ✅
```

### Xato 5: Tag ichida `strings`ni o'zgartirishga urinish

```javascript
function bad(strings) { strings[0] = 'x'; }  // ❌ frozen: strict mode'da TypeError
```
**Yechim:** nusxa oling: `const copy = [...strings];`

### Xato 6: `strings[i]` har doim satr deb o'ylash

```javascript
function latex(strings) { return strings[0].length; }
latex`\unicode`;   // ❌ strings[0] === undefined → TypeError
```
**Yechim:** DSL'larda `strings.raw` ishlating.

### Xato 7: Call site identity'ga noto'g'ri tayanish

```javascript
const cache = new WeakMap();
function f(strings) { return cache.get(strings); }

f`abc`; f`abc`;   // ❌ ikki xil call site → ikki xil kalit, kesh "ishlamaydi"
```
Identity kafolati faqat **bir xil joy** uchun. Shablonni dinamik qurmang (`eval`, `new Function`) — har safar yangi `strings` hosil bo'ladi.

### Xato 8: Escape qilingan HTML'ni *hamma kontekst uchun yetarli* deb hisoblash

Matn kontekstidagi escaping `href`, `style`, `<script>`, event handler kontekstlarini himoyalamaydi. Framework'ning kontekstga mos escaping'idan foydalaning.

### Xato 9: Juda dinamik qiymatlar uchun stilni sinfga aylantirish

```jsx
const Dot = styled.div`
  left: ${(p) => p.$x}px;   /* ❌ har koordinata — yangi sinf */
`;
```
**Yechim:** tez o'zgaradigan qiymatlar uchun `style={{ left: x }}` yoki `.attrs(p => ({ style: { left: p.$x } }))`.

### Xato 10: Tag'ni oddiy funksiya kabi chaqirib, `strings.raw` yo'qligidan xato olish

```javascript
function t(strings) { return strings.raw[0]; }
t(['salom']);   // ❌ TypeError: Cannot read properties of undefined (reading '0')
t`salom`;       // ✅
```
Qo'lda massiv uzatilsa, `raw` property bo'lmaydi.

---

## 16. Interview savollari va qisqa javoblar

**S: Tagged template nima?**
> J: Template literal oldiga funksiya (tag) qo'yib yozilgan chaqiruv shakli. Dvigatel tag'ni chaqirib, unga birinchi argument sifatida statik matn bo'laklari massivini (`strings`, `raw` property'li), qolgan argument sifatida interpolatsiya qiymatlarini (o'zgartirilmagan holda) uzatadi. Tag istalgan tur qiymat qaytara oladi.

**S: `` tag`a${x}b${y}c` `` qanday chaqiriladi?**
> J: Taxminan `tag(['a', 'b', 'c'], x, y)` ga teng — lekin `strings` maxsus (frozen, `raw` property'li, har chaqiruv joyi uchun keshlanadigan) obyekt. `strings.length === values.length + 1` doim.

**S: `strings` va `strings.raw` farqi nima?**
> J: `strings` — cooked (escape'lar qayta ishlangan, masalan `\n` — haqiqiy yangi qator); `strings.raw` — manba kodda yozilgan aynan o'sha belgilar. Tagged template'da noto'g'ri escape (`\u`) bo'lsa, cooked qiymat `undefined` bo'ladi, `raw` esa doim satr.

**S: `String.raw` nima uchun kerak?**
> J: Backslash'larni qayta ishlamasdan, xom holda saqlash uchun — Windows yo'llari, regex manbalari, LaTeX. Qiymatlar baribir o'rniga qo'yiladi.

**S: `strings` obyekti nega keshlanadi va bu nimaga yaxshi?**
> J: Spetsifikatsiya (ES2016'dan) har bir chaqiruv joyi uchun template obyektini bir marta yaratib saqlaydi. Natijada `strings` identity'si kalit bo'lib xizmat qiladi: `lit-html` va `htm` shablonni bir marta parse/kompilyatsiya qilib, `WeakMap`da saqlaydi; keyingi chaqiruvlarda faqat `values` yangilanadi.

**S: Ikkita bir xil matnli tagged template'ning `strings`i `===` bo'ladimi?**
> J: Yo'q, agar ular turli joyda yozilgan bo'lsa. Faqat *bir xil chaqiruv joyi* (masalan, funksiya ichida takror chaqirilganda yoki siklda) bir xil obyekt beradi.

**S: `styled.div` ichkarida qanday ishlaydi?**
> J: `styled.div` — tayyor funksiya (`styled('div')` natijasi), u tag sifatida chaqiriladi. Tag `strings` va interpolatsiyalarni (jumladan funksiyalarni) eslab qolib, React komponenti qaytaradi. Har renderda komponent props bilan interpolatsiyalarni hisoblaydi, tayyor CSS matnidan hash olib sinf nomi yasaydi, bu CSS oldin kiritilmagan bo'lsa preprocessor (stylis) orqali `<style>`ga yozadi va elementni shu sinf bilan render qiladi.

**S: Nega `${props => ...}` ishlaydi, oddiy template literal'da esa funksiya matnga aylanib ketadi?**
> J: Oddiy template literal qiymatni `ToString` bilan satrga aylantiradi. Tagged template'da esa tag qiymatni "xom" holda oladi, shuning uchun funksiyani saqlab, keyinroq (render paytida) props bilan chaqira oladi.

**S: Nega `css` yordamchi tag kerak?**
> J: Qayta ishlatiladigan CSS bo'lagidagi funksiya interpolatsiyalari yo'qolmasligi uchun. Oddiy template literal funksiyani matnga aylantiradi; `css` esa bo'laklarni va funksiyalarni massiv sifatida saqlab, tashqi styled komponent render paytida hisoblashiga imkon beradi.

**S: Tagged template SQL injection'dan qanday himoya qiladi?**
> J: Tag statik matn va qiymatlarni ajratadi: matn `$1, $2` parametrli so'rovga aylanadi, qiymatlar alohida massivda drayverga uzatiladi va hech qachon SQL matniga qo'shilmaydi. Cheklov: faqat qiymatlar parametrlashadi, jadval/ustun nomlari emas.

**S: Tagged template nega Babel/SWC bilan yaxshi optimallashtiriladi?**
> J: Chunki bu sintaksis AST'da aniq (`TaggedTemplateExpression`: `tag` + `quasi`) va statik bo'laklar kompilyatsiya vaqtida ma'lum. Plugin CSS'ni minify qiladi, `displayName` qo'shadi, SSR ID beradi; Linaria kabilar esa CSS'ni build vaqtida faylga chiqarib oladi.

**S: Nima uchun `$primary` kabi transient prop'lar kerak?**
> J: Oddiy prop'lar DOM elementiga atribut sifatida uzatiladi va React'da noma'lum atribut ogohlantirishini keltirib chiqaradi. `$` bilan boshlangan prop'lar faqat stil hisoblashda ishlatiladi va DOM'ga o'tkazilmaydi.

**S: Natijani ayting:**

```javascript
function tag(strings, ...values) { return strings.raw[0]; }
console.log(tag`Salom\nDunyo`);
```
> J: `Salom\nDunyo` — `raw` bo'lgani uchun `\n` yangi qatorga aylanmaydi, backslash va `n` belgilari sifatida chiqadi.

**S: Natijani ayting:**

```javascript
const t = (s, ...v) => s.length + ':' + v.length;
const a = 1, b = 2;
console.log(t`${a}${b}`);
```
> J: `'3:2'` — `strings = ['', '', '']` (uchta bo'sh bo'lak), `values = [1, 2]`.

**S: Runtime CSS-in-JS ning kamchiliklari nima?**
> J: Qo'shimcha JS hajmi va har renderdagi hisoblash/hash/injection narxi, juda dinamik qiymatlarda sinflar ko'payishi, SSR/hydration sozlamalari va React Server Components bilan cheklangan moslik. Shuning uchun zero-runtime yondashuvlar (CSS Modules, Tailwind, Linaria, Panda CSS) ommalashmoqda.

---

## 17. Xulosa

> **Tagged template** — template literal oldiga funksiya qo'yib yozilgan chaqiruv: `` tag`A ${x} B` `` ≈ `tag(['A ', ' B'], x)`. Tag **statik matn bo'laklarini** (`strings`, `raw` bilan) va **dinamik qiymatlarni** (xom holda) **alohida** oladi va istalgan narsani — satr, obyekt, funksiya, DOM, komponent — qaytara oladi.
>
> **Uch asosiy imkoniyat:** (1) statik/dinamik qismlarni ajratish → xavfsizlik (SQL, HTML, shell) va keshlash; (2) qiymatlarni xom holda olish → kechiktirilgan hisoblash (`${props => ...}`); (3) `strings` obyektining har bir chaqiruv joyi uchun bir martalik, frozen, keshlanadigan identity'si → `WeakMap` asosidagi samarali shablon kesh (`lit-html`, `htm`).
>
> **`styled-components`** shu g'oyalarning amaliy ko'rinishi: `styled.button` — tag; u `strings` va interpolatsiyalarni eslab qolib komponent qaytaradi; render paytida props bilan interpolatsiyalar hisoblanadi, CSS matnidan hash olinib sinf yasaladi, bir xil CSS bir marta `<style>`ga kiritiladi. Tagged template sintaksisi statik tahlil qilinishi mumkin bo'lgani uchun Babel/SWC plugin'lari va zero-runtime yechimlar (Linaria) ham mumkin.
>
> **Amaliy qoidalar:** styled komponentlarni modul darajasida e'lon qiling, qayta ishlatiladigan bo'laklar uchun `css` ishlating, DOM'ga o'tmasligi kerak prop'lar uchun `$` prefiksi, SQL/HTML/shell uchun parametrlashtiruvchi yoki escaping qiluvchi tag'lar, DSL'larda `strings.raw`, tez o'zgaradigan qiymatlar uchun `style` prop yoki CSS o'zgaruvchilari.
>
> Bu mavzuni chuqur tushunish — intervyuda "`styled.div` ichkarida qanday ishlaydi?", "tagged template va oddiy template literal farqi?", "`strings` nega frozen va keshlanadi?" kabi savollarga ishonchli javob berish hamda `lit-html`, `gql`, `sql` kabi kutubxonalarning "sehri" aslida oddiy funksiya chaqiruvi ekanini anglash uchun muhim asos.

---

*Tayyorlandi: JS texnik intervyu tayyorgarligi uchun*
