# Media Brain — Build- und Research-Plan v3.2 (Stand Mai 2026)

> **Versionierung:** Vollständige Neuausgabe gegenüber v3.1. Enthält **alle** Sektionen, Funktionen und Empfehlungen aus v3.1 unverändert plus die im Experten-Review identifizierten strukturellen Verbesserungen rund um Stufe 2.5 (Transcript-Aware Micro-Cut Engine). Änderungen markiert mit **[NEU SEIT v3.1]**, **[GEÄNDERT GEGENÜBER v3.1]**, **[BESTÄTIGT]**.
>
> **Caveat zu Provider-Zahlen:** Preise und WER-Werte in diesem Plan stammen aus dem v3.1-Stand plus dem Experten-Review (Mai 2026). Wo öffentliche Transparenz uneinheitlich ist (insb. ElevenLabs Scribe v2 Preise/Benchmarks, Speechmatics DE/EN-WER-Einzelwerte), ist das **explizit markiert**. Alle Zahlen sind in Sprint G auf eigenen Daten zu re-verifizieren, bevor sie produktiv bindend werden.

---

## 0. Executive Summary der Änderungen v3.1 → v3.2

**Was sich ändert — sieben strukturelle Verbesserungen aus dem Experten-Review:**

1. **[NEU SEIT v3.1] Verbatim-First-Prinzip.** Das Master-Transkript wird **niemals** bereinigt. Es gibt zwei strikt getrennte Modi: `verbatim_master` (roh, alle Füllwörter/Disfluencies/false starts erhalten — die **einzige persistierte Wahrheit**) und `display_clean` (bereinigte UI-Ansicht, **abgeleitet**, nicht als Wahrheit persistiert). Begründung: STT-Anbieter entfernen Füllwörter teils standardmäßig oder optional — wer auf bereinigtem Text schneidet, verliert Reversibilität und produziert nicht-auditierbare Cuts.
2. **[GEÄNDERT GEGENÜBER v3.1] STT-Provider-Auswahl ergebnisoffen.** v3.1 hatte ElevenLabs Scribe v2 als Default-Transcript-Provider für Micro-Cut gesetzt. **v3.2 legt keinen Default vorab fest.** Sprint G entscheidet ergebnisoffen zwischen vier gleichberechtigten Kandidaten: Speechmatics Ursa 2, AssemblyAI Universal-3 Pro, ElevenLabs Scribe v2, WhisperX. **AssemblyAI Universal-3 Pro** wird als **Benchmark-Anker** aufgenommen (transparente offizielle DE/EN-WER-Werte).
3. **[NEU SEIT v3.1] Verbatim-First-Referenzarchitektur** als kanonische sechsstufige Pipeline für Stufe 2.5, mit dem Kernprinzip **„JSON zuerst, Render später"**.
4. **[NEU SEIT v3.1] MCP-Video-Server-Bewertungstabelle.** Fünf Server verglichen (mcp-video, video-audio-mcp, video-editing-mcp, ffmpeg-mcp-lite, ffmpeg-mcp-comp). Kernaussage: öffentliche MCP-Video-Server sind die **„Hände"**, nicht das **„Gehirn"** — die Schnittentscheidungslogik bleibt eigene, testbare Logik.
5. **[NEU SEIT v3.1] Export-Hierarchie fixiert.** Kanonische Reihenfolge: internes JSON = Wahrheit → FCPXML/OTIO/EDL abgeleitet → NLE-Import → optionale Render-/Caption-Layer.
6. **[GEÄNDERT GEGENÜBER v3.1] Datenmodell erweitert** um acht neue Tabellen und zahlreiche Verbatim-/Evidenz-Felder.
7. **[GEÄNDERT GEGENÜBER v3.1]** Sprint G um eine **Micro-Cut-Testbank** (50–100 DE/EN-Clips, vier Klassen) und schärfere Eval-Metriken erweitert; **Kostenmodell neu kalibriert** (STT günstiger als angenommen); **Risikoregister** um konkrete Ausfallmuster ergänzt.

**Was unverändert bleibt:** Cloud-Backbone (GCP), KI/ML-Stack (Gemini 3.1 + Marengo 3.0 + Voyage), NLE-Integration, Three-Stage-Pipeline-Architektur, Editor-Stack (BlockNote/Tiptap/Yjs/Hocuspocus/Liveblocks/Loro), Three Presets, HyperFrames als Caption-Layer, Transkript-Editing-UI, Tool-Registry, Provider-Abstraktion über ADK + MCP, 48-Wochen-Roadmap-Grundstruktur.

---

## 1. Vision & Designprinzipien

**Vision.** Media Brain ist ein KI-gestütztes Video-Produktions-Betriebssystem, das die gesamte Wertschöpfung von Roh-Footage über Skript, Vector-Retrieval, Rough-Cut, Feinschnitt, Shortform-Repurposing bis zur NLE-Übergabe agentisch orchestriert. NLE-agnostisch, Provider-agnostisch, mandantenfähig.

**Designprinzipien:**

- **[BESTÄTIGT] Plan A/B/C für jede kritische Funktion.**
- **[BESTÄTIGT] Provider-Abstraktion über ADK + MCP** — kein Modell, kein Tool hart verdrahtet.
- **[BESTÄTIGT] Reversibilität** — keine destruktiven Edits ohne CDL/Snapshot.
- **[BESTÄTIGT] Postgres Row Level Security als harte Tenant-Grenze.**
- **[BESTÄTIGT] OpenTelemetry-Spans pro Pipeline-Schritt** + `tool_health`-Heartbeats.
- **[BESTÄTIGT] Self-Healing Tool Plugin** im ADK.
- **[BESTÄTIGT] Audio is source-of-truth für Cuts** — Cut nur wenn Transkript-Lücke UND Audio-Silence-Detection sich bestätigen.
- **[NEU SEIT v3.1] Verbatim-First.** Die erste persistierte Wahrheit ist ein **roher, lückenloser Wort-Token-Strom**, niemals ein für UI/Untertitel bereinigtes Transkript. Bereinigung ist eine **abgeleitete View**, kein Speicherzustand.
- **[NEU SEIT v3.1] JSON zuerst, Render später.** FCPXML, EDL, OTIO, FFmpeg-Commands und HyperFrames-Compositions sind **abgeleitete Artefakte** aus einem einzigen JSON-Run, niemals der Primärdatensatz. Eure Cut-Engine entscheidet **was** geschnitten wird; FFmpeg/HyperFrames/NLEs entscheiden nur noch **wie** dargestellt wird.

---

## 2. Architektur-Überblick

### 2.1 Cloud-Backbone (Google Cloud, [BESTÄTIGT])

