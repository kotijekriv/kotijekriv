# WhatsApp AI bot: brainstorm i istraživanje

> Status: istraživanje, još nema koda. Datum: 2. 10. 2026.
> Cilj: bot za mene i ekipu u WhatsApp grupi koji (1) piše u stilu naučenom iz
> exportanih razgovora, (2) komentira dnevnu politiku "na našu foru", (3) ima
> "mehaničke" funkcije (trigger riječi, provjera novih članaka…), i kasnije (4)
> da bude open source pa ga i drugi mogu složiti za sebe.

---

## 1. Kako se bot uopće spaja na WhatsApp

Postoje tri puta. Za bot u grupi prijatelja realan je samo prvi.

### A) Neslužbeno: bot kao "povezani uređaj" (kao WhatsApp Web)

Ovo gotovo sigurno koristi i prijatelj sa screenshota.

- Bot je program koji se WhatsAppu predstavi kao **još jedan povezani uređaj**,
  isto kao WhatsApp Web ili Desktop.
- Na mobitelu s brojem bota: *Postavke → Povezani uređaji → Poveži uređaj* i
  skeniraš **QR kod koji ispiše bot** (ili upišeš pairing kod).
- Od tog trenutka bot prima sve poruke iz razgovora i grupa u kojima je taj broj
  i može slati poruke u njegovo ime.
- **Ne treba otvorena desktop aplikacija ni browser.** Biblioteka sama
  "priča" WhatsAppov protokol preko WebSocketa.
- Tvoj broj = račun bota. Dodaš taj broj u grupu kao bilo kojeg člana.

Biblioteke i alati:

| Alat | Jezik | Kako radi | Napomena |
|---|---|---|---|
| **Baileys** (`WhiskeySockets/Baileys`) | JS/TS (Node 20+) | izravno WebSocket, bez browsera | najpopularniji, lagan (~50 MB RAM), MIT licenca; 7.x je još RC, 6.7.x stabilan |
| **whatsapp-web.js** | JS | pokreće skriveni Chrome s WhatsApp Webom | jednostavniji API, ali težak (~0,5 GB RAM) i puca kad WhatsApp promijeni web |
| **whatsmeow** / **neonize** | Go / Python wrapper | izravno protokol | dobar ako se ide u Python |
| **WAHA** | Docker + REST | omotač oko gornjih (WEBJS, NOWEB, GOWS) | pokreneš kontejner, skeniraš QR, dobivaš webhookove; bot onda pišeš u bilo kojem jeziku |
| **Evolution API** | Docker + REST | omotač oko Baileysa (+ službeni API) | slično kao WAHA, ima integracije (n8n, Typebot…) |

**Mane:**

- **Krši WhatsAppove uvjete korištenja → postoji rizik bana broja.** Najrizičnije
  je masovno slanje porukama nepoznatima (spam, marketing). Bot koji odgovara
  ekipi u jednoj ili dvije grupe ima puno manji rizik, ali nije nula. Zato je
  **zaseban broj (bonbon) dobra ideja**: tvoj glavni broj nije u igri.
- **Mobitel s tim brojem mora se spojiti na internet barem jednom u 14 dana**,
  inače se povezani uređaji (dakle i bot) odjave.
- Kad WhatsApp promijeni protokol, biblioteke ponekad neko vrijeme ne rade dok
  ih ne ažuriraju.
- Baileys 7.x prelazi na WhatsAppove "LID" identifikatore umjesto brojeva u
  grupama. To je bitno kod prepoznavanja tko je što napisao i @spominjanja.

### B) Službeno: WhatsApp Business Platform (Cloud API, Meta)

Meta ti preko webhooka šalje poruke na tvoj server, a ti odgovaraš preko
njihovog API-ja. Legalno je i stabilno, **ali za ovaj projekt ne valja**:

1. **Od 15. 1. 2026. Meta je zabranila "general-purpose AI chatbotove"** na
   Business API-ju (ChatGPT, Perplexity i sl. su morali otići). Dopušten je AI
   samo uz neku poslovnu svrhu: podrška, rezervacije, narudžbe.
