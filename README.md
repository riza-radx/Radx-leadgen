# VegaSolar — Sistem Inteligjent i Gjenerimit të Leads-eve

Automatizim n8n që skanon tenderat publike europiane për projekte **fotovoltaike**,
i filtron me AI sipas profilit real të VegaSolar, i përkthen në shqip dhe dërgon
leads-et e vlefshme në Telegram.

> **Burimi i së vërtetës:** [`docs/00-KONTEKSTI-MASTER.md`](docs/00-KONTEKSTI-MASTER.md).
> Lexoje të parin në çdo sesion pune.

---

## Arkitektura aktuale

```
[Schedule Trigger · 07:00 Europe/Tirane]
        │
        ▼
[HTTP Request] ──► TED Search API v3  (publik, pa API key)
        │          DE + IT · vetëm cn-standard/cn-social · dritare 3-ditore
        ▼
[Code: Normalizer] ─► pastrim, filtër mbetjesh, dedupe brenda run-it
        │
        ▼
[Remove Duplicates] ─► heq tenderat e dërguar në ekzekutime të mëparshme (çelës: source_id)
        │
        ▼
[AI Agent · gpt-5-mini] + Structured Output Parser
        │              relevancë · klasifikim B2B/B2C · përkthim në shqip
        ▼
[Filter: is_relevant] ──► [Telegram]
```

Nyja e Google Sheets rri **e shkëputur** me kërkesë — e ruajtur me mapping-un e plotë
nëse arkivi duhet rikthyer.

## Struktura

```
vegasolar-leadgen/
├── docs/          # specifikimi, vendimet, promptet — lexoji sipas numrit
└── n8n/           # eksporti JSON i workflow-it (backup i importueshëm)
```

## Fakte teknike të verifikuara live

Mos i rihetuar — të gjitha të provuara kundrejt TED-it real më 03.09.2026:

| Fakt | Vlera |
|---|---|
| TED Search API | `POST https://api.ted.europa.eu/v3/notices/search` — **pa API key** |
| Emrat e fushave | eForms **kebab-case** (`classification-cpv`) |
| `IN (...)` | vlerat ndahen me **hapësirë**, jo presje |
| `limit` max | **100** |
| Afati | fusha e saktë është `deadline-receipt-request`, **nuk** `deadline` |
| `links` | **nuk ka `self`**; `Object.values()[0]` kthen objekt → ndërto URL-në nga `publication-number` |
| `notice-title` | kthehet në 24 gjuhë; `eng` mund të mungojë |
| `total-value` | i pabesueshëm — nuk përdoret |
| Vëllimi | FT+DE/IT+cn: ~20/javë · 33% e njoftimeve pa filtër `notice-type` janë kontrata të dhëna |
| **Shqipëria në TED** | **0 rezultate në 90 ditë** — TED nuk mund të mbulojë tregun shqiptar |

## Rikthimi i workflow-it

n8n → Workflows → `⋯` → **Import from File** → zgjidh JSON-in nga `n8n/`.

Pas importit duhen rilidhur kredencialet me dorë (ID-të e tyre nuk kalojnë mes instancave).

## 🔐 Rregull sigurie

Tokenat e Telegram-it dhe API key-t **nuk hyjnë kurrë në këto file**. Ato jetojnë vetëm
në **Credentials** të n8n-it, i cili i ruan të kriptuara. Në workflow-in e eksportuar
shfaqen vetëm si referencë ID, jo si vlerë.

Eksporti i workflow-it përmban Chat ID-në e Telegram-it dhe URL-në e Google Sheet-it —
jo sekrete, por gjysmë-private. **Mbaje repo-n private.**
