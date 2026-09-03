# VegaSolar – Sistem Inteligjent i Gjenerimit të Projekteve (Lead Generation)

> **KONTEKST MASTER** – Ky file është burimi i vetëm i së vërtetës për projektin.
> Lexoje në fillim të çdo sesioni pune. Përditësohet në fund të çdo faze.
>
> Krijuar: 2026-09-03 · Pronar: anisa.sinaj@radx.app · Kompania: VegaSolar

---

## 1. ROLI I ASISTENTIT

Arkitekt Specialist i Automatizimeve dhe AI (n8n & LLM Expert).
Detyra: ndërtimi hap pas hapi i një sistemi inteligjent Lead Generation për VegaSolar.

## 1.b PROFILI REAL I KOMPANISË (burimi: https://vegagroup.al/vegasolar/)

Ky profil përcakton se **çfarë është vërtet një lead i mirë** — përdoret nga System Prompt-i i Fazës 3.

| Aspekti | Realiteti |
|---|---|
| **Fokusi** | **Fotovoltaik (PV)** – projektim, inxhinieri, instalim |
| Shërbimet | Projektim & inxhinieri · Prokurim & logjistikë · Menaxhim projekti · Instalim & konfigurim · Konsulencë efiçencë energjetike · Konsulencë financimi · **Mirëmbajtje & monitorim** |
| **Shkalla e projekteve** | **500 kWp – 1.26 MWp** (C&I / publik, jo retail banesor) |
| Referenca | Aeroporti Ndërkombëtar i Tiranës (1 MWp) · Te Stela Resort (1.26 MWp) · Mielli "Atlas" (1.2 MWp) |
| Klientët | Rezidencial · Komercial · Industrial · Objekte publike |
| **Tregu kryesor** | **Shqipëri** (HQ: Rr. Ndre Mjeda, Kompleksi Magnet, Ndërtesa Ara, Tiranë) + "dhe përtej" |
| Kontakti | +355 69 202 1115 |
| Nuk përmendet | Solar termik, bateri BESS, pompa termike, karikues EV, certifikime, brand-e partnere |

### 🎯 Përfundime për sistemin
1. **Sweet-spot-i është PV mbi çati komerciale/industriale/publike, 300 kWp–1.5 MWp.** Një
   tender për një shtëpi 6 kWp nuk është lead i vërtetë për këtë kompani.
2. **Mirëmbajtja & monitorimi janë shërbim aktiv** → tenderat e mirëmbajtjes së PV-ve
   ekzistuese janë leads valide.
3. **Projektimi/inxhinieria është shërbim aktiv** → tenderat vetëm për projektim PV
   janë valide (ndryshe nga auditimet energjetike pa PV).
4. **Solar termik dhe BESS janë periferike**, jo shërbim i deklaruar → relevancë dytësore.
5. ⚠️ **Mospërputhje strategjike:** referencat dhe tregu deklaruar janë **Shqipëria**,
   ndërsa Faza 2 nis me tenderat gjermanë. Shih Vendimin #7.

## 2. MISIONI I SISTEMIT

Sistemi skanon automatikisht:
- internetin (news, portale, blogje)
- portalet e tenderave publike
- lejet e ndërtimit
- mediat sociale

në 5 rajone/shtete: **Shqipëri, Maqedoni e Veriut, Itali, Gjermani, BE** – bazuar në lista Keywords.

Më pas AI:
1. analizon rezultatet,
2. filtron mbetjet (junk/noise),
3. përkthen detajet kryesore në **Shqip**,
4. dërgon projektet te **Telegram bot** + **Google Sheets**.

## 3. STRUKTURA E TË DHËNAVE DALËSE (OUTPUT FORMAT – E DETYRUESHME)

Çdo projekt duhet të nxjerrë saktësisht këtë strukturë JSON:

```json
{
  "project_name": "Emri i Projektit / Biznesit",
  "country_location": "Shteti dhe Qyteti",
  "project_type": "B2B / Tender i Madh  OSE  B2C / Shtëpi & Biznes i Vogël",
  "summary_sq": "Përmbledhja e kërkesës teknike (E përkthyer në Shqip)",
  "contact_info": "Email, Telefon, Personi kontaktues ose Linku origjinal",
  "source_platform": "Platforma nga ku u mor e dhëna"
}
```

**Rregull:** `project_type` mban vetëm dy vlera të lejuara:
`"B2B / Tender i Madh"` ose `"B2C / Shtëpi & Biznes i Vogël"`.

## 4. BURIMET E TË DHËNAVE SIPAS RAJONEVE

### 4.1 BE (Gjermani, Itali, etj.)
| Burim | Tipi | Përdorimi |
|---|---|---|
| TED – Tenders Electronic Daily API | API zyrtare | Tenderat publike mbi pragun e BE-së |
| Google News API | API | Lajme për investime/parqe fotovoltaike |
| Apify – MyHammer (DE) | Scraper | Kërkesa B2C për instalime |
| Apify – ProntoPro (IT) | Scraper | Kërkesa B2C për instalime |

### 4.2 Shqipëri & Maqedoni e Veriut
| Burim | Tipi | Përdorimi |
|---|---|---|
| APP – Portali i Prokurimeve Publike (AL) | Portal/Scraping | Tenderat publike shqiptare |
| e-Nabavki (MK) | Portal/Scraping | Tenderat publike maqedonase |
| Gazeta / buletine të lejeve të ndërtimit | Scraping | Projekte private në zhvillim |
| Apify – Facebook Groups / LinkedIn | Scraper | Kërkesa dhe sinjale B2C/B2B |

## 5. ARKITEKTURA E NYJEVE NË n8n

