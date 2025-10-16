# Vialundskolans MoodCam — README

En *helt lokal* webb-app som med hjälp av din webbkamera:

* detekterar ansikten, skattar känslor & ålder
* ritar roliga overlays (hattar/glasögon m.m.) som följer ansiktets rörelser
* visar en **moodmätare** som räknar “glad-sekunder”
* kan spela upp *diskreta* “spökklipp” i bakgrunden och maskera ut personer i förgrunden (Halloween-läge)
* sparar enkel daglig besökslogg lokalt och kan exportera CSV

All inferens sker i webbläsaren. **Ingen data lämnar datorn**.

---

## Innehåll

* [Tekniker](#tekniker)
* [En överblick](#en-överblick)
* [Snabbstart](#snabbstart)
* [Mappstruktur & resurser](#mappstruktur--resurser)
* [Inställningar i UI](#inställningar-i-ui)
* [Hur det fungerar (pipeline)](#hur-det-fungerar-pipeline)
* [Overlays & teman](#overlays--teman)
* [Peppande texter (peppjson)](#peppande-texter-peppjson)
* [Halloween: spökbakgrund](#halloween-spökbakgrund)
* [Moodmätare & besökslogg](#moodmätare--besökslogg)
* [Prestanda & kvalitet](#prestanda--kvalitet)
* [Tillgänglighet](#tillgänglighet)
* [Säkerhet & integritet](#säkerhet--integritet)
* [Felsökning](#felsökning)
* [Konfigurationskonstanter (koden)](#konfigurationskonstanter-koden)
* [Vanliga frågor](#vanliga-frågor)

---

## Tekniker

* **TensorFlow.js** (`@tensorflow/tfjs` + automatisk fallback till `tfjs-backend-wasm`)
* **face-api.js** (tinyFaceDetector + expressions + age/gender + 68-landmarks)
* **MediaPipe Selfie Segmentation** (maskar ut personer i förgrunden till spök-komposit)
* Vanilla **HTML/CSS/JS**, inga byggsteg krävs (statisk host räcker)

---

## En överblick

Gränssnittet består av en “lager-wrap” där tre plan ritas:

1. **Composite-canvas** – *bakgrundsspöke + livekamera semitransparent + personmask opak*
2. **Overlay-canvas** – etiketter (känsla/ålder), emojis, pepp, samt frivilliga ansiktsrutor/overlays
3. **Moodmätare** – fast panel i nederkant

Råvideon finns i DOM men är dold: vi visar i stället kompositen så Halloween-effekten fungerar.

---

## Snabbstart

1. **Placera filer** i en mapp med:

   * `index.html` (det här projektet)
   * `models/` (face-api.js modeller, tinyFaceDetector m.fl.)
   * `img/` (overlays, `manifest.json`, samt `assets/meta.json`)
   * `pepp.json` (frivilligt – pepptexter)
   * `ghosts/` (frivilligt – dina mp4-klipp)

2. **Starta en statisk server** (HTTPS rekommenderas för kameran):

   * Valfritt: `npx serve .` eller `python -m http.server` (lokalt kan kamera kräva `https://` eller `localhost` beroende på webbläsare).

3. **Öppna sidan** i en modern webbläsare (Chrome/Edge/Brave/Firefox), tillåt kameran.

4. Klicka **Starta kamera** → appen laddar modeller, startar kamera, och kör.

> **Tips:** Om kameran inte startar – se [Felsökning](#felsökning).

---

## Mappstruktur & resurser

En föreslagen struktur:

```
/ (roten där index.html ligger)
├─ models/                         # face-api.js modelfiler
│  ├─ tiny_face_detector_model-weights_manifest.json
│  ├─ face_expression_model-weights_manifest.json
│  ├─ age_gender_model-weights_manifest.json
│  ├─ face_landmark_68_model-weights_manifest.json
│  └─ ... (tillhörande bin)
├─ img/
│  ├─ manifest.json                # lista över overlays (kan ersättas av teman)
│  ├─ assets/                      # filer & per-filens passform i meta.json
│  │  ├─ meta.json
│  │  ├─ hats/..., glasses/... etc
│  └─ tema/
│     ├─ halloween/manifest.json
│     ├─ jul/manifest.json
│     └─ ...
├─ ghosts/                         # mp4-klipp för spökbakgrund
│  ├─ peek_01.mp4
│  ├─ walk_wall_02.mp4
│  └─ flash_03.mp4
└─ pepp.json                       # { "kategori": ["text", ...], ... }
```

### `pepp.json` (exempel)

```json
{
  "Allmänt": [
    "Heja dig!",
    "Du är grym!",
    "Snyggt fokus!"
  ],
  "Halloween": [
    "Buu... 😉",
    "Modigt leende!"
  ]
}
```

### `img/manifest.json` (exempel)

```json
{
  "hats": [
    { "file": "assets/hats/party_01.png", "prob": 1, "fit": "hat", "wFactor": 1.1, "yOffset": -0.12 }
  ],
  "glasses": [
    { "file": "assets/glasses/round_02.png", "prob": 1.2, "fit": "glasses", "wFactor": 2.0 }
  ],
  "eyes": [], "mouth": [], "facial_hair": []
}
```

### `img/assets/meta.json` (exempel)

Per fil-override för passform, skala m.m. (sammanslås över `manifest.json`):

```json
{
  "assets/glasses/round_02.png": { "fit": "glasses", "wFactor": 2.05, "yOffset": 0.06 }
}
```

> **fit** stöds: `eyes`, `glasses`, `mouth`, `beard` (*facial_hair*), `hat`
> Default gissas från sökvägen om `fit` saknas.

---

## Inställningar i UI

Öppna **Settings** uppe till vänster. Viktiga kontroller:

* **Kamera**: välj videokälla.
* **Konfidens (etikett)**: min. känslo-konfidens för att visa etiketten.
* **Detector size**: inmatningsstorlek till TinyFaceDetector (större = mer noggrant men tyngre).
* **Detektor-tröskel**: score-threshold (0.40–0.80).

**Peppande one-liners**

* Aktivera/avaktivera.
* Välj *Kategori* (fylls från `pepp.json`).

**Overlays & teman**

* Aktivera overlays.
* Välj **Tema** (läser `img/tema/<tema>/manifest.json`).
* Slå av/på Hattar/Glasögon/Ögon/Munnar/Skägg.
* Visa ansiktsruta (debug/estetik).

**Kamera – bild (filter)**

* Ljusstyrka, Kontrast, Mättnad, Ton, Temperatur (endast visuell effekt).
* **Spegelvänd bild** (spegelvänder videokällan, overlay kompenserar).

**Besökslogg**

* Ladda ner CSV.
* Nollställ (tömmer `localStorage` för loggen).

**Bakgrundseffekt (Halloween)**

* Aktivera.
* **Frekvens**: Sällan / Ibland / Ofta

  * *Sällan:* snitt ~60 min
  * *Ibland:* snitt ~30 min
  * *Ofta:* snitt ~10 min
* **Bara när någon syns** (kräver ansikte i bild).
* **Bakgrundsgenomskinlighet** (hur tydlig live-bakgrunden är under spökklipp).
* **Spela spöke nu** (forcerar ett klipp direkt).

**Moodmätare – mål**

* Mål (sekunder glad-tid): anger 100% i mätaren.

---

## Hur det fungerar (pipeline)

1. **Init & backend**
   Försöker `webgl` → fallback till `wasm`. Laddar face-modeller; försöker även 68-landmarks (om det går **aktiveras precisionspassning** för overlays).

2. **Kamerastart**
   `getUserMedia` öppnar vald kamera. Dold `<video>` fyller **composite-canvas** (visas), och **overlay-canvas** ovanpå.

3. **Ansikten & känslor (60 fps best effort)**

   * `TinyFaceDetector` (med `inputSize` & `scoreThreshold` från UI)
   * Expressions + Age/Gender
   * (Valfritt) 68-landmarks för ögon/mun/käklinje → ger stabil **pose** (ögon-centrum, ögonavstånd, vinkel).
   * En enkel **IoU-baserad spårning** matchar nya rutor till tidigare *tracks*.
   * Smoothing:

     * bbox: pos α=0.35, storlek α=0.75
     * pose: α=0.15
     * ålder: glidande 10 s fönster
   * **Warmup 3 s** innan glad-tid börjar räknas för en person.

4. **Ritning**

   * **Composite:** ghost (botten) → live (semi-opak) → **personer opakt** (via MediaPipe-mask).
   * **Overlay:** etiketter, emoji, pepptexter, *valfria* ansiktsrutor och 2D-overlays som följer pose.

5. **Moodmätare**
   Summerar glad-sekunder över alla synliga. Konvergerar mjukt mot målet (ease 0.07).

6. **Logg**
   Per dag (YYYY-MM-DD) ökas besökare första gången en track uppnår warmup. Lagring i `localStorage`.

---

## Overlays & teman

* **Teman** väljs i UI.

  * Standard: `img/manifest.json`
  * Tema: `img/tema/<namn>/manifest.json`

* **Viktning:** varje objekt kan ha `"prob"` för att styra sannolikhet.

* **Passform:** från `manifest.json` eller per fil i `img/assets/meta.json`

  * `fit`: `eyes | glasses | mouth | beard | hat`
  * `wFactor`: relativ bredd utifrån referens (ögonavstånd/munbredd/ansiktsbredd)
  * `xOffset`/`yOffset`: i **referensbredd** (positiva nedåt/höger)
  * `rot`: `false` för att **inte** rotera med ögonlinjen
  * `flipX`: spegelvänd just denna asset

> Om 68-landmarks inte kunde laddas **deaktiveras overlay-passning** (ritas ej) tills landmarks blir tillgängliga.

---

## Peppande texter (`pepp.json`)

* Format: `{ "Kategori1": ["text1", "text2"], "Kategori2": [...] }`
* Välj *Alla* eller specifik kategori i UI.
* En slumpad text **låses per person** tills spåret försvinner.

---

## Halloween: spökbakgrund

**Idé:** Spela upp korta mp4-klipp *bakom* livevideon medan personer i förgrunden maskas tillbaka ovanpå (så spöken upplevs i bakgrunden).

* Bakgrundskomposit kör i en egen `requestAnimationFrame`-loop *efter* kameran startats.
* **Selfie Segmentation** ger en binär mask per bildruta.
  Vi renderar: *ghost* → *live α=ghostBgAlpha* → *live klippt av mask (person = 1.0 opacitet)*.
* **Frekvens (slump):**

  * *Sällan:* ~3600 s medel (60 min)
  * *Ibland:* ~1800 s (30 min)
  * *Ofta:* ~600 s (10 min)
    (Poisson-liknande: varje sekund chans = 1/medel. Minsta gap: 120 000 ms.)
* **Bara när någon syns**: kräver att ansikten detekteras just då.
* **Knappar:** “Spela spöke nu 👻” för en direkt trigger.
* **Lägg till klipp:** placera `.mp4` i `ghosts/` och lägg filnamn i `GHOST_CLIPS` i koden.

**Tips för video:**
H.264, 720p eller lägre, korta klipp 1–5 s, loopvänliga eller tydliga “ender”, **tyst** (muted).

---

## Moodmätare & besökslogg

* **Mätare:** ökar med tiden som *glada ansikten* finns (efter warmup 3 s).
  Om flera glada samtidigt → **adderas** deras bidrag.
  Bidrag kan viktas med konfidens (på i koden).
* **Mål:** justeras i UI. 100% = uppnått mål (sekunder).
* **Daglig reset:** kl 19:30 (`RESET_H = 19`, `RESET_M = 30`).
* **Logg:** *första gången* varje person räknas (när warmup passerats).
  Lagra per dag i `localStorage`.
  **Export:** “Ladda ner CSV”.

---

## Prestanda & kvalitet

* **WebGL** används om möjligt – annars **WASM**.
* **Detector size**: 224–512 (UI). Större = mer noggrant, men tyngre.
* **Detector threshold**: 0.40–0.80 (UI).
* **Adaptiv input** (i koden): om många ansikten (≥3) → sänk `inputSize` till 288 automatiskt.
* **Landmarks** kräver mer GPU/CPU men ger mycket bättre overlay-passning.
* **Canvas storlek** synkas mot videons upplösning.
* **FPS** påverkas av:

  * svag hårdvara
  * många ansikten
  * stora overlaybilder
  * tunga ghost-videor

---

## Tillgänglighet

* Mätarens numeriska värde annonseras med `aria-live="polite"` (skärmläsare).
* Kontrast & storlekar uppfyller god läsbarhet; justera i CSS/`UI`-objektet vid behov.

---

## Säkerhet & integritet

* All beräkning körs **lokalt i webbläsaren**.
* Inga nätverksanrop för bild/ljud (förutom att hämta statiska script och dina lokala resurser).
* Ingen spårning. Logg sparas **endast** i din webbläsare (`localStorage`).

---

## Felsökning

**Kameran startar inte**

* Kör sidan över **HTTPS** eller via `localhost`.
* Kontrollera kamerabehörighet i webbläsaren / OS.
* Stäng andra appar som använder kameran (Teams/Meet/Zoom).
* Se konsolen för fel (t.ex. “Kunde inte läsa in AI-modellerna” → kontrollera `/models/`).

**Modeller laddas inte**

* Säkerställ att `models/` innehåller *weights_manifest.json* och binärer med korrekta filnamn.
* Kolla rätt sökväg i koden (`const base='./models/'`).

**Overlays syns inte**

* Landmarks kan saknas. Se logg: “Landmarks aktiverade”.
  Om ej aktiverade → kontrollera att `faceLandmark68Net` filer finns i `models/`.
* Kontrollera att `img/manifest.json` och krävd fil finns.
  Se nätverksfliken i devtools (“404” på bildfiler?).

**Ghost-effekt syns inte**

* `ghosts/` tomt eller fel filnamn i `GHOST_CLIPS`.
* Frekvensen kan vara låg (Sällan/Ibland). Testa **Spela spöke nu 👻**.
* `Bakgrundsgenomskinlighet` för hög → spöket drunknar. Testa lägre (t.ex. 0.75–0.85).

**Låg FPS**

* Sänk *Detector size* i UI (t.ex. 288–320).
* Avaktivera overlays/landmarks (tillfälligt).
* Minska video-upplösningen i `startCamera()` (t.ex. 960×540).
* Använd lättare ghost-klipp (kortare, lägre upplösning).

---

## Konfigurationskonstanter (koden)

I `index.html` högst upp i `<script>`:

```js
const UI = { labelFontSize:16, labelPad:4, pepFontSize:16, pepPad:4, faceBoxWidth:2 };
let EMO_CONF_MIN = 0.40;          // min konfidens för etikett
let DETECTOR_SIZE = 320;          // inmatningsstorlek till TinyFaceDetector
let DETECTOR_SCORE = 0.60;        // scoreThreshold

const HAPPY_WARMUP_SECONDS = 3;   // tid innan glad räknas
const CONTRIB_WEIGHT_BY_CONF = true;

const RESET_H = 19, RESET_M = 30; // daglig reset av mätare

// Smoothing
const POSE_ALPHA = 0.15;
const BBOX_POS_ALPHA  = 0.35;
const BBOX_SIZE_ALPHA = 0.75;

// Adaptiv input
const FRAME_OPTS = { ADAPTIVE_INPUT: true, MANY_FACES_THRESHOLD: 3, LOW_INPUT_SIZE: 288 };

// Pepp & ålder
const AGE_SMOOTH_WINDOW_MS = 10_000;

// Halloween-ghost
let ghostEnabled = true;
let ghostFacesOnly = true;
let ghostMode = 'sometimes'; // rare / sometimes / often
let ghostAvgSec = 120;       // skrivs över av modeToAvgSec()
let ghostBgAlpha = 0.88;     // live-bakgrunds opacitet
let minGapMs=120000;         // minsta gap mellan spök-klipp (~2 min)
```

Kartläggning för frekvens (i koden):

```js
function modeToAvgSec(m){
  if (m==='rare') return 3600;  // 60 min
  if (m==='often') return 600;  // 10 min
  return 1800;                  // 30 min (sometimes)
}
```

> **Vill du ändra defaultvärdena i UI?** Justera attributen `value`/`checked` på respektive `<input>` i HTML.

---

## Vanliga frågor

**Kan jag köra utan internet?**
Ja – om du hostar alla scripts/statik lokalt (inkl. MediaPipe Selfie Segmentation). I nuvarande kod laddas MediaPipe från jsdelivr. Byt `locateFile` till lokal sökväg och inkludera asset-filerna.

**Loggar ni något?**
Nej. Allt körs lokalt och logg sparas endast i din webbläsare.

**Hur byter jag tema?**
Lägg ett `manifest.json` i `img/tema/<namn>/` och välj temat i UI.

**Hur lägger jag till spökklipp?**
Placera `.mp4` i `ghosts/` och lägg till filnamnen i listan `GHOST_CLIPS` i koden.

**Hur ändrar jag typsnitt/utseende på etiketter/mätare/pepp?**
Justera `UI`-objektet (fontstorlekar/padding) samt CSS för paneler.

**Fungerar det på en äldre dator?**
Ja, men sänk *Detector size*, stäng av overlays eller ghost, och/eller låt WASM-backend ta över.

---

## Avslutning

MoodCam är avsiktligt **självförsörjande**, enkel att hosta och modifiera, och med ett UI som gör de vanligaste valen utan att du behöver ändra kod. Anpassa `pepp.json`, dina **tema-manifest**, och fyll `ghosts/` med korta, smakfulla klipp för bästa upplevelse.

Lycka till – och *SMILE*! 😄