| Schicht | Komponente | Stand v3.2 |
|---|---|---|
| Compute (stateless) | Cloud Run (2nd gen) | [BESTÄTIGT] |
| Compute (long-running) | Cloud Run Worker Pools (GA) | [BESTÄTIGT] |
| Compute (GPU) | Cloud Run GPU mit NVIDIA L4 (Standard) und NVIDIA RTX PRO 6000 Blackwell (Premium-Render) | [BESTÄTIGT] |
| OLTP | Cloud SQL PostgreSQL 17 + pgvector 0.8 | [BESTÄTIGT] |
| Object Storage | Google Cloud Storage (Multi-Region eu, Autoclass) | [BESTÄTIGT] |
| Vector Search | Vertex AI Vector Search 2.0 (GA, Hybrid Dense+Sparse) | [BESTÄTIGT] |
| Eventing | Eventarc Advanced + Pub/Sub | [BESTÄTIGT] |
| Agent Platform | Gemini Enterprise Agent Platform | [BESTÄTIGT] |
| Agent SDK | Google ADK 2.1.0 mit Self-Healing Tool Plugin | [BESTÄTIGT] |
| Tool Registry | Cloud API Registry + YAML-Manifest im Repo | [BESTÄTIGT] |

### 2.2 KI/ML-Stack

| Aufgabe | Plan A | Plan B | Plan C / Notbackup |
|---|---|---|---|
| Multimodale Embeddings | Gemini Embedding 2 | Voyage `voyage-multimodal-3.5` | Vertex Multimodal Embedding |
| Premium Video-Index | TwelveLabs Marengo 3.0 | Marengo 2.x | — |
| Visual Document Retrieval | Voyage voyage-multimodal-3.5 | ColPali Self-Hosted | — |
| Skript / Vision / Hook-Scoring | Gemini 3.1 Pro | Gemini 3.1 Flash-Lite (Floor) | Claude Sonnet 4.6 (Cross-Check) |
| Cross-Check-LLM | Claude Sonnet 4.6 | Gemini 3.1 Pro | — |
| Transkription Long-Form-Master (Doku/Podcast) | Speechmatics Enhanced | ElevenLabs Scribe v2 | WhisperX large-v3 |
| **[GEÄNDERT GEGENÜBER v3.1] Transkription Micro-Cut (word-level)** | **ergebnisoffen — Sprint G entscheidet** | siehe Stufe 2.5 / Sprint G | — |
| Notbackup STT | Deepgram Nova-3 | AssemblyAI (Streaming) | — |

**[GEÄNDERT GEGENÜBER v3.1] Begründung:** Der in v3.1 fest gesetzte Default „Scribe v2 für Micro-Cut" wird zurückgenommen. Die STT-Wahl für Stufe 2.5 ist ein **ergebnisoffenes Race in Sprint G** (Detail in 4.3.6 und 8.1). Bis Sprint G abgeschlossen ist, läuft Stufe 2.5 hinter einem Feature-Flag mit AssemblyAI Universal-3 Pro als **Referenz-Provider für die Eval-Harness** (Benchmark-Anker), nicht als Produktions-Default.

### 2.3 NLE-Integration ([BESTÄTIGT])

- **Adobe Premiere Pro:** UXP-Plugin (UXP-first). CEP-Sunset-Migration bis Q3 2026.
- **[GEÄNDERT GEGENÜBER v3.1] Legacy-Bridge präzisiert:** Eine **kleine, isolierte ExtendScript/CEP-Bridge** bleibt bis September 2026 bewusst erhalten — ausschließlich für FCPXML-Importe via `app.openFCPXML(path, projPath)` (im Premiere-Scripting-Guide weiterhin dokumentiert) und vereinzelte Legacy-Übergangsfälle. **Keine neue Geschäftslogik** landet dort; die Bridge ist read-/import-only und vom Rest der Codebase entkoppelt.
- **DaVinci Resolve 20.x / 21 Beta:** Headless Python-API, IntelliScript als Pre-LLM-Rough-Cut-Backend.
- **Sync-Backbone:** LucidLink Filespace (Bulk-Footage), Frame.io V4 REST-API (Review/Kommentare), `rclone bisync` (Fallback).
- **Companion-App:** Tauri 2.
- **[BESTÄTIGT] Text-Based-Editing-Brücke** zu Premiere als Export-Ziel.

### 2.4 Editor & Frontend ([BESTÄTIGT])

- **Editor-Kern:** BlockNote auf Next.js. **Migrationsspur:** Tiptap 3.
- **Realtime:** Yjs + Hocuspocus (Self-Hosted) + Liveblocks (Managed-Fallback). **Offline:** `y-indexeddb`. **CRDT-Migrationsspur:** Loro.
- **Skript-Markup:** Fountain mit eckigen Klammern.
- **[BESTÄTIGT] `transcript-block`** BlockNote-Variante mit Modi Read / Edit-Words / Edit-Cuts (Detail in Sektion 7).

### 2.5 Datenmodell

**[BESTÄTIGT] Bestand:** `projects`, `media`, `embedding_runs`, `agent_traces`, `tool_health`, `tenants`, `users`, `roles`, `cost_events`, `rough_cuts`, `shortform_jobs`, plus die v3.1-Tabellen `transcript_words`, `cut_decision_list`, `micro_cut_runs`. Alle mit Postgres RLS auf `tenant_id`.

**[NEU SEIT v3.1] Acht neue Tabellen / Entitäten:**

