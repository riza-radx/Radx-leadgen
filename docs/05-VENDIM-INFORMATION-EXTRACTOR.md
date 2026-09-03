# VENDIM TEKNIK — Information Extractor apo Basic LLM Chain?

Vendimi #9 · 2026-09-03 · Kërkuar nga pronari
Konteksti: [00-KONTEKSTI-MASTER.md](00-KONTEKSTI-MASTER.md) · [04-FAZA-3](04-FAZA-3-SYSTEM-PROMPT-OPENAI.md)

---

## PËRFUNDIMI

| Burimi | Nyja e saktë |
|---|---|
| **TED API** (Faza 2–3, tani) | ✅ **Basic LLM Chain + Structured Output Parser** — mos e ndrysho |
| **APP Shqipëri / e-Nabavki MK** (Faza 5) | ✅ **Information Extractor** + pastaj Basic LLM Chain |

**Përgjigja e shkurtër:** Information Extractor është nyja e gabuar për Fazën 3, por nyja
**e duhur** për burimet shqiptare dhe maqedonase që vijnë më pas. Ideja është e mirë —
vetëm në vendin e gabuar të pipeline-it.

---

## 1. PSE JO PËR FAZËN 3 (TED)

Detyra e Fazës 3 nuk është *nxjerrje* e të dhënave. Janë **tre** detyra, dhe vetëm një
prej tyre është nxjerrje:

| # | Detyra | Natyra | Information Extractor e bën? |
|---|---|---|---|
| 1 | `is_relevant` — a e vlen për VegaSolar? | **Gjykim biznesi** | 🟡 Me mundim |
| 2 | `project_type` — B2B apo B2C? | **Klasifikim** | ✅ Po (me `enum`) |
| 3 | `summary_sq` — përmbledhje në Shqip | **GJENERIM + PËRKTHIM** | ❌ Jo — nuk është nxjerrje |

**Problemi thelbësor:** `summary_sq` kërkon tekst që **nuk ekziston** në burim. Përshkrimi
origjinal është në gjermanisht; ne duam 2–4 fjali të reja në shqip. Information Extractor
është ndërtuar për të *nxjerrë* fusha që gjenden në tekst — jo për të prodhuar tekst të re.

**Problemi i dytë, praktik:** dokumentacioni i n8n thotë që Information Extractor
**shton automatikisht udhëzimet e vet të formatimit** mbi System Prompt Template-in tënd.
Kur prompti yt ka 120 rreshta me rregulla sjelljeje (relevancë me tre nivele, prompt-
injection guard, rregulla përkthimi), udhëzimet e n8n-it hyjnë sipër dhe nuk kontrollon
plotësisht se çfarë merr modeli. Me Basic LLM Chain, System Message-i shkon i pacenuar.

