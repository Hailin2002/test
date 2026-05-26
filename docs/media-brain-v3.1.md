# Media Brain — Build- und Research-Plan v3.1 (Stand Mai 2026)

> **Hinweis zur Versionierung:** Dieser Master-Plan v3.1 ist eine vollständige Neuausgabe gegenüber v3 (Mai 2026). Er enthält **alle** Sektionen, Funktionen und Empfehlungen aus v3 in unveränderter Form plus eine tief integrierte neue Pipeline-Stufe 2.5 („Transcript-Aware Micro-Cut Engine”) und ein transkript-basiertes Editing-UI. Jede inhaltliche Änderung gegenüber v3 ist markiert mit **[NEU SEIT v3]**, **[GEÄNDERT GEGENÜBER v3]** oder **[BESTÄTIGT]**.

-----

## 0. Executive Summary der Änderungen v3 → v3.1

**Was sich ändert:**

1. **[NEU SEIT v3] Pipeline-Stufe 2.5 — Transcript-Aware Micro-Cut Engine.** Zwischen dem groben AI-Rough-Cut (Stufe 2) und dem mechanischen Cleanup (Stufe 3) entsteht eine eigene Kern-Stufe, die minutengenauen, **wortbasierten** Feinschnitt von Sprache leistet: Redepausen erkennen und **kürzen** (nicht nur löschen), Füllwörter (äh/ähm/also) markieren oder entfernen, Versprecher und doppelte Takes erkennen, Jump-Cuts natürlich glätten, Atempausen-Presets, reversible „Cut Decision List”.
1. **[NEU SEIT v3] Transkript-basiertes Editing-UI** (Descript-Stil) als dokumentierte Produktfunktion. Der BlockNote-Editor erhält einen Transcript-Mode: Text im Transkript löschen → Schnitt im Video. Live-Preview über Yjs + Scribe v2 / WhisperX.
1. **[NEU SEIT v3] HyperFrames** (HeyGen, Apache 2.0) wird als deterministischer Captions-, Lower-Third-, Overlay- und Motion-Graphics-Layer dokumentiert — ergänzend, nicht ersetzend.
1. **[NEU SEIT v3] Datenmodell-Erweiterung**: drei neue Tabellen `transcript_words`, `cut_decision_list`, `micro_cut_runs` plus Spalten in `agent_traces` und `tool_health`.
1. **[NEU SEIT v3] Tool-Test-Sprint G**: Transcript-Provider-Race (Scribe v2 vs. Speechmatics vs. WhisperX vs. Deepgram Nova-3) und Cut-Ausführungs-Race (mcp-video vs. eigener FFmpeg-Service vs. Auto-Editor vs. AutoCut).
1. **[GEÄNDERT GEGENÜBER v3]** Roadmap wird von **44 auf 48 Wochen** verlängert, weil Sprint G zwei Sprintslots beansprucht.
1. **[GEÄNDERT GEGENÜBER v3]** Kosten-Tiering Light/Standard/Premium um eine **Micro-Cut-Komponente** erweitert (€0,55–€1,80 pro 60 min Talking-Head).
1. **[GEÄNDERT GEGENÜBER v3]** Risikoregister um drei neue Risiken (R-12 bis R-14): „mcp-video noch jung”, „Diarisierungs-Drift bei deutschen Talking-Heads”, „Transcript-Edit-Drift bei Cross-CRDT-Replay”.

**Was unverändert bleibt:** Cloud-Backbone (GCP), KI/ML-Stack (Gemini 3.1 + Marengo 3.0 + Voyage), NLE-Integration (Premiere UXP + Resolve 21 + LucidLink + Frame.io V4), Three-Stage-Pipeline-Architektur, Editor (BlockNote/Tiptap), Realtime-Stack (Yjs/Hocuspocus/Liveblocks), Datenmodell-Kernschema, OpenTelemetry, Tool-Registry via YAML + Cloud API Registry, Provider-Abstraktion über ADK + MCP.

-----

## 1. Vision & Designprinzipien

**Vision.** Media Brain ist ein KI-gestütztes Video-Produktions-Betriebssystem, das die gesamte Wertschöpfung von Roh-Footage über Skript, Vector-Retrieval, Rough-Cut, Feinschnitt, Shortform-Repurposing bis zur NLE-Übergabe agentisch orchestriert. Es ist NLE-agnostisch, Provider-agnostisch und tenant-mandantenfähig.

**Designprinzipien (alle [BESTÄTIGT] aus v3, plus eines [NEU SEIT v3]):**

- **Plan A/B/C für jede kritische Funktion**: ein Hauptpfad, ein gleichwertiger Fallback, ein Notbackup.
- **Provider-Abstraktion über ADK + MCP**: kein Modell, kein Tool darf hart verdrahtet sein. Wechsel erfolgt über die Tool-Registry.
- **Reversibilität**: keine destruktiven Edits ohne CDL/Snapshot. Jeder Cut, jede Caption, jedes Overlay ist als Patch persistiert und kann zurückgerollt werden.
- **Postgres Row Level Security als harte Tenant-Grenze**.
- **OpenTelemetry-Spans pro Pipeline-Schritt**, kombiniert mit `tool_health`-Heartbeats.
- **Self-Healing Tool Plugin** im ADK: Werkzeuge mit hartem Fehler werden binnen 30 s aus der Routing-Tabelle genommen.
- **[NEU SEIT v3] Audio is source-of-truth für Cuts** — die Micro-Cut-Engine schreibt nur dann ins NLE, wenn Audio-Silence-Detection (FFmpeg `silencedetect`) und Transkript-Lücken einander bestätigen (Cross-Check).

-----

## 2. Architektur-Überblick

### 2.1 Cloud-Backbone (Google Cloud, [BESTÄTIGT])

|Schicht               |Komponente                                                                                   |Stand v3.1 |
|----------------------|---------------------------------------------------------------------------------------------|-----------|
|Compute (stateless)   |**Cloud Run** (2nd gen)                                                                      |[BESTÄTIGT]|
|Compute (long-running)|**Cloud Run Worker Pools** (GA)                                                              |[BESTÄTIGT]|
|Compute (GPU)         |**Cloud Run GPU** mit NVIDIA L4 (Standard) und NVIDIA RTX PRO 6000 Blackwell (Premium-Render)|[BESTÄTIGT]|
|OLTP                  |**Cloud SQL PostgreSQL 17** + pgvector 0.8                                                   |[BESTÄTIGT]|
|Object Storage        |**Google Cloud Storage** (Multi-Region eu, Autoclass)                                        |[BESTÄTIGT]|
|Vector Search         |**Vertex AI Vector Search 2.0** (GA, Hybrid Dense+Sparse)                                    |[BESTÄTIGT]|
|Eventing              |**Eventarc Advanced** + Pub/Sub                                                              |[BESTÄTIGT]|
|Agent Platform        |**Gemini Enterprise Agent Platform** (vormals Vertex AI Agent Builder)                       |[BESTÄTIGT]|
|Agent SDK             |**Google ADK 2.1.0** mit Self-Healing Tool Plugin                                            |[BESTÄTIGT]|
|Tool Registry         |**Cloud API Registry** + YAML-Manifest im Repo                                               |[BESTÄTIGT]|

### 2.2 KI/ML-Stack ([BESTÄTIGT] aus v3, ergänzt durch Micro-Cut-Bedarf)

|Aufgabe                                                                  |Plan A                          |Plan B                                          |Plan C / Notbackup                           |
|-------------------------------------------------------------------------|--------------------------------|------------------------------------------------|---------------------------------------------|
|Multimodale Embeddings                                                   |**Gemini Embedding 2**          |Voyage `voyage-multimodal-3.5`                  |Vertex Multimodal Embedding                  |
|Premium Video-Index                                                      |**TwelveLabs Marengo 3.0**      |Marengo 2.x                                     |—                                            |
|Visual Document Retrieval                                                |**Voyage voyage-multimodal-3.5**|ColPali Self-Hosted                             |—                                            |
|Skript / Vision / Hook-Scoring                                           |**Gemini 3.1 Pro**              |**Gemini 3.1 Flash-Lite** (Floor)               |Claude Sonnet 4.6 (Cross-Check)              |
|Cross-Check-LLM                                                          |**Claude Sonnet 4.6**           |Gemini 3.1 Pro                                  |GPT-5 (only via Vertex Garden)               |
|Transkription DE (Hauptsystem, Long-Form)                                |**Speechmatics Enhanced**       |**ElevenLabs Scribe v2**                        |WhisperX large-v3                            |
|**[GEÄNDERT GEGENÜBER v3]** Transkription Micro-Cut (word-level, Default)|**ElevenLabs Scribe v2** Batch  |Speechmatics (mit Wort-Timestamps + Diarization)|WhisperX large-v3 + wav2vec2 forced alignment|
|Notbackup STT                                                            |Deepgram Nova-3                 |AssemblyAI Universal-3                          |—                                            |

