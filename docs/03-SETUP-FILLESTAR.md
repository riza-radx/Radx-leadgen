# FAZA 0 – SETUP FILLESTAR (n8n + Telegram + Google Sheets + OpenAI)

Statusi: 🟡 Veprime për ty · Krijuar: 2026-09-03
Konteksti: [00-KONTEKSTI-MASTER.md](00-KONTEKSTI-MASTER.md)

> **Koha totale:** ~45–60 minuta. Google OAuth është pjesa më e mërzitshme (~25 min),
> të tjerat janë të shpejta.

---

## 🖥️ HAPI 1 — n8n (10 min)

**Gjendja e makinës tënde (e kontrolluar):**
| Software | Versioni | Statusi |
|---|---|---|
| Node.js | v24.19.0 | ⚠️ Shumë i ri për n8n |
| npm | 11.17.0 | ✅ |
| Docker | 29.7.2 | ✅ **Rruga e rekomanduar** |

### ✅ Rruga A — Docker (rekomandimi)

n8n zyrtarisht mbështet Node **20–22**. Ti ke **24**, që mund të bllokohet nga
kontrolli i versionit (`engines`) i n8n-it. Docker e shmang plotësisht këtë problem,
sepse image-i vjen me Node-in e vet të saktë.

Hap **PowerShell** dhe ekzekuto:

```powershell
docker volume create n8n_data
docker run -d --name n8n --restart unless-stopped -p 5678:5678 -v n8n_data:/home/node/.n8n -e GENERIC_TIMEZONE="Europe/Tirane" -e TZ="Europe/Tirane" docker.n8n.io/n8nio/n8n
```

Shpjegim i flamurëve:
| Flamuri | Pse |
|---|---|
| `-d` | Punon në sfond (jo i lidhur me terminalin) |
| `--restart unless-stopped` | Rindizet vetë pas restartit të kompjuterit |
| `-v n8n_data:/home/node/.n8n` | **Kritik** – ruan workflow-et & kredencialet. Pa këtë, humbet çdo gjë |
| `-e GENERIC_TIMEZONE` | Cron-i `0 7 * * *` punon me orën e Tiranës, jo UTC |

Komanda të dobishme:
```powershell
docker logs -f n8n          # shiko log-et
docker stop n8n             # ndal
docker start n8n            # nis
docker pull docker.n8n.io/n8nio/n8n   # update (pastaj rm + run perseri)
```

### Rruga B — npx (nëse Docker nuk punon)
```powershell
npx n8n
```
Nëse jep gabim versioni Node-i, instalo Node 22 me `nvm-windows` dhe provo përsëri.

### Rruga C — n8n Cloud
`https://n8n.io` → Sign up (trial falas). **Përparësi e vërtetë:** për Google OAuth
merr një domain HTTPS të gatshëm dhe të shmang gjysmën e problemeve të Hapit 3.

### ▶️ Nisja e parë
1. Hap **http://localhost:5678**
2. Krijo llogarinë e pronarit (email + password) — ruaji, është lokale
3. **Settings → Timezone → `Europe/Tirane`** (pastaj po ashtu edhe në Workflow Settings)

---

## 🤖 HAPI 2 — Telegram Bot + Chat ID (5 min)

### 2.1 Krijo botin
1. Hap Telegram → kërko **@BotFather** → `/start`
2. `/newbot`
3. **Name** (emri i dukshëm): `VegaSolar Leads`
4. **Username** (duhet të përfundojë me `bot` dhe të jetë unik): p.sh. `vegasolar_leads_bot`
5. BotFather kthen tokenin:
   ```
   8123456789:AAF-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
   ⚠️ **Kjo është si fjalëkalim — mos e ndaj publikisht.** Ruaje.

### 2.2 Merr Chat ID-në
Boti **nuk mund** t'i shkruajë i pari askujt. Duhet ta nisësh ti bisedën:

1. Hap Telegram → kërko username-in e botit tënd → **Start** → shkruaj `pershendetje`
2. Në PowerShell (zëvendëso `<TOKEN>`):
   ```powershell
   curl "https://api.telegram.org/bot<TOKEN>/getUpdates"
   ```
3. Në përgjigje kërko:
   ```json
   "chat": { "id": 123456789, "first_name": "Anisa", "type": "private" }
   ```
   → **`123456789` është Chat ID-ja jote.**

### 2.3 (Rekomandohet) Grup ekipi në vend të bisedës private
Që leads-et t'i shohë i gjithë ekipi i shitjeve, jo vetëm ti:
1. Krijo grup Telegram → shto botin si anëtar
2. Shkruaj një mesazh në grup
3. Ekzekuto përsëri `getUpdates` → merr ID-në e grupit

> 📌 **ID-të e grupeve janë NEGATIVE** (p.sh. `-1001234567890`). Kopjoje me minusin,
> përndryshe Telegram Node kthen `chat not found`.

### 2.4 Kredenciali në n8n
n8n → **Credentials → New → Telegram API** → fusha **Access Token** → ngjit tokenin → Save.
Kliko **Test** — duhet të thotë `Connection tested successfully`.

---

## 📊 HAPI 3 — Google Sheets (25 min, pjesa më e ndërlikuar)

### 3.1 Krijo tabelën
1. Hap `https://sheets.new`
2. Emërtoje: **`VegaSolar – Leads Database`**
3. Riemërto tab-in e poshtëm (`Sheet1`) në → **`Leads`**

