# Eddie-AI-Nachbau — Self-Hosted Build-Plan (Stand Juni 2026)

> **Ziel:** Die Kernfunktionen von [Eddie AI](https://www.heyeddie.ai/) selbst nachbauen — als self-hosted / lokal laufende Pipeline mit Premiere-Pro-Anbindung — mit **kompromissloser Qualität bei Transkription und Story-KI**. Plus eigene Extra-Funktionen, die Eddie nicht hat.
>
> **Priorität: Qualität vor Preis.** Wir wählen bewusst die *besten* verfügbaren Modelle für STT (wort-genaue Timestamps) und fürs Storytelling — Budget bis ~€100/Monat ist okay. Der angenehme Nebeneffekt: Selbst mit den teuersten Modellen bleibt der Betrieb **weit unter den €250/Monat von Eddie**, weil wir pro Nutzung statt pro Abo zahlen (Details in Abschnitt 7).
>
> Dieses Dokument ist der **pragmatische Bauplan**. Die große Vision-Architektur steht in [`media-brain-v3.1.md`](./media-brain-v3.1.md); hier geht es darum, ein lauffähiges, qualitativ erstklassiges Eddie-Äquivalent zu bekommen.

---

## 0. Kontext & Motivation

Eddie AI ist ein KI-„Pre-Editor": Man wirft rohes Interview-/Talking-Head-/B-Roll-Material rein, Eddie transkribiert, sortiert, wählt die besten Takes, baut daraus einen narrativen Rough-Cut und exportiert ihn als **XML-Sequenz** für Premiere/Resolve/FCP/Avid. Der eigentliche Feinschliff (Musik, Color) passiert dann im NLE.

**Warum nachbauen:** Eddie kostet im relevanten Tier ~€250/Monat (Pro+). Die zugrunde liegende Technik ist aber eine Kette aus frei verfügbaren bzw. einzeln einkaufbaren Bausteinen — der Mehrwert von Eddie ist Integration und UX, nicht ein geheimes Modell. Wir bauen die Kette selbst und stecken das gesparte Abo-Geld in **Premium-Modelle**: erstklassige STT mit Wort-Timestamps und ein Spitzen-LLM fürs Storytelling. Ergebnis: gleiche oder bessere Qualität, volle Kontrolle, offline/privat möglich — und trotzdem deutlich günstiger.

**Hinweis zum Reverse-Engineering des echten Plugins:** Ich (die KI) laufe in einer Cloud-Sandbox und kann die bei dir lokal installierte Eddie-App / das Premiere-Plugin **nicht** direkt einsehen. Wenn der echte Plugin-Code analysiert werden soll, müsste der Plugin-Ordner ins Repo kopiert werden:
- **Premiere UXP-Plugins (macOS):** `~/Library/Application Support/Adobe/UXP/Plugins/External/` bzw. `.../PluginsStorage/`
- **CEP-Extensions (alt):** `~/Library/Application Support/Adobe/CEP/extensions/`
- **Windows:** `%APPDATA%\Adobe\UXP\Plugins\` bzw. `%APPDATA%\Adobe\CEP\extensions\`

Diesen Plan kann ich aber auch ohne den echten Code umsetzen — die Architektur ist aus dem öffentlichen Funktionsumfang klar rekonstruierbar.

---

## 1. Eddie-AI-Funktionsinventar (was wir nachbauen)

| # | Eddie-Funktion | Was sie tut |
|---|----------------|-------------|
| F1 | **AI Rough-Cut / Story-Frameworks** | LLM wählt ein narratives Gerüst und webt Soundbites zu einer Story (bis ~40 min). |
| F2 | **Transcript-/Text-based Editing** | Cut wie ein Google-Doc bearbeiten: Text löschen = Schnitt im Video. |
| F3 | **Scripted Mode** | Skript/Outline/nichts vorgeben → Eddie findet Takes, wählt den besten, schneidet aufs Skript. |
| F4 | **Clean-Up** | Füllwörter (äh/ähm) + Stille + schlechte Takes entfernen. |
| F5 | **Logging / Metadata** | A-Roll vs. B-Roll trennen, Summaries, Topic-Listen, beste-Soundbite-Tags, B-Roll-Visual-Tagging. |
| F6 | **Footage-Search** | Über mehrere Interviews hinweg nach Antworten/Themen suchen, Stringouts kompilieren. |
| F7 | **Multicam (Podcasts)** | Multicam-Material syncen, Kamerawinkel kontextabhängig wählen. |
| F8 | **Night Shift** | Async über Nacht: Link zu Frame.io/Drive/Dropbox schicken → morgens fertiger Rough-Cut. |
| F9 | **Conversational UI** | Chat-/Prompt-gesteuertes Editing („ChatGPT für Videoschnitt"). |
| F10 | **NLE-Export** | XML für Premiere/Resolve/FCP/Avid, Auto-Relink zu Multicam & Source-Assets. |

---

## 2. Feature → Nachbau-Mapping (der Kern des Plans)

| Eddie-Funktion | Unser Nachbau | Tool / Technik | Lizenz |
|----------------|---------------|----------------|--------|
| F1 Story-Rough-Cut | Transkript → LLM-Prompt „wähle & ordne Segmente zu Story, gib JSON" | **Claude Opus 4.8** (primär, beste editorische Taste); Fable 5 für Härtefälle; Gemini 3.1 Pro als Fallback — Details §2.5 | API |
| F2 Transcript-Editing-UI | Web-UI, Wort-Klick = Timeline-Seek, Löschen = Cut-Patch | Fork von **OpenCut-AI** / **OpenScript** als UI-Basis | MIT |
| F3 Scripted Mode | Skript-Embeddings ↔ Transkript-Embeddings matchen, beste Takes wählen | Sentence-Embeddings + Cosine; LLM für Best-Take | open |
| F4 Clean-Up | Stille/Filler erkennen + schneiden | **auto-editor** (`--edit audio`) + FFmpeg `silencedetect` + LLM-Filler-Pass | Unlicense (Public Domain) |
| F5 Logging/Metadata | Diarisierung trennt Sprecher; LLM erzeugt Summaries/Topics; CLIP taggt B-Roll | **WhisperX** (Diarization) + CLIP + LLM | BSD-2 / MIT |
| F6 Footage-Search | Frames + Transkript embedden, Vektor-Suche | **OpenCLIP** + **LanceDB** (embedded, file-based) | Apache-2.0 |
| F7 Multicam-Sync | Audio-Waveform-Alignment der Kameras | FFmpeg + Cross-Correlation (z. B. `audalign`) | open |
| F8 Night Shift | Watch-Folder / Webhook → Job-Queue → Batch-Run | Cron/Worker + optional Frame.io-/Drive-API | eigen |
| F9 Conversational UI | Chat-Layer über der Pipeline, der Parameter/Presets setzt | LLM-Function-Calling auf unsere Pipeline-Tools | eigen |
| F10 NLE-Export | Edit Decision List → FCP7-XML / Resolve / OTIO | **OpenTimelineIO** + auto-editor-Export | Apache-2.0 |

**Kernaussage:** Jede einzelne Eddie-Funktion hat ein hochwertiges Äquivalent. Wir wählen pro Schicht das *beste* Modell (nicht das billigste) — die konkrete, qualitätsgetriebene Auswahl steht in §2.5.

---

## 2.5 Modellauswahl — qualitätsgetrieben (das Herzstück)

> **Leitsatz:** Bei STT und Story-LLM wird **nicht** gespart. Wir nehmen die besten verfügbaren Modelle. Budget bis ~€100/Monat ist freigegeben — und selbst damit bleiben wir locker drunter (§7).

### 2.5.1 STT — Wort-genaue Transkription (das Fundament von allem)

Der gesamte Schnitt hängt an **wort-genauen Timestamps** — wir schneiden auf Wortgrenzen, also ist Timestamp-Präzision die wichtigste Einzelmetrik. Zwei Wege, je nach Datenschutz-Anforderung:

**A) Cloud-Premium — beste Out-of-the-box-Qualität (empfohlen):**
- **ElevenLabs Scribe** *(Primär)* — Liefert word-level Timestamps mit `type`-Klassifikation (`word`/`spacing`/`audio_event`), Diarisierung bis **32 Sprecher** und **Audio-Event-Tagging** (`[breath]`/`[laugh]`/`[um]`). Letzteres ist ein echter Qualitätsvorteil: Atmer und Füllwörter kommen direkt aus dem STT, ohne separaten Erkennungs-Pass. Sehr stark auf Deutsch. *(Genauen aktuellen Versionsnamen — v1 vs. „v2"/Realtime — auf der ElevenLabs-Pricing-Page verifizieren; ~$0,22–0,40/h.)*
- **Speechmatics Enhanced** *(Cross-Check / Alternative)* — Exzellent bei deutschen **Dialekten und Akzenten** (sächsisch/bayrisch), Diarisierung als Kernfeature ohne Aufpreis, **On-Prem-Deployment** möglich → der Pfad für sensible/embargo Inhalte, die keine US-Cloud sehen dürfen.

**B) Self-hosted — max. Privatsphäre / offline (kostenlos, braucht GPU):**
- **WhisperX** *(sicherer Default)* — Whisper large-v3 + deutsches wav2vec2 **forced alignment** (Default `VOXPOPULI_ASR_BASE_10K_DE`; besser: ein stärkeres deutsches wav2vec2 von HuggingFace einsetzen) + pyannote-Diarisierung. ~**±50 ms** Wortgrenzen. Läuft auf **einer 8–12 GB GPU** (RTX 3060 12 GB / 4070 / A2000). Erfüllt alle drei Anforderungen (Präzision + Deutsch + Diarisierung).
- **CrisperWhisper** *(Präzisions-Spitze)* — Whisper-Variante, gezielt auf scharfe Wortgrenzen getrimmt (Segmentierungs-F1 0,79 vs. 0,66 bei Whisper-v3), **explizit auf Deutsch demonstriert**. Keine eingebaute Diarisierung → pyannote separat. Head-to-head gegen WhisperX evaluieren, wenn Wortpräzision das absolute Top-Kriterium ist.
- **Nicht** für Timestamps nehmen: `faster-whisper` / `whisper.cpp` standalone (nutzen DTW statt forced alignment → ungenauer). **NVIDIA Parakeet/Canary**: sehr schnell und starke deutsche WER (~7 % FLEURS), aber Timestamps v. a. auf **Segment-Ebene** dokumentiert → für frame-genaues Schneiden erst empirisch verifizieren.

**C) Qualitäts-Maximum für Härtefälle — Zwei-Engine-Cross-Check:** Scribe (oder Speechmatics) **und** WhisperX-Forced-Alignment parallel laufen lassen, Wortgrenzen abgleichen; bei Abweichung entscheidet FFmpeg-`silencedetect` als Schiedsrichter. Das ist exakt das Prinzip „Audio is source-of-truth" aus [`media-brain-v3.1.md`](./media-brain-v3.1.md) (§1) — teurer, aber maximal robust.

**Empfehlung:** **Scribe** als Primär-STT (beste Qualität + Audio-Events out of the box), **WhisperX** als self-hosted Fallback/Privacy-Pfad, **Cross-Check** für Premium-/kritische Projekte.

### 2.5.2 Story-LLM — die narrative Intelligenz

Für die „Story" (beste Soundbites wählen, schwache Takes raus, Dramaturgie ordnen) zählt **editorische Taste**, nicht Benchmark-Reasoning.

- **Primär: Claude Opus 4.8** (`claude-opus-4-8`) — beste editorische Taste der aktuellen Modelle, **1M-Token-Context** (schluckt ein ganzes Interview), native structured-JSON-Ausgabe. Mit `effort: high` und adaptivem Thinking betreiben.
- **Upgrade für lange/komplexe Mehrstunden-Interviews: Claude Fable 5** (`claude-fable-5`) — höchstes Reasoning + langer Horizont, ~2× Kosten. Nur wo Opus an Komplexität stößt.
- **Fallback (Vendor-Redundanz): Gemini 3.1 Pro** — 2M-Context, günstigste der Spitzenklasse; bei Anthropic-Ratelimits oder als Zweitmeinung. *(Reasoning-stark, aber Output gilt als „generischer" → nicht erste Wahl für Kuratierung.)*
- **Lokal-optional (reine Privatsphäre, starke GPU nötig): Qwen2.5-72B / Llama-3.3-70B** — adäquat für Segmentauswahl, editorisch unter den Frontier-Modellen.

### 2.5.3 Embeddings (Footage-Suche F6) — auch hier nicht sparen

- **Visuell:** SigLIP 2 oder OpenCLIP (große Variante) statt Mini-Modelle — bessere B-Roll-Treffer.
- **Text/semantisch:** ein starkes **multilinguales** Embedding für deutsche Inhalte (z. B. Voyage-multimodal oder ein Top-Open-Modell).
- **Vektor-DB:** LanceDB (embedded) — die Qualität steckt im Embedding-Modell, nicht in der DB.

---

## 3. Architektur (lean, local-first)

```
                 ┌─────────────────────────────────────────────┐
                 │   Frontend (Web-UI, Next.js / React)        │
                 │   • Transcript-Editor (F2)                  │
                 │   • Chat-Steuerung (F9)                     │
                 │   • Footage-Suche (F6)                      │
                 │   • Live-Preview-Player                     │
                 └───────────────┬─────────────────────────────┘
                                 │ REST / WebSocket
                 ┌───────────────▼─────────────────────────────┐
                 │   Backend-Service (Python, FastAPI)          │
                 │                                              │
                 │   [1] Ingest  → FFmpeg Audio-Extract         │
                 │   [2] ASR     → WhisperX (Wort-TS + Diariz.) │
                 │   [3] Index   → CLIP-Frames + LanceDB        │
                 │   [4] Story   → LLM (Gemini Flash/Haiku)     │
                 │   [5] CleanUp → auto-editor / FFmpeg         │
                 │   [6] EDL     → OpenTimelineIO               │
                 │   [7] Export  → FCP7-XML / Resolve / MP4     │
                 └───────────────┬─────────────────────────────┘
                                 │ XML / OTIO Datei
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                         ▼
  Premiere Pro            DaVinci Resolve            MP4-Preview
  (FCP7-XML-Import   (FCP7-XML / headless     (Frame.io / Web)
   ODER UXP-Panel)    Python-API)
```

**Designprinzipien:**
- **Local-first:** Standardmäßig läuft alles auf einer Maschine (eigene GPU oder gemietete Cloud-GPU stundenweise). Kein Zwang zur Cloud, kein Daten-Upload — DSGVO-freundlich.
- **EDL als Single Source of Truth:** Alle Schnitte sind eine **reversible Cut-Decision-List** (JSON). Nichts wird destruktiv gerendert; jeder Schnitt ist zurückrollbar. (Konzept aus `media-brain-v3.1.md`, Abschnitt 4.3.8.)
- **NLE-agnostisch:** Wir geben Dateien aus, kein Lock-in. Primärziel Premiere, Resolve als „leichter zu automatisieren"-Bonus.

---

## 4. Premiere-Pro-Integration — zwei Wege

### Weg A (MVP, sofort): FCP7-XML-Import
Premiere importiert **FCP7-XML** nativ (`Datei → Importieren`, NICHT FCPX-XML!). Wir bauen die Timeline einmal in **OpenTimelineIO** und exportieren über den `fcp_xml`-Adapter. auto-editor kann Premiere-XML auch direkt schreiben.
- **Pro:** Kein Plugin nötig, läuft heute, kein Adobe-Beta-Risiko.
- **Contra:** Kein Live-Editing im NLE; OTIOs FCP7-Adapter hat bekannte In/Out-Fidelity-Bugs → **Frame-Genauigkeit testen** (Sprint 1 Akzeptanzkriterium).

### Weg B (Produktiv, später): UXP-Panel
Ein echtes Premiere-Panel wie Eddies — **UXP** (CEP ist abgekündigt; CEP 12 ist die letzte Version).
- **Architektur:** UXP-Panel (React/HTML, Boilerplate **`hyperbrew/bolt-uxp`**) ↔ spricht via `fetch`/`WebSocket` mit unserem lokalen Backend ↔ wendet die zurückgegebene EDL über die DOM-API auf die Timeline an.
- **Premiere-DOM-API (`require("premierepro")`):** Sequenzen/Tracks/Clips/Marker. Editing-Primitive:
  - `SequenceEditor.createInsertProjectItemAction` / `createOverwriteItemAction` (ProjectItem + `TickTime` In/Out + Track-Index)
  - `createAddMarkerAction` (Name, Typ, Startzeit, Kommentar — ideal für Cut-Begründungen)
  - Alle Mutationen laufen über `executeTransaction(callback, undoString)` **mit `lockedAccess`** — beide zwingend zusammen, sonst „Script Action failed to execute".
- **Wichtige Manifest-Hürde:** Jeder Backend-Host (auch `localhost`) muss in `manifest.json` unter `requiredPermissions.network` deklariert sein, sonst blockt UXP alle Verbindungen.
- **Pro:** Echtes In-App-Erlebnis, Round-Trip. **Contra:** UXP für Premiere ist Beta, API-Lücken bei Timeline-Operationen → Features gegen die aktuelle Beta verifizieren.

### Resolve-Alternative (am einfachsten zu automatisieren)
DaVinci Resolve hat eine **headless Python-API** (`resolve` → ProjectManager → MediaPool → Timeline). Deutlich einfacher als Premiere UXP für Automation — **aber nur Resolve Studio (kostenpflichtig, einmalig ~€295)** hat Scripting, die Gratis-Version nicht. Für reines Prototyping der Cut-Logik ideal.

**Empfehlung:** Backend-first bauen, Cut-Logik über **Resolve-Python ODER FCP7-XML-Import** validieren, UXP-Panel erst als Produktions-Frontend bauen, wenn die Pipeline steht.

---

## 5. Phasen-Roadmap

| Phase | Dauer | Inhalt | Ergebnis |
|-------|-------|--------|----------|
| **P0 — Setup** | ~3 Tage | Repo-Struktur, FastAPI-Skeleton, FFmpeg + WhisperX lokal lauffähig | Audio rein → Wort-genaues Transkript raus |
| **P1 — Clean-Up MVP** | ~1 Woche | auto-editor + silencedetect-Cross-Check, FCP7-XML-Export | Rohvideo → entstörter Cut, in Premiere importierbar (F4, F10) |
| **P2 — Story-Engine** | ~1–2 Wochen | LLM-Segment-Auswahl & -Ordering, JSON-EDL, OTIO-Timeline | Interview → narrativer Rough-Cut (F1, F3) |
| **P3 — Transcript-UI** | ~2 Wochen | OpenCut-AI/OpenScript forken, Wort-Klick-Seek, Löschen=Cut, Live-Preview | Descript-artiges Editing im Browser (F2, F9) |
| **P4 — Logging & Search** | ~1–2 Wochen | Diarisierung-Auswertung, CLIP-Frame-Index, LanceDB, Summaries | A/B-Roll-Sortierung + semantische Footage-Suche (F5, F6) |
| **P5 — Multicam & Night Shift** | ~1–2 Wochen | Audio-Sync mehrerer Kameras, Watch-Folder/Webhook-Queue | Multicam-Cut + Async-Batch über Nacht (F7, F8) |
| **P6 — Premiere UXP-Panel** | ~2–3 Wochen | bolt-uxp-Panel, DOM-API-Edits, manifest-Network-Perms | Echtes In-App-Plugin wie Eddie (Weg B) |

**MVP-Schnitt:** Schon nach **P1+P2** hat man ein nutzbares Eddie-Äquivalent (Rough-Cut + Clean-Up + Premiere-Import) — der teuerste Eddie-Use-Case, für quasi 0 € Betriebskosten.

---

## 6. Die „Extra-Funktionen" (was wir besser machen als Eddie)

Vorschläge — bitte priorisieren/streichen nach Geschmack:

1. **Deutsch-first.** Eddie ist englisch-zentriert. Wir tunen WhisperX large-v3 + `wav2vec2-large-xlsr-53-german`, Filler-Whitelist (äh/ähm/also/halt/quasi), Umgang mit Dialekt (sächsisch/bayrisch) und Komposita.
2. **Echte Reversibilität mit Branching.** Cut-Decision-List als Patch-Baum (`parent_cdl`) — jeden einzelnen Schnitt zurückrollen, Varianten vergleichen. (Eddie kann das nicht so granular.)
3. **Audio↔Transkript-Cross-Check.** Schnitt nur, wenn FFmpeg-`silencedetect` UND Transkript-Lücke übereinstimmen → weniger Fehlschnitte. (Aus `media-brain-v3.1.md`, Designprinzip „Audio is source-of-truth".)
4. **Shortform-Repurposing eingebaut.** Ein `shorts_aggressive`-Preset, das aus dem Rough-Cut direkt 9:16-Clips + Auto-Captions macht — spart ein zweites Tool (OpusClip etc.).
5. **100 % offline/privat.** Kompletter Lauf ohne Cloud-Upload (lokales LLM optional) — für sensible/embargo Inhalte, die nie Eddies Server sehen dürfen.
6. **Presets** (`natural_podcast`, `youtube_tight`, `shorts_aggressive`) mit einstellbaren Pausen-/Filler-/Buffer-Schwellen statt One-Size-Fits-All.
7. **Eigener Cost-/Audit-Log:** pro Projekt sehen, was die LLM-API gekostet hat (Transparenz, die Eddie nicht bietet).

---

## 7. Kostenvergleich (qualitäts-first gerechnet)

Wichtig: Wir rechnen hier mit den **teuersten/besten** Modellen (Scribe-STT + Claude Opus 4.8), nicht mit Sparvarianten — und liegen trotzdem weit unter Eddie.

| Posten | Eddie AI | Unser Nachbau (Premium-Modelle) |
|--------|----------|---------------------------------|
| Lizenz/Abo | **~€250/Monat** (Pro+) | **€0** (kein Abo, pay-per-use) |
| Transkription (60-min-Interview) | inkludiert | ElevenLabs Scribe ≈ **€0,25–0,40** *(oder WhisperX self-hosted = nur GPU-Strom)* |
| Story-LLM (60-min-Interview) | inkludiert | Claude Opus 4.8 ≈ **€0,18** (~20k in + 4k out); Fable 5 ≈ €0,40 |
| Cut-Engine / Export | inkludiert | auto-editor / FFmpeg / OTIO = **€0** |
| GPU (nur falls self-hosted STT) | (Eddie-Server) | eigene GPU **oder** Cloud-GPU stundenweise (~€0,30–0,80/h L4) |
| **Kosten pro 60-min-Projekt** | (im Abo) | **≈ €0,45–0,80** mit Premium-Modellen |
| **Effektiv / Monat (50 Projekte)** | **~€250** | **≈ €25–40** — also ~6–10× günstiger, trotz Bestmodellen |

**Fazit:** Selbst kompromisslos qualitäts-first (Scribe + Opus 4.8, plus optionalem Cross-Check) bleibst du bei realistischer Nutzung klar **unter deinem €100-Budget** und um ein Vielfaches unter Eddies €250-Abo. Kosten sind hier kein limitierender Faktor — die Modellwahl darf rein nach Qualität erfolgen.

---

## 8. Konkreter Tech-Stack

| Schicht | Tool | Lizenz | Warum |
|---------|------|--------|-------|
| STT Cloud-Premium *(Primär)* | **ElevenLabs Scribe** | kommerziell | Beste Wort-Timestamps + Audio-Events (breath/filler) + Diarisierung out of the box |
| STT Cross-Check / On-Prem | **Speechmatics Enhanced** | kommerziell | Top bei dt. Dialekten, Diarisierung inkl., On-Prem für sensible Inhalte |
| STT self-hosted *(Default/Privacy)* | **WhisperX** (large-v3 + dt. wav2vec2) | BSD-2 | ~±50 ms Wortgrenzen via forced alignment, läuft auf 8–12 GB GPU |
| STT Präzisions-Spitze (Eval) | **CrisperWhisper** | OSS | Schärfste Wortgrenzen, auf Deutsch getestet (pyannote separat) |
| Clean-Up / Auto-Cut | **auto-editor** (WyattBlue) | Unlicense | Reifste OSS-Cut-Engine, exportiert Premiere-XML/FCPXML/Resolve |
| Story-LLM *(Primär)* | **Claude Opus 4.8** | kommerziell | Beste editorische Taste, 1M-Context, native JSON |
| Story-LLM (Härtefälle) | **Claude Fable 5** | kommerziell | Höchstes Reasoning für lange/komplexe Interviews |
| Story-LLM (Fallback) | Gemini 3.1 Pro | kommerziell | 2M-Context, Vendor-Redundanz |
| Story-LLM (lokal, optional) | Qwen2.5-72B / Llama-3.3-70B | Apache/Llama | Reine Privatsphäre, wenn starke GPU vorhanden |
| Footage-Embeddings | SigLIP 2 / OpenCLIP (groß) | MIT | Visuelle Suche + B-Roll-Tagging in hoher Qualität |
| Vektor-DB | **LanceDB** | Apache-2.0 | Embedded, file-based, kein Server für Single-User |
| Timeline-Interchange | **OpenTimelineIO** | Apache-2.0 | EDL-Hub → FCP7-XML / EDL / AAF |
| Video-Processing | FFmpeg 7.x | LGPL/GPL | `silencedetect`, `select`, `concat`, `xfade`/`acrossfade` |
| Multicam-Sync | audalign / Cross-Correlation | MIT | Audio-Waveform-Alignment |
| Backend | Python + FastAPI | MIT | — |
| Frontend / Transcript-UI | Fork von **OpenCut-AI** / **OpenScript** / **CutScript** | MIT | Fertige Transcript-Editing-Basis |
| Premiere-Panel | UXP + **bolt-uxp** | MIT | Standard-Boilerplate für UXP-Panels |

---

## 9. Vorgeschlagene Repo-Struktur

```
eddie-clone/
├── backend/
│   ├── ingest/          # FFmpeg Audio-Extract, Proxy-Gen
│   ├── asr/             # WhisperX-Wrapper (Wort-TS + Diarization)
│   ├── index/           # CLIP-Frames + LanceDB
│   ├── story/           # LLM-Segment-Auswahl, Prompt-Templates
│   ├── cleanup/         # auto-editor / silencedetect Cross-Check
│   ├── edl/             # Cut-Decision-List Modell (JSON, reversibel)
│   ├── export/          # OTIO → FCP7-XML / Resolve / MP4
│   ├── presets/         # natural_podcast / youtube_tight / shorts_aggressive
│   └── api/             # FastAPI Routes + WebSocket-Progress
├── frontend/            # Next.js Transcript-Editor + Chat + Suche
├── premiere-uxp/        # bolt-uxp Panel (Phase 6)
├── resolve-scripts/     # headless Python für Resolve-Prototyping
└── docs/                # dieses Dokument + API-Specs
```

---

## 10. Verifikation & nächste Schritte

**Akzeptanzkriterien pro Phase:**
- **P1:** Ein 12-min-Talking-Head-Video → Clean-Up-Cut, in Premiere importiert, **Schnitte frame-genau** (OTIO-FCP7-Fidelity manuell prüfen), hörbar keine Audio-Klicks an Schnitten.
- **P2:** Ein 60-min-Interview → kohärenter Rough-Cut < 10 min, Story-Reihenfolge plausibel, LLM-Kosten protokolliert (< €0,50).
- **P3:** Wort im Transkript löschen → Schnitt erscheint in < 1 s in der Live-Preview; ⌘Z rollt den Cut-Patch zurück.
- **P4:** „Finde die Stelle, wo über X gesprochen wird" liefert die richtige Clip-Position.

**Sofort startbar (Phase 0/1), sobald du grünes Licht gibst:**
1. Repo-Struktur + FastAPI-Skeleton + WhisperX-Lauf anlegen.
2. auto-editor + FCP7-XML-Export als erstes End-to-End durchziehen.
3. Mit einem deiner echten Clips testen (du legst eine Datei ins Repo oder einen Pfad).

**Offene Entscheidungen (deine Wahl, Defaults in Klammern):**
- Primäres NLE: **Premiere** (Default, weil du das Plugin nutzt) oder Resolve (leichter zu automatisieren)?
- Story-LLM: Cloud-API (Default, billig) oder zwingend lokal (Privatsphäre, braucht GPU)?
- Welche der Extra-Funktionen aus Abschnitt 6 haben Priorität?

> Wenn du den echten Eddie-Plugin-Code reverse-engineered haben willst, kopier den Plugin-Ordner (Pfade in Abschnitt 0) ins Repo — dann analysiere ich die konkrete Panel↔Backend-Kommunikation und übernehme bewährte Details.