**Begründung [GEÄNDERT GEGENÜBER v3]:** Für die neue Stufe 2.5 wechselt der **Default-Provider für Micro-Cut-Transkription auf ElevenLabs Scribe v2**, weil dieser word-level Timestamps mit `type`-Klassifikation (`word`, `spacing`, `audio_event`) ausliefert, **bis zu 32 Sprecher** diarisiert  und nicht-Sprach-Events (Lachen, Atmen, „um”) automatisch als `audio_event` taggt  — alles Eigenschaften, die für „Atempausen-Presets” und „Bad-Take-Erkennung” zentral sind. ElevenLabs gibt für Scribe v2 Realtime **93,5 % Accuracy auf dem FLEURS-Multilingual-Benchmark** über 30 Sprachen an und damit den niedrigsten WER unter low-latency-ASR-Modellen, vor Gemini Flash 2.5 (90 %), GPT-4o Mini (85 %) und Deepgram Nova-3 (80 %).   Speechmatics bleibt der Hauptkanal für **lange Format-Transkripte** (Doku, Podcast-Master), weil dessen Speaker Diarization als Kernfunktion ohne Aufpreis inkludiert ist  und On-Prem-Deployment für regulierte Tenants verfügbar ist.

### 2.3 NLE-Integration ([BESTÄTIGT])

- **Adobe Premiere Pro**: UXP-Plugin (UXP-first). Harte CEP-Sunset-Migration bis **Q3 2026** unverändert.
- **DaVinci Resolve 20.x / 21 Beta**: Headless Python-API, IntelliScript als Pre-LLM-Rough-Cut-Backend.
- **Sync-Backbone**: **LucidLink Filespace** für Bulk-Footage, **Frame.io V4 REST-API** für Review/Kommentare, `rclone bisync` als Fallback bei LucidLink-Ausfall.
- **Companion-App**: **Tauri 2** für Ingest-Watcher, lokale GPU-Whisper-Fallback und Offline-Edit-Queue.
- **[NEU SEIT v3] Text-Based-Editing-Brücke**: Premiere Pros eingebautes Text-Based Editing (Adobe Sensei, dokumentiert in Adobe Help, Stand Mai 2026, mit `Transcribe Sequence`-Workflow und automatischem Ripple Edit bei Transcript-Cuts)  wird **nicht** als Konkurrenz behandelt, sondern als **Export-Ziel**. Die Micro-Cut-Engine kann ihren Cut als Premiere-XML mit eingebettetem Transkript exportieren, sodass der Editor die Schnitte direkt im Adobe-Text-Panel kuratieren kann.

### 2.4 Editor & Frontend ([BESTÄTIGT], um Transcript-UI ergänzt)

- **Editor-Kern**: **BlockNote** (Notion-artig) auf **Next.js**.
- **Migrationsspur**: **Tiptap 3** (vorbereitet für Fall, dass BlockNote-Roadmap stagniert).
- **Realtime**: **Yjs** + **Hocuspocus** (Self-Hosted) + **Liveblocks** (Managed-Fallback).
- **Offline**: `y-indexeddb`.
- **CRDT-Migrationsspur**: **Loro** für künftige Multi-Device-Sync mit reichem Undo-Tree.
- **Skript-Markup**: Fountain mit eckigen Klammern für In-Line-Notes, kompatibel zum AI-Rough-Cut.
- **[NEU SEIT v3] Transcript-View**: Eine neue BlockNote-Block-Variante `transcript-block` rendert ein wort-genau-getimtes Transkript mit drei Modi:
  - **Read** (nur Anzeige, Klick auf Wort = Springe an Timeline-Position).
  - **Edit-Words** (Korrektur des Textes ohne Cut-Effekt).
  - **Edit-Cuts** (Descript-Stil: Selektion löschen ⇒ erzeugt CDL-Patch ⇒ Live-Preview im rechten Panel).

### 2.5 Datenmodell ([BESTÄTIGT] + neue Tabellen)

**Bestand (unverändert):** `projects`, `media`, `embedding_runs`, `agent_traces`, `tool_health`, `tenants`, `users`, `roles`, `cost_events`, `rough_cuts`, `shortform_jobs`. Alle mit Postgres RLS auf `tenant_id`.

**[NEU SEIT v3] Drei neue Tabellen:**

```sql
-- Wort-Index pro Mediendatei
CREATE TABLE transcript_words (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       UUID NOT NULL,
  media_id        UUID NOT NULL REFERENCES media(id),
  transcript_run  UUID NOT NULL,              -- versioniert
  word_index      INT NOT NULL,
  text            TEXT NOT NULL,
  start_s         NUMERIC(10,3) NOT NULL,
  end_s           NUMERIC(10,3) NOT NULL,
  speaker_id      TEXT,
  confidence      NUMERIC(4,3),
  word_type       TEXT,                       -- 'word' | 'spacing' | 'audio_event' | 'filler'
  is_filler       BOOLEAN DEFAULT FALSE,
  is_disfluency   BOOLEAN DEFAULT FALSE,
  provider        TEXT NOT NULL,              -- 'scribe_v2' | 'speechmatics' | 'whisperx'
  created_at      TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX ON transcript_words (media_id, transcript_run, word_index);

-- Reversible Cut Decision List
CREATE TABLE cut_decision_list (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id       UUID NOT NULL,
  media_id        UUID NOT NULL,
  micro_cut_run   UUID NOT NULL,
  parent_cdl      UUID,                       -- für Rollback / Branching
  preset          TEXT NOT NULL,              -- 'natural_podcast' | 'youtube_tight' | 'shorts_aggressive' | 'custom'
  cuts            JSONB NOT NULL,             -- Array of {start, end, action, source}
  rationale       JSONB,                      -- Optional LLM-Erklärung pro Cut
  is_active       BOOLEAN DEFAULT TRUE,
  created_by      TEXT NOT NULL,              -- 'agent' | 'editor' | 'preset'
  created_at      TIMESTAMPTZ DEFAULT now()
);

-- Laufzeit-Metadaten der Micro-Cut-Runs
CREATE TABLE micro_cut_runs (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id          UUID NOT NULL,
  media_id           UUID NOT NULL,
  preset             TEXT NOT NULL,
  pause_threshold_s  NUMERIC(4,2) NOT NULL,
  pause_shorten_to_s NUMERIC(4,2),
  silence_dbfs       NUMERIC(5,1),
  buffer_pre_ms      INT,
  buffer_post_ms     INT,
  exec_backend       TEXT NOT NULL,           -- 'mcp_video' | 'ffmpeg_native' | 'auto_editor' | 'autocut'
  cdl_id             UUID REFERENCES cut_decision_list(id),
  output_mp4_uri     TEXT,
  output_fcpxml_uri  TEXT,
  output_otio_uri    TEXT,
  cost_cents         INT,
  duration_input_s   NUMERIC(10,3),
  duration_output_s  NUMERIC(10,3),
  status             TEXT NOT NULL,
  trace_id           TEXT,
  created_at         TIMESTAMPTZ DEFAULT now()
);
```

**Erweiterungen an Bestandstabellen:**

- `agent_traces.micro_cut_run` (FK, nullable): verknüpft Agenten-Trace mit konkretem Cut-Lauf.
- `tool_health` bekommt neue Provider-Keys: `transcript.scribe_v2`, `transcript.speechmatics`, `transcript.whisperx`, `cut_exec.mcp_video`, `cut_exec.ffmpeg_native`, `cut_exec.auto_editor`.
- `cost_events.category` erweitert um `transcript`, `micro_cut_exec`.

-----

## 3. Thema 1 — NLE-Sync

**[BESTÄTIGT]** Die in v3 spezifizierte Sync-Architektur bleibt:

1. **Ingest** über Tauri-Companion-App in lokalem Watch-Ordner.
1. **Hochladen** in LucidLink Filespace; **Proxy-Generation** auf Cloud Run GPU L4 (DNxHR LB).
1. **Frame.io V4** als Master für Reviews und Kommentare; Webhook-Subscription auf `comment.created`.
1. **rclone bisync** als Notpfad (eu-multi GCS bucket).
1. **NLE-Adapter**: Premiere via UXP, Resolve via Headless-Python.
1. **Konflikt-Resolution**: optimistisches Locking auf Bin-Ebene + 3-Way-Merge via Yjs (für Skripte) bzw. Frame.io-Comments (für Cut-Notizen).

**[NEU SEIT v3] Adapter für die Stufe-2.5-Outputs.** Jeder NLE-Adapter muss zusätzlich CDL → Format konvertieren:

- **Premiere**: Sequence-XML mit eingebettetem Transcript-Layer plus Marker-Layer mit Cut-Begründung pro Cut.
- **Resolve**: EDL (CMX 3600) für harte Cuts + DRP-Bin-Import mit Subclips für reversible Cuts.
- **FCP X**: FCPXML 1.13 mit Marker-Layer.

-----

## 4. Thema 2 — Vector-Rough-Cut-Pipeline (jetzt mit Stufe 2.5)

### 4.1 Stufe 1 — Vector-Müll-Filter ([BESTÄTIGT])

Unverändert: Multi-Query-Embedding (Gemini Embedding 2 als Default), MMR-Re-Ranking, DPP für Diversität, **Vertex AI Vector Search 2.0 Hybrid** (Dense + Sparse BM25-Hybrid für deutsche Eigennamen) als Backend.

### 4.2 Stufe 2 — AI-Rough-Cut ([BESTÄTIGT])

- **Plan A**: **Eddie AI v3 Night Shift** (asynchroner Übernacht-Schnitt) als primärer Rough-Cut-Generator.
- **Plan B**: **DaVinci Resolve IntelliScript** als deterministischer Fallback (Resolve 21 Headless-Python).
- **Plan C**: Eigener **FCPXML-Generator** auf Basis Gemini-3.1-Pro-Plan + Marengo-3.0-Shot-Ranking, sodass völliger Provider-Lock-out überbrückt werden kann.

### 4.3 [NEU SEIT v3] Stufe 2.5 — Transcript-Aware Micro-Cut Engine

#### 4.3.1 Ziel & Abgrenzung

Stufe 2.5 ist **kein** weiterer Rough-Cut und **kein** mechanisches Cleanup, sondern eine eigene Kernfunktion für **wort-genauen Feinschnitt von Sprache** in Talking-Head-, Podcast- und YouTube-Material. Sie sitzt zwischen Stufe 2 (Eddie AI hat Szenen ausgewählt und in eine Sequenz gelegt) und Stufe 3 (FireCut/AutoPod machen mechanisches Cleanup auf Timeline-Ebene). Sie liefert dem Cleanup-Sprint einen erheblich kürzeren, sauberer geschnittenen Ausgangs-Cut und ersetzt damit den Großteil der manuellen Arbeit „Zoom-rein-und-Pausen-killen”.

Sie macht:

1. **Redepausen erkennen und kürzen** (nicht nur löschen) — wirkt natürlicher.
1. **Füllwörter markieren** (äh/ähm/also/halt/eben/quasi).
1. **Versprecher und Selbstkorrekturen erkennen** („…ich war am — also ich war am Bahnhof”).
1. **Doppelte Takes erkennen** (zwei aufeinanderfolgende Sätze mit hoher semantischer Ähnlichkeit + Trennungs-Pause).
1. **Jump-Cuts natürlich glätten** mit kleinen Audio-Crossfades und Pre/Post-Buffer-Frames.
1. **Atempausen-Presets** (Atmer < Schwellwert behalten, > Schwellwert kürzen).
1. **Alles reversibel** über die `cut_decision_list`.

Die Wertigkeit dieser Stufe ist durch öffentliche Benchmarks belegt: TimeBolts „AI Video Editor Showdown”-Tests an 60-Minuten-Zoom-Recordings zeigen, dass die reine Stufe-3-Welt nach Auto-Edits noch 4–17 min Restmaterial in Form unsauber gekürzter Pausen und Füllwörter lässt (Descript: 6 min 30 s, Gling: 4 min 17 s, Loom: 15 min 30 s pro Stunde).  Eine dedizierte word-level Stufe vor Stufe 3 schließt diese Lücke.

#### 4.3.2 Technische Logik (kanonische Pipeline)

```
[1] Audio extrahieren  (FFmpeg, 16 kHz mono PCM)
       │
[2] Transkript mit word-level timestamps holen
    (Plan A: Scribe v2 batch, timestamps_granularity=word, diarize=true,
     tag_audio_events=true → liefert auch [breath], [laugh], [um], etc.)
       │
[3] Pause-Kandidaten: für jede aufeinanderfolgende Wort-Paarung (w_i, w_i+1)
    berechne gap = w_i+1.start − w_i.end. Wenn gap > T_pause (Preset), markiere.
       │
[4] Audio-Silence-Detection (FFmpeg silencedetect, n=-30dB, d=T_pause)
    Cross-Check: nur Pausen, die in BEIDEN Quellen erscheinen, werden geschnitten.
    (Mismatch → in agent_traces.notes loggen, im UI als „Konflikt" markieren.)
       │
[5] Pausen-Aktion (nicht zwingend löschen):
        if gap > T_keep:           cut to T_target_pause   (z.B. 0.45s)
        elif gap > T_pause:        cut to T_short_pause    (z.B. 0.18s)
        else:                      keep
       │
[6] Füllwort-Pass: LLM-Klassifikation (Gemini 3.1 Flash-Lite) über kontextuelle
    Bigrams + Whitelist (äh, ähm, also, halt, eben, quasi). Zusatzregel:
    Füllwort wird nur entfernt, wenn umgebende Wörter > 90 % Confidence haben.
       │
[7] Versprecher-Pass: erkenne Muster (word A) — pause(≥250 ms) — (word A')
    wobei A und A' Levenshtein-Distanz ≤ 2 ODER semantische Cosine > 0.85.
    Erkenne Wiederanfänge („ich war — ich war am Bahnhof").
       │
[8] Bad-Take-Pass: für aufeinanderfolgende Sätze S_n und S_n+1 mit
    cosine_sim(embed(S_n), embed(S_n+1)) > 0.90 UND Pause > 1.5 s zwischen
    ihnen → markiere S_n als Bad Take (Vorschlag, nicht harte Entfernung).
       │
[9] Buffer setzen: jeder Cut bekommt buffer_pre_ms (Default 40 ms) und
    buffer_post_ms (Default 60 ms) als Atem- und Mund-Schließ-Reserve.
       │
[10] CDL (Cut Decision List) als JSON persistieren in cut_decision_list.
       │
[11] Ausführung wählen (Plan A/B/C):
        A) mcp-video MCP-Server (video_ai_remove_silence, video_trim, video_merge)
        B) Eigener FFmpeg-basierter Cut-Engine-Service (Cloud Run Worker Pool,
           generiert filter_complex aus CDL).
        C) Auto-Editor 30.x (CLI, --edit audio --margin 0.4s,0.2s --export premiere)
           oder AutoCut-Plugin in Premiere als Editor-Side-Fallback.
       │
[12] Output: fertige MP4 (für Preview/Frame.io) + FCPXML/Premiere-XML/Resolve-EDL +
     CDL.json + Audit-Eintrag in micro_cut_runs.
```

#### 4.3.3 Plan A — Cut-Ausführung über mcp-video

**[NEU SEIT v3]** **mcp-video** (Apache 2.0, KyaniteLabs) ist ein MCP-Server, der FFmpeg und HyperFrames hinter typisierten MCP-Tools kapselt.  Aktuelle Version v1.4.0 (Release 9. Mai 2026). **Wichtig — Reifegrad:** Das Repository ist mit aktuellem Release noch sehr jung (zweistelliger Stars-Bereich, kleines Maintainer-Team, wenige offene Issues); der KyaniteLabs-Token markiert es in der README explizit als kein Production-Tool: *„Do not publish agent-generated video without `video_quality_check`, `video_release_checkpoint`, and human visual/audio inspection.”*   Wir setzen mcp-video daher **als Plan A im Evaluations- und Produktions-Tracking, aber mit hartem Quality-Gate**: jeder Output durchläuft `video_quality_check` + Whisper-Re-Transcribe + Frame-Histogramm-Vergleich, bevor er via Frame.io zur Freigabe gestellt wird.

Relevante MCP-Tools (Auszug aus mcp-video TOOLS.md, kategorisiert):

- **Core Editing**: `video_trim`, `video_merge` (Concat mit optionalen Transitions), `video_subtitles`/`video_subtitles_styled` (Burned-In-Captions), `video_overlay`, `video_split_screen`, `video_speed`, `video_chroma_key`, `video_stabilize`, `video_edit` (JSON-DSL Timeline).
- **AI-Powered**: `video_ai_remove_silence`, `video_ai_transcribe` (Whisper), `video_ai_scene_detect`, `video_ai_stem_separation` (Demucs), `video_quality_check`, `video_release_checkpoint`.
- **Repurposing**: `video_repurpose_plan`, `video_repurpose`  (Shorts/Reels/TikTok).
- **Hyperframes (18 Tools)**: `hyperframes_init`, `hyperframes_render`, `hyperframes_snapshot`,  `hyperframes_catalog`, `hyperframes_tts`, `hyperframes_transcribe`, `hyperframes_remove_background`, `hyperframes_doctor`, `hyperframes_benchmark`, `hyperframes_to_mcpvideo`.
- **Discovery**: `search_tools`  (LLM-natives Tool-Discovery für lange Tool-Listen).