### 3.2 Merr Sheet ID-në
Nga URL-ja e shfletuesit:
```
https://docs.google.com/spreadsheets/d/1a2B3cD4eFgHiJkLmNoPqRsTuVwXyZ_abcdefghij/edit#gid=0
                                      └──────────── SHEET ID ────────────────────┘
```
→ Kopjo pjesën midis `/d/` dhe `/edit`.

### 3.3 Rreshti i kokës (rreshti 1) — kopjoji saktësisht

Ngjiti këto 11 kolona në rreshtin 1, secila në një celulë (A1 → K1):

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| `date_added` | `source_id` | `project_name` | `country_location` | `project_type` | `summary_sq` |

| G | H | I | J | K |
|---|---|---|---|---|
| `contact_info` | `source_platform` | `deadline` | `source_url` | `status` |

**Pse 11 kolona kur output-i ka vetëm 6:**
- `source_id` → **e detyrueshme për dedupe.** Pa të, `lookback_days = 3` do të mbushë
  tabelën me tre kopje të çdo tenderi.
- `date_added`, `deadline`, `source_url` → praktikë për punën e vërtetë të shitjeve
- `status` → kolona ku ekipi shkruan manualisht: `I ri` / `Kontaktuar` / `Ofertë e dërguar` / `Refuzuar`

### 3.4 Google Cloud Console — kredencialet OAuth2

> ⚠️ **Pse OAuth2 dhe jo Service Account:** Google i ndalon Service Account-et e krijuara
> **pas 15.04.2025** t'i lexojnë skedarët e "My Drive" — vetëm Shared Drives. Meqë do e
> krijoje tani, do të bllokohej. OAuth2 punon normalisht me llogari personale Gmail.

**a) Projekti dhe API-t**
1. Hap `https://console.cloud.google.com`
2. Lart → **Select a project → New Project** → emri `vegasolar-n8n` → Create
3. **APIs & Services → Library** → kërko dhe kliko **Enable** për **DY** API:
   - ✅ **Google Sheets API**
   - ✅ **Google Drive API** ← n8n e kërkon për të gjetur tabelën. Harrohet shpesh.

**b) OAuth consent screen**
1. **APIs & Services → OAuth consent screen**
2. User Type: **External** → Create
3. App name: `VegaSolar n8n` · User support email: emaili yt · Developer email: emaili yt
4. Save and Continue (Scopes → Save, Test users → hapi tjetër)
5. **Test users → + Add users → shto emailin tënd** (`anisa.sinaj@radx.app`)
   ⚠️ Pa këtë hap merr `Error 403: access_denied` kur provon të lidhesh.

**c) OAuth Client ID**
1. **APIs & Services → Credentials → + Create credentials → OAuth client ID**
2. Application type: **Web application**
3. Name: `n8n`
4. **Authorized redirect URIs → + Add URI** → ngjit:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
   *(Për n8n Cloud: `https://<subdomain-i-yt>.app.n8n.cloud/rest/oauth2-credential/callback`)*
   
   💡 Mos e shkruaj me dorë — n8n e shfaq **saktësisht** këtë URL në ekranin e kredencialit
   (hapi d). Kopjo prej tij dhe ngjit.
5. Create → ruaj **Client ID** dhe **Client Secret**

**d) Kredenciali në n8n**
1. n8n → **Credentials → New → Google Sheets OAuth2 API**
2. Kopjo **OAuth Redirect URL** që shfaq n8n → sigurohu që është identik me atë në Google
3. Ngjit **Client ID** + **Client Secret**
4. Kliko **Sign in with Google** → zgjidh llogarinë → *"Google hasn't verified this app"* →
   **Advanced → Go to VegaSolar n8n (unsafe)** → Allow
   *(Ky paralajmërim është normal — aplikacioni është i yti, në modalitet Testing.)*