```sql
-- Ein Transkriptionslauf pro Mediendatei pro Provider (versioniert, verbatim)
CREATE TABLE transcript_runs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id       UUID NOT NULL,
  media_id        UUID NOT NULL REFERENCES media(id),
  provider        TEXT NOT NULL,           -- 'speechmatics' | 'assemblyai' | 'scribe_v2' | 'whisperx'
  provider_model  TEXT NOT NULL,           -- 'ursa_2' | 'universal_3_pro' | 'scribe_v2' | 'large_v3'
  mode            TEXT NOT NULL,           -- 'verbatim_master' | 'display_clean'  (display_clean ist abgeleitet)
  language        TEXT,
  is_master       BOOLEAN DEFAULT FALSE,   -- genau EIN verbatim_master pro media gilt als Wahrheit
  wer_estimate    NUMERIC(5,3),
  der_estimate    NUMERIC(5,3),
  created_at      TIMESTAMPTZ DEFAULT now()
);
-- Sprecher-Segmente getrennt vom Wort-Index
CREATE TABLE speaker_segments (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       UUID NOT NULL,
  transcript_run  UUID NOT NULL REFERENCES transcript_runs(id),
  speaker_id      TEXT NOT NULL,           -- 'S1','S2',... oder benannt
  start_s         NUMERIC(10,3) NOT NULL,
  end_s           NUMERIC(10,3) NOT NULL,
  confidence      NUMERIC(4,3)
);
-- Nicht-Sprach-Events (Atmer, Lachen, Applaus, Musik)
CREATE TABLE audio_events (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       UUID NOT NULL,
  transcript_run  UUID NOT NULL REFERENCES transcript_runs(id),
  event_type      TEXT NOT NULL,           -- 'breath' | 'laugh' | 'applause' | 'music' | 'noise'
  start_s         NUMERIC(10,3) NOT NULL,
  end_s           NUMERIC(10,3) NOT NULL,
  source          TEXT NOT NULL            -- 'provider_tag' | 'vad' | 'classifier'
);
-- Ein Micro-Cut-Lauf (ersetzt/ergänzt micro_cut_runs konzeptionell als "cut_runs")
CREATE TABLE cut_runs (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id          UUID NOT NULL,
  media_id           UUID NOT NULL,
  transcript_run     UUID NOT NULL REFERENCES transcript_runs(id),
  preset             TEXT NOT NULL,        -- 'natural_podcast' | 'youtube_tight' | 'shorts_aggressive' | 'custom'
  params             JSONB NOT NULL,       -- Schwellwerte, Buffer, dBFS
  exec_backend       TEXT NOT NULL,        -- 'mcp_video' | 'ffmpeg_native' | 'auto_editor'
  cdl_id             UUID,                 -- FK auf cut_decision_list
  status             TEXT NOT NULL,
  cost_cents         INT,
  trace_id           TEXT,
  created_at         TIMESTAMPTZ DEFAULT now()
);
-- Evidenz pro einzelnem Cut (separat, damit auditierbar)
CREATE TABLE cut_evidence (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       UUID NOT NULL,
  cut_run         UUID NOT NULL REFERENCES cut_runs(id),
  cut_local_id    TEXT NOT NULL,           -- 'c01', 'c02', referenziert cut_decision_list.cuts[].id
  gap_ms          INT,
  vad_silence_ms  INT,
  speaker_change  BOOLEAN,
  filler_flag     BOOLEAN,
  plosive_near    BOOLEAN,                 -- riskanter Cut nahe Plosiv
  score           NUMERIC(4,3),
  decision_reason TEXT
);
-- Editor-Revisionen / Branching der CDL
CREATE TABLE edit_revisions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id       UUID NOT NULL,
  cdl_id          UUID NOT NULL,
  parent_revision UUID,
  author          TEXT NOT NULL,           -- 'agent' | 'editor:<user_id>' | 'preset'
  human_override  BOOLEAN DEFAULT FALSE,
  accepted_by_editor BOOLEAN,
  diff            JSONB NOT NULL,
  created_at      TIMESTAMPTZ DEFAULT now()
);
-- Abgeleitete NLE-/Interchange-Exporte (FCPXML/OTIO/EDL)
CREATE TABLE nle_exports (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id       UUID NOT NULL,
  cut_run         UUID NOT NULL REFERENCES cut_runs(id),
  format          TEXT NOT NULL,           -- 'fcpxml' | 'otio' | 'edl' | 'premiere_xml' | 'srt'
  uri             TEXT NOT NULL,
  derived_from_json_run UUID NOT NULL,     -- Pflicht: jeder Export verweist auf seinen JSON-Run
  import_success  BOOLEAN,
  created_at      TIMESTAMPTZ DEFAULT now()
);
-- QC-Metriken pro Render/Export
CREATE TABLE render_qc (
  id                          BIGSERIAL PRIMARY KEY,
  tenant_id                   UUID NOT NULL,
  cut_run                     UUID NOT NULL REFERENCES cut_runs(id),
  timestamp_coverage_rate     NUMERIC(5,3),
  word_overlap_rate           NUMERIC(5,3),
  speaker_swap_rate           NUMERIC(5,3),
  cut_regret_rate             NUMERIC(5,3),    -- nach Human Review
  fcpxml_import_success_rate  NUMERIC(5,3),
  preview_vs_export_diff_ms   INT,
  subtitle_drift_ms           INT,
  stream_reconnect_reset_detected BOOLEAN,
  black_frame_rate            NUMERIC(5,3),
  silent_render_rate          NUMERIC(5,3),
  created_at                  TIMESTAMPTZ DEFAULT now()
);
```

**[GEÄNDERT GEGENÜBER v3.1] Erweiterte Felder in `transcript_words`:**

```sql
ALTER TABLE transcript_words ADD COLUMN provider_word_id     TEXT;
ALTER TABLE transcript_words ADD COLUMN char_offsets         INT4RANGE;
ALTER TABLE transcript_words ADD COLUMN is_disfluency        BOOLEAN DEFAULT FALSE;
ALTER TABLE transcript_words ADD COLUMN display_text         TEXT;   -- bereinigte Ansicht
ALTER TABLE transcript_words ADD COLUMN verbatim_text        TEXT;   -- roh, Wahrheit
ALTER TABLE transcript_words ADD COLUMN language             TEXT;
ALTER TABLE transcript_words ADD COLUMN source_provider_run_id UUID REFERENCES transcript_runs(id);
```

**[GEÄNDERT GEGENÜBER v3.1] Erweiterte Felder in `cut_decision_list.cuts[]` (JSONB-Schema):** zusätzlich `reversible`, `decision_reason`, `gap_ms`, `vad_silence_ms`, `speaker_change`, `filler_flag`, `human_override`, `accepted_by_editor`, `score`, `evidence`.

**Invariante:** Pro `media_id` existiert **genau ein** `transcript_runs`-Eintrag mit `mode='verbatim_master'` und `is_master=true`. Alle `display_clean`-Runs und alle CDLs leiten von ihm ab. `display_clean` wird **nicht** als unabhängige Wahrheit persistiert, sondern als deterministische View berechnet.

---

## 3. Thema 1 — NLE-Sync ([BESTÄTIGT], um Export-Hierarchie ergänzt)

Die in v3.1 spezifizierte Sync-Architektur bleibt: Ingest über Tauri-Companion → LucidLink Filespace → Proxy-Generation auf Cloud Run GPU L4 → Frame.io V4 als Review-Master → `rclone bisync` als Notpfad → NLE-Adapter (Premiere UXP, Resolve Headless-Python) → Konflikt-Resolution via Bin-Locking + Yjs/Frame.io-Merge.

**[NEU SEIT v3.1] Export-Hierarchie fixiert.** Jeder NLE-Adapter folgt der **kanonischen Reihenfolge**:

```
[1] Internes JSON (CDL + Run)        ← die einzige Wahrheit
        │
[2] Ableitung in Interchange-Format:
        ├── FCPXML 1.13   → Premiere-Import (via UXP, Fallback Legacy-Bridge app.openFCPXML)
        ├── OTIO 0.19     → herstellerneutraler Mid-Layer, v.a. Resolve-orientierte lokale Handoffs
        └── EDL (CMX 3600)→ absichtlich simples Cut-only-Sicherheitsnetz (nur In/Out-Punkte)
        │
[3] NLE-Import
        │
[4] Optionale Render-/Caption-Layer (HyperFrames, Submagic)
```