2. **Grupe:** Groups API postoji, ali samo za *Official Business Account*
   (verificirani, "zelena kvačica"), **najviše 8 članova** po grupi, a članovi
   ulaze samo preko invite linka.
3. Treba Meta Business verifikacija, naplata po porukama ili predlošcima, puno
   birokracije.

Ovo ima smisla tek za neki kasniji ozbiljan ili poslovni projekt.

### C) Zaobilazno

- **Matrix bridge** (`mautrix-whatsapp`): WhatsApp se premosti u Matrix pa se bot
  piše za Matrix. Ispod opet koristi whatsmeow (povezani uređaj), dakle isti
  rizik, a infrastrukture je više. Nema smisla za početak.
- **Telegram / Discord**: ako ekipa nije baš vezana za WhatsApp, tamo su botovi
  **službeno podržani, besplatni i bez rizika bana**. Telegram Bot API je daleko
  najjednostavniji. Vrijedi imati na umu za open-source verziju (konektor kao
  zamjenjivi modul).

---

## 2. Što konkretno s brojem (bonbon)

1. Registriraj WhatsApp (ili **WhatsApp Business** app, besplatna) s tim brojem.
   SMS ili poziv za verifikaciju treba primiti samo jednom.
   - WhatsApp Business app može stajati **na istom mobitelu** kao tvoj običan
     WhatsApp, s drugim brojem. Ne treba poseban mobitel.
   - Ili stari Android u ladici na Wi-Fi-ju s tim brojem.
2. Postavi ime i profilnu da se vidi da je bot (npr. "🤖 Botko").
3. Kad bot bude postojao: povežeš ga kao uređaj (QR), dodaš broj u grupu i to je
   to.
4. Pazi:
   - otvori tu aplikaciju barem jednom u 14 dana (vidi gore);
   - **prepaid broj se gasi ako dugo nema nadoplate.** Ako broj istekne, operater
     ga s vremenom može dodijeliti nekom drugom, a taj netko onda može preuzeti
     WhatsApp račun. Provjeri bonbon pravila i nadoplaćuj povremeno.

---

## 3. Server ili kompjuter?

Bot je **program koji mora stalno raditi** jer drži otvorenu vezu s WhatsAppom.
Gdje god on radi 24/7, tamo je "server". Opcije:

| Gdje | Cijena | Za što |
|---|---|---|
| **Tvoj laptop** | 0 € | razvoj i testiranje. Kad zatvoriš laptop, bot "spava". Poruke ne propadaju (dobije ih kad se spoji), ali ne odgovara odmah. |
| **Raspberry Pi / stari laptop doma** | jednokratno + malo struje | trajno rješenje, sve ostaje doma |
| **VPS** (npr. Hetzner, ~4–5 €/mj; Oracle Cloud ima besplatni tier) | par eura mjesečno | trajno rješenje, radi i kad nema struje doma |

Bitno: **sam AI model ne radi na tvom serveru** (osim ako želiš lokalni model).
Server samo prima poruke i zove API (Claude, DeepSeek, OpenAI…). Zato je dovoljan
najmanji VPS ili Raspberry Pi, posebno uz Baileys koji ne treba browser.

---

## 4. Arhitektura (koncept)

```mermaid
flowchart LR
    WA[WhatsApp grupa] <--> C[Konektor<br/>Baileys / WAHA]
    C <--> R[Jezgra bota<br/>usmjeravanje poruka]
    R --> T[Trigger riječi<br/>Perišić, Bilić…]
    R --> M[@spominjanje / reply<br/>na bota]
    S[Raspored / cron] --> N[Dnevni komentar vijesti]
    W[Watcheri<br/>RSS, nove kolumne] --> R
    T --> L[LLM sloj]
    M --> L
    N --> L
    L --> P[(persona.md +<br/>primjeri stila)]
    L --> H[(povijest grupe<br/>SQLite)]
    L --> API[Claude / DeepSeek /<br/>OpenRouter API]
```

