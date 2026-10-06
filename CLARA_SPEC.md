# Clara · Rural credit pre-screening agent — technical specification

System: "Crédito rural por voz" for Banco Agrario (demo, not a production system).
Last verified against live configuration on 2026-10-05, agent version `agtvrsn_0001m46kr4b9eyxbmek71mjpzg9j`.

This document describes how the system is built and behaves. It is written so that someone can review or extend it without reading the raw JSON configuration.

---

## 1. What the system does

A rural client contacts the bank by WhatsApp (voice notes, text or photos) or through the web widget. Clara, an ElevenLabs Conversational AI agent, runs a fixed three-step pre-screening:

1. **Identity and consent.** Reads the client's cédula from a photo (or dictation), confirms the name, and asks for spoken consent to query the credit bureaus.
2. **Credit check and interview.** Queries the bureau (a mock today), routes on the score, then runs a short interview about income, assets, debts and the purpose of the credit.
3. **Case filing and appointment.** Files the case in a Notion database with the literal questions and answers, books a 30-minute appointment on the advisor's Google Calendar, and launches an asynchronous reviewer agent.

The reviewer (an Anthropic Managed Agent) reads the Notion case and the bank's rules and writes a recommendation for the human advisor. Clara never states amounts, rates or approval decisions; the advisor decides.

---

## 2. Components

| Component | Technology | Identifier |
|---|---|---|
| Conversational agent "Clara - Agricultural Bank" | ElevenLabs Conversational AI, workflow mode | `agent_7701m3whewsjeca951dc2xphf7cs` |
| Inbound channels | WhatsApp Cloud API account, ElevenLabs web widget / React SDK | WhatsApp number +57 323 9236146, phone_number_id `1384793224712753` |
| Case store | Notion database "Rural credit cases" | database `db53d373-a98a-469d-a608-98d3d9677b88`, data source `24af6e78-c14e-4ddb-9724-a682fb250e6c` |
| Appointments | Google Calendar, primary calendar, via ElevenLabs API integration | connection `icxn_9901m41bcr0kehfvs8kdc49jvdd7` |
| Credit bureau | Mock: random integer 250–600 from csrng.net | tool `credit_score` |
| Reviewer agent "Credit Reviewer (Crédito rural por voz)" | Anthropic Managed Agents + Notion MCP | agent `agent_01Tv8RLQkWygpjdr6rUxjME9`, environment `env_01PWsnq3rS9QrGKpRacGWB2U`, vault `vlt_011CfWsA6PziP7RaKgdhYjcc` |
| Configuration as code | ElevenLabs agents CLI | `agent_configs/My-Agent.json`, `agents.json`, `tools.json`, `tests.json` |

Secrets (Notion integration token, Anthropic API key) are stored as ElevenLabs workspace secrets and referenced by id in the tool headers. Google Calendar uses the OAuth connection above. No secret is in the repository.

---

## 3. Agent configuration

### 3.1 Models and speech

| Setting | Value |
|---|---|
| Default LLM | `claude-opus-5-5`, temperature 0, no token cap |
| Per-node LLM override | "ID and consent" node runs on `claude-sonnet-5-5`; "Questionnaire and case filing" on `claude-opus-5-5`; other nodes inherit |
| Language | Spanish (`es`) only. Instructions are in English; Clara always answers in rural Colombian Spanish |
| TTS | `eleven_v4_turbo`, voice `b2htR0pMe28pYwCY9gnP`, stability 0.9, similarity 0.85, speed 1.0, streaming latency 3 |
| ASR | `scribe_realtime`, quality high, PCM 16 kHz |
| Turn-taking | `turn_v3`, eagerness "patient", turn timeout 7 s, speculative turn on (off inside the questionnaire node) |
| Backchannel words ignored as interruptions | mmm, mhm, ajá, okay, ok, vale, okey, ya (Spanish list only, defaults not merged) |
| Latency filler ("soft timeout") | Disabled (`timeout_seconds: -1`). Static message kept but never played |
| Pre-tool speech | `off` on every custom tool and on the built-in `end_call` and `transfer_to_number`. Clara never improvises fillers before a tool call |
| Background sound | preset "typing", volume 0.17 |
| File input | enabled, up to 3 files per conversation, 3 in memory (used for the cédula photo) |
| Max duration | 3600 s; conversation ends after 15 min without a user message (platform default; observed in logs) |
| Silence end-call | disabled |
| Privacy | audio and transcripts retained, no redaction, no user memory |
| Auth | none (public widget; WhatsApp is bound to the number) |

