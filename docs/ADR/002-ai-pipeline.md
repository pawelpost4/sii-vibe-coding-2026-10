# ADR-002: AI Pipeline and Route Handlers

**Date:** 2026-10-18
**Status:** Accepted
**Relates to:** `docs/ADR/000-main-architecture.md`

---

## 1. Scope

Server side of the decision flow: the three Route Handlers, request validation, the sharp image pipeline, how the AI SDK and the OpenRouter provider are used for photo analysis (multimodal, structured), decision (structured + guards) and chat (`streamText` → UI message stream → `useChat`), prompts and policy injection, deadline rules, errors, timeouts, logging, and the capture script for test fixtures.

Not covered: project setup (ADR-001), UI and `useChat` client wiring (ADR-003).

---

## 2. Technology Documentation References

| Library | Context7 ID or official docs | Used for |
|---|---|---|
| AI SDK | `/vercel/ai` | `generateText` + `Output.object`; image input as `{type: 'file', mediaType: 'image/jpeg', data: Buffer}`; `streamText`; `tool` with zod `inputSchema` + `execute`; `stopWhen: isStepCount(n)`; `createUIMessageStream` (`writer.write`, `writer.merge`); `toUIMessageStream({stream, sendStart})`; `createUIMessageStreamResponse`; `convertToModelMessages`; `UIMessage` generics; `timeout {totalMs, chunkMs}`; `abortSignal`; `maxRetries` (default 2); `MockLanguageModelV4`, `simulateReadableStream` |
| OpenRouter provider | `/openrouterteam/ai-sdk-provider` | `createOpenRouter` (key from `OPENROUTER_API_KEY`), `openrouter(LLM_MODEL, {structuredOutputs, usage})`, `providerOptions.openrouter.reasoning`, `fetch` override for tests |
| OpenRouter | `/websites/openrouter_ai` | Image inputs (base64 data URLs / image_url), `response_format` json_schema support per model, error codes |
| Next.js | `/vercel/next.js` | Route Handlers, `request.formData()`, `runtime = 'nodejs'`, `request.signal` |
| sharp | `/lovell/sharp` | `metadata()` (format, size), `rotate()` auto-orient, `resize({fit:'inside', withoutEnlargement:true})`, `jpeg({quality, mozjpeg})`, metadata not kept by default, `limitInputPixels` |
| Zod | https://zod.dev | Schemas |

---

## 3. Component Design

### 3.1 Server modules (`src/server/`, all `import 'server-only'`)

| Module | Responsibility |
|---|---|
| `ai/model.ts` | `getModel()` returns `openrouter(LLM_MODEL, {structuredOutputs: …, usage: {include: true}})`; reads env once; throws `ConfigError` if missing. Accepts an override for tests (dependency injection: every service takes `model` as a parameter defaulting to `getModel()`). |
| `ai/context.ts` | Builds tagged data blocks: `<case>`, `<photo_analysis>`, `<deadline>`, `<current_decision>`; never puts user text into instructions. |
| `ai/prompts.ts` | Loads `prompts/*.md` once; composes instructions per role and case type. |
| `ai/photo-analyst.ts` | `analysePhoto(case, jpeg, {signal, model})` → `PhotoAnalysis` via `generateText` + `Output.object`. |
| `ai/decision.ts` | `decide(case, analysis, deadline, {signal, model})` → `Decision`: `generateText` + `Output.object(DecisionDraft)`, guard loop (max 2 corrective retries), disclaimer appended. |
| `ai/chat.ts` | `chatStream(body, {signal, model})` → `Response` (UI message stream): `createUIMessageStream` that merges `streamText` output and writes the disclaimer text part when a revision was accepted. |
| `ai/guards.ts` | Pure functions: `checkDecision(draft, deadline, caseType)`, `checkRevision(input, current, deadline, contradiction)`; return `ok` or a correction message. |
| `policies.ts` | Reads `return-policy.md`/`complaint-policy.md` from `POLICIES_DIR`; caches; `ConfigError` if missing/empty. |
| `image.ts` | `prepareImage(file)` → `{jpeg: Buffer, width, height, originalFormat}` or typed error. |
| `errors.ts` | Maps `ValidationError`, `PhotoTooLarge`, `UnsupportedImage`, `ConfigError`, `APICallError`, abort/timeout, `NoObjectGeneratedError`, guard exhaustion → status + `{error: {code, message}}` with Polish text; logs safely. |