**Regel:** Jeder Eintrag in `nle_exports` trägt `derived_from_json_run` als Pflicht-FK. Ein Export, der nicht eindeutig auf einen JSON-Run zurückführbar ist, wird vom QC-Gate abgelehnt. Das verhindert das v3.1-Ausfallmuster „NLE-Import gelingt formal, driftet aber semantisch".

---

## 4. Thema 2 — Vector-Rough-Cut-Pipeline

### 4.1 Stufe 1 — Vector-Müll-Filter ([BESTÄTIGT])

Multi-Query-Embedding (Gemini Embedding 2), MMR-Re-Ranking, DPP für Diversität, Vertex AI Vector Search 2.0 Hybrid (Dense + Sparse BM25) als Backend.

### 4.2 Stufe 2 — AI-Rough-Cut ([BESTÄTIGT])

- **Plan A:** Eddie AI v3 Night Shift.
- **Plan B:** DaVinci Resolve IntelliScript (Headless-Python).
- **Plan C:** Eigener FCPXML-Generator (Gemini-3.1-Pro-Plan + Marengo-3.0-Shot-Ranking).

### 4.3 Stufe 2.5 — Transcript-Aware Micro-Cut Engine

#### 4.3.1 Ziel & Abgrenzung ([BESTÄTIGT])

Wort-genauer Feinschnitt von Sprache zwischen grobem Rough-Cut (Stufe 2) und mechanischem Cleanup (Stufe 3): Redepausen erkennen und **kürzen** (nicht nur löschen), Füllwörter markieren/entfernen, Versprecher und doppelte Takes erkennen, Jump-Cuts glätten, Atempausen-Presets, alles reversibel über die `cut_decision_list`.

#### 4.3.2 [NEU SEIT v3.1] Verbatim-First-Referenzarchitektur (kanonische sechsstufige Pipeline)

```
[1] VERBATIM STT
    Audio extrahieren (FFmpeg 16 kHz mono PCM).
    Transkription IMMER im verbatim_master-Modus:
      - alle Füllwörter, Disfluencies, false starts ERHALTEN
      - provider-seitige "clean"/"no_verbatim"/filler-removal-Optionen DEAKTIVIERT
    Ergebnis → transcript_runs (mode='verbatim_master', is_master=true)
        │
[2] NORMIERUNG
    Alle Provider-Antworten in EIN einheitliches internes Schema überführen:
      words[]          → transcript_words (verbatim_text, display_text, start/end, speaker,
                         confidence, is_disfluency, provider_word_id, char_offsets, language)
      utterances[]     → abgeleitete Satz-/Turn-Gruppierung
      speaker_segments[]→ speaker_segments
      audio_events[]   → audio_events (breath/laugh/applause/music/noise)
        │
[3] CUT-ANALYZER → KANDIDATEN
    Erzeuge Cut-Kandidaten aus:
      - Wortlücken: gap = w_i+1.start − w_i.end
      - Disfluency-Markern (provider-Tag ODER Regex/Lexikon: äh, ähm, also, halt, eben, quasi)
      - Sprecherwechseln (aus speaker_segments)
      - Bad-Take-Mustern (semantische Ähnlichkeit aufeinanderfolgender Sätze)
        │
[4] CROSS-CHECK MIT AKUSTIK
    Jede Kandidaten-Pause gegen akustisches Signal prüfen:
      - eigener VAD-Layer (Silero VAD primär, WebRTC VAD als Leichtgewicht-Fallback)
      - ODER provider-seitiges VAD/Endpointing
      - FFmpeg silencedetect als zusätzlicher Beleg
    Nur Cuts, die in lexikalischer UND akustischer Evidenz konsistent sind, bleiben "stark".
    Mismatch → score senken, im UI als "Konflikt" markieren, in cut_evidence loggen.
        │
[5] SMOOTHING
    - sehr nahe Kandidaten mergen
    - Head-/Tail-Padding (buffer_pre_ms / buffer_post_ms) ergänzen
    - riskante Cuts herabstufen: nahe Plosiven (plosive_near=true), nahe Sprecherwechseln,
      nahe Atmern/Lachern aus audio_events
        │
[6] REVERSIBLE CUT DECISION LIST
    Ergebnis = JSON-CDL in cut_decision_list, jeder Cut mit voller Evidenz.
    DIESE CDL ist die Wahrheit. Erst danach:
      → FFmpeg / mcp-video / Auto-Editor (Ausführung)
      → FCPXML / OTIO / EDL (Interchange, abgeleitet)
      → HyperFrames (Captions/Graphics, abgeleitet)
```

**Kernprinzip [NEU SEIT v3.1]:** *JSON zuerst, Render später.* Die CDL ist der einzige Primärdatensatz. Alle Renderings und Exporte sind abgeleitete, jederzeit reproduzierbare Artefakte.

#### 4.3.3 [NEU SEIT v3.1] MCP-Video-Server — Bewertungstabelle

Kernaussage: **Öffentliche MCP-Video-Server sind heute die „Hände", nicht das „Gehirn".** Die Schnittentscheidungslogik (Schritte 1–6 oben) bleibt **eigene, testbare Logik** — niemals delegiert an einen MCP-Server.

| Server | Reife-Signale | Production-Readiness | Rolle in Media Brain |
|---|---|---|---|
| **mcp-video** (KyaniteLabs) | Aktive Release-Historie, Changelog, Tests, CLI + Python-Client, HyperFrames-Anbindung, Guardrails/Preflight/QA-Checkpoints, explizite Philosophie gegen „shell-command roulette" | **Mittel–gut** für internen Executor | **Plan A Execution-Layer** — bevorzugt wegen Guardrails + QA-Checkpoints. **Nicht** Source of Truth. |
| **video-audio-mcp** | Solider FFmpeg-Werkzeugkasten (Trim, Overlays, Transitions, B-Roll, Subtitle-Burn-in, Silence Removal), FastMCP, aber **keine veröffentlichten Releases**, sichtbar einfacher | Mittel | **Plan B / Fallback-Adapter** oder Referenzimplementierung |
| **video-editing-mcp** (Video Jungle) | Produktnäher: Upload, semantische Suche, Edit-Generierung, OTIO-Export zu Resolve Studio — aber **externer kommerzieller Dienst + API-Key nötig**, Cloud-Abhängigkeit | Mittel (im Produktkontext), niedrig als neutraler Baustein | **Benchmark-/Produkt-Referenz**, nicht Backbone |
| **ffmpeg-mcp-lite** | Basale FFmpeg-Operationen, klein, PyPI-fähig, geringe Reife (wenige Commits) | Niedrig–mittel | **Plan-C-Referenz** / Minimal-Fallback |
| **ffmpeg-mcp-comp** | Umfassender Prototyp (18 Tools), aber **Projekt sagt selbst: „exploration only — not for production use"**, nicht aktiv gepflegt | Niedrig | **Nicht übernehmen** — nur Negativ-/Lernreferenz |