5. Save

---

## 🧠 HAPI 4 — OpenAI API Key (5 min)

1. Hap `https://platform.openai.com/api-keys`
2. **Create new secret key** → emri `vegasolar-n8n` → kopjo (`sk-proj-...`)
   ⚠️ Shfaqet **vetëm një herë**. Ruaje menjëherë.
3. **Billing → Add payment method** → shto krediun minimal (~5 USD)
   Pa kredi, API-ja kthen `insufficient_quota` edhe me key valid — falas nuk punon.
4. n8n → **Credentials → New → OpenAI** → ngjit key-in → Save

**Kostoja e pritshme:** me `gpt-4o-mini` dhe ~40 tenderë/muaj (pas optimizimit të
Fazës 2 që kursen 88% të tokenave) → **cent të pakët në muaj**, praktikisht e
papërfillshme. 5 USD të mbajnë muaj me radhë. *(Verifiko çmimet aktuale në
`openai.com/api/pricing` — ndryshojnë periodikisht.)*

---

## ✅ CHECKLIST — çfarë duhet të kem para Fazës 4

Plotësoje dhe ma dërgo (ose thjesht konfirmo që i ke):

```
[ ] n8n punon në http://localhost:5678
[ ] Timezone = Europe/Tirane (Settings + Workflow Settings)
[ ] Telegram Bot Token:  ________________________
[ ] Telegram Chat ID:    ________________________   (negativ nese grup)
[ ] Google Sheet ID:     ________________________
[ ] Emri i tab-it:       Leads
[ ] 11 kolonat ne rreshtin 1: DONE
[ ] Kredenciali Google Sheets OAuth2 ne n8n: Testuar OK
[ ] Kredenciali OpenAI ne n8n: Testuar OK
[ ] Billing OpenAI aktiv
```

> 🔐 **Sigurie:** tokenat dhe key-t **mos i shkruaj në këto file** dhe mos i ngarko në git.
> Ato hyjnë vetëm në **Credentials** të n8n-it, që i ruan të kriptuara në volumin `n8n_data`.
> Në workflow-et dhe në JSON-in e eksportuar shfaqen vetëm si referencë ID, jo si vlerë.

---

## 🔨 NDËRKOHË — ndërto Fazën 2 në n8n

Pa pritur asnjë kredencial (TED nuk kërkon API key), mund të ndërtosh dhe testosh
menjëherë 5 nyjet e Fazës 2:

1. **Workflows → + Add workflow** → emri `VegaSolar – Lead Gen`
2. **Settings → Timezone → `Europe/Tirane`** (bëje të parën)
3. Shto nyjet sipas [02-FAZA-2-TRIGGER-DHE-TED-API.md](02-FAZA-2-TRIGGER-DHE-TED-API.md):
   `Schedule Trigger` → `Edit Fields (Set)` → `HTTP Request` → `Code` → `Filter`
4. Në **Set: CONFIG** vendos përkohësisht `lookback_days = 30`
5. **Execute Workflow** → pritje: ~38 items nga TED, ~35–38 pas Code Node-it
6. Kontrollo në output që `contact_email` dhe `description_raw` kanë vlera reale
7. Kthe `lookback_days` në `3`

Këtë mund ta bëj bashkë me ty; ose kur mbaron Hapin 1, kaloj në **Fazën 3** (System
Prompt + OpenAI Node) dhe në **Fazën 4** të jap workflow-in e plotë JSON gati për
**Import from File** — atëherë nuk të duhet të ndërtosh nyje me dorë fare, vetëm të
lidhësh kredencialet.

---

## 🔄 RENDI I REKOMANDUAR I PUNËS

| Radha | Veprimi | Koha | Bllokon çfarë |
|---|---|---|---|
| 1 | Docker → n8n punon | 10 min | Të gjithë |
| 2 | Telegram bot + Chat ID | 5 min | Fazën 4 |
| 3 | Google Sheet + 11 kolonat | 5 min | Fazën 4 |
| 4 | OpenAI key + billing | 5 min | Fazën 3 |
| 5 | Google Cloud OAuth | 25 min | Fazën 4 |
| 6 | Ndërto/testo Fazën 2 | 15 min | — |

> 💡 **Nis me 1 → 4** (25 min gjithsej). Google OAuth (hapi 5) është më i mërzitshmi
> dhe nuk bllokon Fazën 3 — bëje ndërkohë që ne punojmë System Prompt-in.