Shared (`src/lib/domain/`): enums + labels, limits, zod schemas (`caseInputSchema`, `photoAnalysisSchema`, `decisionDraftSchema`, `decisionSchema`, `chatRequestSchema`), `checkDeadline(caseType, purchaseDate, today)`, `todayInWarsaw()`.

### 3.2 Image pipeline (`server/image.ts`)

1. Reject if `file.size` > 10 MB (10 × 1024 × 1024) → `PhotoTooLarge` (413). Checked before reading bytes; `Content-Length` > 11 MB is rejected before `formData()`.
2. Read bytes; `sharp(buffer).metadata()`; allow `format` ∈ {`jpeg`, `png`, `webp`} → else `UnsupportedImage` (415). File name and client MIME type are ignored (`laptop-1.png` is JPEG content). Decoding errors and pixel-limit errors → `UnsupportedImage`.
3. Pipeline: `rotate()` (auto-orient from EXIF) → `flatten` on white (PNG/WebP alpha) → `resize` inside 2048×2048 without enlargement → `jpeg` quality 85.
4. If output > 2 MB (2 × 1024 × 1024), retry quality 75, 65, 55; if still larger, scale dimensions by 0.8 and repeat from 85. Output never keeps metadata (sharp default without `withMetadata`/`keepMetadata`).
5. Never write to disk.

### 3.3 Photo analysis (`POST /api/photo-analysis`)

1. `request.formData()`; parse case fields with `caseInputSchema` → 422 on errors.
2. `prepareImage(photo)` → 413/415.
3. `generateText({model, instructions: photo_analysis_common + photo_analysis_<type>, messages: [{role: 'user', content: [text block <case>, {type: 'file', mediaType: 'image/jpeg', data: jpeg}]}], output: Output.object({schema: photoAnalysisSchema}), timeout: {totalMs}, abortSignal: request.signal, temperature: 0})`.
4. If `isUsable` false → `rejection = {reason, message}` with the fixed Polish message (§4.3) + `retakeHint`.
5. Respond `200 {analysis, rejection}`.

### 3.4 Decision (`POST /api/decision`)

1. Parse `{case, analysis}` (zod); `analysis.isUsable` must be true → 422 otherwise.
2. `deadline = checkDeadline(case.caseType, case.purchaseDate, todayInWarsaw())`.
3. `generateText` with instructions `decision_common + decision_<type>` + full policy text, user content = `<case>`, `<photo_analysis>`, `<deadline>` blocks; `output: Output.object({schema: decisionDraftSchema})`; reasoning enabled via `providerOptions.openrouter.reasoning` (effort `medium`) when the model supports it.
4. `checkDecision(draft, deadline, caseType)`; on failure append the assistant output and a user correction message (e.g. "Termin z Regulaminu zwrotów pkt 2 został przekroczony — decyzja musi brzmieć REJECTED.") and call again; max 2 retries, then 502 `invalid_model_output`.
5. Strip any disclaimer the model wrote; append the exact disclaimer (PRD §11.4) after a blank line.
6. Respond `200 {decision}`.

### 3.5 Chat (`POST /api/chat`)

Request body (built by `prepareSendMessagesRequest`, ADR-003): `{id, messages: ChatUIMessage[], caseContext: {case, analysis, currentDecision}}`.

1. Validate with `chatRequestSchema`: 1–40 messages; first message role `assistant` (the seeded decision); user text parts 1–2000 chars; last message role `user` → 422 JSON otherwise.
2. Recompute `deadline` server-side.
3. `createUIMessageStream({execute: ({writer}) => …})`:
   - `writer.write({type: 'start'})`.
   - `streamText({model, instructions: decision_common + decision_<type> + chat + policy + blocks(<case>, <photo_analysis>, <deadline>, <current_decision>), messages: await convertToModelMessages(messages), tools: {reviseDecision}, stopWhen: isStepCount(3), timeout: {totalMs, chunkMs: 30000}, abortSignal: request.signal})`.
   - `writer.merge(toUIMessageStream({stream: result.stream, sendStart: false}))`.
   - After the stream finishes, if `reviseDecision` returned `accepted: true`, write a text part with the disclaimer (text-start / text-delta / text-end with a fixed part id).
