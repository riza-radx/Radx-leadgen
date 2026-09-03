# FAZA 2 – Schedule Trigger + Mbledhja e të Dhënave (TED API · DE + IT)

Statusi: 🟢 Testuar live më 2026-09-03 · Vendimi #1: **Opsioni A – TED API (Gjermani + Itali)**
Konteksti: [00-KONTEKSTI-MASTER.md](00-KONTEKSTI-MASTER.md) · Keywords: [01-FAZA-1-KEYWORDS.md](01-FAZA-1-KEYWORDS.md)

---

## 0. ✅ REZULTATE TË TESTUARA LIVE (jo teori)

API-ja u testua me `curl` përpara konfigurimit. Faktet e konfirmuara:

| Çështja | Rezultati real |
|---|---|
| Endpoint | `POST https://api.ted.europa.eu/v3/notices/search` |
| **API Key** | ❌ **NUK duhet.** Search API është publik, pa autentikim |
| Statusi i testit | `HTTP 200`, JSON valid |
| Volumi DE (CORE CPV, 30 ditë, ACTIVE) | **34 tenderë** |
| Volumi IT (CORE CPV, 30 ditë, ACTIVE) | **4 tenderë** |
| Volumi DE+IT (7 ditë) | **12 tenderë** |
| `today(-7)` në query | ✅ funksionon |
| Fusha kontakti (email, tel, person) | ✅ ekzistojnë dhe kthejnë të dhëna reale |
| Numri total i fushave të disponueshme | **1830** |

**Shembull i vërtetë i kapur nga testi (03.09.2026):**
```
publication-number:  607540-2026
buyer:               Universitätsklinikum Bonn AöR
city:                Bonn (53127)
title (eng):         Germany – Electrical wiring and fitting work – PV-Anlagen Teil 3, Parkhäuser
email:               baueinkauf@ukbonn.de
tel:                 022828716693
contact person:      Stabsstelle Baurecht und Baubeschaffung
submission-url:      https://www.evergabe.nrw.de/VMPSatellite/notice/CXPNY5YD2ST
deadline:            2026-09-09T11:00:00+02:00
```
→ Ky është pikërisht një lead B2B i gatshëm: PV në 3 parkingje të spitalit universitar të Bonit.

---

## 1. 🚨 DY ZBULIME KRITIKE NGA TESTIMI

### Zbulimi #1: `links` kthehet GJITHMONË dhe peshon 4.491 bytes për tender
TED e kthen fushën `links` edhe kur **nuk** e kërkon (24 gjuhë × 4 formate = PDF/HTML/XML).
Një tender i papërpunuar = **8.958 bytes**; pa `links` = **4.456 bytes**.

→ **Zgjidhja:** e heqim me Code Node dhe ndërtojmë linkun vetë:
`https://ted.europa.eu/en/notice/-/detail/{publication-number}`

### Zbulimi #2: `notice-title` kthehet në **24 gjuhë** njëherësh
`description-proc` kthehet vetëm në gjuhën origjinale (`deu` për Gjermani).

→ **Zgjidhja:** mbajmë vetëm `eng` (titulli) + gjuhën origjinale (përshkrimi).

### 💰 Efekti financiar
| | Bytes / tender | Kosto për 40 tenderë/muaj (gpt-4o-mini) |
|---|---|---|
| Pa optimizim | 8.958 | ~100% |
| Pas optimizimit | ~1.100 | **~12%** |

**Kursim ~88% i tokenave në input**, pa humbur asnjë informacion të nevojshëm.

### ⚠️ Zbulimi #3: `total-value` është i pabesueshëm
Në testin real u kthye `total-value: 1` – TED shpesh nuk e publikon vlerën në njoftimin
fillestar. Mos e përdor për prioritizim; vlera reale nxirret nga PDF-i i tenderit.

### ⚠️ Zbulimi #4: Itali ka volum të ulët në TED (4/muaj)
Arsyeja: tenderat italiane nën pragun e BE-së publikohen vetëm në platformat kombëtare
(ANAC / MEPA / portalet regjionale), **jo** në TED. TED kap vetëm tenderat e mëdha.
→ Për volum real në Itali, në një fazë të mëpasme duhet shtresa ANAC/MEPA.

---