**Entscheidung [GEÄNDERT GEGENÜBER v3.1]:** mcp-video bleibt **Plan A Execution-Layer**, aber mit hartem QC-Gate (`video_quality_check` + Whisper-Re-Transcribe + Frame-Histogramm-Vergleich) und der ausdrücklichen Festlegung, dass es **kein Source of Truth** ist. video-audio-mcp wird in Sprint G als Plan-B-Adapter mitgetestet.

#### 4.3.4 Plan B — Eigener FFmpeg-basierter Cut-Engine-Service ([BESTÄTIGT])

Cloud Run Worker Pool, FFmpeg 7.x static, CDL → `filter_complex`. **Best Practice:** für Video-Cuts **nicht** `silenceremove` allein (reiner Audio-Filter), sondern `silencedetect` → `ametadata` → Python-Postprozess → `select`/`aselect`-Inverses mit `acrossfade` 50 ms und `setpts`/`asetpts`-Resets.

#### 4.3.5 Plan C — Auto-Editor ([BESTÄTIGT])

Auto-Editor (Public Domain / Unlicense) als CLI-Fallback im Tauri-Companion und für Massen-Roughing über Nacht. Native Exporte: Premiere XML, FCPXML, Resolve, ShotCut, Kdenlive. **Kein** OTIO-Export.

#### 4.3.6 [GEÄNDERT GEGENÜBER v3.1] Transcript-Provider — ergebnisoffenes Race

**v3.1 hatte Scribe v2 als Default gesetzt. v3.2 legt keinen Default fest.** Vier Kandidaten gehen gleichberechtigt in Sprint G; **AssemblyAI Universal-3 Pro ist Benchmark-Anker**.

| Kandidat | Wort-Timestamps | Diarisierung | Disfluency/Filler | Verbatim-Modus | EU-Residenz | Verifizierbarkeit (Mai 2026) |
|---|---|---|---|---|---|---|
| **AssemblyAI Universal-3 Pro** *(Benchmark-Anker)* | ja | ja, +$0,02/h | per Prompting verbatim-nah steuerbar | ja | EU-Endpunkte dokumentiert | **Hoch** — offizielle DE/EN-WER publiziert: ~4,88 % DE / ~4,38 % EN (FLEURS Batch); Streaming ~11,79 % DE / ~8,43 % EN; Latenzen dokumentiert |
| **Speechmatics Ursa 2** | ja | ja, Sprecherlabel pro Wort | Disfluency-Tags ODER -Entfernung | ja (Tags statt Entfernung wählen) | SaaS + On-Prem | Mittel — meldet 18 % WER-Reduktion über 50+ Sprachen, **aber keine sauber publizierte DE/EN-WER-Einzeltabelle** |
| **ElevenLabs Scribe v2** | ja, + Zeichen-Timestamps + Audio-Event-Tagging | ja, bis 32 Sprecher | Audio-Events (`[breath]`,`[um]`); Realtime bietet `no_verbatim` (deaktivieren!) | ja (no_verbatim deaktiviert lassen) | EU-/India-Residency, Zero Retention | **Uneinheitlich** — Scribe-v2-Blog betont „lowest WER", aber öffentliche Sprachseiten zeigen teils v1-gelabelte Benchmarkblöcke; Batch-Preis in Primärquellen unklar (Sekundärvergleiche nennen ~$0,22/h) |
| **WhisperX (self-hosted)** | ja, wav2vec2 forced alignment | ja, pyannote 3.1 | keine native Filler-Klassifikation; eigener Lexikon-Pass nötig | ja (immer roh) | vollständig (eigene Infra) | Software frei; WER/DER hängen von Modell + Ops ab; bekannter Alignment-Drift bei deutschen Komposita |

**Verbatim-Pflicht für alle Kandidaten:** Provider-seitige Bereinigungsoptionen werden **deaktiviert** — Speechmatics: Disfluencies *taggen*, nicht entfernen; Deepgram: `filler_words=true`; AssemblyAI: verbatim-nahes Prompting; ElevenLabs: `no_verbatim` **aus**. Der Master-Run ist immer roh.

#### 4.3.7 Drei Presets ([BESTÄTIGT])

Unverändert aus v3.1: **`natural_podcast`** (Pausen >1,2 s → 0,45 s, Filler nur markieren, keine harten Jump-Cuts), **`youtube_tight`** (Pausen >0,6 s → 0,18 s, Filler optional entfernen, Versprecher zur Review markieren), **`shorts_aggressive`** (Pausen >0,35 s → 0,08 s, Filler/Disfluency entfernen, harte Jump-Cuts, Auto-Caption-Burn-in, Hook-Zooms).

#### 4.3.8 Cut Decision List & OTIO ([BESTÄTIGT], Evidenz-Felder erweitert)

CDL als JSON-Patch-Liste mit Branching (`parent_cdl`), jeder Cut jetzt mit voller Evidenz (`gap_ms`, `vad_silence_ms`, `speaker_change`, `filler_flag`, `plosive_near`, `score`, `decision_reason`, `reversible`, `human_override`, `accepted_by_editor`). Internes Format = JSON-CDL; **OTIO 0.19 als sekundärer Export** (herstellerneutraler Mid-Layer, v.a. Resolve-Handoffs), EDL und FCPXML/Premiere-XML als Pflicht-Exporte.

#### 4.3.9 HyperFrames als Caption-/Overlay-/Motion-Graphics-Layer ([BESTÄTIGT])

HyperFrames (HeyGen, Apache 2.0) rendert deterministisch HTML+CSS+GSAP/Lottie/Three.js in MP4/WebM — **Layer-Generator** für Captions, Lower-Thirds, Overlays, Hook-Zooms, **nicht** Cutter. Läuft im selben Worker-Pool wie mcp-video.

#### 4.3.10 [NEU SEIT v3.1] Ausfallmuster (Failure Modes)