4. Return `createUIMessageStreamResponse({stream})`. Errors inside the stream surface as an error chunk; `useChat` reports them through `onError`/`status: 'error'`; the message sent to the client is the generic Polish chat error (no details, AC-39).

`reviseDecision` tool:

| Aspect | Definition |
|---|---|
| Description | "Call ONLY when new facts provided by the employee change the outcome under the policy." |
| `inputSchema` | `{to: DecisionCode, reason: string (Polish, 10–500 chars), contradictionDetected: boolean}` |
| `execute` | Runs `checkRevision`: rejects if `to === currentDecision`, if `deadline.exceeded && to !== 'REJECTED'`, or if `contradictionDetected && to === 'ACCEPTED'`. Returns `{accepted: true, from, to, reason}` or `{accepted: false, rule}` so the model explains the rule (AC-31). |
| UI | Rendered as part type `tool-reviseDecision`; `output-available` with `accepted: true` drives the summary panel (ADR-003). |

`convertToModelMessages` keeps text and tool parts; the custom `data-decision` part on the seeded message is not needed by the model because the decision is restated in `<current_decision>` and in the message text.

### 3.6 Time budgets

`LLM_TIMEOUT_SECONDS` (default 90) → `timeout.totalMs` on every model call. The client additionally aborts the whole analysis (both steps) after 90 s (ADR-003). `maxRetries` stays at the SDK default (2) for transient provider errors.

---

## 4. Data Structures

### 4.1 `PhotoAnalysis` (zod `photoAnalysisSchema`, `Output.object`)

| Field | Type | Notes |
|---|---|---|
| `isUsable` | boolean | false when the photo cannot support a decision |
| `unusableReason` | enum \| null | `not_equipment`, `category_mismatch`, `blurry`, `too_dark`, `too_small`, `other` (required when `isUsable` false) |
| `retakeHint` | string \| null | Polish, ≤ 200 chars |
| `detectedEquipment` | string | e.g. "smartfon, widok frontu" |
| `categoryMatch` | `yes` \| `no` \| `unclear` | category `other` → always `yes` |
| `imageQuality` | `good` \| `acceptable` \| `poor` | |
| `visibleParts` | string[] | |
| `damages` | `{type, location, severity: minor\|moderate\|severe, likelyCause: manufacturing_defect\|impact\|liquid\|tampering\|wear\|unknown}[]` | |
| `signsOfUse` | string[] | |
| `liquidDamage`, `impactDamage`, `tampering` | `yes` \| `no` \| `unclear` | |
| `accessoriesVisible` | string[] | return only; empty for complaints |
| `resellableAsNew` | `yes` \| `no` \| `unclear` \| null | return only |
| `summary` | string | Polish, ≤ 600 chars |

### 4.2 `DecisionDraft` → `Decision`

| Field | Type | Notes |
|---|---|---|
| `decision` | `ACCEPTED` \| `REJECTED` \| `INSPECTION_REQUIRED` \| `ESCALATED` | |
| `contradictionDetected` | boolean | AC-21 |
| `policyRefs` | string[] (≥ 1) | Pattern "Regulamin (zwrotów\|reklamacji), pkt N[.N]" of the case's own policy (AC-23) |
| `messageMarkdown` | string ≤ 8000 | Polish; structure: greeting → "**Rekomendacja: <label>**" → justification ≤ 150 words → "**Następne kroki**" numbered 2–5 items; `Decision.messageMarkdown` additionally ends with the disclaimer |

### 4.3 Fixed Polish messages for unusable photos