## 2. ARKITEKTURA E FAZËS 2 (5 nyje)

```
[1. Schedule Trigger 07:00]
            ↓
[2. Set: CONFIG]  ← CPV, shtetet, lookback (një vend i vetëm për t'i ndryshuar)
            ↓
[3. HTTP Request: TED Search API]  ← POST, pa API key
            ↓
[4. Code: SLIM & NORMALIZE]  ← hiq links, zgjidh gjuhën, ndërto URL, dedupe CPV
            ↓
[5. Filter: NEGATIVE KEYWORDS]  ← hedh mbetjet PARA OpenAI-t
            ↓
      ➡️ FAZA 3 (OpenAI Node)
```

---

## 3. NYJA 1 — Schedule Trigger

| Parametri | Vlera |
|---|---|
| Node | **Schedule Trigger** |
| Trigger Interval | `Custom (Cron)` |
| Cron Expression | `0 7 * * *` |
| Workflow Timezone | `Europe/Tirane` ← **Settings → Timezone**, jo default |

> ⚠️ Pa e caktuar timezone-in, n8n përdor UTC dhe workflow-i niset në 09:00 me orën e Tiranës.

---

## 4. NYJA 2 — Set Node: `CONFIG`

Node: **Edit Fields (Set)** · Mode: `Manual Mapping` · **Include Other Input Fields: OFF**

| Emri i fushës | Tipi | Vlera |
|---|---|---|
| `cpv_core` | String | `09331000 09331200 09332000 45261215 31712331` |
| `countries` | String | `DEU ITA` |
| `lookback_days` | Number | `3` |

**Pse `lookback_days = 3` dhe jo 1:**
TED publikon vetëm në ditë pune. Me `1`, e hëna humbet tenderat e së premtes dhe një
dështim i vetëm i workflow-it humbet një ditë përgjithmonë. Me `3` ka mbivendosje të
sigurt — duplikatat i heq dedupe-i me `source_id` në Google Sheets (Faza 4).

**Zgjerimi i mëvonshëm (CPV të gjera – opsional):**
`09300000` (energji elektrike) dhe `71314100` (shërbime energjetike) e çojnë Gjermaninë
nga 34 → **229 tenderë/muaj**, por me shumë zhurmë (ngrohje, gaz, energji bërthamore).
Rekomandimi: mbaji jashtë tani; shtoji si stream i dytë vetëm kur AI-filtri i Fazës 3
të provohet i besueshëm.

---

## 5. NYJA 3 — HTTP Request: TED Search API

| Parametri | Vlera |
|---|---|
| Method | `POST` |
| URL | `https://api.ted.europa.eu/v3/notices/search` |
| Authentication | `None` ✅ |
| Send Body | `ON` |
| Body Content Type | `JSON` |
| Specify Body | `Using JSON` |
| Options → Timeout | `30000` |
| Options → Response → Never Error | `OFF` (dua të dijë kur dështon) |

### JSON Body (aktivizo Expression-in `=` në këtë fushë)

```json
{
  "query": "classification-cpv IN ({{ $json.cpv_core }}) AND buyer-country IN ({{ $json.countries }}) AND publication-date >= today(-{{ $json.lookback_days }}) SORT BY publication-date DESC",
  "fields": [
    "publication-number",
    "notice-title",
    "description-proc",
    "notice-type",
    "contract-nature",
    "buyer-country",
    "place-of-performance",
    "classification-cpv",
    "publication-date",
    "deadline-receipt-request",
    "total-value",
    "total-value-cur",
    "organisation-name-buyer",
    "organisation-city-buyer",
    "organisation-post-code-buyer",
    "organisation-email-buyer",
    "organisation-tel-buyer",
    "organisation-contact-point-buyer",
    "organisation-internet-address-buyer",
    "submission-url-lot"
  ],
  "limit": 100,
  "page": 1,
  "scope": "ACTIVE",
  "paginationMode": "PAGE_NUMBER"
}
```

### Shënime mbi query-n (sintaksë e verifikuar)
- Emrat e fushave janë **eForms kebab-case** (`classification-cpv`), **jo** kodet e vjetra
  dy-shkronjore (`PC`, `CY`). Sintaksa para-2023 kthen 0 rezultate.