| Ausfallmuster | Ursache | Gegenmaßnahme |
|---|---|---|
| **Diarisierungs-Swaps** | Speaker-Labels springen → unnatürliche Cuts an Sprecherwechseln | `speaker_change` als Evidenz-Feld; Cuts nahe Sprecherwechsel-Grenzen herabstufen; Bad-Take-Aktion nur `mark_for_review` außer bei `shorts_aggressive` |
| **Falsche Pausen** | „Pause" ist Atmer, Lacher, dramaturgischer Hold | `audio_events`-Tabelle: Atmer/Lacher ausgenommen; `breath_keep_ms`-Preset; Cross-Check mit VAD |
| **Reconnect-Timestamp-Reset** | Streaming-Reconnect → Timestamps starten neu bei 00:00:00 (z. B. bei Deepgram dokumentiert) | Streaming **nur** für Live-Captions/Monitoring, **nie** für Master-Cut; `stream_reconnect_reset_detected`-Flag in `render_qc` |
| **Reversibilitäts-Verlust** | Schnitt auf bereinigtem `display_clean`-Transkript statt `verbatim_master` | Verbatim-First-Invariante; CDL referenziert immer `verbatim_master`-Run |
| **Semantischer NLE-Drift** | FCPXML/OTIO/EDL aus unterschiedlichen Runs erzeugt | `nle_exports.derived_from_json_run` Pflicht-FK; QC-Gate lehnt nicht-rückführbare Exporte ab |

### 4.4 Stufe 3 — Cleanup ([BESTÄTIGT])

AutoCut, FireCut, AutoPod, Phantom Editor, PremiereCopilot als Premiere-Plugins. Da Stufe 2.5 bereits Pausen/Filler entfernt hat, übernimmt Stufe 3 stärker B-Roll- und Zoom-Funktionen.

---

## 5. Thema 3 — Shortform-Pipeline ([BESTÄTIGT])

Plan A OpusClip, Plan B Vizard.ai, Plan C Submagic (Caption-/B-Roll-Layer), Reap als MCP-Kandidat, Floor-Pipeline Gemini 3.1 Flash-Lite + FFmpeg + WhisperX. Shortform-Trigger über CDL: `shorts_aggressive`-Output dient direkt als Input für OpusClip/Vizard.

---

## 6. Tool-Registry & Provider-Abstraktion ([BESTÄTIGT], STT-Einträge ergänzt)

YAML-Manifest, alle vier STT-Kandidaten als gleichberechtigte `plan: eval`-Einträge bis Sprint G entscheidet:

```yaml
tools:
  - id: transcript.assemblyai_u3pro
    kind: stt
    plan: eval            # Benchmark-Anker
    capabilities: [word_timestamps, diarization, verbatim, eu_endpoint]
  - id: transcript.speechmatics_ursa2
    kind: stt
    plan: eval
    capabilities: [word_timestamps, diarization, disfluency_tags, on_prem]
  - id: transcript.scribe_v2
    kind: stt
    plan: eval
    capabilities: [word_timestamps, char_timestamps, diarization_32, audio_events]
  - id: transcript.whisperx
    kind: stt
    plan: eval
    capabilities: [word_timestamps, forced_alignment, diarization, self_hosted]
  - id: cut_exec.mcp_video
    kind: video_edit
    plan: A
    quality_gate: required
  - id: cut_exec.video_audio_mcp
    kind: video_edit
    plan: B
  - id: cut_exec.ffmpeg_native
    kind: video_edit
    plan: B
  - id: cut_exec.auto_editor
    kind: video_edit
    plan: C
```

Nach Sprint G wird genau **ein** STT-Kandidat auf `plan: A` gehoben, ein zweiter auf `plan: B`; die übrigen bleiben als Fallback registriert.

---

## 7. Transkript-basiertes Editing-UI ([BESTÄTIGT])

`transcript-block` im BlockNote-Editor, Descript-Stil: Klick auf Wort = Seek, Selektion + Entfernen = `delete`-Patch in aktiver CDL mit Live-Preview, `youtube_tight`-Preset-Pausen-Shorten, Rechtsklick = Füllwort/Bad-Take/Keep markieren. Drei Modi: **Read**, **Edit-Words** (nur Yjs, keine CDL-Wirkung), **Edit-Cuts** (erzeugt CDL-Patches). **[NEU SEIT v3.1]** Edit-Words arbeitet auf der `display_clean`-View; Cuts referenzieren immer den `verbatim_master`-Run — Edit-Words triggert daher keine CDL-Invalidierung.

---

## 8. Phasen-Roadmap mit Tool-Test-Sprints ([BESTÄTIGT] 48 Wochen)

| Phase | Wochen | Fokus | Sprint |
|---|---|---|---|
| 0 — Setup | 1–4 | GCP-Foundation, Cloud SQL, RLS, Tool-Registry-Stub | — |
| 1 — Ingest & NLE-Sync | 5–10 | LucidLink, Frame.io V4, Tauri, Premiere UXP-Plugin | Sprint A |
| 2 — Embeddings & Vector | 11–16 | Gemini Embedding 2, Marengo 3.0, Vector Search 2.0 | Sprint B |
| 3 — Stufe 1 + Stufe 2 | 17–24 | Eddie AI v3, IntelliScript, FCPXML-Gen | Sprint C |
| 3.5 — Stufe 2.5 Micro-Cut | 25–32 | Verbatim-STT, Cut-Engine, mcp-video, Cut-Exec | **Sprint G** |
| 4 — Stufe 3 Cleanup | 33–37 | AutoCut, FireCut, AutoPod | Sprint D |
| 5 — Shortform | 38–42 | OpusClip, Vizard, Submagic, Reap | Sprint E |
| 6 — Transcript-Editing-UI | 43–46 | transcript-block, Live-Preview, Conflict-UI | Sprint F |
| 7 — Hardening & GA | 47–48 | Load-Tests, RLS-Audit, QC-Gate-Härtung | — |

### 8.1 [GEÄNDERT GEGENÜBER v3.1] Sprint G — Transcript- & Cut-Exec-Race (Wochen 25–32)

**[NEU SEIT v3.1] Micro-Cut-Testbank.** 50–100 manuell gelabelte DE/EN-Clips in vier Klassen — **Podcast**, **Interview**, **Street/Noisy**, **Doku/VO**. Pro Clip Ground Truth für: Wortgrenzen, Pause-Typen (echte Pause / Atmer / dramaturgischer Hold), editorisch akzeptierte Cuts.

**Eval-Kriterien Transcript-Provider** (ergebnisoffen, alle vier Kandidaten):

| Metrik | Definition | Plan-A-Threshold |
|---|---|---|
| WER-DE / WER-EN | gegen Ground-Truth-Transkript | ≤ 6 % / ≤ 5 % |
| Wort-Timestamp-Drift | vs. Montreal-Forced-Aligner | ≤ 80 ms |
| Diarization Error Rate | 2–4 Sprecher | ≤ 12 % |
| Audio-Event-Recall | korrekt erkannte breath/laugh/um | ≥ 80 % |
| Verbatim-Treue | Anteil erhaltener Füllwörter im Master-Run | 100 % |
| Preis pro Stunde inkl. Diarisierung | verifiziert | ≤ $0,40/h |

**[NEU SEIT v3.1] Eval-Kriterien Cut-Qualität** (nicht nur WER/DER):