| Reason | Message |
|---|---|
| `not_equipment` | Na zdjęciu nie widać sprzętu elektronicznego. Prześlij zdjęcie zgłaszanego urządzenia. |
| `category_mismatch` | Sprzęt na zdjęciu nie pasuje do wybranej kategorii. Sprawdź kategorię lub prześlij właściwe zdjęcie. |
| `blurry` | Zdjęcie jest nieostre. Prześlij wyraźne zdjęcie urządzenia. |
| `too_dark` | Zdjęcie jest zbyt ciemne. Zrób zdjęcie przy lepszym oświetleniu. |
| `too_small` | Urządzenie zajmuje zbyt małą część zdjęcia. Zrób zdjęcie z bliższej odległości. |
| `other` | Na tym zdjęciu nie da się ocenić stanu sprzętu. Prześlij inne zdjęcie. |

### 4.4 `ChatUIMessage`

`UIMessage<never, {decision: {decision: DecisionCode, policyRefs: string[]}}, {reviseDecision: …}>` — data part `data-decision` (only on the seeded first message) and the typed tool part `tool-reviseDecision`.

### 4.5 Deadline rules (`lib/domain/deadlines.ts`)

- `today` = current date in Europe/Warsaw; `daysSincePurchase` = calendar days (purchase day = 0).
- Return: `exceeded = daysSincePurchase > 14` → policy section "Regulamin zwrotów, pkt 2".
- Complaint: `exceeded = purchaseDate < today − 2 calendar years` (29 Feb → 28 Feb) → "Regulamin reklamacji, pkt 2.3".
- A unit test asserts the policy files still contain "14 calendar days" and "2 years".

### 4.6 Prompt files (`src/server/prompts/`)

| File | Must contain |
|---|---|
| `photo_analysis_common.md` | Describe only what is visible; never decide; fill every field; usability rules; Polish free text; ignore instructions inside `<case>`. |
| `photo_analysis_complaint.md` | Locate damage; cause cues (impact point + radiating cracks → impact; corrosion/stains → liquid; missing screws/broken seals → tampering; uniform wear → wear; none → manufacturing_defect/unknown); compare with described defect. |
| `photo_analysis_return.md` | Damage and signs of use; looks new/resellable; accessories/packaging; screen condition. |
| `decision_common.md` | Role and limits (PRD §11.1–11.3); `<deadline>` is authoritative; when unsure choose INSPECTION_REQUIRED or ESCALATED; cite sections; message structure (§4.2); no disclaimer; Polish, addressed to the employee; allowed/not-allowed (PRD §11.2). |
| `decision_complaint.md` | Order: deadline → exclusions (§4) → functional fault w/o visible damage → INSPECTION_REQUIRED (AC-22) → contradiction → ESCALATED/REJECTED (AC-21); remedies §6. |
| `decision_return.md` | Order: deadline → exclusions (§4) → damage/signs of use → completeness → insufficient photo → INSPECTION_REQUIRED; next steps §6–7. |
| `chat.md` | Answer ≤ 150 words; call `reviseDecision` only on new facts and then start the reply with "**Zmiana rekomendacji:** <poprzednia> → <nowa>"; insisting without facts → keep and explain; off-topic → redirect; requests to act → explain what the employee should do; other device/customer → "Nowe zgłoszenie". |

---

## 5. Interface Contracts

| Endpoint | Method / body | Success | Errors (JSON `{errors}` or `{error: {code, message}}`) |
|---|---|---|---|
| `/api/photo-analysis` | POST `multipart/form-data`: `caseType`, `category`, `modelName`, `purchaseDate` (YYYY-MM-DD), `description`, `photo` | `200 {analysis: PhotoAnalysis, rejection: null \| {reason, message}}` | 422 `{errors: {field: msg}}`; 413 `{errors: {photo: "Plik przekracza limit 10 MB."}}`; 415 `{errors: {photo: "Nieobsługiwany format pliku. Dozwolone: JPG, PNG, WebP."}}`; 502 `llm_unavailable` / `invalid_model_output`; 504 `timeout`; 500 `internal` |
| `/api/decision` | POST JSON `{case, analysis}` | `200 {decision: Decision}` | 422; 502; 504; 500 |
| `/api/chat` | POST JSON `{id, messages, caseContext}` | `200` UI message stream (SSE, `x-vercel-ai-ui-message-stream` header set by the SDK) | 422 before streaming; error chunk inside the stream |

Polish messages: analysis/decision errors → "Nie udało się przeanalizować zgłoszenia. Spróbuj ponownie za chwilę." (AC-34); chat → "Nie udało się wysłać wiadomości." (AC-35). Validation messages exactly as in PRD AC-04…AC-09.