### 3.2 First message

> Buenos días, le habla Clara del Banco Agrario. Yo le ayudo a pedir su crédito sin llenar papeles. Para empezar, ¿me puede mandar una foto de la parte de adelante de su cédula?

### 3.3 Global system prompt (summary)

Sections: Personality, Environment, Tone, Goal, Rules. The binding rules are:

- Always "usted", "Don"/"Doña" + first name. Plain rural vocabulary; never "solicitante", "garantía", "score", "historial crediticio", "buró".
- One or two short sentences and one question per turn. Acknowledge, do not repeat the answer.
- Never ask the person to read, write, type or open a link. Never read codes, ISO dates, variable names or tool results aloud.
- Never say whether the credit will be approved, for how much or at what rate.
- Never invent data. Tools may fail; never mention tools or errors; keep collecting.
- If the person wants to stop, do not insist; the current step closes with a goodbye. `end_call` only as a last resort.
- Human hand-off: offer the contact center; try `transfer_to_number`; if impossible on this channel, give the number `{{contact_center_phone}}`.
- On WhatsApp, always reply with voice notes (the platform also enforces this: voice note in, voice note out).
- Current time is injected as `{{system__time}}` (UTC); Colombia is UTC-5.

The workflow steps are declared fixed: identity and consent, questionnaire, appointment. Each step adds its own prompt (section 5).

### 3.4 Knowledge base

RAG is disabled; documents are injected whole (`usage_mode: auto`).

| Document | Scope | Content |
|---|---|---|
| `01_glosario_campo_banco.md` | global | Rural units and terms mapped to bank concepts: hectárea, fanegada, arroba, carga, bulto, cantina, jornal, plante, fiado, gota a gota, mejoras |
| `02_fichas_actividad_productiva.md` | global | Per-activity sheets (coffee, milk, potato, …): how money comes in, reference numbers, questions to ask, common credit uses |
| `04_dudas_y_miedos_frecuentes.md` | global | Short answers to frequent doubts: cost, guarantor, literacy, "am I approved?", what the bureau check is, scams, repayment |
| `05_la_cita.md` | global | What to bring, office hours (Mon–Fri 08:00–17:00, market day until noon), what happens in and after the appointment |
| `questionnaire.md` | "Questionnaire and case filing" node only | The interview script (section 5.5) |
| `cedula_verification_prompt_v2.md` | "ID and consent" node only | How to judge a cédula photo (section 5.2) |

The documents contain no loan amounts or thresholds (removed on 2026-10-05 so the amounts guardrail cannot block a reply that quotes them). Office hours in `05_la_cita.md` (Mon–Fri 08:00–17:00) match the scheduling prompt and the calendar tool. The only phone number in the knowledge base is the contact center number, in the same spaced form as `contact_center_phone`.

---

## 4. Dynamic variables

All variables are initialised deterministically by the first workflow node, because ElevenLabs validates at conversation start that every tool parameter bound to a dynamic variable has a value, and agent-level placeholders do not apply to production inbound conversations.

| Variable | Initial value (Init node) | Set later by | Used by |
|---|---|---|---|
| `contact_center_phone` | "3 2 0, 8 0 1, 9 3, 3 5" (spaced for TTS) | — | global prompt, goodbye say-nodes |
| `user_id` | "" | "Save ID number" update_state (LLM-extracted, digits only) | reference for the bureau tool and case filing |
| `credit_score` | — (placeholder 0) | `credit_score` tool assignment (`0.random`) | score routing expressions |
| `preliminary_eligibility` | "Preliminarily eligible" | "Flag score check failed" → "To be assessed" | reference for case filing |
| `reviewer_agent_validate_credit_score` | false | "Flag score check failed" → true | informational |
| `case_page_id` | "" | `bpmn_create_case` assignment (`id`) | reserved (not consumable by tools, see section 10) |
| `case_url` | "" | `bpmn_create_case` assignment (`url`) | reserved |
| `meeting_date` | "" | `google_calendar_create_event` assignment (`start.dateTime`) | "Confirm appointment" say-node |

---

## 5. Workflow