| Metrik | Definition | Threshold |
|---|---|---|
| `cut_precision` | Anteil korrekter Cuts vs. Ground Truth | ≥ 0,90 |
| `cut_regret_rate` | vom Editor zurückgerollte Cuts nach Review | ≤ 8 % |
| `plosive_clip_rate` | Cuts, die Plosive abschneiden | ≤ 2 % |
| `false_pause_rate` | als Pause geschnittene Atmer/Holds | ≤ 5 % |
| `speaker_swap_after_cut` | unnatürliche Speaker-Jumps nach Cut | ≤ 3 % |
| `preview_to_nle_drift_ms` | Drift Preview ↔ NLE-Import | ≤ 1 Frame |
| `human_keep_rate` | vom Editor unverändert akzeptierte Cuts | ≥ 85 % |

**Eval-Kriterien Cut-Ausführung:** `reverse_timeline_reste` ≤ 5 s / 30 min, Audio-Klick-Rate ≤ 1/10 min, Frame-Sync-Drift ≤ 1 Frame, QC-Erfolg ≥ 98 %, CDL-Round-Trip 100 %.

**Sprint-G-Tasks:** Verbatim-STT-Adapter für alle vier Kandidaten · Normierungs-Layer (Schritt 2 der Referenzarchitektur) · eigener VAD-Layer (Silero VAD primär) · mcp-video-Sidecar + QC-Gate · eigener FFmpeg-Cut-Service · Auto-Editor-Wrapper · video-audio-mcp als Plan-B-Adapter · Testbank aufbauen + MFA-Ground-Truth · Entscheidungs-Doc + Tool-Registry final.

**[NEU SEIT v3.1] Zwei getrennte Betriebsmodi:** **Offline-Master-Transcript** (verbatim, für Micro-Cuts) und **Live-Transcript** (Streaming, für Captions/Monitoring) werden architektonisch getrennt. Streaming treibt **nie** den Master-Cut.

---

## 9. Eval-Kriterien-Kataloge ([BESTÄTIGT] A–F, Sprint G siehe 8.1)

**[NEU SEIT v3.1] Hartes Health- & QC-Set** (Tabelle `render_qc`): `timestamp_coverage_rate`, `word_overlap_rate`, `speaker_swap_rate`, `cut_regret_rate`, `fcpxml_import_success_rate`, `preview_vs_export_duration_diff_ms`, `subtitle_drift_ms`, `stream_reconnect_reset_detected`, `black_frame_rate`, `silent_render_rate`. Diese Metriken hängen an der **eigenen Cut-Engine**, nicht am MCP-Server.

**Querkriterien:** OpenTelemetry-Spans für 100 % der Aufrufe · Tool-Health-Heartbeat < 60 s · Kosten-Event pro Aufruf · RLS-Test bestanden.

---

## 10. [GEÄNDERT GEGENÜBER v3.1] Kosten- & Skalierungsmodell

**Neu kalibriert.** Die öffentlichen STT-Preise 2026 sind niedriger als oft angenommen — die reine transcript-aware Micro-Cut-Vorstufe ist eine **Cent- bis niedrige-Zehn-Cent-Frage pro Audiostunde**, solange nicht doppelt/dreifach cross-gecheckt wird.

**Verifizierbare STT-Anhaltspunkte (Mai 2026, in Sprint G zu re-verifizieren):**

| Provider | Batch | Streaming | Diarisierung |
|---|---|---|---|
| Speechmatics Ursa 2 | ab ~$0,24/h | verfügbar | inkludiert/Enhanced |
| AssemblyAI | ~$0,15–0,21/h | ~$0,45/h (U3 Pro Streaming) | +$0,02/h |
| Deepgram Nova-3 | ~$0,0048–0,0058/min (~$0,29–0,35/h) | ~$0,0077/min | extra |
| ElevenLabs Scribe v2 | ~$0,22/h *(Sekundärquelle, unbestätigt)* | <150 ms Realtime | inkludiert |
| WhisperX | nur GPU-Compute (~€0,03/h L4) | — | pyannote inklusive |

**Cost-Tiering Stufe 2.5 (neu kalibriert):**

| Posten | Light | Standard | Premium |
|---|---|---|---|
| STT (verbatim_master) | ~€0,03/h (WhisperX GPU) | ~€0,20/h (Cloud-STT) | ~€0,45/h (Cloud + Cross-Check) |
| LLM Filler/Disfluency | ~€0,02/h (Flash-Lite) | ~€0,02/h | ~€0,12/h (Pro) |
| VAD-Layer | ~€0,01/h (Silero, CPU) | ~€0,01/h | ~€0,01/h |
| Cut-Exec Compute | ~€0,10/h | ~€0,15/h | ~€0,25/h |
| HyperFrames Render | — | ~€0,15/h | ~€0,30/h |
| **Σ pro Audio-Stunde** | **~€0,16** | **~€0,53** | **~€1,13** |

**[GEÄNDERT GEGENÜBER v3.1]** Die v3.1-Schätzung (€0,55 / €0,90 / €1,80) war zu hoch — v3.2 korrigiert nach unten. GPU wird vor allem bei selbst gehostetem WhisperX relevant; bei gehosteter STT ist der teure Teil die Transkription, nicht das Herunterschneiden bereits entschiedener Segmente.

---

## 11. Risikoregister ([BESTÄTIGT] R-01 bis R-14, ergänzt)

R-01 bis R-14 unverändert aus v3.1 (UXP/CEP-Sunset, Marengo-Preis, Gemini-Outage, LucidLink-Ausfall, Yjs-OOM, Frame.io-Rate-Limits, Vector-Search-Cost, ADK-Breaking-Change, Eddie-AI-Instabilität, OpusClip-Throttle, DSGVO/US-STT, mcp-video-Jugend, Diarization-Drift, Transcript-Edit-Drift).

**[NEU SEIT v3.1] Ergänzte Risiken:**

| ID | Risiko | Wahrscheinlichkeit | Impact | Gegenmaßnahme |
|---|---|---|---|---|
| R-15 | Schnitt auf bereinigtem statt verbatim Transkript zerstört Reversibilität | mittel | hoch | Verbatim-First-Invariante; CDL referenziert nur `verbatim_master`; Provider-Clean-Optionen deaktiviert |
| R-16 | Streaming-Reconnect-Timestamp-Reset zerlegt Cut-Timing | mittel | hoch | Streaming nur für Live-Captions; `stream_reconnect_reset_detected`-Flag |
| R-17 | NLE-Export semantisch driftet (Exporte aus verschiedenen Runs) | mittel | hoch | `nle_exports.derived_from_json_run` Pflicht-FK; QC-Gate-Ablehnung |
| R-18 | STT-Provider-Transparenz-Lücke (Scribe v2 / Speechmatics) führt zu Fehlentscheidung | mittel | mittel | Ergebnisoffenes Sprint-G-Race auf eigener Testbank; AssemblyAI als Benchmark-Anker; keine Vorab-Festlegung |
| R-19 | Falsche Pausen-Cuts (Atmer/Lacher/Holds) | mittel | mittel | `audio_events`-Tabelle, VAD-Cross-Check, `false_pause_rate`-Metrik, `breath_keep_ms`-Preset |