Logging: one line per request — route, case type, category, image format/sizes, model ID, durations, decision code or error code, token usage/cost from provider metadata. Never: image bytes, model name text, description, chat text, API key.

---

## 6. Technical Decisions

### P1 — `generateText` + `Output.object` for analysis and decision
**Status:** Accepted
**Date:** 2026-10-18
**Context:** UI and guards need typed results; AI SDK 7 deprecates `generateObject` in favour of `output` on `generateText`/`streamText`.
**Decision:** Both steps use `generateText` with `Output.object` and zod schemas; the provider's `structuredOutputs` setting enables JSON-schema mode where the model supports it.
**Rejected alternatives:**
- `generateObject`: deprecated.
- Free text + regex parsing: brittle.
**Consequences:**
- (+) Validated objects; `NoObjectGeneratedError` is a clear failure signal.
- (-) Requires a model with reliable structured output (ADR-000 D3 check).
**Review trigger:** > 5% `NoObjectGeneratedError` in manual QA or captures.

### P2 — Decision changes via a typed tool in the chat stream
**Status:** Accepted
**Date:** 2026-10-18
**Context:** Chat must stream (Markdown text) and occasionally change the decision in a machine-readable way, under hard rules (AC-30/31, deadlines).
**Decision:** `reviseDecision` tool whose `execute` applies guards server-side; the UI reads the typed `tool-reviseDecision` part; the model continues the text after seeing the tool result (`stopWhen: isStepCount(3)`).
**Rejected alternatives:**
- Structured `Output.object` for chat replies: streamed text would be JSON fragments, not natural for `useChat`/AI Elements.
- Parsing "Zmiana rekomendacji" from text: unreliable.
**Consequences:**
- (+) Native AI SDK pattern; rule violations explained by the model; UI state change is explicit.
- (-) The model may forget to call the tool when it changes its mind in text; mitigated by `chat.md` and a review of captures.
**Review trigger:** If manual QA finds text/tool mismatches.

### P3 — Image sent as a `file` part with a Buffer
**Status:** Accepted
**Date:** 2026-10-18
**Context:** AI SDK multimodal input; provider converts image file parts to OpenRouter `image_url` (data URL).
**Decision:** After sharp, pass `{type: 'file', mediaType: 'image/jpeg', data: Buffer}` in the user message of the analysis call only.
**Rejected alternatives:**
- Public image URL: requires storage; photos must not be persisted.
**Consequences:**
- (+) No storage; ≤ 2 MB payload.
- (-) Base64 increases request size by ~33% (≤ ~2.7 MB).
**Review trigger:** If OpenRouter limits inline image size below 2 MB for the chosen model.

---

## 7. Diagrams

### Component Diagram

```mermaid
classDiagram
    class PhotoAnalysisRoute { POST(multipart) }
    class DecisionRoute { POST(json) }
    class ChatRoute { POST(useChat body) }
    class ImagePipeline { prepareImage(file) }
    class PhotoAnalyst { analysePhoto(case, jpeg) }
    class DecisionMaker { decide(case, analysis, deadline) }
    class ChatStreamer { chatStream(body) }
    class Guards { checkDecision() checkRevision() }
    class ContextBlocks { caseBlock() analysisBlock() deadlineBlock() }
    class Prompts { compose(role, caseType) }
    class Policies { forCaseType(type) }
    class ModelFactory { getModel() }
    class Deadlines { checkDeadline() todayInWarsaw() }
    PhotoAnalysisRoute --> ImagePipeline
    PhotoAnalysisRoute --> PhotoAnalyst
    DecisionRoute --> DecisionMaker
    DecisionRoute --> Deadlines
    ChatRoute --> ChatStreamer
    ChatRoute --> Deadlines
    PhotoAnalyst --> ModelFactory
    PhotoAnalyst --> Prompts
    DecisionMaker --> ModelFactory
    DecisionMaker --> Guards
    DecisionMaker --> Prompts
    DecisionMaker --> ContextBlocks
    ChatStreamer --> ModelFactory
    ChatStreamer --> Guards
    ChatStreamer --> Prompts
    ChatStreamer --> ContextBlocks
    Prompts --> Policies
```