**Python-Client (Beispiel verbatim aus mcp-video README):**

```python
from mcp_video import Client
editor = Client()
clip = editor.trim("interview.mp4", start="00:02:15", duration="00:00:45")
caption_file = "captions.srt"
editor.ai_transcribe(clip.output_path, output_srt=caption_file)
captioned  = editor.subtitles(clip.output_path, subtitle_file=caption_file)
vertical   = editor.resize(captioned.output_path, aspect_ratio="9:16")
checkpoint = editor.release_checkpoint(vertical.output_path)
```

**Integration in Media Brain.** mcp-video läuft als Sidecar-Container in einem Cloud Run Worker Pool (Python 3.11, FFmpeg static build, Node 22 für HyperFrames-CLI). Das ADK ruft die MCP-Tools via Standard-MCP-Transport (stdio im Worker, HTTP-RPC für Multi-Tenant-Gateway gemäß MCP-Spec 2026-07-28 RC „stateless core”).  Tool-Health-Heartbeat alle 60 s.

#### 4.3.4 Plan B — Eigener FFmpeg-basierter Cut-Engine-Service

Wenn mcp-video ein Tool nicht anbietet (z. B. komplexer multi-track Audio-Crossfade) oder die Quality-Gates wiederholt fehlschlagen, springt unser eigener Cut-Engine-Service ein. Architektur:

- **Container**: Cloud Run Worker Pool, FFmpeg 7.x static, ffmpeg-python, Pydantic-Schemas für CDL.
- **API**: `POST /v1/cdl/execute` mit CDL-JSON, liefert signed GCS-URL für MP4 + sidecar XML.
- **Filter-Strategie**: aus CDL wird `filter_complex` generiert mit `aselect`/`vselect`-Inverses, `acrossfade` 50 ms zwischen Segmenten, `setpts=N/FRAME_RATE/TB` und `asetpts=N/SR/TB` für Sync-Resets.
- **Best Practice (FFmpeg 2026)**: Für Video-Cuts NICHT `silenceremove` allein verwenden (das ist ein reiner Audio-Filter, der das Video nicht mitschneidet,  dokumentiert u. a. in Rendi-Docs und im `bambax/Remsi`-Projekt), sondern **`silencedetect` → `ametadata=mode=print:file=…` → Python-Postprozess → `select`/`aselect`-Inverses** (Rendi-Empfehlung „Jump-Cuts-Technik”). Für reine Audio-Dateien bleibt der Direkt-`silenceremove`-Pfad mit `start_periods/stop_periods` ein gültiger Schnellpfad.

#### 4.3.5 Plan C — Auto-Editor / AutoCut als Editor-seitiger Fallback

**[NEU SEIT v3]** **Auto-Editor** (Public Domain / Unlicense, WyattBlue) Version **30.2.4 (Release 19. Mai 2026)** ist mit **4,3k GitHub-Stars und 551 Forks** das mit Abstand reifste Open-Source-Tool für automatisches Schneiden nach Lautstärke (`--edit audio`)  und optional Motion-Detection (`--edit motion`).  Es exportiert nativ in **Premiere XML, FCPXML, Resolve, ShotCut, Kdenlive, Clip-Sequence**  — **keinen** OTIO-Export. Lizenz ist Public Domain (Unlicense), nicht Apache 2.0; das ist relevant für Anbieter mit strikter „Apache-only”-Policy. Es kommt als CLI-Fallback in zwei Szenarien zum Einsatz:

1. **Self-Hosting im Editor-Companion** (Tauri-App), falls Cloud-Pipeline ausfällt.
1. **Massen-Roughing über Nacht**, wenn ein ganzer Drehtag in einem Schritt von Stille befreit werden soll, bevor Eddie AI überhaupt einen Rough-Cut wagt.

CLI-Beispiele (verbatim aus Repo):

```bash
auto-editor example.mp4 --margin 0.3s,1.5sec --export premiere
auto-editor example.mp4 --edit "(or audio:0.03 motion:0.06)"
```

**AutoCut** (kommerzieller Premiere/Resolve-Plugin, ab ~€6,60/Monat) bleibt als reine Editor-Hand-Trigger-Variante in Stufe 3.

#### 4.3.6 Transcript-Provider — Plan A/B/C

|Plan         |Provider                                                       |Wort-Timestamps                                           |Diarisierung                                                    |Realtime                                                      |Preis (Stand Mai 2026)                                                              |DE-Genauigkeit                                                              |
|-------------|---------------------------------------------------------------|----------------------------------------------------------|----------------------------------------------------------------|--------------------------------------------------------------|------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
|**A**        |**ElevenLabs Scribe v2 Batch**                                 |ja, `type`-klassifiziert (`word`/`spacing`/`audio_event`) |bis **32 Sprecher**                                             |Scribe v2 Realtime, ~150 ms p50 (separate Variante)           |**$0,22/h** Batch + $0,070/h Entity Detection optional + $0,050/h Keyterm Prompting |sehr hoch; Realtime-Variante 93,5 % FLEURS                                  |
|**B**        |**Speechmatics Enhanced**                                      |ja, mit Confidence                                        |inkludiert als Kernfeature (S1, S2, …, UU)  ohne Add-on-Aufpreis|ja, WebSocket                                                 |**ab $0,28/h Pro-Plan**, 480 min/Monat gratis                                       |sehr hoch, On-Prem möglich (regulierte Tenants)                             |
|**C**        |**WhisperX large-v3 + wav2vec2**                               |ja, sub-100 ms via forced phoneme alignment  (CTC-basiert)|pyannote 3.1, ~90 % bei 2–3 Sprechern, 80–88 % bei 4–6          |nein (batched, ~70× RT auf RTX 4090, ~250× auf large-v3-turbo)|nur Compute-Kosten (~€0,03/h auf L4)                                                |gut, schwankt bei deutschen Dialekten; bekannter MFA-Drift bei dt. Komposita|
|**Notbackup**|Deepgram Nova-3 / AssemblyAI Universal-3 / Mistral Voxtral Mini|ja                                                        |ja                                                              |ja                                                            |$0,003–$0,21/min                                                                    |mittel                                                                      |

**Begründung Plan A:** Scribe v2 ist nach unserer Recherche der einzige produktreife Provider, der word-level Timestamps mit `type='word'|'spacing'|'audio_event'` ausliefert. Das `audio_event`-Tag liefert direkt `[breath]`, `[laugh]`, `[um]` — kritisch für Atempausen-Preset und Filler-Erkennung ohne separaten LLM-Pass. **Preis:** $0,22/h Batch  ist günstig genug, dass selbst 100 h/Tag (≈ €22) im Premium-Tier unbedenklich sind.

**Begründung Plan B:** Speechmatics bleibt für Long-Form-Master (>2 h) wegen integrierter Diarisierung ohne Add-on-Aufschlag, der unbestrittenen On-Prem-Option für regulierte Tenants und der dokumentierten 55+ Sprach-Support-Reichweite (inkl. bilingualer Packs).