```
[Schedule Trigger 07:00]
        │
        ├─→ [HTTP Request]  → API-të zyrtare (TED, Google News, APP, e-Nabavki)
        └─→ [Apify Node]    → Social Media / Marketplace Scrapers
                    │
                    ▼
        [OpenAI Node: gpt-4o-mini / gpt-4o]
        + Structured Output Parser  → filtrim + përkthim SQ + JSON i saktë
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
[Google Sheets Node]     [Telegram Node]
   (baza e të dhënave)      (alert real-time)
```

- **Trigger:** Schedule Trigger – çdo ditë në orën **07:00**
- **Data Retrieval:** HTTP Request Node (API zyrtare) / Apify Node (scrapers)
- **AI Processing:** OpenAI Node, model `gpt-4o-mini` ose `gpt-4o`, me Structured Output Parsers
- **Storage & Alert:** Google Sheets Node + Telegram Node

## 6. RREGULLAT E PUNËS – PHASING (HAP PAS HAPI)

**RREGULL KRYESOR:** Mos jep të gjithë kodin/arkitekturën në një përgjigje të vetme.
Punohet fazë pas faze, me konfirmim para kalimit në fazën tjetër.

| Faza | Përmbajtja | Statusi |
|---|---|---|
| **FAZA 0** | Setup: n8n (Docker), Telegram bot, Google Sheet, OpenAI key | 🟡 Veprime për pronarin |
| **FAZA 1** | Lista e Keywords ndërkombëtare (DE, IT, MK, SQ, EN) + zgjedhja e burimeve të para | ✅ Kryer |
| **FAZA 2** | Konfigurimi i Trigger-ave dhe mbledhjes së të dhënave (HTTP Request & Apify) në n8n | ✅ Kryer (TED · DE+IT, testuar live) |
| **FAZA 3** | System Prompt + ndërlidhja e OpenAI Node për filtrim & përkthim në Shqip | ✅ Kryer (prompt gati) |
| **FAZA 4** | Google Sheets + Telegram Bot + gjenerimi i JSON-it për Import në n8n | 🟡 Radha (kërkon kredencialet e Fazës 0) |

## 7. STRUKTURA E FILE-VE TË PROJEKTIT

```
vegasolar-leadgen/
├── docs/
│   ├── 00-KONTEKSTI-MASTER.md              ← ky file
│   ├── 01-FAZA-1-KEYWORDS.md               ← keywords 5 gjuhë + CPV + burimet
│   ├── 02-FAZA-2-TRIGGER-DHE-TED-API.md    ← Schedule Trigger + TED API + Code Node
│   ├── 03-SETUP-FILLESTAR.md               ← n8n, Telegram, Google Sheets, OpenAI (veprime)
│   ├── 04-FAZA-3-SYSTEM-PROMPT-OPENAI.md   ← System Prompt + LLM Chain + Output Parser
│   └── 05-VENDIM-INFORMATION-EXTRACTOR.md  ← Information Extractor: jo Faza 3, po Faza 5
└── n8n/                                    ← workflow JSON (Faza 4)
```

## 8. FAKTE TEKNIKE TË VERIFIKUARA (testuar live, mos i rihetuar)

- **TED Search API:** `POST https://api.ted.europa.eu/v3/notices/search` — **pa API key**, publik.
- Emrat e fushave janë **eForms kebab-case** (`classification-cpv`), jo kodet e vjetra (`PC`, `CY`).
- `IN (...)` pranon vlera të ndara me **hapësirë**, jo presje. `limit` max = 100.
- Fusha `links` kthehet **gjithmonë** (4.5 KB/tender) — duhet hequr me Code Node.
- `notice-title` kthehet në **24 gjuhë**; `description-proc` vetëm në gjuhën origjinale.
- `total-value` shpesh i pabesueshëm (kthen `1`).
- Volumi CORE-CPV: **DE ~34/muaj, IT ~4/muaj** (Itali e ulët — tenderat nën pragun e BE-së
  shkojnë në ANAC/MEPA, jo në TED).

## 9. VENDIME (DECISION LOG)

| # | Pyetja | Statusi |
|---|---|---|
| 1 | Nga cili shtet/burim nis testimi i parë? | ✅ **A – TED API (Gjermani + Itali)** |
| 2 | API key për TED | ✅ **Nuk duhet** – Search API është publik |
| 3 | Plani Apify (falas / me pagesë) | ⚪ Shtyhet (jo e nevojshme për TED) |
| 4 | Chat ID i Telegram bot-it + ID e Google Sheet | 🟡 Udhëzimet: [03-SETUP-FILLESTAR.md](03-SETUP-FILLESTAR.md) |
| 5 | Të shtohen CPV-të e gjera `09300000` + `71314100`? | ❓ Vendim pas testimit të Fazës 3 |
| 6 | AI-filtri: strikt apo i gjerë (elektrike/EV pa PV)? | ✅ **Strikt me derë të hapur** – PV/solar/BESS i detyrueshëm; EV & elektrike vetëm bashkë me PV. Ndryshohet me 1 rresht (04-FAZA-3 §3) |
| 7 | Gjermani/Itali apo Shqipëri si treg parësor? | ❓ **E hapur.** Profili (§1.b) tregon treg shqiptar; TED-i DE+IT është ndërtuar dhe punon. Rekomandim: shto burimin **APP Shqipëri** si Fazë 5 |
| 9 | Të përdorim nyjen **Information Extractor**? | ✅ **Jo në Fazën 3** (detyra ka përkthim = gjenerim). **Po në Fazën 5** për APP/e-Nabavki. [05-VENDIM](05-VENDIM-INFORMATION-EXTRACTOR.md) |
| 8 | Të shtohet `lead_priority` + `capacity_kwp` në output? | ❓ Opsion i propozuar në 04-FAZA-3 §10 – prioritizim sipas shkallës 300 kWp–1.5 MWp |