### Sequence — decision with guard retry

```mermaid
sequenceDiagram
    participant R as /api/decision
    participant D as DecisionMaker
    participant M as Model (OpenRouter)
    participant G as Guards
    R->>D: decide(case, analysis, deadline{exceeded: true})
    D->>M: generateText(Output.object DecisionDraft)
    M-->>D: {decision: ACCEPTED, ...}
    D->>G: checkDecision
    G-->>D: fail: "deadline exceeded → REJECTED, Regulamin zwrotów pkt 2"
    D->>M: generateText(previous output + correction)
    M-->>D: {decision: REJECTED, ...}
    D->>G: checkDecision → ok
    D->>D: strip + append disclaimer
    D-->>R: Decision
```

### Sequence — chat stream with rejected revision

```mermaid
sequenceDiagram
    participant C as useChat
    participant R as /api/chat
    participant S as streamText
    participant T as reviseDecision.execute
    C->>R: {messages, caseContext{currentDecision: REJECTED}}
    R->>S: stream (tools, stopWhen 3)
    S-->>C: text deltas
    S->>T: {to: ACCEPTED, reason}
    T->>T: deadline.exceeded → reject
    T-->>S: {accepted: false, rule: "Regulamin zwrotów, pkt 2"}
    S-->>C: tool part output-available (accepted: false)
    S-->>C: text explaining the rule (AC-31)
    Note over R: no disclaimer part, decision unchanged
```

---

## 8. Testing Strategy

### Test scenarios for this area

| Scenario | Type | Input | Expected output | Edge cases |
|---|---|---|---|---|
| Deadlines | Unit | Return 14/15 days; complaint 2y / 2y+1d | false/true | 29 Feb; 23:30 UTC = next day in Warsaw |
| Policy numbers pinned | Unit | Policy files | Contain "14 calendar days", "2 years" | — |
| Guards | Unit | Each rule violated / satisfied | Correction message / ok | Ref to the other policy; empty refs |
| Context blocks | Unit | Description with prompt-injection text | Appears only inside `<case>`; instructions unchanged | Empty description |
| Prompt composition | Unit | complaint vs return | Correct policy text included, other excluded | — |
| Image pipeline | Integration (real sharp) | `laptop-1.png`, `laptop-2.webp`, `phone-2.jpeg`, text bytes, 10 MB + 1 | JPEG ≤ 2048 px ≤ 2 MB no EXIF; webp ok; small not enlarged; 415; 413 | Truncated JPEG → 415 |
| Photo route | Integration | Real request + `MockLanguageModelV4` replaying capture | 200 `{analysis, rejection: null}`; model received an image file part and the type-specific prompt | Unusable capture → `rejection` with fixed message |
| Decision route | Integration | Mock returns ACCEPTED then REJECTED with exceeded deadline | 200 REJECTED; model called twice; disclaimer once at end | Always invalid → 502 after 3 calls |
| Chat route | Integration | Mock stream with text + `reviseDecision` call | UI message stream contains text, tool part, disclaimer part | Revision rejected → no disclaimer; 41 messages → 422 |
| Errors | Integration | Mock throws `APICallError`; mock delays > `totalMs` | 502 / 504 with Polish message, no internals | — |
| Live capture | Manual (`npm run capture:llm`) | 5 example images × 2 case types | Captures saved and valid against schemas | Run only with a key |

### Technical acceptance criteria

- **TAC-002-01** With `deadline.exceeded`, no response from `/api/decision` and no accepted `reviseDecision` has a decision other than `REJECTED`.
- **TAC-002-02** With `contradictionDetected: true`, no returned or accepted decision is `ACCEPTED`.
- **TAC-002-03** Every `Decision.messageMarkdown` ends with the exact disclaimer and contains it once.
- **TAC-002-04** For `laptop-1.png` the image passed to the model is JPEG, longer side 2048 px, ≤ 2,097,152 bytes, without EXIF.
- **TAC-002-05** The decision and chat calls never contain image data; the photo-analysis call never contains policy text.
- **TAC-002-06** Every capture in `tests/fixtures/captures/` validates against the current schemas.