### Provë nga vetë n8n
Në listën e template-ve që dërgove ([n8n.io – Information Extractor](https://n8n.io/workflows/categories/other/?integrations=Information+Extractor)),
38 workflow-e përdorin këtë nyje. Ato që shfaqen janë: *Resume screening*, *CV analysis*,
*LinkedIn profile extraction*, *Resume generation*, *Upwork job alert processing*.

Të gjitha kanë të njëjtin profil: **një dokument i çrregullt hyn, fusha të strukturuara
dalin.** Asnjë përkthim, asnjë gjykim biznesi. Kjo është pikërisht natyra e nyjes.

---

## 2. KRAHASIMI

| | Information Extractor | Basic LLM Chain + Parser |
|---|---|---|
| Nyje gjithsej | 2 (nyja + model) | 3 (chain + model + parser) |
| Skema | E brendshme, 3 mënyra | Sub-node më vete |
| Kontrolli i System Prompt-it | 🟡 n8n shton udhëzimet e vet sipër | ✅ I plotë |
| Përkthim / gjenerim teksti | ❌ Jashtë qëllimit | ✅ Po |
| Prompt i gjatë me rregulla sjelljeje | 🟡 I ngushtë | ✅ Natyral |
| Nxjerrje nga tekst i çrregullt (HTML/PDF) | ✅ **Më e mira** | 🟡 Punon, më shumë punë |
| Output | `$json.output` | `$json.output` |

Të dyja kërkojnë një **Chat Model** sub-node dhe të dyja e kthejnë rezultatin në
`$json.output`. Ndryshimi nuk është teknik — është **përshtatje me detyrën**.

---

## 3. PSE PO — PËR FAZËN 5 (Shqipëri & Maqedoni)

Këtu situata përmbyset plotësisht.

TED jep **JSON të strukturuar** — Code Node-i i Fazës 2 e bëri nxjerrjen falas, me kod, pa
asnjë token. Nuk kemi nevojë për AI-nxjerrje.

APP Shqipëri dhe e-Nabavki **nuk kanë API**. Do të marrim:
- HTML të papërpunuar nga buletini
- ose tekst nga PDF
- pa fusha, pa etiketa, pa strukturë — emri i autoritetit kontraktor, objekti i
  prokurimit, fondi limit, afati dhe kontakti të gjitha të përziera në tekst të lirë shqip

**Kjo është saktësisht detyra për të cilën është ndërtuar Information Extractor.** Një
Code Node me regex do të prishej në javën e dytë, sapo buletini ndryshon formatin.

### Arkitektura e propozuar për Fazën 5

```
[HTTP Request: APP buletin]
        ↓
[HTML / Extract from PDF]  ← tekst i papërpunuar
        ↓
[Information Extractor]  ← NXJERRJE: autoriteti, objekti, fondi, afati, kontakti
        ↓                   (fakte që NDODHEN në tekst)
[Basic LLM Chain]        ← GJYKIM + PËRKTHIM (i njëjti System Prompt i Fazës 3)
        ↓
[Filter → Set → Sheets/Telegram]
```

Ndarja e përgjegjësive:
- **Information Extractor** → nxjerr *fakte*. Nuk gjykon, nuk përkthen, nuk shpik.
- **Basic LLM Chain** → gjykon relevancën dhe përkthen. I merr fakte të pastra, jo HTML.

Përfitimi: kur buletini i APP-së ndryshon format, ndryshon **vetëm** nyja e nxjerrjes.
Prompti i gjykimit — pjesa më e vështirë e punës — mbetet i paprekur.

> ⚠️ **Kosto:** dy nyje AI = dy thirrje API për njoftim. Me volumet e pritshme
> (dhjetëra në muaj) është e papërfillshme. Nëse APP-ja nxjerr qindra njoftime në ditë,
> vendos një filtër keywords **para** Information Extractor-it.

---

## 4. LEXIMI I DYTË — "extractor" si shërbim scraping-u

Nëse me *extractor* nënkuptonim një shërbim të jashtëm nxjerrjeje (jo nyje n8n), këtu
qëndron pengesa e vërtetë e Fazës 5: APP dhe e-Nabavki nuk kanë API.

| Mjeti | Përshtatja | Shënim |
|---|---|---|
| **HTTP Request + HTML node (n8n)** | 🥇 Provoje të parën | Falas. Punon nëse buletini është HTML i thjeshtë |
| **Firecrawl** | 🥈 | Kthen markdown të pastër nga faqe të komplikuara; ka `/extract` me skemë |
| **Apify** | 🥉 | Fleksibël, por kërkon actor të shkruar për portalin |
| **Browse AI / Octoparse** | 4 | Pa kod, mirë për tabela; abonim mujor |

**Rendi i saktë i punës:** para se të paguajmë për ndonjë shërbim, testojmë a mjafton
`HTTP Request` bosh kundrejt APP-së. Kjo verifikohet në 10 minuta — dhe TED-i tregoi që
testimi paraprak kursen supozime të gabuara (shih Zbulimet #1–#4 të [02-FAZA-2](02-FAZA-2-TRIGGER-DHE-TED-API.md)).

---

## 5. VEPRIMI

- ❌ **Mos ndrysho** asgjë në Fazën 3. Basic LLM Chain mbetet.
- ✅ **Mbaje** Information Extractor për Fazën 5 — është shënuar në planin e arkitekturës.
- ⏭️ Faza 4 vazhdon si planifikuar (Sheets + Telegram + JSON import).