**Begründung Plan C:** WhisperX self-hosted (m-bain/whisperX) ist der einzige Pfad für Tenants mit „No-Cloud-STT”-Policy. Bekannte Schwäche: forced alignment kann bei deutschen Komposita und Eigennamen abweichen  (siehe MFA-vs-WhisperX-Issue #1247 im Repo) — daher für Edit-Cuts nur, wenn Cross-Check mit `silencedetect` übereinstimmt.

#### 4.3.7 Presets

Drei eingebaute Presets in `cut_decision_list.preset`, plus `custom`:

**1. `natural_podcast`** — für Talk- und Interview-Material, das Atmungs- und Denkpausen behalten soll.

```yaml
pause_threshold_s: 1.20
pause_shorten_to_s: 0.45
filler_action: mark_only        # äh/ähm bleiben, werden nur als Marker angezeigt
disfluency_action: mark_only
bad_take_action: suggest_only
silence_dbfs: -30
buffer_pre_ms: 60
buffer_post_ms: 80
allow_hard_jumpcuts: false
breath_keep_ms: 300              # Atmer ≤ 300 ms bleiben
```

**2. `youtube_tight`** — für YouTube-Erklärformate.

```yaml
pause_threshold_s: 0.60
pause_shorten_to_s: 0.18
filler_action: remove_optional   # User entscheidet im UI per Toggle
disfluency_action: mark_for_review
bad_take_action: mark_for_review
silence_dbfs: -32
buffer_pre_ms: 40
buffer_post_ms: 60
allow_hard_jumpcuts: true
breath_keep_ms: 180
```

**3. `shorts_aggressive`** — für 9:16-Shorts und Reels, max. Energiedichte.

```yaml
pause_threshold_s: 0.35
pause_shorten_to_s: 0.08
filler_action: remove
disfluency_action: remove
bad_take_action: remove
silence_dbfs: -34
buffer_pre_ms: 20
buffer_post_ms: 40
allow_hard_jumpcuts: true
breath_keep_ms: 0
auto_caption: burn_in            # Submagic-Stil oder HyperFrames-Captions
hook_zoom: true                  # automatische Zoom-Ins auf Schlüsselwörter
```

#### 4.3.8 Cut Decision List — Format & Reversibilität

Die CDL ist als **JSON-Patch-Liste** modelliert, **nicht** als gerendertes Timeline-File. Damit kann jeder Cut individuell zurückgerollt werden (`is_active=false`), und es entsteht ein Branching-Baum (`parent_cdl`).

```jsonc
{
  "version": "1.0",
  "media_id": "…",
  "preset": "youtube_tight",
  "cuts": [
    {
      "id": "c01",
      "start": 12.480,
      "end": 13.117,
      "action": "shorten",            // delete | shorten | keep | mark_bad_take
      "target_duration": 0.180,
      "reason": "pause>0.6s, cross-checked silencedetect=-32dB",
      "source_words": [134, 135],
      "buffer_pre_ms": 40,
      "buffer_post_ms": 60
    },
    {
      "id": "c02",
      "start": 47.220,
      "end": 47.560,
      "action": "delete",
      "reason": "filler:'ähm', confidence=0.97",
      "source_words": [512]
    }
  ]
}
```

**OTIO-Frage (recherchiert).** **OpenTimelineIO** (Academy Software Foundation, aktive Releases im 0.17/0.18/0.19-Branch, Apache-2.0) wäre als Industrie-Standard-Interchange-Format prinzipiell attraktiv: laut offizieller Doku „a modern Edit Decision List (EDL) that also includes an API for reading, writing, and manipulating editorial data”  mit Adaptern für CMX EDL, FCP 7 XML  und (über `OpenTimelineIO-Plugins`) für AAF und FCPXML. Wir setzen es als **sekundären Export** ein (CDL → OTIO via Adapter), **nicht** als primäres internes Format, weil OTIO als Timeline-Modell (Stack/Sequence/Clip) zu **schwer** für eine reine Cut-Patch-Liste ist und Reversibilität/Branching nicht nativ ausdrückt. Auto-Editor unterstützt OTIO ebenfalls nicht nativ, wodurch wir keinen Open-Source-Verbündeten im Format-Krieg unter unseren Tools hätten. **Entscheidung:** Internes Format = unser JSON-CDL; OTIO-Export via eigener Adapter, EDL und Premiere-XML als Pflicht-Exporte.

#### 4.3.9 HyperFrames als Caption-/Overlay-/Motion-Graphics-Layer

**[NEU SEIT v3]** **HyperFrames** (HeyGen, Apache 2.0, aktuell **v0.6.14** released 16. Mai 2026, **20,3k GitHub-Stars und 1,7k Forks**) wird **nicht** als Stufe-2.5-Bestandteil, sondern als **Layer-Generator für Captions, Lower-Thirds, Overlays und Motion-Graphics** verwendet, der zwischen Stufe 2.5 und Stufe 3 läuft. HyperFrames rendert deterministisch HTML+CSS+GSAP/Lottie/Three.js/WAAPI via Puppeteer+FFmpeg in MP4/WebM und ist laut README „byte-identisch reproduzierbar”. Die offizielle Pitch: *„HyperFrames lets AI agents compose videos by writing HTML, CSS & JS”*  — Open-Source unter Apache 2.0,  ohne Per-Render-Fees oder Seat-Caps.

Aus der SKILL.md (verbatim): *„Create video compositions, animations, title cards, overlays, captions, voiceovers, audio-reactive visuals, and scene transitions in HyperFrames HTML. Use when asked to … add captions or subtitles synced to audio … add animated text highlighting (marker sweeps, hand-drawn circles, burst lines, scribble, sketchout), or add transitions between scenes (crossfades, wipes, reveals, shader transitions).”*

Unterstützte Animations-Runtimes über Adapter-Skills: **GSAP** (Hauptempfehlung), **Anime.js**, **CSS-Animationen**, **Lottie** (lottie-web + dotLottie), **Three.js**, **WAAPI** (Web Animations API).  Frame-Adapter-Pattern integriert beliebige Animationstechnologie, solange sie frame-genau seekbar ist.

Konkrete Nutzung in Media Brain:

- **Captions**: Stufe-2.5-CDL liefert Wort-Timestamps; HyperFrames `captions.md`-Pattern erzeugt karaoke-style highlighting deterministisch (per-word styling, text overflow prevention, caption exit guarantees).
- **Lower-Thirds**: aus Skript-Metadaten generiert (README-Beispiel: „Add a lower third at 0:03 with my name and title.”).
- **Hook-Zooms** im `shorts_aggressive`-Preset: CSS-Transform-Overlay als transparente WebM, via FFmpeg auf den Cut gelegt.
- **Animated Charts** und Daten-Inserts für YouTube-Erklärformate.

HyperFrames-CLI läuft im selben Worker-Pool wie mcp-video (Node 22 + FFmpeg + Puppeteer/Playwright). Asset-Bundle wird in GCS pro Tenant getrennt versioniert.

#### 4.3.10 Output-Formate (Stufe 2.5)

Pro Lauf werden **immer** geschrieben:

|Format                    |Verwendung                                                                                       |
|--------------------------|-------------------------------------------------------------------------------------------------|
|**MP4 (H.264 / AAC)**     |Frame.io-Preview, schnelle Review                                                                |
|**CDL JSON**              |Internes Reversibilitäts-Format, in `cut_decision_list.cuts`                                     |
|**FCPXML 1.13**           |Final Cut Pro X und Apple-Editor-Kette                                                           |
|**Premiere Sequence XML** |Premiere Pro Import (mit Marker-Layer und eingebettetem Transkript für Text-Based-Editing-Brücke)|
|**Resolve EDL (CMX 3600)**|DaVinci Resolve harte Cuts                                                                       |
|**OTIO 0.19**             |Sekundärer Industrie-Standard-Export, via OpenTimelineIO-Plugins-Pakete                          |
|**SRT/VTT**               |Captions für Web-Plattformen, generiert aus Wort-Timestamps                                      |

### 4.4 Stufe 3 — Cleanup ([BESTÄTIGT])

Unverändert: **AutoCut**, **FireCut**, **AutoPod**, **Phantom Editor**, **PremiereCopilot** als Premiere-Plugins für mechanische Politur (Zoom-In auf Sprecher, B-Roll-Vorschläge, Multicam-Sync, Caption-Burn-In, automatische Kapitelmarken). FireCut ist nach den verfügbaren Quellen das stärkste Werkzeug für deutsche Talking-Head-Pipelines und bleibt Plan A. **[Implizit GEÄNDERT GEGENÜBER v3]:** Da Stufe 2.5 bereits Pausen und Filler entfernt hat, übernimmt Stufe 3 in v3.1 stärker die B-Roll- und Zoom-Funktionen und weniger das Silence-Removal-Doppel.

-----

## 5. Thema 3 — Shortform-Pipeline ([BESTÄTIGT], um Stufe-2.5-Trigger erweitert)

- **Plan A**: **OpusClip Premium** für Premium-Tenants (Hook-Scoring, Auto-Captions, Auto-Reframe).
- **Plan B**: **Vizard.ai Standard** als kostengünstigerer Fallback.
- **Plan C**: **Submagic** als Caption- und B-Roll-Layer (kann auf bestehende Cuts aufsetzen).
- **MCP-Kandidat**: **Reap** als experimenteller MCP-Provider, im Sprint F evaluiert.
- **Floor-Pipeline**: **Gemini 3.1 Flash-Lite + FFmpeg + WhisperX**, gehostet auf Cloud Run GPU L4.

**[NEU SEIT v3] Shortform-Trigger über CDL.** Wenn ein Mediastream mit dem `shorts_aggressive`-Preset durch Stufe 2.5 läuft, wird die resultierende MP4 direkt als Input für OpusClip/Vizard verwendet. Das spart den OpusClip-internen Silence-Pass und reduziert die Anzahl der durch OpusClip generierten Clips, weil die Kompaktheit bereits höher ist (gemessener Effekt in TimeBolt-Benchmarks: 4–10× weniger Restmaterial). **Submagic** wird in v3.1 zusätzlich zum HyperFrames-Caption-Render als Plan-B-Caption-Engine verwendet, weil Submagic stärkere Animations-Templates für deutschsprachige Captions hat.

-----

## 6. Tool-Registry & Provider-Abstraktion ([BESTÄTIGT], erweitert)

YAML-Manifest (Auszug v3.1, neue Einträge fett):

```yaml
tools:
  - id: transcript.scribe_v2
    kind: stt
    plan: A
    provider: elevenlabs
    model: scribe_v2
    capabilities: [word_timestamps, diarization, audio_events, multi_lang]
    price_per_hour: 0.22
    region: us, eu
  - id: transcript.speechmatics
    kind: stt
    plan: B
    provider: speechmatics
    capabilities: [word_timestamps, diarization, on_prem]
    price_per_hour: 0.28
  - id: transcript.whisperx
    kind: stt
    plan: C
    provider: self_hosted
    runtime: cloud_run_gpu_l4
    capabilities: [word_timestamps, diarization, forced_alignment]
  # NEU SEIT v3
  - id: cut_exec.mcp_video
    kind: video_edit
    plan: A
    provider: kyanitelabs
    version: "1.4.0"
    transport: mcp_stdio
    capabilities: [trim, concat, silence_removal, hyperframes, qc_checkpoint]
    quality_gate: required
  - id: cut_exec.ffmpeg_native
    kind: video_edit
    plan: B
    provider: self_hosted
    runtime: cloud_run_worker
  - id: cut_exec.auto_editor
    kind: video_edit
    plan: C
    provider: oss
    version: "30.2.4"
    license: unlicense
  - id: layer.hyperframes
    kind: motion_graphics
    plan: A
    provider: heygen
    version: "0.6.14"
    license: apache-2.0
    runtime: node22 + ffmpeg + puppeteer
```

**Cloud API Registry** spiegelt jedes Tool als API Resource und liefert per `tools/list`-Cache (gemäß MCP-Spec 2026-07-28 RC — Stateless Core, kein Sticky-Session-Zwang) die für den Agenten aktiven Endpoints.

**Self-Healing Tool Plugin (ADK).** Wenn `cut_exec.mcp_video` zwei Quality-Gate-Failures in einer Stunde hat, fällt das Routing automatisch auf `cut_exec.ffmpeg_native`.

-----

## 7. [NEU SEIT v3] Transkript-basiertes Editing-UI

### 7.1 Produktbeschreibung

Im Media-Brain-Editor erhält jedes Mediendokument eine **Transcript-View** neben dem Skript-Editor (zweispaltiger Layout, resizable). Diese View ist **kein neuer Editor**, sondern eine spezialisierte BlockNote-Block-Variante `transcript-block`, die mit der Micro-Cut-Engine sprechen kann. Das Pattern ist nachweislich industriell etabliert: Descript ist seit 2026 mit einer **API in Open Beta** verfügbar (Release 14. Mai 2026, Descripts erste programmatische Schnittstelle, mit Underlord-Agent-Aktionen wie „remove filler words”,  MCP-Connector für Claude), Premiere Pro hat Text-Based Editing nativ („Cut, copy, and paste text in the sequence transcript, and the edits automatically reflect in the timeline”  — Adobe Help, aktuell Mai 2026 dokumentiert), und Captions.app, Gling, TimeBolt etc. setzen vergleichbare Muster.

### 7.2 UX-Patterns (orientiert an Descript, Premiere Text-Based Editing, Captions.app)

|Gesture                    |Effekt                                                                      |NLE-Pendant                 |
|---------------------------|----------------------------------------------------------------------------|----------------------------|
|Klick auf Wort             |Springe an Timeline-Position im rechten Vorschau-Panel                      |Premiere “Click word → seek”|
|Selektion + ⌘X / Entfernen |Erzeugt einen `delete`-Patch in der aktiven CDL, Live-Preview rendert sofort|Descript Composition Edit   |
|Selektion + „Shorten Pause”|Erzeugt `shorten`-Patch mit Target-Pause aus Preset                         |Eigene Innovation           |
|Rechtsklick auf Wort       |Kontext: „Als Füllwort markieren”, „Bad Take”, „Keep”                       |Descript Filler/Take-Menu   |
|⌘Z                         |Rollback des CDL-Patches (nicht Yjs-Undo des Textes!)                       |—                           |
|Speakers-Lane              |Klick auf Speaker-Bubble = filtert Transkript                               |Descript Speakers           |
|„Edit Words” Toggle        |Korrektur des Wortlauts (nur Yjs-Edit, keine CDL-Wirkung)                   |Premiere Edit Transcript    |
|„Edit Cuts” Toggle         |Selektion erzeugt CDL-Patches                                               |Descript Default            |

### 7.3 Technische Brücke BlockNote ↔ Micro-Cut-Engine

```
[BlockNote transcript-block]
   – Yjs Doc (Wort-Knoten als Inline-Annotation mit start_s/end_s)
   – Edit-Cuts-Modus emittiert CDL-Patch über Hocuspocus → API
        │
        ▼
[Micro-Cut Service]
   – validiert Patch gegen transcript_words.start_s/end_s
   – appendet zu cut_decision_list.cuts
   – triggert preview-render (Cloud Run Worker)
        │
        ▼
[Live-Preview MP4 Player]
   – HLS-Streaming aus GCS
   – Wort-Highlight synchron zur Audio-Spur (mediaState.currentTime)
```

### 7.4 Kollaborations- und Konflikt-Modell

Mehrere Editoren können gleichzeitig im Transcript-Mode arbeiten. **Edit-Words** geht durch Yjs (CRDT-Merge ist trivial, weil textbasiert). **Edit-Cuts** hingegen erzeugt **lineare** CDL-Patches — wenn zwei Editoren denselben Wortbereich schneiden, gewinnt der erste committete Patch, der zweite wird als „Konflikt” zurückgegeben und im UI als „Suggested” markiert.

-----

## 8. Phasen-Roadmap mit Tool-Test-Sprints

**[GEÄNDERT GEGENÜBER v3]: 48 Wochen statt 44 (durch neuen Sprint G)**

|Phase                                      |Wochen   |Fokus                                                     |Sprint                                                 |
|-------------------------------------------|---------|----------------------------------------------------------|-------------------------------------------------------|
|0 — Setup                                  |1–4      |GCP-Foundation, Cloud SQL, RLS, Tool-Registry-Stub        |—                                                      |
|1 — Ingest & NLE-Sync                      |5–10     |LucidLink, Frame.io V4, Tauri, Premiere UXP-Plugin        |Sprint A (NLE-Race)                                    |
|2 — Embeddings & Vector                    |11–16    |Gemini Embedding 2, Marengo 3.0, Vertex Vector Search 2.0 |Sprint B (Embedding-Race)                              |
|3 — Stufe 1 + Stufe 2 Rough-Cut            |17–24    |Eddie AI v3, IntelliScript, eigener FCPXML-Gen            |Sprint C (Rough-Cut-Race)                              |
|**3.5 — [NEU SEIT v3] Stufe 2.5 Micro-Cut**|**25–32**|**mcp-video, Scribe v2, WhisperX, eigener FFmpeg-Service**|**Sprint G (Transcript-Provider-Race + Cut-Exec-Race)**|
|4 — Stufe 3 Cleanup                        |33–37    |AutoCut, FireCut, AutoPod, Phantom Editor                 |Sprint D (Cleanup-Race)                                |
|5 — Shortform                              |38–42    |OpusClip, Vizard, Submagic, Reap MCP                      |Sprint E (Shortform-Race)                              |
|6 — Transcript-Editing-UI                  |43–46    |BlockNote transcript-block, Live-Preview, Conflict-UI     |Sprint F (UI-Polish + Captions-Race)                   |
|7 — Hardening & GA                         |47–48    |Load-Tests, RLS-Audit, SOC2-Vorbereitung                  |—                                                      |

### 8.1 [NEU SEIT v3] Sprint G — Transcript & Cut-Exec Race (Wochen 25–32)

**Ziel:** ermitteln, welcher Transcript-Provider und welche Cut-Ausführung in echten Media-Brain-Workloads die besten Resultate liefert.

#### Eval-Datensätze

- **DE-Podcast** (3 Episoden × 60 min, 2–4 Sprecher).
- **DE-Talking-Head YouTube** (5 Videos × 12 min, 1 Sprecher, Studiomic).
- **DE-Talking-Head schwierig** (5 Videos × 12 min, Headset-Mic, Hintergrundgeräusche, sächsisch/bayrisch).
- **EN-Mixed-Lang** (Code-Switching DE↔EN, 2 × 20 min).
- **9:16-Mobile-Recordings** (Shorts-Material, 20 × 60 s).

#### Eval-Kriterien Transcript-Provider

|Metrik                      |Definition                                     |Plan-A-Threshold|
|----------------------------|-----------------------------------------------|----------------|
|WER-DE                      |Word-Error-Rate Deutsch                        |≤ 6 %           |
|Wort-Timestamp-Drift        |mittlere Abweichung vs. Montreal-Forced-Aligner|≤ 80 ms         |
|Diarization Error Rate (DER)|bei 2–4 Sprechern                              |≤ 12 %          |
|Audio-Event-Recall          |Anteil korrekt erkannter [breath]/[laugh]/[um] |≥ 80 %          |
|Latency Batch               |Sekunden Audio pro Sekunde Wallclock           |≥ 30× RT        |
|Preis pro Stunde            |inkl. Diarization                              |≤ $0,40/h       |

#### Eval-Kriterien Cut-Ausführung

|Metrik                 |Definition                                               |Plan-A-Threshold|
|-----------------------|---------------------------------------------------------|----------------|
|Reverse-Timeline-Reste |Sekunden Stille/Filler übrig nach Run (TimeBolt-Methodik)|≤ 5 s pro 30 min|
|Cut-Audio-Klick-Rate   |hörbare Pops zwischen Segmenten                          |≤ 1 / 10 min    |
|Frame-Sync-Drift       |Audio↔Video-Drift nach Concat                            |≤ 1 Frame       |
|Output-Robustheit      |Anteil Jobs ohne QC-Fehler                               |≥ 98 %          |
|Reversibilität         |CDL-Round-Trip identisch zu UI-State                     |100 %           |
|Kosten pro Stunde Input|Compute + STT + Provider-Fees                            |≤ €0,80/h       |

#### Sprint-G-Tasks

- **G.1** Scribe-v2-Batch-Adapter implementieren (`timestamps_granularity=word`, `diarize=true`, `tag_audio_events=true`). Schwellwerte aus Preset injecten.
- **G.2** Speechmatics-Adapter (batch + WebSocket-Live für künftige Live-Caption-Use-Case).
- **G.3** WhisperX-Worker (Cloud Run GPU L4, `large-v3` + `wav2vec2-large-xlsr-53-german`).
- **G.4** mcp-video-Sidecar deployen, Quality-Gate-Hooks (`video_quality_check` + Whisper-Re-Transcribe) verdrahten.
- **G.5** Eigener FFmpeg-Cut-Service als Cloud Run Worker Pool, CDL → `filter_complex`.
- **G.6** Auto-Editor-Wrapper im Tauri-Companion (Offline-Fallback).
- **G.7** Eval-Harness, Goldset-Transkripte erstellen (manuelles MFA-Alignment als Ground Truth).
- **G.8** Entscheidungs-Doc Sprint G veröffentlichen, Tool-Registry final konfigurieren.

-----

## 9. Eval-Kriterien-Kataloge (Gesamtüberblick)

**[BESTÄTIGT]** Sprints A–F übernehmen ihre Eval-Kriterien-Kataloge unverändert aus v3 (NLE-Race, Embedding-Race, Rough-Cut-Race, Cleanup-Race, Shortform-Race, UI/Captions-Race). **[NEU SEIT v3]** Sprint G ergänzt Tabelle 8.1.

**Querkriterien gelten für alle Sprints:**

- OpenTelemetry-Spans existieren für 100 % der Aufrufe.
- Tool-Health-Heartbeat < 60 s alt.
- Kosten-Event pro Aufruf in `cost_events`.
- RLS-Test bestanden (cross-tenant read = 0 Treffer).

-----

## 10. Kosten- und Skalierungsmodell

### 10.1 Cost-Tiering ([GEÄNDERT GEGENÜBER v3])

|Tier        |STT (Stufe 2.5)                         |Cut-Exec                                 |LLM Stufe 2.5        |Layer (HyperFrames)    |Beispielkosten / 60 min Talking-Head|
|------------|----------------------------------------|-----------------------------------------|---------------------|-----------------------|------------------------------------|
|**Light**   |WhisperX self-hosted (Cloud Run L4)     |Eigener FFmpeg-Service                   |Gemini 3.1 Flash-Lite|optional               |~€0,55                              |
|**Standard**|**Scribe v2 Batch** ($0,22/h)           |mcp-video (Default)                      |Gemini 3.1 Flash-Lite|HyperFrames Captions ja|~€0,90                              |
|**Premium** |**Scribe v2 + Speechmatics Cross-Check**|mcp-video + FFmpeg-Service Cross-Validate|Gemini 3.1 Pro       |HyperFrames + Submagic |~€1,80                              |

**[NEU SEIT v3] Micro-Cut-Komponenten** in der Kostenmatrix:

|Posten                |Light                |Standard       |Premium              |
|----------------------|---------------------|---------------|---------------------|
|STT                   |~€0,03/h (GPU)       |$0,22/h ≈ €0,20|$0,22 + $0,28 ≈ €0,45|
|LLM-Filler/Disfluency |~€0,02/h (Flash-Lite)|~€0,02/h       |~€0,12/h (Pro)       |
|Cut-Exec Compute      |~€0,10/h             |~€0,15/h       |~€0,25/h             |
|HyperFrames Render    |—                    |~€0,15/h       |~€0,30/h             |
|Submagic              |—                    |—              |~€0,40/h             |
|**Σ pro Audio-Stunde**|**~€0,55**           |**~€0,90**     |**~€1,80**           |

Skalierung: Cloud Run Worker Pools horizontal, GPU-Pool L4 mit max. 8 concurrent jobs, RTX-PRO-6000-Pool nur für Premium-Render-Pässe und HyperFrames-4K-Output.

### 10.2 Sensitivitäten

- **Scribe v2 Realtime** ($0,39/h,  ~150 ms p50, Quelle: ElevenLabs API-Pricing-Page Stand Mai 2026) erhöht die Stundenkosten um Faktor 1,8 gegenüber Batch — lohnt sich nur für Live-Sessions; **keine** Standard-Wahl für Stufe-2.5-Batch.
- **HyperFrames** ist rein Compute (Puppeteer + FFmpeg), keine API-Fees, Apache-2.0 ohne Per-Render-Fees.  Render-Zeit dominiert (≈ 1,2× Realtime auf L4).
- **mcp-video** ist kostenlos (Apache 2.0), aber QC-Re-Transcribe per Whisper kostet Compute (~€0,03/h).
- **Auto-Editor** ist Public Domain (Unlicense, WyattBlue), reine Compute-Kosten im Tauri-Companion oder GPU-Worker.

-----

## 11. Risikoregister

|ID                    |Risiko                                                                                                                  |Wahrscheinlichkeit       |Impact|Gegenmaßnahme                                                                                                                                                                         |
|----------------------|------------------------------------------------------------------------------------------------------------------------|-------------------------|------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|R-01                  |UXP/CEP-Sunset zerstört Premiere-Plugin                                                                                 |mittel                   |hoch  |UXP-first, CEP-Brücke nur Read-only                                                                                                                                                   |
|R-02                  |Marengo 3.0-Preis-Sprung                                                                                                |mittel                   |mittel|Voyage als B, Self-Hosted-ColPali als C                                                                                                                                               |
|R-03                  |Gemini-3.1-Region-Outage                                                                                                |mittel                   |hoch  |Claude Sonnet 4.6 als sofort schaltbarer Cross-Check                                                                                                                                  |
|R-04                  |LucidLink-Ausfall                                                                                                       |niedrig                  |hoch  |rclone bisync + lokaler Tauri-Cache                                                                                                                                                   |
|R-05                  |Yjs-Server-OOM bei großen Dokumenten                                                                                    |mittel                   |mittel|Hocuspocus Sharding + Loro-Migrationsspur                                                                                                                                             |
|R-06                  |Frame.io-V4-Rate-Limits                                                                                                 |mittel                   |mittel|Eigener Comment-Cache + Pub/Sub-Batch                                                                                                                                                 |
|R-07                  |Vertex Vector Search-Cost-Spike                                                                                         |mittel                   |mittel|Hybrid-Index, geringere Replicas Light-Tier                                                                                                                                           |
|R-08                  |ADK-Breaking-Change in 2.x                                                                                              |hoch                     |mittel|Provider-Abstraktion, alle Tools MCP-konform                                                                                                                                          |
|R-09                  |Eddie AI v3 Night Shift unstable                                                                                        |mittel                   |hoch  |Resolve IntelliScript als gleichwertiger Plan B                                                                                                                                       |
|R-10                  |OpusClip-API-Throttle                                                                                                   |mittel                   |mittel|Vizard + eigene Floor-Pipeline                                                                                                                                                        |
|R-11                  |DSGVO-Konflikt mit US-STT                                                                                               |hoch (Enterprise-Tenants)|hoch  |Speechmatics On-Prem oder WhisperX Plan C                                                                                                                                             |
|**R-12 [NEU SEIT v3]**|**mcp-video noch jung (v1.4.0 Mai 2026, kleines Team, README warnt explizit vor unbeaufsichtigter Veröffentlichung)**   |hoch                     |mittel|**Strenger QC-Gate (`video_quality_check` + `video_release_checkpoint` + Whisper-Re-Transcribe Pflicht), Plan B (eigener FFmpeg-Service) gleichwertig getestet, kein Vendor-Lock-Out**|
|**R-13 [NEU SEIT v3]**|**Diarization-Drift bei deutschen Talking-Heads / Headset-Mics führt zu falschen Bad-Take-Markierungen**                |mittel                   |hoch  |**Bad-Take-Aktion in `natural_podcast` und `youtube_tight` per Default `mark_for_review`, nicht `remove`; nur `shorts_aggressive` entfernt hart**                                     |
|**R-14 [NEU SEIT v3]**|**Transcript-Edit-Drift bei Cross-CRDT-Replay (Yjs-Edit-Words räumt Wörter um, CDL referenziert alte `source_words[]`)**|mittel                   |hoch  |**CDL referenziert `start_s`/`end_s`, nicht Yjs-IDs; Yjs-Edit-Words triggert keine CDL-Invalidierung. Cross-Check beim Render-Pass verwirft inkonsistente Patches mit Editor-Notiz.** |

-----

## 12. Konsistenzcheck — keine v3-Funktion verloren

Alle in der v3-Spezifikation enumerierten Funktionen sind in v3.1 erhalten:

✅ Cloud Run + Worker Pools GA + GPU L4/Blackwell — 2.1  
✅ Cloud SQL 17 + pgvector 0.8 + RLS — 2.1 / 2.5  
✅ Vertex Vector Search 2.0 + Eventarc Advanced + Pub/Sub — 2.1  
✅ Gemini Enterprise Agent Platform + ADK 2.1.0 + Self-Healing Tool Plugin — 2.1  
✅ Gemini Embedding 2 + Marengo 3.0 + Voyage multimodal-3.5 — 2.2  
✅ Gemini 3.1 Pro / Flash-Lite + Claude Sonnet 4.6 — 2.2  
✅ Speechmatics + Scribe v2 + WhisperX + Deepgram Nova-3 — 2.2 / 4.3.6  
✅ Premiere UXP, CEP-Sunset Q3 2026 — 2.3  
✅ Resolve 20.x/21 Python-Headless + IntelliScript — 2.3  
✅ LucidLink + Frame.io V4 + rclone bisync — 2.3 / 3  
✅ Tauri 2 Companion — 2.3  
✅ Three-Stage-Pipeline (Stufe 1/2/3) — 4.1 / 4.2 / 4.4  
✅ Vector-Müll-Filter MMR + DPP + Hybrid — 4.1  
✅ Eddie AI v3 Night Shift + IntelliScript + eigener FCPXML-Gen — 4.2  
✅ AutoCut + FireCut + AutoPod + Phantom Editor + PremiereCopilot — 4.4  
✅ OpusClip + Vizard + Submagic + Reap + Floor-Pipeline — 5  
✅ BlockNote + Tiptap + Next.js — 2.4  
✅ Yjs + Hocuspocus + Liveblocks + y-indexeddb + Loro — 2.4  
✅ Fountain Markup — 2.4  
✅ Postgres RLS Multi-Tenancy — 2.5  
✅ `embedding_runs`, `agent_traces`, `tool_health` — 2.5  
✅ OpenTelemetry — 2.1 / 9  
✅ Cost-Tiering Light/Standard/Premium — 10  
✅ Tool-Registry YAML + Cloud API Registry — 6  
✅ Provider-Abstraktion ADK + MCP — 6  
✅ 6 Tool-Test-Sprints (A–F) — 8 (+ Sprint G NEU)  
✅ 44-Wochen-Roadmap → 48 Wochen (+ Sprint G, NEU)

**Ergebnis: 0 Funktionen verloren, 7 Funktionsblöcke ergänzt.**

-----

## 13. Änderungsprotokoll v3 → v3.1

|# |Sektion|Art     |Beschreibung                                                             |
|--|-------|--------|-------------------------------------------------------------------------|
|1 |0      |NEU     |Executive Summary v3.1                                                   |
|2 |1      |NEU     |Designprinzip „Audio is source-of-truth”                                 |
|3 |2.2    |GEÄNDERT|Scribe v2 wird Default-STT für Micro-Cut (Word-Timestamps + Audio-Events)|
|4 |2.3    |NEU     |Text-Based-Editing-Brücke zu Premiere                                    |
|5 |2.4    |NEU     |`transcript-block` BlockNote-Variante mit Read / Edit-Words / Edit-Cuts  |
|6 |2.5    |NEU     |3 Tabellen `transcript_words`, `cut_decision_list`, `micro_cut_runs`     |
|7 |3      |NEU     |NLE-Adapter für CDL → Format (Premiere/Resolve/FCP)                      |
|8 |**4.3**|**NEU** |**Stufe 2.5 Transcript-Aware Micro-Cut Engine, komplettes Kapitel**      |
|9 |4.3.3  |NEU     |mcp-video v1.4.0 als Plan A der Cut-Ausführung mit QC-Gate               |
|10|4.3.5  |NEU     |Auto-Editor 30.2.4 als Plan C                                            |
|11|4.3.6  |NEU     |Transcript-Provider-Race-Tabelle                                         |
|12|4.3.7  |NEU     |Drei Presets `natural_podcast`, `youtube_tight`, `shorts_aggressive`     |
|13|4.3.8  |NEU     |CDL-Format spezifiziert; OTIO-Position geklärt                           |
|14|4.3.9  |NEU     |HyperFrames v0.6.14 als Caption/Overlay/Motion-Graphics-Layer            |
|15|4.3.10 |NEU     |Output-Format-Tabelle Stufe 2.5                                          |
|16|5      |GEÄNDERT|Shortform-Trigger über CDL, Submagic als Plan B Caption-Engine           |
|17|6      |GEÄNDERT|Tool-Registry um STT- und Cut-Exec-Provider erweitert                    |
|18|7      |NEU     |Transkript-basiertes Editing-UI (komplettes Kapitel)                     |
|19|8      |GEÄNDERT|Roadmap 44 → 48 Wochen, Sprint G eingefügt                               |
|20|8.1    |NEU     |Sprint G Detail-Spec + Eval-Kriterien                                    |
|21|10     |GEÄNDERT|Cost-Tiering mit Micro-Cut-Komponenten                                   |
|22|11     |GEÄNDERT|3 neue Risiken R-12, R-13, R-14                                          |
|23|12     |NEU     |Konsistenzcheck-Tabelle                                                  |
|24|13     |NEU     |Dieses Änderungsprotokoll                                                |

-----

## 14. Schlussempfehlung

**Für die nächsten 4 Wochen:** mcp-video v1.4.0 in der Sandbox aufsetzen, Sprint-G-Eval-Datensatz zusammenstellen (vor allem deutsches Headset-Mic-Material), und Scribe v2 Batch + WhisperX als doppelt verdrahteten Test-Adapter in das ADK integrieren. Parallel das `transcript-block`-Spike im BlockNote-Editor bauen.

**Hartes Go/No-Go-Kriterium für Stufe 2.5:** Wenn Sprint G zeigt, dass das Cross-Check-Prinzip (Transcript-Gap + Audio-Silence-Detect) auf deutschem Material **WER ≤ 6 %**, **Drift ≤ 80 ms** und **Reverse-Timeline-Reste ≤ 5 s / 30 min** erreicht, geht die Engine in den Standard-Tier; sonst bleibt sie hinter einem Feature-Flag, und wir liefern v3.1 zunächst ohne aktivierte 2.5 aus.

**Hartes Go/No-Go-Kriterium für mcp-video:** Wenn nach 4 Wochen Eval mehr als 5 % der QC-Gates fehlschlagen oder das Repo bis Sprint-G-Ende keine signifikante Maintainer-Aktivität / wachsende Community zeigt, wird Plan B (eigener FFmpeg-Service) zum Default und mcp-video bleibt nur als HyperFrames-Wrapper.

**Hartes Go/No-Go-Kriterium für Transcript-Editing-UI:** Wenn der CDL-Round-Trip in 100 % der Eval-Fälle deterministisch ist und die Konflikt-Rate < 2 % bei parallelen Editoren bleibt, wird das UI GA-fähig; sonst bleibt es als Beta-Feature hinter einem Tenant-Flag, mit reinem Read+Edit-Words als Default, bis CDL-Stabilität erreicht ist.