- `IN (...)` pranon vlera të ndara me **hapësirë**, jo me presje.
- `scope: "ACTIVE"` = vetëm tenderë të hapur. `"ALL"` = arkiva e plotë (mos e përdor ditë për ditë).
- `limit` max = **100**. Kur `totalNoticeCount > 100`, rrit `page` (2, 3…) me Loop node.
  Me volumin aktual (34/muaj) nuk ka nevojë.
- Përgjigja jep `totalNoticeCount` — përdore për monitorim.

---

## 6. NYJA 4 — Code Node: `SLIM & NORMALIZE`

Node: **Code** · Mode: **Run Once for All Items** · Language: JavaScript

```javascript
// ===== VegaSolar · TED Normalizer =====
// Hyrja: 1 item me {notices:[...], totalNoticeCount:N}
// Dalja: 1 item per tender, i pastruar dhe i lehte per OpenAI (Faza 3)

const COUNTRY_SQ = {
  DEU: 'Gjermani', ITA: 'Itali', ALB: 'Shqiperi', MKD: 'Maqedoni e Veriut',
  AUT: 'Austri', GRC: 'Greqi', HRV: 'Kroaci', SVN: 'Slloveni', ESP: 'Spanje',
  FRA: 'France', NLD: 'Holande', BEL: 'Belgjike', POL: 'Poloni',
};

// Filtri i mbetjeve (Faza 1, seksioni G) - hedh perpara se te paguajme tokena
const NEG = [
  'stellenangebot', 'praktikum', 'offerta di lavoro', 'job vacancy',
  'webinar', 'seminar', 'schulung', 'fortbildung', 'corso di formazione',
  'solarium', 'sonnenstudio', 'crema solare', 'solar eclipse',
];

const MAX_DESC = 1200; // karaktere - mbron buxhetin e tokenave

const pickLang = (v, prefer = ['eng', 'deu', 'ita', 'fra', 'spa']) => {
  if (v == null) return '';
  if (typeof v === 'string') return v;
  if (Array.isArray(v)) return v.filter(Boolean).join(' | ');
  for (const l of prefer) {
    if (v[l]) return Array.isArray(v[l]) ? v[l].filter(Boolean).join(' ') : String(v[l]);
  }
  const first = Object.values(v)[0];
  if (first == null) return '';
  return Array.isArray(first) ? first.filter(Boolean).join(' ') : String(first);
};

const one = (v) => (Array.isArray(v) ? (v.find(Boolean) || '') : (v == null ? '' : v));
const uniq = (a) => [...new Set((a || []).filter(Boolean))];
const clean = (s) => String(s || '').replace(/\s+/g, ' ').trim();

const out = [];
const seen = new Set();
let skipped = 0;

for (const item of $input.all()) {
  const notices = item.json.notices || [];

  for (const n of notices) {
    const id = n['publication-number'];
    if (!id || seen.has(id)) { skipped++; continue; }   // dedupe brenda run-it
    seen.add(id);

    const titleEng    = clean(pickLang(n['notice-title'], ['eng']));
    const titleNative = clean(pickLang(n['notice-title'], ['deu', 'ita', 'fra', 'spa']));
    const desc        = clean(pickLang(n['description-proc'])).slice(0, MAX_DESC);

    // FILTRI I MBETJEVE
    const hay = `${titleEng} ${titleNative} ${desc}`.toLowerCase();
    if (NEG.some((w) => hay.includes(w))) { skipped++; continue; }

    const cc = one(n['buyer-country']);

    out.push({
      json: {
        // --- Identiteti & burimi ---
        source_id: id,
        source_platform: 'TED (Tenders Electronic Daily)',
        source_url: `https://ted.europa.eu/en/notice/-/detail/${id}`,
        submission_url: one(n['submission-url-lot']),

        // --- Teksti per AI-n (Faza 3) ---
        title_eng: titleEng,
        title_native: titleNative,
        description_raw: desc,

        // --- Bleresi & lokacioni ---
        buyer_name: clean(pickLang(n['organisation-name-buyer'], ['deu', 'ita', 'eng'])),
        country_code: cc,
        country_sq: COUNTRY_SQ[cc] || cc,
        city: one(n['organisation-city-buyer']),
        postcode: one(n['organisation-post-code-buyer']),
        nuts: uniq(n['place-of-performance']).filter((x) => x.length > 3).join(', '),

        // --- Kontakti (per fushen contact_info) ---
        contact_person: one(n['organisation-contact-point-buyer']),
        contact_email: one(n['organisation-email-buyer']),
        contact_phone: one(n['organisation-tel-buyer']),
        buyer_website: one(n['organisation-internet-address-buyer']),

        // --- Klasifikimi ---
        cpv: uniq(n['classification-cpv']).join(', '),
        contract_nature: one(n['contract-nature']),
        notice_type: n['notice-type'] || '',

        // --- Vlera & datat (total_value i pabesueshem - shih Zbulimin #3) ---
        total_value: n['total-value'] ?? '',
        currency: one(n['total-value-cur']),
        publication_date: n['publication-date'] || '',
        deadline: one(n['deadline-receipt-request']),

        // --- Meta ---
        fetched_at: new Date().toISOString(),
        ted_total_found: item.json.totalNoticeCount ?? null,
      },
    });
  }
}

