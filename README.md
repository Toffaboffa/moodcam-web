# Vialundskolans MoodCam

**Kortfattat:** En helt lokal webb-app som använder din webbkamera för att upptäcka ansikten, estimera känslor & ålder, lägga på roliga overlays, spela diskreta ”spökklipp” i bakgrunden och räkna klassens ”glad-sekunder” i en moodmätare. **Allt körs i webbläsaren – ingen data lämnar datorn.**

Skapad av **Kristoffer**.

---

## Vad gör den?

- Detekterar ansikten och skattar känslor (t.ex. *Glad, Förvånad, Neutral*) och ålder.
- Lägger på *overlays* (hattar/glasögon m.m.) som följer ansiktet.
- Visar en **moodmätare** som ökar när personer är glada.
- Kan spela *spökklipp i bakgrunden* (Halloween-läge) och maskera tillbaka personer i förgrunden.
- Loggar antal ”besökare” per dag lokalt och kan exportera CSV.
- Kan köras helt offline om du hostar allt lokalt.

---

## Innehåll
- [Tekniker](#tekniker)
- [Snabbstart](#snabbstart)
- [Mappstruktur & resurser](#mappstruktur--resurser)
- [Inställningar i UI](#inställningar-i-ui)
- [Hur det fungerar (pipeline)](#hur-det-fungerar-pipeline)
- [Overlays & teman](#overlays--teman)
- [Peppande texter (peppjson)](#peppande-texter-peppjson)
- [Halloween: spökbakgrund](#halloween-spökbakgrund)
- [Moodmätare & besökslogg](#moodmätare--besökslogg)
- [Prestanda & kvalitet](#prestanda--kvalitet)
- [Tillgänglighet](#tillgänglighet)
- [Säkerhet & integritet](#säkerhet--integritet)
- [Felsökning](#felsökning)
- [Konfigurationskonstanter (koden)](#konfigurationskonstanter-koden)
- [Vanliga frågor](#vanliga-frågor)

---

## Tekniker
- **TensorFlow.js** (`@tensorflow/tfjs` + fallback till `tfjs-backend-wasm`)
- **face-api.js** (tinyFaceDetector + expressions + age/gender + 68-landmarks)
- **MediaPipe Selfie Segmentation** (för personmask i komposit)
- Vanilla **HTML/CSS/JS** (statisk host räcker)

---

## Snabbstart
1. Lägg filerna i en mapp:
   - `index.html`
   - `models/` (face-api.js-modeller)
   - `img/` (overlays + `manifest.json`, `assets/meta.json`)
   - `pepp.json` (valfritt)
   - `ghosts/` (valfritt, mp4-klipp)
2. Starta en statisk server (HTTPS/localhost för kamera):
   - `npx serve .` eller `python -m http.server`
3. Öppna sidan i en modern webbläsare och tillåt kamera.
4. Klicka **Starta kamera**.

> Om kameran inte startar – se **Felsökning**.

---

## Mappstruktur & resurser

```

/ (roten)
├─ index.html
├─ models/
├─ img/
│  ├─ manifest.json
│  ├─ assets/
│  │  ├─ meta.json
│  │  ├─ hats/ ... glasses/ ... eyes/ ... mouth/ ... facial_hair/ ...
│  └─ tema/
│     ├─ halloween/manifest.json
│     ├─ jul/manifest.json
│     └─ ...
├─ ghosts/
└─ pepp.json

````

**`pepp.json` (exempel)**
```json
{
  "Allmänt": ["Heja dig!", "Du är grym!", "Snyggt fokus!"],
  "Halloween": ["Buu... 😉", "Modigt leende!"]
}
````

**`img/manifest.json` (exempel)**

```json
{
  "hats":    [{ "file": "assets/hats/party_01.png", "prob": 1,   "fit": "hat", "wFactor": 1.1, "yOffset": -0.12 }],
  "glasses": [{ "file": "assets/glasses/round_02.png", "prob": 1.2, "fit": "glasses", "wFactor": 2.0 }],
  "eyes": [], "mouth": [], "facial_hair": []
}
```

**`img/assets/meta.json` (exempel)**

```json
{ "assets/glasses/round_02.png": { "fit": "glasses", "wFactor": 2.05, "yOffset": 0.06 } }
```

---

## Inställningar i UI

Öppna **Settings**:

* Kamera, Konfidens, Detector size, Detektor-tröskel
* **Peppande one-liners** (läses från `pepp.json`)
* **Overlays & teman** (tema byter *filer*, passform från `assets/meta.json`)
* **Kamera – bild (filter)** (påverkar bara visning)
* **Besökslogg** (export/nollställ)
* **Bakgrundseffekt** (spökklipp: Sällan/Ibland/Ofta, ”bara när någon syns”)
* **Moodmätare – mål** (sekunder = 100%)

---

## Hur det fungerar (pipeline)

1. Backend väljs (WebGL → WASM fallback). Modeller laddas.
2. Kamera startar, composite-canvas visas (live + ev. spökbakgrund + personmask).
3. Ansikten detekteras; känslor/ålder estimeras; landmarks ger stabil pose.
4. Overlays ritas enligt pose; pepp/etikett/emoji visas per ansikte.
5. Moodmätaren summerar *glad-sekunder* och eased mot målet.
6. Besökslogg sparar första gången en person passerar warmup.

---

## Overlays & teman

* Välj tema i UI: `img/manifest.json` eller `img/tema/<namn>/manifest.json`.
* Varje resurs kan ha `prob` (vikt), `fit`, `wFactor`, `xOffset`, `yOffset`, `rot`, `flipX`.
* Om 68-landmarks saknas pausas passningen och overlays ritas inte.

---

## Peppande texter (`pepp.json`)

* Format: `{ "Kategori": ["text", ...] }`.
* Välj **Alla** eller specifik kategori i UI.
* En slumpad text **låses per person** tills spåret lämnar bild.

---

## Halloween: spökbakgrund

* Spelar korta mp4-klipp bakom livevideon.
* Personmask (MediaPipe) lägger tillbaka personer opakt.
* Frekvens: **Sällan ~60 min**, **Ibland ~30 min**, **Ofta ~10 min** (stochastiskt, min-gap ~2 min).
* Knapp: **Spela spöke nu 👻** (forcerar).
* Tips: 720p, 1–5 s, tysta klipp.

---

## Moodmätare & besökslogg

* Warmup innan glädje börjar räknas, konfidens kan vikta bidraget.
* Mål (sek) = 100%.
* Daglig reset (se konstanter).
* Logg per datum (localStorage), exportera CSV från UI.

---

## Prestanda & kvalitet

* WebGL om möjligt; WASM fallback.
* Adaptiv input size vid många ansikten.
* Landmarks ger bäst passning men kostar prestanda.
* Tunga overlays/klipp påverkar FPS – sänk `inputSize`/bitrate vid behov.

---

## Tillgänglighet

* Mätarens siffra har `aria-live="polite"`.
* Kontrast & storlek anpassad för läsbarhet.

---

## Säkerhet & integritet

* All beräkning sker lokalt.
* Ingen spårning, ingen nätverksuppladdning av kamera.
* Logg ligger i din webbläsare.

---

## Felsökning

**Kameran startar inte**

* Kör via HTTPS eller `localhost`, ge kamerabehörighet, stäng andra kameraappar.

**Modeller laddas inte**

* Kontrollera `models/` (manifest + binärer) och sökväg i koden.

**Overlays syns inte**

* Kontrollera att landmarks laddats och att overlay-filer finns (nätverksfliken).

**Ghost syns inte**

* Fyll `ghosts/`, justera frekvensen, prova **Spela spöke nu 👻**,
  sänk bakgrundsopacitet om spöket drunknar.

---

## Vanliga frågor

**Kan jag köra offline?** Ja, om allt hostas lokalt (inkl. MediaPipe-filer).
**Byta tema?** Lägg `manifest.json` i `img/tema/<namn>` och välj i UI.
**Lägga till spökklipp?** Lägg `.mp4` i `ghosts/` och lista filerna i koden.

---

*Skapad av Kristoffer. Lycka till — och **SMILE**! 😄*