Moduli (svaki uključiv ili isključiv u configu):

- **Odgovor kad ga netko spomene** (`@Botko`) ili odgovori na njegovu poruku.
- **Trigger riječi**: "Perišić", "Bilić" → s nekom **vjerojatnošću** (npr. 30 %)
  i **cooldownom** (npr. najviše jednom na sat), da ne postane naporan.
- **Nasumično ubacivanje** u razgovor (rijetko).
- **Dnevni komentar politike** u zadano vrijeme (cron).
- **"Mehaničke" provjere** bez AI-ja: "ima li nova kolumna na Indexu", "novi
  članak o X". To je samo RSS ili scraping plus pamćenje što je već javljeno.
- **Memorija**: zadnjih N poruka grupe kao kontekst, da bot zna o čemu se priča.

---

## 5. Učenje stila iz exportanih poruka

### Format exporta

WhatsApp export je `.txt`, otprilike:

- iOS: `[02.10.2026. 14:35:12] Ivan: poruka`
- Android: `02.10.2026. 14:35 - Ivan: poruka`

Format datuma ovisi o jeziku i regiji mobitela. Neke poruke idu u više redova,
a ima i posebnih redova (`<Medij izostavljen>`, "Ova poruka je izbrisana",
sistemske poruke). Export ide **do 40.000 poruka bez medija** (10.000 s medijima)
po razgovoru. Prvi korak je skripta koja to parsira u strukturirani oblik:
`{vrijeme, autor, tekst, odgovor_na}`.

### Četiri razine (od najjednostavnije)

**1. Samo persona prompt (`persona.md`)**
Opišeš osobnost, poštapalice, interpunkciju (sve malim slovima? bez kvačica?
emojiji?), duljinu poruka, omiljene teme i fore. Brzo je, ali model "glumi" i
često zvuči generički.

**2. Persona + puno stvarnih primjera ("many-shot")**: preporuka za start
Skripta iz exporta izvuče stotine do par tisuća karakterističnih poruka osobe
(i kratke razmjene: poruka → njen odgovor) i stavi ih u system prompt. Današnji
modeli imaju ogroman kontekst (Claude do 1M tokena), a **prompt caching**
ponavljanje istog dugog prompta čini puno jeftinijim. Ovo daje najbolji omjer
truda i rezultata.

**3. RAG (pretraživanje po prošlim porukama)**
Sve stare poruke se indeksiraju (embeddingi). Kad se u grupi pojavi tema (npr.
"Plenković", "porez", "Dinamo"), bot dohvati **kako je ta osoba prije reagirala
na slične teme** i ubaci to kao primjere. Tako se hvata i "sadržaj": stavovi,
interne fore i šale koje se ponavljaju.

**4. Fine-tuning (treniranje modela)**
Model se trenira na parovima *(zadnjih N poruka razgovora → odgovor osobe X)*.

- Opcije: open-weights model (Qwen, Gemma, Llama, Mistral, **EuroLLM** koji
  službeno podržava hrvatski) + **LoRA/QLoRA** preko **Unsloth** (Google Colab
  ili iznajmljeni GPU; 7–8B model stane u 8–16 GB VRAM-a). Druga opcija je
  hosted fine-tuning (OpenAI, Together, Fireworks…).
- Claude se ne može fine-tunati preko javnog API-ja, s njim ide samo pristup 1–3.
- Prednost: najbolje hvata "glas" (kratke poruke, sleng, tipfelere).
- Mane: mali modeli su slabiji u hrvatskom i slabije "kuže" politiku; model
  treba negdje hostati (GPU) ili platiti hosting; treba par tisuća dobrih
  primjera; teže se održava.
- Moguć je i hibrid: veliki model smisli *što* reći, a mali fine-tunani model to
  *prepiše* u "glas" osobe.

**Plan:** krenuti s razinom 2, dodati 3, a 4 probati kao eksperiment tek ako
treba.

