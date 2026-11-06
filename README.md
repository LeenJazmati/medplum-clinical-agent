# Oncology Medical Assistant (n8n workflow)

An n8n AI-agent workflow that powers the physician-facing chat assistant of a hospital system built on [Medplum](https://www.medplum.com/) (an open-source FHIR R4 EHR). Staff talk to the assistant in natural language (Arabic or English, text or voice) and the agent reads and writes FHIR resources through the Medplum REST API on their behalf.

This repository contains only the n8n workflow. Medplum itself, the clinical frontend, and the other parts of the surrounding system are not included.

## What the assistant can do

| Area | Capabilities |
|---|---|
| Patients | Search (with retry on alternate name spellings), create, update, count, get by ID |
| Clinical data | Create and search conditions, allergies, vital signs, medication statements, family member history |
| Coded data | Resolves real codes before writing: ICD-10-CM (conditions), UNII (allergens), RxTerms (medications), family relationship codes, allergy reaction codes. It never invents a code |
| Orders | Lab and imaging orders (ServiceRequest plus fulfillment Task created together), pinned AI questionnaires, LOINC-based catalog search with explicit user selection before ordering |
| Scheduling | Practitioner conflict check, visit booking, care plan lookup, PlanDefinition apply, general practitioner assignment |
| Voice | Voice input transcribed with Gemini; optional spoken replies with Gemini TTS |

## Architecture

```
Webhook Trigger (POST)
   └─ Route Source
        ├─ text input ──────────────────────────────┐
        ├─ audio input → Decode audio → Transcribe ─┤
        │                 (Gemini) → Tag voice      ▼
        │                                   Normalize Input
        │                                          ▼
        │                                      AI Agent ◄── Gemini Chat Model
        │                                       │  ▲    ◄── Chat Memory (10 messages, per chatId)
        │                                       │  └── 34 Medplum / terminology tools (HTTP)
        │                                       ▼
        │                               Respond to Webhook  { "reply": "..." }
        └─ ttsOnly → Gemini TTS → PCM to WAV → Respond to Webhook
```

## Webhook contract

`POST /webhook/oncology-assistant` with JSON body:

| Field | Required | Description |
|---|---|---|
| `input` | one of `input` / `audio` / `ttsOnly` | User text message |
| `audio`, `audioMimeType` | | Base64 voice message (and MIME type, default `audio/webm`) |
| `ttsOnly`, `text` | | Convert `text` to speech only, with no agent call |
| `chatId` | recommended | Session key for conversation memory |
| `medplumToken` | yes for agent calls | Medplum access token of the logged-in user, forwarded as `Authorization: Bearer ...` on every FHIR call |
| `requester` | optional | Current practitioner reference, e.g. `Practitioner/<id>` |
| `patientId`, `topicId`, `model`, `messages`, `type`, `replyWithVoice` | optional | Extra context passed through `Normalize Input` |

Response: `{ "reply": "<assistant text>" }` (or WAV audio for `ttsOnly`).

Because the caller's own Medplum token is used for every request, the assistant only has the permissions of the logged-in user (Medplum AccessPolicy is enforced server-side).

## Requirements

- n8n (self-hosted)
- A running Medplum server. The workflow expects it at `http://medplum-server:8103` (Docker service name on the same network as n8n). Change the host in the HTTP tool nodes if yours differs.
- Google Gemini API credential (chat model, transcription, TTS)

## Setup

1. In n8n, go to **Workflows → Import from file** and choose `workflow.json`.
2. Attach your **Google Gemini (PaLM) API** credential to: `Gemini Chat Model`, `Transcribe Voice`, `Generate Speech (TTS Only)`.
3. If Medplum is not reachable as `medplum-server:8103` from n8n, update the base URL in the Medplum HTTP tool nodes.
4. Create the Medplum resources the workflow refers to (PlanDefinitions, Questionnaires, code systems under `http://medplum.com/graduationproject/...`), or change those canonical URLs to your own.
5. Activate the workflow and call the webhook from your frontend or a Medplum Bot.

## Security notes

- The webhook is set to `allowedOrigins: *`. Restrict it to your frontend origin in production.
- The webhook path is a fixed placeholder. Use a hard-to-guess path or add authentication before exposing it.
- No API keys are stored in this file; credentials are referenced by n8n credential ID only.
- Patient records cannot be deleted through the assistant by design.

## Author

Leen Jazmati