console.log(`[VegaSolar] Kaluan: ${out.length} | U hodhen: ${skipped}`);
return out;
```

> **Kujdes:** `links` **nuk** kopjohet fare — kjo është ku ndodh kursimi i 88% të tokenave.

---

## 7. NYJA 5 — Filter (shtresë e dytë, opsionale)

Filtri i mbetjeve është brenda Code Node-it. Nëse e do si nyje të veçantë e të dukshme
(më e lehtë për ta ndryshuar pa prekur kodin):

Node: **Filter** · Condition: `String` → `{{ $json.title_eng }}` → `does not contain` →
fjalët negative (një kusht për fjalë, Combine: `AND`).

Filtër shtesë i dobishëm — hiq tenderat me afat të kaluar ose shumë të shkurtër:
`{{ $json.deadline }}` → `Date & Time` → `is after` → `{{ $now.plus(3, 'days') }}`

> Arsyeja: një tender me afat pas 2 ditësh nuk është lead i përdorshëm — ofertat teknike
> për PV kërkojnë më shumë kohë përgatitje.

---

## 8. 🧪 SI TA TESTOSH NË n8n

1. Krijo workflow-in e ri: `VegaSolar – Lead Gen`
2. **Settings → Timezone → `Europe/Tirane`** (bëje të parën, harrohet lehtë)
3. Shto 5 nyjet sipas rendit të mësipërm
4. Në **Set: CONFIG** vendos përkohësisht `lookback_days = 30` (për të pasur më shumë data testi)
5. Kliko **Execute Workflow** (jo prit orën 07:00)
6. **Pritje:** ~38 items në output të HTTP Request, dhe ~35–38 items pas Code Node-it
7. Kontrollo në output të Code Node-it: `source_url`, `contact_email`, `description_raw`
   duhet të kenë vlera reale
8. Kthe `lookback_days` në `3` para se ta aktivizosh

**Nëse merr 0 rezultate:** kontrollo që `cpv_core` përdor **hapësira** (jo presje) dhe që
Expression-i (`=`) është aktiv në fushën JSON Body.

---

## 9. STATUSI I VENDIMEVE (përditësim)

| # | Pyetja | Statusi |
|---|---|---|
| 1 | Nga cili shtet/burim nis testimi? | ✅ **A – TED API (DE + IT)** |
| 2 | API key për TED | ✅ **E anuluar – nuk duhet API key** |
| 3 | Plani Apify | ⚪ Shtyhet (jo e nevojshme për TED) |
| 4 | Telegram Chat ID + Google Sheet ID | ❓ Duhen në **Fazën 4** |
| 5 | CPV të gjera (09300000, 71314100)? | ❓ Vendim pas testimit të Fazës 3 |

## 10. HAPI TJETËR — FAZA 3

Ndërtimi i System Prompt-it për OpenAI Node që:
- klasifikon `project_type` (B2B / Tender i Madh vs B2C / Shtëpi & Biznes i Vogël)
- përkthen `summary_sq` në Shqip
- bashkon fushat e kontaktit në `contact_info`
- hedh çdo tender që nuk është realisht fotovoltaik (CPV-të janë të përafërta — testi
  nxori edhe `45311000` punime elektrike të përgjithshme)
- kthen saktësisht 6 fushat e Structured Output-it