**Kako mjeriti je li dobro, i to zabavno:** "slijepi test" s ekipom. Pomiješaš
prave stare poruke i generirane pa ekipa pogađa koje su prave.

---

## 6. Komentiranje dnevne politike "na foru"

- **Izvor vijesti:**
  - **RSS** je besplatan, pouzdan i deterministički. Index ima feedove, npr.
    `index.hr/rss/vijesti-hrvatska`, `.../vijesti-svijet`. Slično imaju N1,
    Telegram.hr, HRT, Jutarnji…
  - ili **web search alat** samog modela (npr. Claude web search), fleksibilnije
    ali skuplje.
- **Tok:** cron (npr. 8:00 i 16:00) → pokupi naslove i sažetke → model odabere
  1–2 najsočnije teme → napiše komentar u stilu persone → pošalje u grupu,
  **s linkom**.
- **Na zahtjev:** "@Botko šta kažeš na ovo" + link → bot pročita članak i
  prokomentira.
- **Pazi na izmišljanje:** neka komentira samo ono što piše u dohvaćenom članku.
- Modeli imaju vlastite granice: sarkazam i zajebancija na račun politike prolaze,
  uvrede i kleveta konkretnih ljudi ne (i ne trebaju). Različiti modeli su
  različito "zločesti". To treba isprobati.

---

## 7. Modeli i okvirni trošak

| Model | Cijena (ulaz / izlaz po 1M tokena) | Napomena |
|---|---|---|
| Claude Sonnet 5.5 | $2 / $10 (cache read $0,20) | odličan hrvatski i humor, prijatelj kaže "Sonnet je dovoljan" |
| Claude Haiku 4.5 | $1 / $5 | jeftiniji, brži |
| Claude Opus 5.5 | $4 / $20 | najjači, ovdje vjerojatno nepotreban |
| DeepSeek V4 Flash | ~$0,15–0,44 / ~$0,60–1,32 (ovisi o dobu dana) | jako jeftino, to prijatelj koristi; serveri su u Kini |
| Lokalni model (Ollama) | 0 € po poruci | treba jak hardver, hrvatski slabiji |
| OpenRouter | ovisi o modelu | jedan API za sve modele, lako mijenjati i uspoređivati |

**Gruba procjena:** jedna kratka poruka ≈ 15 tokena, pa je persona s ~300
primjera ≈ 5.000 tokena. Uz kontekst razgovora to je ~7.000 tokena po pozivu.
Uz ~50 odgovora bota dnevno:

- Sonnet 5.5: oko **10–20 €/mj** bez cachea, s cacheom manje;
- Haiku: oko pola toga;
- DeepSeek: **oko 1 €/mj**.

Trigger riječi i cooldown filtriraju se u kodu, pa se model zove samo kad
stvarno treba. Što više primjera stila u promptu, to je skuplje po pozivu.

---

## 8. Privatnost, etika, pravila

- **Exportani razgovori sadrže tuđe poruke i osobne podatke → nikad u git
  repo.** Mapa `data/` ide u `.gitignore`. Open source znači samo kod i
  izmišljeni primjer persone, nikad prave poruke.
- Ako bot imitira **konkretnog prijatelja**, pitaj ga. Bot neka bude jasno
  označen kao bot.
- Ekipa treba znati da bot **čita grupu i šalje dijelove poruka vanjskom API-ju**
  (Anthropic, DeepSeek…).
- Rizik bana broja (vidi 1A). Zato bez spama, bez masovnog dodavanja u grupe i
  bez pisanja nepoznatima.

---

## 9. Ideja za open-source strukturu

Cilj je da svatko može uzeti projekt, ubaciti svoju personu i uključiti module
koje želi:

```
whatsapp-persona-bot/
├── personas/
│   └── primjer.md          # osobnost (prompt.md iz screenshota)
├── config.example.yaml     # grupe, triggeri, vjerojatnosti, cooldown, raspored, model
├── src/
│   ├── connectors/         # whatsapp (Baileys), kasnije telegram…
│   ├── modules/            # mention, triggers, random, daily-news, rss-watch
│   └── llm/                # pozivi modela, sastavljanje prompta, RAG
├── tools/
│   ├── parse-export        # WhatsApp .txt → JSON
│   └── build-style-examples# odabir karakterističnih poruka za personu
├── data/                   # (gitignored) exporti, baza, sesija
├── docker-compose.yml      # deploy jednom naredbom
└── README.md
```

- **Jezik:** Baileys je JS/TS, pa je **TypeScript (Node.js)** prirodan izbor za
  bota (i prijatelj koristi JS). Python ima smisla za alate oko podataka i
  fine-tuninga. Moguće je i kombinirati.
- **Licenca:** MIT ili Apache-2.0.
- Projekt bi trebao imati **vlastiti repo**, ne ovaj profilni
  (`kotijekriv/kotijekriv` je repo čiji README se prikazuje na GitHub profilu).

---

## 10. Prijedlog faza

0. **Odluke** (pitanja ispod).
1. **MVP:** Baileys + testna grupa (ti + bot) + odgovor na @spominjanje preko
   LLM-a + jednostavan `persona.md`.
2. **Trigger riječi** s vjerojatnošću i cooldownom (Perišić/Bilić fora).
3. **Parser exporta + persona s pravim primjerima stila** → slijepi test s
   ekipom.
4. **Vijesti:** RSS + dnevni komentar + komentar na poslani link.
5. **Deploy** na VPS ili Raspberry Pi (Docker), da radi 24/7.
6. **Eksperimenti:** RAG, fine-tuning.
7. **Open source:** čišćenje, README, primjeri, licenca.

## 11. Otvorena pitanja

1. **JS/TS ili Python?**
2. **Gdje će bot živjeti:** laptop za početak, pa Raspberry Pi ili VPS?
3. **Koji model i budžet:** Claude (kvaliteta) ili DeepSeek (cijena), ili oboje
   preko OpenRoutera pa usporedba?
4. **Čiji stil:** tvoj, stil konkretnog prijatelja ili izmišljeni lik složen od
   stila cijele ekipe? Može i više persona.
5. **Export:** koliko poruka, koliko ljudi, iOS ili Android format?
6. **Ekipa:** jesu li OK da bot čita grupu i da imitira nekoga?
7. **Novi repo** za projekt: ime?

---

## Izvori

- Baileys: https://github.com/WhiskeySockets/Baileys · https://www.npmjs.com/package/baileys
- Baileys vs whatsapp-web.js: https://whatsapp.checkleaked.cc/blog/baileys-vs-whatsapp-web-js
- WAHA: https://github.com/devlikeapro/waha
- Evolution API: https://github.com/evolution-foundation/evolution-api
- Meta zabrana general-purpose AI chatbotova (od 15. 1. 2026.): https://respond.io/blog/whatsapp-general-purpose-chatbots-ban · https://techradar.com/ai-platforms-assistants/meta-will-ban-rival-ai-chatbots-from-whatsapp
- Službeni Groups API (OBA, max 8 članova): https://developers.facebook.com/documentation/business-messaging/whatsapp/groups · https://www.unipile.com/whatsapp-group-api/
- Rizik bana kod neslužbenih alata: https://zylos.ai/research/2026-01-26-whatsapp-api-automation/ · https://achiya-automation.com/en/blog/whatsapp-spam-detection-2026/
- Povezani uređaji i pravilo 14 dana: https://www.mursa.me/blog/whatsapp-multi-device · https://gallabox.com/blog/how-to-use-whatsapp-on-two-phones
- Index.hr RSS: https://www.index.hr/rss/info
- Unsloth (fine-tuning): https://unsloth.ai/docs/new/studio/start
- EuroLLM (podržava hrvatski): https://huggingface.co/blog/eurollm-team/eurollm-22b
- DeepSeek cijene: https://deepseek.ai/pricing