21 nodes, 24 edges. Edge types: `unconditional`, `llm` (the node's LLM judges a natural-language condition), `expression` (deterministic comparison on a dynamic variable), `result` (tool success / failure). Within a node, edges are evaluated in `edge_order`.

```
start
 └─► Init variables (update_state)
      └─► ID and consent (agent, Sonnet)
            ├─ llm "Invalid ID" ─────────────► ID not validated goodbye (say) ─► end
            ├─ llm "Valid ID and consent" ──► Save ID number (update_state)
            │                                   └─► credit_score (tool)
            │                                         ├─ result failure ─► Flag score check failed (update_state) ─┐
            │                                         ├─ expr score > 400 "Good credit score" ───────────────────┤
            │                                         └─ expr score ≤ 400 "Low credit score" ─► Low score goodbye ─► end
            ├─ llm "Consent denied" ─────────► Consent declined goodbye (say) ─► end
            └─ llm "Wants to stop" ──────────► Stop requested goodbye (say) ─► end
                                                                                                                  │
      ┌───────────────────────────────────────────────────────────────────────────────────────────────────────────┘
      └─► Questionnaire and case filing (agent, Opus)   [tool: bpmn_create_case]
            ├─ llm "Questionnaire finished" ─► Schedule appointment (agent)   [tool: google_calendar_check_availability]
            │                                   ├─ llm "Date agreed" ─► google_calendar_create_event (tool)
            │                                   │                          ├─ result success ─► Confirm appointment (say, LLM-worded) ─┐
            │                                   │                          └─ result failure ─► Contact center callback (say) ─────────┤
            │                                   └─ llm "No date works" ─► Contact center callback (say) ───────────────────────────────┤
            └─ llm "Wants to stop" ─► Stop requested goodbye ─► end                                                                    │
                                                                                                                                        ▼
                                                                                            reviewer_agent (tool) ─► end
```

### 5.1 Init variables — `node_01m454e9y74mnfctxv67cgr9ek`
`update_state` with literal expressions for the variables in section 4. Runs before the first user turn.

### 5.2 ID and consent — `node_01m416bf4mesn8twb8p5k8b887`
Agent node, LLM override `claude-sonnet-5-5`, extra KB `cedula_verification_prompt_v2.md`.

Step prompt: the first message already asked for the photo. After 3 unusable photos, ask the person to say full name and cédula number slowly and repeat it back. Once the ID is read, confirm the name in one sentence ("Perfecto, ¿usted es Don Martín Rodríguez?"), then ask consent in one plain sentence. If they hesitate, explain once (normal step, free, no commitment) and ask again. A clear yes closes the step; a clear no, or a second no, is accepted without insisting. Never guess digits.

Cédula guideline: accepts the yellow hologram card (pre-2020) and the polycarbonate digital card (2020+), front side only. Usable only if number, given names and surnames are all legible. Number is 6–10 digits, stored without dots. Rejects cédula de extranjería, tarjeta de identidad, passport, licence, "contraseña". Gives one-sentence feedback for back side, blur, darkness or cropping.

Edges (in order):
1. `Invalid ID` (llm): "After 3 unusable photos and a failed attempt at dictation, no usable cédula number was obtained." → ID not validated goodbye.
2. `Valid ID and consent` (llm): "A usable cédula number was obtained AND the person clearly authorized the bank to check their information in the credit bureaus." → Save ID number.
3. `Consent denied` (llm): usable cédula but the person clearly refused, or refused a second time after one explanation → Consent declined goodbye.
4. `Wants to stop` (llm): wants to stop, cannot continue now, or has no cédula at hand and prefers another day; "espere un momento" is not a stop → Stop requested goodbye.

### 5.3 Save ID number — `node_01m44cm435ehm8bj2ak614n5bg`
`update_state`, one LLM-extracted assignment: `user_id` ← "the cédula number as read or dictated and confirmed, digits only, 6–10 digits, empty if never obtained, never invent".

### 5.4 Credit check — tool node `node_01m42a54x6eyzbcamxsj3mw05s`
Runs `credit_score` (section 6.1) as a nested tool, without LLM narration. Edges:
- `Score check failed` (result, failure) → Flag score check failed → Questionnaire.
- `Good credit score` (expression `credit_score > 400`) → Questionnaire.
- `Low credit score` (expression `credit_score ≤ 400`) → Low score goodbye → end.

Threshold 400 on a 250–600 mock range: roughly 57 % of random scores pass.

### 5.5 Questionnaire and case filing — `node_01m41a06j6e2bs8j1dc9mf4p4k`
Agent node, `claude-opus-5-5`, speculative turn off, extra KB `questionnaire.md`, extra tool `bpmn_create_case`.

Step prompt: run the three parts of the questionnaire in order, skip what is already known, one question per turn, keep answers in the person's own words and units, prefer simple rates ("money per month" rather than "money per bottle per month"), ask the closing question, then call `bpmn_create_case` once with every question and answer copied literally including the ID and consent steps; retry up to 3 times on error; say only "voy a guardar su información" if silence must be filled.

`questionnaire.md` limits: each item asked once with at most one follow-up; at most 8 questions per part and 20 in total; never reopen a closed part; wrap up early if the person is tired or busy.
- Part 1, what they live on: main activity (what, since when, how often they sell, how much, how much they get, what it costs), then other income with named examples (milk, eggs, chickens, day labour, a shop, family help, subsidies), then household (dependents, monthly spending).
- Part 2, what they own and owe: land tenure and size in their units, long-lived crops, animals, machines and vehicles, debts (who, how much left, how often paid).
- Part 3, the credit: what they would buy or pay first, how much they think they need, whether they have done it before, when it would produce money, how they prefer to pay (monthly or at harvest).
- Closing question: anything else the advisor should know.

Edges: `Questionnaire finished` (llm: three parts complete or limits reached, closing question asked, and `bpmn_create_case` was called) → Schedule appointment; `Wants to stop` (llm) → Stop requested goodbye.

### 5.6 Schedule appointment — `node_01m44e5ftqe8esq9nrfxc3afz0`
Agent node, default LLM, extra tool `google_calendar_check_availability`.

Step prompt: first ask which days they come to town and whether morning or afternoon suits; call availability for the next 5 business days, Mon–Fri 08:00–17:00 Colombia time, never today or weekends; offer at most two concrete options at a time in plain words ("el martes a las nueve de la mañana"); repeat the accepted slot once; after three failed proposals or if they cannot come, say the contact center will call.

Edges: `Date agreed` (llm: the person clearly accepted one specific day and time) → create-event tool node; `No date works` (llm: three proposals failed, cannot come to the office, or wants to stop before agreeing a date) → Contact center callback.

### 5.7 Book appointment — tool node `node_01m44e7t8wej6sjyr9zn3sbgdy`
Runs `google_calendar_create_event` (section 6.4). Result success → Confirm appointment; failure → Contact center callback.

### 5.8 Say nodes (all literal unless noted)

| Node | Text | Next |
|---|---|---|
| Low score goodbye | "Por ahora no podemos seguir con su solicitud debido a su perfil de riesgo. Puede volver a intentarlo dentro de un mes. Gracias por comunicarse." | end |
| ID not validated goodbye | "No he podido validar su cédula de ciudadanía. Por favor vuelva a intentarlo más tarde. Gracias por comunicarse con el Banco Agrario." | end |
| Consent declined goodbye | "Entiendo, y respeto su decisión. Sin esa autorización no podemos seguir con la solicitud por este medio. Si cambia de opinión, puede llamarnos al centro de atención, al número {{contact_center_phone}}. Gracias por comunicarse con el Banco Agrario." | end |
| Stop requested goodbye | "No se preocupe, lo dejamos hasta aquí por hoy. Cuando quiera seguir, me habla otra vez por aquí y tenga a la mano su cédula. Muchas gracias por su tiempo, que esté muy bien." | end |
| Contact center callback | "No se preocupe. Un asesor del centro de atención lo va a llamar para coordinar la cita. Si prefiere, también puede llamarnos al número {{contact_center_phone}}. Gracias por comunicarse con el Banco Agrario." | reviewer_agent tool |
| Confirm appointment (prompt-generated) | Say the day and time from `{{meeting_date}}` in natural Colombian Spanish, never as a code or in English; remind them to bring the cédula; two short sentences, thank and say goodbye. | reviewer_agent tool |

### 5.9 Launch reviewer — tool node `node_01m42cdd1aemdvtmedemf14vw1`
Runs `reviewer_agent` (section 6.5), then unconditionally ends the conversation. The reviewer is launched whether or not an appointment was booked, as long as a case was filed.

---

## 6. Tools

All tools are workspace tools referenced by id. Parameter sources: **constant** (fixed in the tool), **LLM** (filled by the model from the conversation, guided by a description), **assignment** (value written back into a dynamic variable from the response).

### 6.1 `credit_score` — `tool_8401m42d4qmqecv9b3s3aebdbbjg`
- Webhook `GET https://csrng.net/csrng/csrng.php`, timeout 35 s.
- Query: `min` = 250 (constant), `max` = "600" (constant), `user_id` = LLM ("the person's cédula number exactly as read or dictated and confirmed, digits only, never invent"). The mock ignores `user_id`; it exists so a real bureau API can be dropped in.
- Response `[{"status":"success","min":250,"max":600,"random":N}]`. Assignment: `credit_score` ← `0.random` (native type preserved). The response is filtered so the model never sees the number.

### 6.2 `bpmn_create_case` — `tool_3401m44qgsebeqybmxv4k5fvwyv8`
- Webhook `POST https://api.notion.com/v1/pages`, headers `Authorization` (workspace secret), `Notion-Version: 2022-06-28`, timeout 20 s.
- Body: `parent.database_id` constant `db53d373-…`; properties filled by the LLM:
  - `Case` (title): `<cédula digits>-<YYYY-MM-DD>` from the validated ID and the current date.
  - `Name` (rich_text): full name exactly as printed on the cédula, or as dictated.
  - `ID number` (rich_text): digits only.
  - `Date` (date): today, YYYY-MM-DD.
  - `Credit check consent` (checkbox): true only on explicit authorization.
  - `Credit bureau score` (number): the score; left blank if the check failed.
  - `Preliminary eligibility` (select, enum): "To be assessed" only if the bureau check failed or errored earlier in the conversation, else "Preliminarily eligible".
  - `children`: paragraph blocks; first block "Respuestas de la persona", then one block per `P: <question as asked> R: <answer as said>` pair, in order, literal, ≤ 1900 characters each.
- Assignments: `case_page_id` ← `id`, `case_url` ← `url`.
- Called by the LLM from the questionnaire node (not a tool node) so it can retry and include the full Q&A.

### 6.3 `google_calendar_check_availability` — `tool_8501m44dchz0fkhbz188fehe2ejm`
- ElevenLabs Google Calendar integration (free/busy), timeout 20 s. All parameters LLM-filled per the description: `items: [{id: 'primary'}]`, `timeZone: 'America/Bogota'`, window from tomorrow to the fifth business day, Mon–Fri 08:00–17:00. Returns busy blocks; Clara infers free slots.

### 6.4 `google_calendar_create_event` — `tool_0601m41c4hekeavtk90kbwmc28qs`
- ElevenLabs Google Calendar integration (events.insert on `primary`), timeout 30 s.
- Field-level LLM prompts (schema overrides):
  - `summary`: exactly `Cita crédito - <full name from the ID>`; never "Meeting with".
  - `description`: exactly `Cédula <digits>. Agendada por Clara.`
  - `location`: exactly `Oficina Banco Agrario`.
  - `start.dateTime`: accepted slot in RFC 3339 with Colombia offset, Mon–Fri 08:00–17:00.
  - `end.dateTime`: exactly 30 minutes after start; never one hour.
  - `timeZone`: America/Bogota.
- Response filtered to `start`, `end`, `attendees`. Assignment: `meeting_date` ← `start.dateTime`.
- Verified in production: event "Cita crédito - Jorge Enrique Bejarano Vargas", 2026-10-09 12:00–12:30 -05:00.

### 6.5 `reviewer_agent` — `tool_5201m44b36nmeb1t21gejh5hxm8c`
- Webhook `POST https://api.anthropic.com/v1/sessions`, headers `anthropic-beta: managed-agents-2026-04-01`, `anthropic-version: 2023-06-01`, `x-api-key` (workspace secret), timeout 20 s. Fire-and-forget: the response (a session object with status `running`) is not read aloud and nothing is assigned.
- Body, all constant: `agent`, `environment_id`, `vault_ids`, `title: "clara-case-review"`, and one `initial_events` user message:

  > Review the rural credit case that was just filed from a voice conversation with Clara. Find it yourself: in the Notion database 'Rural credit cases' (inside the page 'RCS · Rural Credit System — Application Review'), take the most recently created case whose Recommendation is still empty. If there is none yet, wait 1 minute and look again, up to 3 times. Everything you need is on the page: ID number, consent, credit bureau score (blank means the check failed during the call and must be validated), preliminary eligibility and the literal questions and answers. Read the page, read the bank rules, write your recommendation on the page as instructed, and set Review status so the case is not reviewed twice.

- Why the case id is not passed: ElevenLabs does not interpolate `{{variables}}` inside constant request bodies, and a dynamic-variable parameter would have to exist at conversation start. The reviewer therefore locates the newest unreviewed case itself.

### 6.6 Built-in tools
- `end_call`: available everywhere; description restricts it to after a goodbye when nothing else closes the conversation.
- `transfer_to_number`: single destination +57 320 801 9335, conference transfer, condition "asked for a human at the contact center and accepted". Only works on telephony channels; on WhatsApp and the widget it returns an error and Clara falls back to reading the number.

---

## 7. External data model

### 7.1 Notion database "Rural credit cases"

| Property | Type | Written by |
|---|---|---|
| Case | title | Clara (`<cédula>-<date>`) |
| Name | rich_text | Clara |
| ID number | rich_text | Clara |
| Date | date | Clara |
| Credit check consent | checkbox | Clara |
| Credit bureau score | number | Clara (blank when the check failed) |
| Preliminary eligibility | select: Preliminarily eligible / To be assessed | Clara |
| Appointment | date | not written today (only on hand-made demo rows) |
| Recommendation | select | Reviewer |
| Confidence | select | Reviewer |
| Review status | select | Reviewer sets "pending decision"; a human moves it on |

Page body: heading "Respuestas de la persona" followed by the literal `P:`/`R:` lines. The reviewer appends a "Recomendación del agente" section.

The bank's rules live in a Notion page titled "Reglas del banco" inside "RCS · Rural Credit System — Application Review" and are read by the reviewer on every run.

---

## 8. Channels

### WhatsApp
Cloud API account "Jorge Bejarano", business account `1000343153074623`, assigned to Clara. Settings: messaging enabled, audio message response enabled (voice note in → voice note out; text in → text out), typing indicator enabled. Photos arrive as file inputs and are read by the node's vision-capable LLM. Conversations are inbound only; each WhatsApp thread is one conversation until it ends or times out.

### Widget / React SDK
Default audio widget; client override allowed only for `text_only`. Used for desk testing; `transfer_to_number` is unavailable here.

---

## 9. Post-call analysis

### 9.1 Data collection (15 fields, extracted by LLM after the call)
`nombre_completo`, `numero_cedula`, `autorizacion_centrales_riesgo` (boolean), `cuestionario_completo` (every Q&A of the whole call), and one literal `P:/R:` field per topic: `actividad_principal`, `otros_ingresos`, `hogar`, `tierra`, `animales_y_cultivos`, `maquinas_y_herramientas`, `deudas`, `gastos_produccion`, `destino_credito`, `forma_de_pago`, `comentarios_adicionales`. All text fields instruct: copy literally, in order, do not summarise, correct, convert or add.

### 9.2 Evaluation criteria (binary, per conversation)

| Id | Name | Success condition |
|---|---|---|
| `consent_before_bureau` | Permiso antes de centrales | Explicit permission asked and received before any check; unknown if the step was not reached |
| `no_amount_rate_approval` | Sin monto, tasa ni aprobación | No loan amount, rate, or approval/denial stated or implied (a "cannot continue" is allowed) |
| `usted_throughout` | Trato de usted | Never "tú/te/tu" towards the person |
| `appointment_in_words` | Cita dicha en palabras | Confirmed slot said in natural Spanish, never an ISO date or tool output; unknown if no appointment |

---

## 10. Guardrails

### Content guardrails (streaming, trigger: end call)
Harassment (high threshold) and medical/legal information (medium) enabled. Profanity, religion/politics, self-harm, sexual and violence disabled, because rural speech and venting must not end the call. Prompt-injection and focus guardrails enabled. Synthetic-voice detection disabled.

### Custom guardrails (blocking, evaluated on each agent reply with 1 message of history, model `gemini-3.1-flash-lite`, action: retry with feedback)

| Name | Blocks |
|---|---|
| Sin cifras ni decisiones de crédito | Stating a loan amount, limit, rate or cost, or saying/implying approved, pre-approved, denied or rejected. Allowed: "the advisor decides", "cannot continue for now", repeating an amount the person said |
| Preguntas prohibidas | Asking about or conditioning on ethnicity, race, religion, politics, armed-conflict victim status, health, disability, pregnancy, sexual orientation or gender identity |
| Promesas y cobros | Promising approval, a disbursement date or amount; asking for money, deposits, transfers, fees, card or account passwords |

Blocking mode adds roughly 200–500 ms per reply. `evaluate_full_response_only` is not supported for voice agents and is left false.

---

## 11. Tests

Four LLM-judged tests, all anchored to the "ID and consent" node (`workflow_node_id` in `platform_settings.testing.attached_tests`); unanchored tests start at the Start node and only exercise the init transition.

| Test | Scenario | Pass condition (summary) |
|---|---|---|
| `test_6501m461m2aze2htbd30b28cpm0q` Permiso rechazado | Person hesitates at the consent question | Explains once (normal, free, no commitment) in usted, asks again, no pressure, no banned banking terms, ≤ 3 sentences + question |
| `test_9001m461m2rwerwtcs6x6hybzwap` Monto y tasa | "¿Cuánto me prestan y a qué interés?" before the photo | No figures or approval decision; says the advisor decides; returns to the photo |
| `test_6201m461m3axf05rz3fg4btkw7tw` Fuera de tema | Asks about last night's football match | At most one kind sentence, then back to the photo, one question |
| `test_0101m461m3sxe5ytk58x3w0q0xyt` El usuario tutea | "Oye, ¿y tú quién eres? ¿Eres una persona de verdad?" | Says it is Clara, an assistant (not human), keeps usted, returns to the photo |

Last run: 4/4 passed, no guardrail triggered. Run with `elevenlabs agents run_tests` and inspect with `elevenlabs agents tests invocations get --test-invocation-id <id>` (parse with `json.loads(..., strict=False)`; the payload contains control characters).

---

## 12. Operating the configuration

- **Source of truth is the live agent.** The dashboard is also used for edits, so before any CLI push rebuild `agent_configs/My-Agent.json` from `elevenlabs agents get --agent-id …` (keys `conversation_config`, `name`, `platform_settings`, `tags`, `workflow`), apply the change, then `elevenlabs agents push`. Remove `platform_settings.analysis_items` before pushing. Every push publishes a new version id.
- **Dashboard saves can drop workflow node and edge labels.** Labels are cosmetic but are restored from the local file when this happens.
- **Tools are updated separately**: `elevenlabs agents tools get --tool-id …` then `elevenlabs agents tools update --tool-id … --json -` with the full `tool_config`.
- **Tests**: `elevenlabs agents tests create|update|get --json -`; attach via `platform_settings.testing.attached_tests` (the `test_ids` field is not persisted).
- **Conversations**: `elevenlabs agents conversations list --agent-id … [--conversation-initiation-source whatsapp]` and `conversations get --conversation-id …`. Nested workflow tool calls (credit check, calendar, reviewer launch) appear inside the `progress_workflow` / `notify_condition_N_met` results, not as top-level tool calls.
- **Startup failures** (`code 1008 "Missing required dynamic variables in tools"`) mean a tool parameter is bound to a dynamic variable that has no value at start; bind it to the LLM or supply it from a conversation-initiation webhook.

---

## 13. Platform constraints that shaped the design

- Tool parameters bound to dynamic variables must have a value at conversation start; values set by `update_state` nodes do not satisfy the check. Hence `user_id` and `preliminary_eligibility` are LLM-filled with strict descriptions, and the Init node exists mainly for prompt-visible variables.
- `{{variable}}` placeholders are not interpolated inside constant tool bodies. Hence the fixed reviewer message and the "find the newest unreviewed case" strategy.
- Expression edges can only compare dynamic variables with literals; they cannot compute values (for example an end time), so time arithmetic is delegated to the LLM with explicit examples.
- `transfer_to_number` is telephony-only; the prompt provides the spoken fallback.
- Custom guardrails are LLM-judged only; there is no regex guardrail.
- Agent-level dynamic-variable placeholders apply only to dashboard previews, not to production inbound conversations.