---

## 12. Konsistenzcheck — keine v3.1-Funktion verloren

✅ Cloud-Backbone (Cloud Run, Worker Pools, GPU, Cloud SQL 17, pgvector 0.8, Vector Search 2.0, Eventarc Advanced, ADK 2.1.0) — 2.1  
✅ KI/ML-Stack (Gemini Embedding 2, Marengo 3.0, Voyage 3.5, Gemini 3.1 Pro/Flash-Lite, Claude Sonnet 4.6) — 2.2  
✅ NLE-Integration (Premiere UXP, CEP-Sunset Q3 2026, Resolve 20.x/21, LucidLink, Frame.io V4, rclone, Tauri 2) — 2.3  
✅ Editor-Stack (BlockNote, Tiptap, Yjs, Hocuspocus, Liveblocks, y-indexeddb, Loro, Fountain) — 2.4  
✅ Datenmodell + RLS (alle v3.1-Tabellen inkl. transcript_words, cut_decision_list, micro_cut_runs) — 2.5  
✅ Three-Stage-Pipeline (Stufe 1/2/2.5/3) — 4  
✅ Stufe 2.5 Micro-Cut-Engine inkl. drei Presets — 4.3  
✅ mcp-video / FFmpeg-Service / Auto-Editor (Plan A/B/C) — 4.3.3–4.3.5  
✅ HyperFrames als Caption/Overlay-Layer — 4.3.9  
✅ Transkript-basiertes Editing-UI — 7  
✅ Shortform-Pipeline (OpusClip/Vizard/Submagic/Reap/Floor) — 5  
✅ Tool-Registry + Provider-Abstraktion (ADK + MCP) — 6  
✅ 48-Wochen-Roadmap, Sprints A–G — 8  
✅ Cost-Tiering Light/Standard/Premium — 10  
✅ OpenTelemetry, QC-Metriken — 9  
✅ Risiken R-01 bis R-14 — 11  

**Ergebnis: 0 Funktionen verloren, 7 strukturelle Verbesserungen integriert.**

---

## 13. Änderungsprotokoll v3.1 → v3.2

| # | Sektion | Art | Beschreibung |
|---|---|---|---|
| 1 | 0, 1 | NEU | Verbatim-First-Prinzip + „JSON zuerst, Render später" als Designprinzipien |
| 2 | 2.2, 4.3.6 | GEÄNDERT | STT-Default für Micro-Cut zurückgenommen; ergebnisoffenes Sprint-G-Race, AssemblyAI U3 Pro als Benchmark-Anker |
| 3 | 2.3 | GEÄNDERT | Premiere-Legacy-Bridge präzisiert (read-/import-only, app.openFCPXML, isoliert) |
| 4 | 2.5 | GEÄNDERT | 8 neue Tabellen (transcript_runs, speaker_segments, audio_events, cut_runs, cut_evidence, edit_revisions, nle_exports, render_qc); Verbatim-/Evidenz-Felder; Verbatim-Master-Invariante |
| 5 | 3 | NEU | Export-Hierarchie fixiert (JSON → FCPXML/OTIO/EDL → NLE → Render), derived_from_json_run Pflicht-FK |
| 6 | 4.3.2 | NEU | Verbatim-First-Referenzarchitektur (kanonische sechsstufige Pipeline) |
| 7 | 4.3.3 | NEU | MCP-Video-Server-Bewertungstabelle (5 Server); „Hände nicht Gehirn"-Festlegung |
| 8 | 4.3.6 | GEÄNDERT | Transcript-Provider-Tabelle mit Verifizierbarkeits-Spalte und Verbatim-Pflicht |
| 9 | 4.3.10 | NEU | Ausfallmuster-Tabelle (Diarisierungs-Swaps, falsche Pausen, Reconnect-Reset, Reversibilitätsverlust, NLE-Drift) |
| 10 | 8.1 | GEÄNDERT | Sprint G um Micro-Cut-Testbank (50–100 Clips, 4 Klassen) und Cut-Qualitäts-Metriken erweitert; zwei getrennte Betriebsmodi |
| 11 | 9 | NEU | Hartes Health- & QC-Set (render_qc-Metriken) |
| 12 | 10 | GEÄNDERT | Kostenmodell neu kalibriert (STT günstiger; Tiering €0,16/€0,53/€1,13) |
| 13 | 11 | GEÄNDERT | Risiken R-15 bis R-19 ergänzt |
| 14 | 12, 13 | NEU | Konsistenzcheck + dieses Änderungsprotokoll |

---

## 14. Schlussempfehlung

**Nächste 4 Wochen:** Micro-Cut-Testbank aufbauen (Priorität: deutsches Street/Noisy- und Interview-Material mit manuellem MFA-Ground-Truth). Verbatim-STT-Adapter für alle vier Kandidaten gleichzeitig in die ADK-Tool-Registry integrieren — **mit deaktivierten Provider-Clean-Optionen**. Eigenen VAD-Layer (Silero VAD) und Normierungs-Schema bauen, bevor irgendein Cut-Code entsteht.

**Hartes Go/No-Go für Stufe 2.5:** Sprint G muss auf der eigenen Testbank `cut_precision ≥ 0,90`, `cut_regret_rate ≤ 8 %`, `human_keep_rate ≥ 85 %` und Wort-Timestamp-Drift ≤ 80 ms erreichen. Sonst bleibt Stufe 2.5 hinter dem Feature-Flag und v3.2 liefert zunächst ohne aktivierte 2.5 aus.

**Hartes Go/No-Go für STT-Provider:** Kein Provider wird vor Sprint-G-Abschluss produktiv gesetzt. Gewinner = bester gewichteter Score aus Qualität (cut_precision, WER, Drift) + Integrationsaufwand + Bedienbarkeit + Kosten — nicht der niedrigste Marketing-WER.

**Hartes Go/No-Go für mcp-video:** > 5 % QC-Gate-Fehlschläge über 4 Wochen Eval → Plan B (eigener FFmpeg-Service / video-audio-mcp) wird Default; mcp-video bleibt nur HyperFrames-Wrapper.

**Unverhandelbar:** Das `verbatim_master`-Transkript wird niemals bereinigt. Jeder Export trägt seinen `derived_from_json_run`. Die Schnittentscheidungslogik bleibt eigener, getesteter Code — kein MCP-Server ist jemals Source of Truth.
