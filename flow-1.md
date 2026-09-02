
# Complete Runtime Flow: AI Voice Chatbot Order Creation Flow

---

## 1. Simple Flow Explanation

1. **Launcher Visibility & Period Check:** The application checks if the user's role has permission to see the chatbot (`SHOW_CHATBOT`). When clicked, it verifies whether the accounting period allows document creation in the current month (`CHECK_IF_INSERTED`).
2. **Audio Capture (Microphone):** The user clicks the microphone button. [AudioRecorderService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/audio-recorder.service.ts) accesses the device microphone via Web Audio API, meters real-time audio volume, and records 16kHz mono PCM audio into a standard WAV blob upon silence detection or manual stop.
3. **Speech-to-Text Transcription:** The WAV blob is sent to [SarvamSpeechService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/sarvam-speech.service.ts) (`saaras:v3` model with `en-IN` language code) or [OpenAiSpeechService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/openai-speech.service.ts). The raw transcript is normalized and validated against non-Latin script characters.
4. **AI Workflow Execution:** The transcribed text is sent via [ChatBotService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.service.ts) to `POST /api/ai/v1/workflow/{WorkflowName}/chat` with a persistent `X-Session-ID` header. The backend maps the active screen code (e.g., `SO`, `INV`, `DN`) to its designated AI workflow.
5. **Live Form Hydration:** If the AI response contains an intermediate `draft` order, [ChatBotComponent](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts) routes to `/sales-order/view/AI` passing `history.state.data = draft`, causing [SalesOrderComponent](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts) to populate form inputs, customer address, and line-item tables live on the screen.
6. **Voice Synthesis (TTS) & Navigation:** The assistant synthesizes spoken audio feedback via Sarvam TTS (`bulbul:v3`, voice: `ishita`). When the workflow status becomes `COMPLETED`, the app resolves the target redirect URL (`wtudTranUrl`) from the master mapping and navigates to the finalized order view `router.navigate([wtudTranUrl, voucherId])`.

---

## 2. ASCII Flow Diagram

```
+----------------------------------------------------------------------------------------------------+
|                                    1. LAUNCHER & PERMISSION GATE                                   |
|  User Clicks Launcher Icon  -->  AppComponent.chatBotShowHide()  -->  PortalService.checkInsertion |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                   2. VOICE CAPTURE & RECORDING                                     |
|  User Taps Mic Icon         -->  ChatBotComponent.startVoiceInput()                                |
|                             -->  AudioRecorderService.startRecording() (Web Audio API 16kHz WAV)  |
|                             -->  Volume Meter stream updates UI animation                          |
|  User Finishes Speaking     -->  AudioRecorderService.stopRecording() --> Emits WAV Blob           |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                   3. SPEECH-TO-TEXT TRANSCRIPTION                                  |
|  ChatBotComponent.onRecordingStopped(blob)                                                         |
|  --> SarvamSpeechService.transcribe(blob) [POST /api/sarvam/speech-to-text (saaras:v3, en-IN)]     |
|  --> Validation: speechDetected == true & containsNonLatinText() == false                          |
|  --> normalizeVoiceTranscript(text)                                                                |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                      4. AI WORKFLOW EXECUTION                                      |
|  ChatBotComponent.sendMessage(fromVoice = true)                                                    |
|  --> ChatBotService.sendMessage(payload, X-Session-ID)                                              |
|  --> POST /api/ai/v1/workflow/Create Sale Order - Sun Hol (Copy)/chat                              |
|  <-- Backend returns: { state: 'IN_PROGRESS'|'COMPLETED', draft: {...}, voucherId: 'SO-101' }      |
+----------------------------------------------------------------------------------------------------+
                                                  |
                         +------------------------+------------------------+
                         |                                                 |
                         v                                                 v
+--------------------------------------------------+  +---------------------------------------------+
|          5. LIVE ERP FORM HYDRATION              |  |         6. TTS & SCREEN NAVIGATION          |
| Draft received:                                  |  | ChatBotComponent.speak(botResponse)        |
| -> Router navigates to /sales-order/view/AI      |  | -> SarvamSpeechService.synthesize()         |
| -> history.state.data = draft                    |  | -> Audio element plays audio in browser     |
| -> SalesOrderComponent.getDetail('AI')           |  | When state == 'COMPLETED':                  |
| -> Populates Header, Items Table, Net Amounts    |  | -> ChatBotService.returnRoute('SO')         |
|                                                  |  | -> router.navigate(['/sales-order/view',    |
|                                                  |  |                     voucherId])             |
+--------------------------------------------------+  +---------------------------------------------+
```

---

## 3. Step-by-Step Runtime Flow Trace

```
User Action
  ↓
HTML (Template)
  ↓
Component
  ↓
Method
  ↓
Service
  ↓
API Endpoint
  ↓
Request Payload
  ↓
Backend Response
  ↓
State / Data Update
  ↓
HTML (Template View)
  ↓
UI Feedback
  ↓
Navigation
```

---

### Step 1: Launcher Display & User Clicks Floating Chatbot Icon

* **Exact File Path:** [src/app/chat-bot/chat-bot.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.html#L386-L389) and [src/app/chat-bot/chat-bot.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts#L420-L435)
* **Class Name:** `ChatBotComponent`
* **Method Name:** `toggleChat()`
* **What Happens:**
  1. The user clicks `<button class="chat-toggle-btn" (click)="toggleChat()">`.
  2. If `this.isOpen` is `false`, `toggleChat()` reads the active route's `erp_txn_code` or `orderType`.
  3. It calls `this.checkInsertion(txnCode)` to check if transactions can be created in the current month.
* **Why It Happens:** Prevents the user from entering orders for locked financial periods or invalid transaction types.
* **What Calls It:** User click event in template.
* **What It Calls Next:** `PortalService.getDynamicList()` with `subcatgid: 'CHECK_IF_INSERTED'`.

---

### Step 2: Accounting Period & Insertion Check API

* **Exact File Path:** [src/app/chat-bot/chat-bot.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts#L365-L418) and [src/app/service/portal.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts#L1796-L1801)
* **Class Name:** `PortalService`
* **Method Name:** `getDynamicList(obj)`
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/common/dynamic/landing/list`
* **Request Payload:**
  ```json
  {
    "ascordesc": "",
    "filter": {
      "TXNCODE": "SO",
      "MONTH": 9
    },
    "fromno": "",
    "orderbycol": "",
    "search": "",
    "subcatgid": "CHECK_IF_INSERTED",
    "tono": ""
  }
  ```
* **Backend Response:**
  ```json
  [
    {
      "trueOrFalse": "1"
    }
  ]
  ```
* **State / Data Update:** If `trueOrFalse === '1'`, `this.isOpen = true`.
* **UI:** The slide-out chat window container `<div class="chat-window" [class.open]="isOpen">` expands on screen.

---

### Step 3: User Clicks Microphone & Starts Audio Recording

* **Exact File Path:** [src/app/chat-bot/chat-bot.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.html#L215-L222), [src/app/chat-bot/chat-bot.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts#L1081-L1115), and [src/app/chat-bot/audio-recorder.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/audio-recorder.service.ts)
* **Class Name:** `AudioRecorderService`
* **Method Name:** `startRecording()`
* **What Happens:**
  1. The user clicks `<button (click)="toggleRecording()">`.
  2. `startVoiceInput()` invokes `AudioRecorderService.startRecording()`.
  3. `AudioRecorderService` calls `navigator.mediaDevices.getUserMedia({ audio: true })`.
  4. It initializes `AudioContext` (sample rate: 16,000 Hz) and attaches a `ScriptProcessorNode` to stream raw audio buffers.
  5. It calculates Root Mean Square (RMS) volume and emits updates via `volume$` Subject.
* **State / Data Update:**
  * `this.recordingState = 'recording'`
  * `this.isRecording = true`
  * `this.isVoiceMode = true`
  * `this.voiceModeActive = true`
* **UI Feedback:** Renders the pulsating recording ripple `<div class="mic-ping" [style.transform]="'scale(' + (1 + volume * 5) + ')'">` and displays *"Listening..."*.
* **What It Calls Next:** Subscribes to `this.recorder.stopped$` awaiting speech termination.

---

### Step 4: Speech Stops & WAV Blob Encoding

* **Exact File Path:** [src/app/chat-bot/audio-recorder.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/audio-recorder.service.ts) and [src/app/chat-bot/chat-bot.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts#L1189-L1265)
* **Class Name:** `ChatBotComponent`
* **Method Name:** `onRecordingStopped(blob: Blob)`
* **What Happens:**
  1. Silence is detected by VAD threshold or user clicks stop.
  2. `AudioRecorderService.stopRecording()` converts PCM float buffers into 16-bit integer mono PCM and packages standard RIFF/WAV headers.
  3. `stopped$` emits the finalized `Blob` (MIME: `audio/wav`).
  4. `ChatBotComponent.onRecordingStopped(blob)` sets `this.recordingState = 'transcribing'`.
* **What It Calls Next:** Calls `SarvamSpeechService.transcribe(blob)` (or `OpenAiSpeechService.transcribe`).

---

### Step 5: Speech-to-Text Transcription API

* **Exact File Path:** [src/app/chat-bot/sarvam-speech.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/sarvam-speech.service.ts#L33-L52)
* **Class Name:** `SarvamSpeechService`
* **Method Name:** `transcribe(audioBlob: Blob)`
* **API Endpoint:** `POST http://192.168.10.115:8089/api/sarvam/speech-to-text`
* **Headers:** `api-subscription-key: sk_nub5vhnf_lmQckKjVhk7rpgMMe9sVcOL8`
* **Request Payload (`FormData`):**
  * `file`: `recording.wav` (binary audio blob)
  * `model`: `saaras:v3`
  * `mode`: `transcribe`
  * `language_code`: `en-IN`
* **Backend Response:**
  ```json
  {
    "request_id": "req-987654",
    "transcript": "Create sale order for customer 1002 add 10 cases of orange juice",
    "language_code": "en-IN"
  }
  ```
* **Validation Pipeline in Component:**
  1. `normalizeVoiceTranscript(transcript)` removes noise and standardizes numbers.
  2. `if (!this.recorder.speechDetected || !normalizedTranscript.trim())` $\rightarrow$ triggers `failVoiceInput('No speech detected. Please try again.')`.
  3. `if (containsNonLatinText(normalizedTranscript))` $\rightarrow$ triggers `failVoiceInput('Please speak in English.')`.
* **State / Data Update:** Sets `this.newMessage = processedText`, sets `this.recordingState = 'idle'`.
* **What It Calls Next:** Triggers `this.sendMessage(true)`.

---

### Step 6: Dispatching Message to AI Workflow Backend

* **Exact File Path:** [src/app/chat-bot/chat-bot.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts#L528-L600) and [src/app/chat-bot/chat-bot.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.service.ts#L45-L121)
* **Class Name:** `ChatBotService`
* **Method Name:** `sendMessage()`
* **Workflow Resolution Logic:**
  * Order type `SO` $\rightarrow$ Workflow Name: `Create Sale Order - Sun Hol (Copy)`
  * Order type `INV` $\rightarrow$ Workflow Name: `Create Invoice - Sun Holding`
  * Order type `SR` $\rightarrow$ Workflow Name: `Create Sale Return - Sun Holding`
  * Order type `DN` $\rightarrow$ Workflow Name: `Create Delivery Note - Sun Holding`
  * Order type `MR` $\rightarrow$ Workflow Name: `Create Material Request - Sun Hol`
* **API Endpoint:** `POST {AI_SERVICE_BASE_URL}/api/ai/v1/workflow/Create Sale Order - Sun Hol (Copy)/chat?locale=en&userId=USR001&companyId=COMP01&tenantId=TEN-001&messageType=VOICE&orderType=SO`
* **Headers:** `X-Session-ID: <current_session_id>`
* **Request Payload:**
  ```json
  {
    "message": "Create sale order for customer 1002 add 10 cases of orange juice"
  }
  ```
* **Backend Response:**
  ```json
  {
    "sessionId": "session-xyz-12345",
    "state": "IN_PROGRESS",
    "response": "I have added 10 cases of Orange Juice for Customer 1002. Total amount is AED 450.",
    "draft": {
      "customerNo": "1002",
      "customerName": "Al Maya Supermarket",
      "orderType": "SO",
      "items": [
        {
          "itemCode": "ITM-OJ-01",
          "itemDesc": "Orange Juice 1L Case",
          "quantity": 10,
          "unitPrice": 45.0,
          "totalAmount": 450.0
        }
      ],
      "totalAmount": 450.0
    },
    "voucherId": null
  }
  ```
* **State / Data Update:**
  * `this.sessionId = response.sessionId`
  * `this.messages.push({ text: userMsg, sender: 'user' })`
  * `this.aiChatService.updateQuotationData(draft)`

---

### Step 7: Live ERP Form Hydration via Route Navigation

* **Exact File Path:** [src/app/chat-bot/chat-bot.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts#L764-L796) and [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L3198-L3225)
* **Class Name:** `SalesOrderComponent`
* **Method Name:** `getDetail('AI')` $\rightarrow$ `getDetailSUbAPI(detail)`
* **What Happens:**
  1. [ChatBotComponent](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts) executes `router.navigate(['sales-order', 'view', 'AI'], { state: { data: draft } })`.
  2. [SalesOrderComponent](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts) detects `this.detailId == 'AI'` in route params.
  3. It reads `const state = history.state.data` and invokes `getDetailSUbAPI(state)`.
  4. `getDetailSUbAPI()` populates:
     * Header customer code and customer delivery addresses via `getAddressDetails()`.
     * `saleOrderForm` controls: terms, currency, salesman codes.
     * Line items table data source with quantities, prices, discounts, and VAT calculations.
* **UI:** The active ERP Sales Order screen immediately reflects the draft order on the screen while the chat panel remains open.

---

### Step 8: Synthetic Text-to-Speech (TTS) Voice Feedback

* **Exact File Path:** [src/app/chat-bot/chat-bot.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts#L999-L1078) and [src/app/chat-bot/sarvam-speech.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/sarvam-speech.service.ts#L54-L83)
* **Class Name:** `SarvamSpeechService`
* **Method Name:** `synthesize(text, speaker = 'ishita')`
* **API Endpoint:** `POST http://192.168.10.115:8089/api/sarvam/text-to-speech`
* **Headers:** `api-subscription-key: sk_nub5vhnf_lmQckKjVhk7rpgMMe9sVcOL8`
* **Request Payload:**
  ```json
  {
    "text": "I have added 10 cases of Orange Juice for Customer 1002. Total amount is AED 450.",
    "target_language_code": "en-IN",
    "speaker": "ishita",
    "model": "bulbul:v3",
    "speech_sample_rate": 22050,
    "pace": 1.0
  }
  ```
* **Backend Response:**
  ```json
  {
    "request_id": "tts-112233",
    "audios": [ "UklGRi4AAABXQVZFZm10IBAAAAABAAEAQB8AAEAfAAABAAgAZGF0YQ..." ]
  }
  ```
* **Playback Execution:**
  1. `SarvamSpeechService` converts the base64 audio string into a blob URL `URL.createObjectURL(blob)`.
  2. `ChatBotComponent` instantiates `this.currentAudio = new Audio(audioUrl)`.
  3. `onplay`: Sets `this.ttsState = 'speaking'` and pushes the assistant message into `this.messages`.
  4. `onended`: Sets `this.ttsState = 'idle'`. If `voiceModeActive` is true and conversation is not finished, it triggers `this.scheduleVoiceRestart(500)` to automatically re-arm the microphone for the user's next turn.

---

### Step 9: Final Order Completion & Screen Navigation

* **Exact File Path:** [src/app/chat-bot/chat-bot.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts#L617-L668) and [src/app/chat-bot/chat-bot.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.service.ts#L122-L137)
* **Class Name:** `ChatBotService`
* **Method Name:** `returnRoute(erpTxnCode, userId)`
* **What Happens:**
  1. User says *"Yes, please place the order"*.
  2. AI workflow response returns: `state === 'COMPLETED'`, `voucherId: 'SO-2026-0042'`.
  3. `ChatBotComponent` calls `ChatBotService.returnRoute('SO', userId)`.
  4. `returnRoute()` queries `PortalService.getChatbotMappingDetail('SO')` to obtain the configured route: `wtudTranUrl: '/sales-order/view'`.
  5. Angular Router executes:
     ```typescript
     this.router.navigate(['/sales-order', 'view', 'SO-2026-0042']);
     ```
* **UI & Navigation Result:** The chat marks `isChatCompleted = true`. The browser navigates to the permanent, read-only finalized Sales Order document view displaying the generated voucher number and order details.

---

## 4. Important Technical Concepts

### 4.1 Angular Concepts
* **Dynamic Template Conditionals (`*ngIf`, `[ngClass]`, `[class.open]`):** Used for managing chat drawer expansion, recording waves, speaking banners, and message alignment (`user` vs `bot`).
* **ViewChild & ElementRef:** `@ViewChild('scrollContainer')` scrolls message history to bottom on new messages; `@ViewChild('chatInput')` handles auto-expanding textarea heights.
* **NgZone Run Outside/Inside Angular:** `this.zone.run()` wraps `Audio.onplay` and `Audio.onended` callbacks to ensure asynchronous Web Audio events trigger Angular's change detection.
* **ChangeDetectorRef:** `this.cdr.detectChanges()` forces UI updates when volume meters tick or async HTTP responses arrive.

### 4.2 TypeScript Concepts
* **Interfaces & Type Annotations:** Strongly typed message contracts:
  ```typescript
  interface Message {
    text: string;
    sender: 'user' | 'bot';
    timestamp: Date;
    link?: any[];
    ref?: string;
  }
  ```
* **Async / Await & Promises:** Converts RxJS observables to promises using `firstValueFrom` or custom promise wrappers for sequential permission checks (`await this.checkInsertion()`).

### 4.3 RxJS Concepts
* **BehaviorSubject & Subject:**
  * `ChatBotService.quotationDataSubject`: Stores and broadcasts the latest draft quotation state.
  * `AudioRecorderService.volume$`: Streams continuous volume changes (0.0 to 1.0) to drive UI animations.
* **Subscription Teardown (`sessionSubs`):** In-flight HTTP requests and audio synthesis observables are grouped in `this.sessionSubs.unsubscribe()` during `confirmClose()` so stale requests cannot speak or push messages into a closed chat.

### 4.4 API & Networking Concepts
* **Multipart Form Data (`FormData`):** Packages raw audio binary arrays with string metadata (`model`, `language_code`) for STT requests.
* **Custom HTTP Headers:** `X-Session-ID` passes the persistent conversation context token across multi-turn chat turns.
* **Blob Object URLs (`URL.createObjectURL`):** Converts Base64 WAV strings into local streaming URLs for browser audio playback without disk writes.

---

## 5. Important Files

| File Path | Role / Purpose |
| :--- | :--- |
| [src/app/chat-bot/chat-bot.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts) | Core component controller: handles recording, STT/TTS coordination, conversation state, and screen navigation. |
| [src/app/chat-bot/chat-bot.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.html) | UI template containing floating launcher, drawer panel, messages history, mic controls, and voice selectors. |
| [src/app/chat-bot/chat-bot.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.service.ts) | Service managing communication with AI backend, mapping transaction codes to workflows, and providing RxJS observables. |
| [src/app/chat-bot/audio-recorder.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/audio-recorder.service.ts) | Web Audio API wrapper that records 16kHz mono audio, detects volume levels, and exports WAV blobs. |
| [src/app/chat-bot/sarvam-speech.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/sarvam-speech.service.ts) | Speech adapter for Sarvam AI (`saaras:v3` STT and `bulbul:v3` TTS). |
| [src/app/chat-bot/openai-speech.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/openai-speech.service.ts) | Alternative speech adapter using OpenAI Whisper STT and TTS models. |
| [src/app/chat-bot/voice-transcript-normalizer.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/voice-transcript-normalizer.ts) | Text sanitization logic to reject non-English scripts and normalize quantity keywords. |
| [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts) | Target ERP sales order screen that reads AI draft data from route state and populates form inputs. |

---

## 6. Important Methods

| Class | Method | Purpose |
| :--- | :--- | :--- |
| `ChatBotComponent` | `toggleChat()` | Opens/closes chat drawer after evaluating month insertion permissions. |
| `ChatBotComponent` | `checkInsertion(TXNcode)` | Validates accounting month rules via `CHECK_IF_INSERTED` before opening chat. |
| `ChatBotComponent` | `startVoiceInput()` | Begins audio recording via `AudioRecorderService`. |
| `ChatBotComponent` | `onRecordingStopped(blob)` | Submits WAV blob to STT API, normalizes transcript, and dispatches message. |
| `ChatBotComponent` | `sendMessage(fromVoice)` | Transmits text query with `X-Session-ID` to AI workflow server. |
| `ChatBotComponent` | `speak(text, message)` | Sends text to TTS synthesis API and plays audio via `HTMLAudioElement`. |
| `ChatBotComponent` | `scheduleVoiceRestart(delay)` | Automatically restarts microphone recording after TTS playback finishes. |
| `ChatBotService` | `returnRoute(erpTxnCode, userId)` | Looks up target redirect URL from master mapping table for completed transactions. |
| `AudioRecorderService` | `startRecording()` | Initializes `AudioContext`, streams microphone input, and begins volume metering. |
| `AudioRecorderService` | `stopRecording()` | Converts float PCM audio buffers into standard WAV Blob. |
| `SarvamSpeechService` | `transcribe(audioBlob)` | Invokes `POST /api/sarvam/speech-to-text` with `saaras:v3` model. |
| `SarvamSpeechService` | `synthesize(text, speaker)` | Invokes `POST /api/sarvam/text-to-speech` with `bulbul:v3` model. |

---

## 7. API List Table

| Method | HTTP | Endpoint | Request Payload | Response | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `checkInsertion` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'CHECK_IF_INSERTED', filter: { TXNCODE, MONTH } }` | `[{ trueOrFalse: '1' }]` | Verify if transactions are permitted in current financial period. |
| `transcribe` | `POST` | `http://192.168.10.115:8089/api/sarvam/speech-to-text` | `FormData` (WAV file, `saaras:v3`, `en-IN`) | `{ transcript: "..." }` | Convert recorded voice audio into English text. |
| `sendMessage` | `POST` | `/api/ai/v1/workflow/{WorkflowName}/chat` | `{ message: "..." }` (Header: `X-Session-ID`) | `{ state, draft, voucherId, response }` | Execute conversational AI order booking step. |
| `synthesize` | `POST` | `http://192.168.10.115:8089/api/sarvam/text-to-speech` | `{ text, speaker: 'ishita', model: 'bulbul:v3' }` | `{ audios: [ base64String ] }` | Synthesize conversational voice feedback. |
| `returnRoute` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'CHATBOT_USER_MAPPING', filter: { TXNCODE } }` | `[{ wtudTranUrl: '/sales-order/view' }]` | Retrieve destination URL mapped to the transaction code. |

---

## 8. Business Rules Summary

1. **Role Authorization Gate:** The chatbot launcher only displays if the user's `SOLIDS_ROLEID` is mapped in generic master `SHOW_CHATBOT`.
2. **Accounting Month Check:** If `CHECK_IF_INSERTED` returns `trueOrFalse !== '1'`, the user is blocked from opening the chatbot and redirected to the list page with an error toast.
3. **Language & Script Validation:** Any transcript containing non-Latin script characters (e.g., Arabic or Hindi scripts) is rejected by `containsNonLatinText()` with a warning: *"Please speak in English."*
4. **Silence / VAD Rejection:** If `speechDetected` is false or the normalized transcript is empty, the voice loop halts and displays *"No speech detected. Please try again."*
5. **Continuous Voice Loop:** When voice mode is active, the app automatically pauses microphone capture during TTS playback and re-opens the microphone 500ms after playback completes (`scheduleVoiceRestart`).
6. **State Completion Navigation:** When the AI workflow returns `state === 'COMPLETED'`, SFA extracts the generated `voucherId`, finds the route in `chat-transaction-user-mapping`, and navigates to `/[wtudTranUrl]/[voucherId]`.

---

## 9. Concepts You Need to Learn

1. **Web Audio API (`AudioContext`, `ScriptProcessorNode`, `Float32Array`):** How raw audio streams are sampled at 16kHz and packaged into binary WAV containers.
2. **RxJS Observable Lifecycle & Unsubscription:** Managing `Subscription` instances and calling `unsubscribe()` on dialog close to prevent background memory leaks or late speech triggers.
3. **Angular Router Navigation with State (`history.state`):** How `router.navigate(url, { state: { data: draft } })` passes complex in-memory objects to destination components without exposing them in URL query parameters.
4. **Session-Header Management:** How `X-Session-ID` maintains multi-turn conversation memory across distinct HTTP requests to the AI server.

---

## 10. Things That Are Still Unclear (Unknowns / Server-Side Logic)

1. **AI Microservice Workflow Logic:** The internal prompt templates, item entity extraction rules, and validation logic inside `POST /api/ai/v1/workflow/{WorkflowName}/chat` are executed on the backend AI server (`192.168.10.115:8089`).
2. **Sarvam AI Deployment Mode:** Whether the endpoint `http://192.168.10.115:8089/api/sarvam/*` is an internal proxy forwarding requests to Sarvam's cloud API (`api.sarvam.ai`) or an on-premise container is defined on the microservice host.
3. **Database Insertion Rules (`CHECK_IF_INSERTED`):** The exact SQL stored procedure logic behind `CHECK_IF_INSERTED` resides in the database backend.





























<!-- 














 -->


Here is a breakdown of **which other modules exist in your codebase** that can be analyzed in this exact end-to-end runtime flow format, along with **actionable suggestions on which ones you should study next**.

* **Module A: AI Voice Chatbot Order Creation Flow** ([chat-bot.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/chat-bot/chat-bot.component.ts) $\rightarrow$ Audio Capture $\rightarrow$ Sarvam/OpenAI STT $\rightarrow$ Workflow API $\rightarrow$ Live ERP Hydration $\rightarrow$ Router Navigation).

Here are the **7 other signature modules** in your repository that follow similar complex architectures:

---

### Module 1: Geofenced Sales Order & Beat Validation Engine
* **Core File:** [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts) *(15,600+ lines)*
* **Runtime Scope:**
  * Salesman selects a customer $\rightarrow$ [GeocodingService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/geocoding.service.ts) gets device GPS coordinates.
  * System queries customer coordinates and calls `calculateCustomerGeoDistance`.
  * Evaluates the **50-meter Geofence rule** and scheduled **Beat Day** (`MON`, `TUE`, etc.).
  * Evaluates role privilege overrides: `OUT_RANGE_ORD`, `OUT_BEAT_ORD`, `OUT_ROUTE_ORD`.
  * Dynamic line items table: stock availability validation, tier pricing, multi-currency exchange rates, discounts, and ERP voucher creation.

---

### Module 2: Background Salesman Geo-Tracking & Telemetry Engine
* **Core File:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts)
* **Runtime Scope:**
  * Runs on app startup when session is detected.
  * Reads `GEO_TRACK_ENABLE_YN`, `GEO_TRACK_INTERVAL`, and checks `GEO_TRACK` in `ACCESS_LIST`.
  * Runs periodic heartbeat: GPS position + device battery level (`navigator.getBattery()`) + reverse geocoded address via Google Maps + public IP.
  * Posts telemetry payloads to `POST /api/map-api-log/save`.
  * **Anti-drain idle protection:** Pauses tracking when the user stays on the same route for $2 \times \text{Interval}$ to save battery.

---

### Module 3: Salesman Route Tracker & Nearest-Neighbor Sequencing
* **Core File:** [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts)
* **Runtime Scope:**
  * Loads scheduled shops for today's beat (`CUSTOMER_GEO_DETAILS_DATED`).
  * Runs `orderLocationsByNearest()` to sort pending shops by proximity to the salesman's current GPS position.
  * Renders color-coded map pins: 🟡 Check-in without order, 🔵/🔴 Order placed, ⚪ Planned stop.
  * Handles **"No Order" visits** with reason codes (`SHOP_CLOSED`, `OWNER_UNAVAILABLE`) and salesman remarks.

---

### Module 4: Supervisor Live Geotracking & Breadcrumb Map
* **Core File:** [src/app/masters/salesman-geotracking/salesman-geotracking.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-geotracking/salesman-geotracking.component.ts)
* **Runtime Scope:**
  * Managers select salesmen and dates to inspect field activity.
  * Draws Google Maps **polylines** showing actual traveled paths.
  * Interactive pins allow managers to view visit times, order values, and check-in statuses.

---

### Module 5: Salesman Performance Dashboard & KPI Analytics
* **Core File:** [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts)
* **Runtime Scope:**
  * Aggregates executive KPIs: Planned vs. Actual Visits, Visit Coverage %, Productive Visit %, Total Order Value.
  * **Route Grid (`GEO_SALES_PERF_3`):** Breaks down inside-geofence vs. outside-geofence orders.
  * **Salesman Grid (`GEO_SALES_PERF_4`):** Individual performance summary.
  * **Visit Timeline Modal (`GEO_SALES_PERF_5`):** Double-clicking a salesman row opens a chronological audit timeline of each customer stop.
  * Excel report download via `PortalService.XXlList('SALESMAN_PERF_XL_EXPORT')`.

---

### Module 6: Geo API Raw Audit Log Inspector
* **Core File:** [src/app/report-geo-api/geo-api-report.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/report-geo-api/geo-api-report.component.ts)
* **Runtime Scope:**
  * Paginated audit log grid for compliance officers.
  * Shows exact timestamp, device UUID, GPS coordinates, accuracy in meters, battery %, resolved street address, and linked ERP sales order numbers.
  * Excel export via `GEO_API_REPORT_XL`.

---

### Module 7: Customer Coordinates & Places Autocomplete Master
* **Core File:** [src/app/masters/customeraddress/customeraddress.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/customeraddress/customeraddress.component.ts)
* **Runtime Scope:**
  * Google Places Autocomplete search to pinpoint store locations.
  * Interactive map with draggable pin to fine-tune $(\text{Lat}, \text{Lng})$.
  * Linking customers to designated Routes and Beats.

---

## 2. Top Recommendations: Which Module to Learn Next?

### 🥇 Recommendation 1: Geofenced Sales Order Booking (Highest Priority)
* **Why:** This is the core revenue-generating screen of the application where all business rules converge (Geofencing, Beat calendars, Privileges, Pricing, and ERP posting).
* **What you will master:**
  1. How the 50m distance math interacts with `OUT_RANGE_ORD` approvals.
  2. How the Beat validation logic matches weekday codes against customer master data.
  3. How item stock check and line-item taxes are calculated before final submission.

---

### 🥈 Recommendation 2: Salesman Route Tracker & Nearest-Neighbor Sequencer
* **Why:** This is the primary mobile screen used by sales reps every morning in the field.
* **What you will master:**
  1. The `orderLocationsByNearest()` TSP proximity sorting algorithm.
  2. The check-in and "No Order" visit submission flow (`saveSalesmanCustomerVisit`).
  3. How visits update color-coded pins dynamically on Google Maps.

---

### 🥉 Recommendation 3: Salesman Performance Dashboard
* **Why:** This is the main screen used by Sales Managers and Executives.
* **What you will master:**
  1. How manager filters (`selectedDateRange`, `selectedManager`, `selectedBeats`) trigger parallel API calls via `forkJoin`.
  2. How double-click modal timelines work (`GEO_SALES_PERF_5`).
  3. How large data grids are exported to Excel files in Angular.

---

## 3. Practical Suggestions for Your Development & Codebase Mastery

1. **Understand `PortalService.getDynamicList()` First:**
   * Almost 80% of data queries in this project (reports, grids, dropdowns, access checks) go through `POST /api/common/dynamic/landing/list` with different `subcatgid` values (e.g. `ACCESS_LIST`, `CHECK_IF_INSERTED`, `GEO_SALES_PERF_4`, `CUSTOMER_GEO_DETAILS_DATED`). Understanding this single pattern unlocks how the entire app talks to the database.
2. **Keep an Eye on `localStorage` Keys:**
   * Global state is heavily stored in `localStorage` (`SOLIDS_DATA`, `SOLIDS_Token`, `SOLIDS_MASTER_URL`, `Route`, `Beat`, `ALQ_ACCESS`). Always check these keys when debugging user permissions or multi-company switches.
3. **Trace Asynchronous Zones for Map & Audio Events:**
   * Whenever Google Maps geocoding or Web Audio playback events occur, look for `this.zone.run(() => ...)` and `this.cdr.detectChanges()`. This is how the codebase ensures browser background APIs synchronize with Angular's UI.

---

**Which module would you like to break down next into the exact runtime flow format?**
* **Option 1:** *Geofenced Sales Order Booking & Beat Validation (`sales-order.component.ts`)*
* **Option 2:** *Salesman Route Tracker & Map Routing (`salesman-tracker.component.ts`)*
* **Option 3:** *Salesman Performance Dashboard (`salesman-performance-dashboard.component.ts`)*
* **Option 4:** *Background Telemetry & Battery Tracking (`app.component.ts`)*






<!-----------------------------------------

































 ------------------------>


 Searched for "autoCompleteCustomerNo"
Viewed sales-order.component.html:315-365
Viewed sales-order.component.ts:2430-2500
Viewed sales-order.component.ts:2501-2700
Searched for "SalesquoteCreate"
Searched for "portalService.quoteCreate"
Searched for "quoteCreate"
Searched for "submitForm"
Searched for "saveSaleOrder"
Searched for "Save"
Viewed sales-order.component.ts:6375-6450
Searched for "generateOrder()"
Viewed sales-order.component.ts:15115-15175
Searched for "generateOrder"
Viewed sales-order.component.ts:4758-4830
Viewed sales-order.component.ts:5270-5350
Viewed sales-order.component.ts:5351-5420

# Complete Runtime Flow: Geofenced Sales Order Booking & Beat Validation

---

## 1. Simple Flow Explanation

1. **Customer Selection & Permissions Check:** When the salesman searches and selects a retail customer in [SalesOrderComponent](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts), the system verifies whether the salesman's role has `GEO_TRACK` access.
2. **Route & Beat Calendar Validation:** SFA retrieves the customer's assigned Route and Beat codes. It maps the current order date to a weekday code (e.g. `MON`, `TUE`) via `getOrderDateBeat()`. If the customer does not belong to today's Beat or the salesman's assigned Route, SFA checks for override privileges (`OUT_BEAT_ORD`, `OUT_ROUTE_ORD`). If privileges are missing, customer selection is blocked.
3. **GPS Acquisition & Distance Calculation:** The browser captures high-accuracy GPS coordinates of the salesman's device using [GeocodingService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/geocoding.service.ts). It queries the customer's registered store coordinates from the database and calls `POST /api/customer/geo/calculate-distance`.
4. **50-Meter Geofence Gate:**
   * **Distance $\le 50\text{m}$:** Geofence check passes immediately.
   * **Distance $> 50\text{m}$:** SFA checks whether the user possesses the `OUT_RANGE_ORD` privilege. If granted, booking proceeds with an *"Out of Range Approval"* notification; if absent, the customer is cleared, form inputs are locked, and an error toast is displayed.
5. **Auto-Registration of Missing Coordinates:** If the customer outlet does not yet have GPS coordinates in the database, SFA prompts a modal offering to auto-register the customer's coordinates using the salesman's current GPS position via `createCustomerGeo`.
6. **Line Items, Pricing & Stock Validation:** The salesman adds catalog items, specifies quantities and discounts, and validates real-time inventory and credit limits.
7. **ERP Order Generation & Telemetry Logging:** Clicking **Save** runs `generateOrder()`, posting the complete document payload to `POST /api/sales/quotation/create` (or update). Upon success, SFA automatically submits a salesman visit record (`createSalesmanCustomerVisit`) with status `Order`, logs physical coordinates to `POST /api/map-api-log/save`, and navigates to `/sales-order/update/:quoteId`.

---

## 2. ASCII Flow Diagram

```
+----------------------------------------------------------------------------------------------------+
|                                    1. CUSTOMER SEARCH & SELECTION                                  |
|  User types customer name  -->  (completeMethod)="customerListTextChange($event)"                  |
|  User selects customer     -->  (onSelect)="customerSelect()"                                      |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                 2. ROUTE & BEAT CALENDAR VALIDATION                                |
|  PortalService.getCustomerGeoByCompanyAndCustomer(compId, customerCode)                            |
|  --> Route Check: Does Customer Route match Salesman Assigned Route?                               |
|      [NO] -> Check 'OUT_ROUTE_ORD' privilege. Missing? -> BLOCK SELECTION.                         |
|  --> Beat Check: getOrderDateBeat(date) -> (e.g. 'MON', 'TUE')                                     |
|      [NO] -> Check 'OUT_BEAT_ORD' privilege. Missing? -> BLOCK SELECTION.                          |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                  3. DEVICE GPS & DISTANCE CALCULATION                              |
|  GeocodingService.getCurrentLocation() -> Captures device (Lat, Lng)                               |
|  Google Maps Geocoder -> Resolves physical street address                                          |
|  Has Customer GPS in DB?                                                                           |
|  ├── [NO]  --> Prompt confirmation modal -> Auto-register store GPS via createCustomerGeo          |
|  └── [YES] --> PortalService.calculateCustomerGeoDistance(userLat, userLng, custLat, custLng)     |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                    4. 50-METER GEOFENCE GATE                                       |
|  Is calculated distance <= 50 meters?                                                              |
|  ├── [YES] --> Pass immediately.                                                                   |
|  └── [NO]  --> Check 'OUT_RANGE_ORD' privilege.                                                     |
|                ├── [MISSING] -> Clear customer input, lock form, Toast: "Beyond 50m limit".        |
|                └── [PRESENT] -> Toast: "You Have Out of Range Order Approval". Continue booking.   |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                               5. LINE ITEMS, PRICING & STOCK CHECK                                 |
|  Salesman inputs item codes, quantities, and discounts                                             |
|  Real-time stock validation, VAT tax calculation, and outstanding credit balance checks            |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                              6. ERP ORDER CREATION & VISIT LOGGING                                 |
|  User clicks 'Save' -> SalesOrderComponent.generateOrder()                                         |
|  --> PortalService.generateOrder(orderPayload) -> Generates quoteId & quoteRef                     |
|  --> Automatically calls PortalService.createSalesmanCustomerVisit() (Status: 'Order')             |
|  --> Automatically dispatches background telemetry to POST /api/map-api-log/save                   |
|  --> Router navigates to /sales-order/update/:quoteId                                              |
+----------------------------------------------------------------------------------------------------+
```

---

## 3. Step-by-Step Runtime Flow Trace

```
User Action
  ↓
HTML (Template)
  ↓
Component
  ↓
Method
  ↓
Service
  ↓
API Endpoint
  ↓
Request Payload
  ↓
Backend Response
  ↓
State / Data Update
  ↓
HTML (Template View)
  ↓
UI Feedback
  ↓
Navigation
```

---

### Step 1: Customer Autocomplete Search & Selection

* **Exact File Path:** [src/app/sales-order/sales-order.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.html#L323-L332) and [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L2436-L2455)
* **Class Name:** `SalesOrderComponent`
* **Method Name:** `customerSelect()`
* **What Happens:**
  1. The user types a customer code or store name into the PrimeNG AutoComplete input `#autoCompleteCustomerNo`.
  2. Clicking a customer triggers `(onSelect)="customerSelect()"`.
  3. `customerSelect()` extracts `customer = this.saleOrderForm.getRawValue()['customerNo']`.
  4. It resets tracking coordinates: `this.salesmanLatitude = undefined`, `this.salesmanLongitude = undefined`, `this.allowedRadius = 0`.
  5. It checks `if (this.hasGeoTrackAccess)` to initiate the validation pipeline.
* **Why It Happens:** Ensures field compliance checks execute before enabling item entry grids or billing calculations.
* **What Calls It:** Template event `(onSelect)="customerSelect()"`.
* **What It Calls Next:** `PortalService.getCustomerGeoByCompanyAndCustomer()`.

---

### Step 2: Fetch Customer Master GPS Coordinates

* **Exact File Path:** [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L2448-L2456) and [src/app/service/portal.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts)
* **Class Name:** `PortalService`
* **Method Name:** `getCustomerGeoByCompanyAndCustomer(companyCode, customerCode)`
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/customer/geo/byCompanyAndCustomer`
* **Request Payload:**
  ```json
  {
    "companyCode": "COMP01",
    "customerCode": "CUST-1002"
  }
  ```
* **Backend Response:**
  ```json
  {
    "status": "SUCCESS",
    "data": {
      "customerGeoId": 412,
      "companyCode": "COMP01",
      "customerCode": "CUST-1002",
      "customerName": "Al Maya Supermarket",
      "latitude": 25.204849,
      "longitude": 55.270782,
      "routeCode": "ROUTE-01",
      "beatCode": "MON,WED,FRI",
      "address": "Al Fahidi St, Bur Dubai"
    }
  }
  ```
* **State / Data Update:** Sets `this.customerRoute = customerGeo.data.routeCode`.
* **What It Calls Next:** Route and Beat validation logic.

---

### Step 3: Route & Beat Calendar Validation

* **Exact File Path:** [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L2457-L2524)
* **Class Name:** `SalesOrderComponent`
* **Method Name:** `getOrderDateBeat()` & `confirmRouteBeat(message)`
* **What Happens:**
  1. **Route Check:** Reads salesman assigned routes `localStorage.getItem('Route')` and compares with `customerGeo.data.routeCode`.
     * If route does not match:
       * If `!this.isOutOfRouteOrder`: Clears customer control, displays error toast: *"Selected customer does not belong to your assigned route."* and halts execution.
       * If `this.isOutOfRouteOrder`: Calls `await this.confirmRouteBeat("Selected customer does not belong to your assigned route. Do you want to proceed?")`.
  2. **Beat Check:** Invokes `todayBeat = this.getOrderDateBeat()` which converts `new Date()` into standard day tokens (`MON`, `TUE`, `WED`, `THU`, `FRI`, `SAT`, `SUN`).
     * Compares `todayBeat` against `customerGeo.data.beatCode.split(',')` and `salesmanBeats`.
     * If customer does not belong to today's Beat:
       * If `!this.isOutOfBeatOrder`: Clears customer control, displays error toast: *"Selected customer does not belong to today's beat."* and halts execution.
       * If `this.isOutOfBeatOrder`: Calls `await this.confirmRouteBeat("Selected customer does not belong to today's beat. Do you want to proceed?")`.
  3. **Outside Booking Flag:** Sets `this.isOutsideBooking = !(isValidRoute && isValidBeat && isValidCustomerBeat)`.
* **What It Calls Next:** `this.getCurrentLocation()` via `GeocodingService`.

---

### Step 4: Device GPS Acquisition & Reverse Geocoding

* **Exact File Path:** [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L2526-L2549) and [src/app/service/geocoding.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/geocoding.service.ts#L19-L72)
* **Class Name:** `GeocodingService`
* **Method Name:** `getCurrentLocation()` and `getAddressFromCoords(lat, lng)`
* **What Happens:**
  1. `GeocodingService.getCurrentLocation()` requests GPS coordinates via `navigator.geolocation.getCurrentPosition` with `enableHighAccuracy: true`, `timeout: 10000`.
  2. If high-accuracy fails due to timeout or hardware, it automatically falls back to low accuracy.
  3. `GeocodingService.getAddressFromCoords(lat, lng)` loads the Google Maps Geocoder script and resolves the physical address string.
* **State / Data Update:**
  * `this.salesmanLatitude = userCoords.latitude`
  * `this.salesmanLongitude = userCoords.longitude`
* **UI Feedback:** Displays info toast: `"Location: Lat 25.204910, Lng 55.270810"`.
* **What It Calls Next:** Distance calculation or auto-registration flow.

---

### Step 5: Auto-Registration of Missing Customer GPS Coordinates

* **Exact File Path:** [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L2551-L2605)
* **Class Name:** `SalesOrderComponent`
* **Method Name:** `confirmCreateCustomerGeo()`
* **Condition:** `if (!customerGeo || !customerGeo.data || !customerGeo.data.latitude || !customerGeo.data.longitude)`
* **What Happens:**
  1. Opens template modal `templateCustomerGeoConfirm` asking: *"Customer location coordinates are not registered. Do you want to register your current location as this customer's store location?"*.
  2. If the salesman confirms:
     * Dispatches `POST {SOLIDS_MASTER_URL}/api/customer/geo/create`:
       ```json
       {
         "companyCode": "COMP01",
         "customerCode": "CUST-1002",
         "customerName": "Al Maya Supermarket",
         "latitude": 25.204910,
         "longitude": 55.270810,
         "address": "Al Fahidi St, Bur Dubai, UAE",
         "locationDetails": "Auto registered from Sales Order screen",
         "createdBy": "USR001",
         "routeCode": "ROUTE-01",
         "beatCode": "MON,WED,FRI"
       }
       ```
     * Sets `bypassDistanceCheck = true` and shows success toast: *"Customer location registered successfully."*.

---

### Step 6: 50-Meter Geofence Distance Calculation API

* **Exact File Path:** [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L2608-L2642) and [src/app/service/portal.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts)
* **Class Name:** `PortalService`
* **Method Name:** `calculateCustomerGeoDistance(lat1, lon1, lat2, lon2)`
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/customer/geo/calculate-distance`
* **Request Payload:**
  ```json
  {
    "lat1": 25.204910,
    "lon1": 55.270810,
    "lat2": 25.204849,
    "lon2": 55.270782
  }
  ```
* **Backend Response:**
  ```json
  {
    "status": "SUCCESS",
    "distance_km": 0.0072,
    "message": "Distance calculated successfully"
  }
  ```
* **Calculated Distance:** $0.0072\text{ km} \times 1000 = 7.20\text{ meters}$.
* **State / Data Update:** Sets `this.allowedRadius = 7.20`.
* **Geofence Gate Evaluation:**
  * **If Distance $\le 50\text{m}$ (or `this.geoAllowedRadius`):** Validation passes silently.
  * **If Distance $> 50\text{m}$:**
    * If `this.isOutOfRangeOrder === true` (User has `OUT_RANGE_ORD` privilege): Displays success toast *"You Have Out of Range Order Approval"* and permits booking.
    * If `this.isOutOfRangeOrder === false`: Clears `saleOrderForm.controls['customerNo'].setValue('')`, displays error toast: *"your location not match exact customer location (Distance: 120.45 meters, Limit: 100 meters)"*, and halts execution.

---

### Step 7: Form Hydration & Outstanding Credit Balance

* **Exact File Path:** [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L2645-L2710)
* **Class Name:** `SalesOrderComponent`
* **Method Name:** `getCustomerDefaults(custCode)` and `getAddressDetails(custCode)`
* **What Happens:**
  1. Fetches payment terms, price lists (`getPL_CODE()`), default ship-to locations, and VAT percentage.
  2. Queries outstanding balance and credit limits via `PortalService.get_quotation_Dropdownlist()`.
  3. Hydrates form fields: `termsCode`, `currencyCode`, `currencyDecimal`, `vat`, `custType`.
* **UI:** Unlocks item search bar `#itemsListSearchBox` and line items grid.

---

### Step 8: Item Addition, Stock Check & Pricing

* **Exact File Path:** [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L4758-L4830)
* **Class Name:** `SalesOrderComponent`
* **Method Name:** `calculateTotalAmount()` and `getItemAtSaveLocationList()`
* **What Happens:**
  1. Salesman adds items with quantity, unit rate, and discount percentage.
  2. SFA validates real-time stock balance in the selected warehouse location.
  3. Form recalculates: `grossTotal`, `discountValue`, `vatAmount`, and `netTotal`.
  4. SFA verifies credit control rules (`creditControlYN === 'Y'`): verifies if `netTotal <= availableBalance`.

---

### Step 9: Order Submission & ERP Document Creation API

* **Exact File Path:** [src/app/sales-order/sales-order.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.html#L32) and [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L4758-L5390)
* **Class Name:** `SalesOrderComponent`
* **Method Name:** `generateOrder()`
* **User Action:** Salesman clicks `<button (click)="generateOrder()"> Save </button>`.
* **Service:** [PortalService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts) $\rightarrow$ `generateOrder(orderObj)`
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/sales/quotation/create` (or `/update`)
* **Request Payload (`orderObj`):**
  ```json
  {
    "companyCode": "COMP01",
    "customerNo": "CUST-1002",
    "customerName": "Al Maya Supermarket",
    "orderType": "SO",
    "orderDate": "2026-09-02T00:00:00.000Z",
    "locationCode": "DXB-WH01",
    "termsCode": "CR-30",
    "currencyCode": "AED",
    "totValue": 450.0,
    "discountValue": 0.0,
    "vat": 22.5,
    "netTotal": 472.5,
    "salesmanLatitude": 25.204910,
    "salesmanLongitude": 55.270810,
    "allowedRadius": 7.20,
    "items": [
      {
        "itemCode": "ITM-OJ-01",
        "quantity": 10,
        "rate": 45.0,
        "amount": 450.0
      }
    ]
  }
  ```
* **Backend Response:**
  ```json
  {
    "status": "SUCCESS",
    "quoteId": 114088,
    "quoteRef": "SON2026-00088",
    "message": "Created Successfully"
  }
  ```

---

### Step 10: Automatic Salesman Customer Visit Log & Navigation

* **Exact File Path:** [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts#L5288-L5365)
* **Class Name:** `PortalService`
* **Method Name:** `createSalesmanCustomerVisit(visitPayload)`
* **API Endpoints:**
  1. `POST {SOLIDS_MASTER_URL}/api/salesman/customer/visit/save`
  2. `POST {SOLIDS_MASTER_URL}/api/map-api-log/save`
* **Visit Payload Dispatched:**
  ```json
  {
    "salesmanCode": "USR001",
    "customerCode": "CUST-1002",
    "companyCode": "COMP01",
    "visitDate": "2026-09-02",
    "visitTime": "10:15:30",
    "salesmanLatitude": 25.204910,
    "salesmanLongitude": 55.270810,
    "allowedRadius": 7.20,
    "flexField01": "SO",
    "flexField02": "SON2026-00088",
    "flexField03": "Order",
    "flexField04": 472.5,
    "flexField08": "INSIDE",
    "flexField09": "ROUTE-01",
    "flexField12": 450.0
  }
  ```
* **UI & Navigation:**
  * Displays success toast: *"Created Successfully!!!"*.
  * Angular Router navigates to:
    ```typescript
    this.router.navigate([`/sales-order/update/${data.quoteId}`], { queryParams: this.queryParamsData });
    ```

---

## 4. Important Technical Concepts

### 4.1 Angular Concepts
* **Reactive Forms (`FormGroup`, `FormControl`):** Manages all header inputs (`saleOrderForm`), dynamic validators, and disabled controls.
* **CDK Drag and Drop (`@angular/cdk/drag-drop`):** Allows salesmen on tablet devices to re-arrange header layout cards dynamically (`cdkDrag`, `cdkDropList`).
* **PrimeNG AutoComplete (`p-autoComplete`):** Asynchronous customer, location, and currency dropdowns with keyboard navigation shortcuts (`(keydown.enter)`).
* **Modal Service (`BsModalService`, `BsModalRef`):** Renders confirmation popups for Beat override (`templateRouteBeatConfirm`) and customer GPS auto-registration (`templateCustomerGeoConfirm`).

### 4.2 TypeScript Concepts
* **Promises with Resolvers:** Dynamic promise pattern used for modal confirmations:
  ```typescript
  confirmRouteBeat(msg: string): Promise<boolean> {
    return new Promise((resolve) => {
      this.confirmRouteBeatResolve = resolve;
      this.modalRef = this.modalService.show(this.templateRouteBeatConfirm);
    });
  }
  ```
* **Date Parsing & ISO Normalization:** `getOrderDateBeat()` extracts UTC day names without timezone drift bugs.

### 4.3 RxJS Concepts
* **firstValueFrom & toPromise():** Awaits HTTP responses inside `async customerSelect()` to ensure sequential execution of Geofence and Beat rules before form rendering.
* **forkJoin:** Used in `validateItemsBeforeApprove()` to run simultaneous warehouse stock checks across all order lines.

### 4.4 API & Math Concepts
* **Haversine Distance Math:** Executed on backend `calculate-distance` API to accurately compute great-circle distance between GPS coordinates over the Earth's sphere.
* **GPS Accuracy Fallback:** High accuracy GPS request $\rightarrow$ retry with low accuracy on timeout.

---

## 5. Important Files

| File Path | Role / Purpose |
| :--- | :--- |
| [src/app/sales-order/sales-order.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts) | Central sales order component (>15,600 lines); executes Geofence check, Beat check, line-item pricing, and order submission. |
| [src/app/sales-order/sales-order.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.html) | Template containing AutoComplete dropdowns, Drag-and-Drop cards, item grids, and confirmation modals. |
| [src/app/service/portal.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts) | API layer for customer GPS coordinates, distance calculation, ERP order creation, and visit logging. |
| [src/app/service/geocoding.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/geocoding.service.ts) | Geolocation hardware wrapper handling GPS coordinates capture and Google Maps reverse geocoding. |
| [src/app/masters/customeraddress/customeraddress.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/customeraddress/customeraddress.component.ts) | Master screen used to configure customer reference coordinates $(\text{Lat}, \text{Lng})$. |

---

## 6. Important Methods

| Class | Method | Purpose |
| :--- | :--- | :--- |
| `SalesOrderComponent` | `customerSelect()` | Primary handler on customer selection: triggers Route, Beat, GPS, and Geofence checks. |
| `SalesOrderComponent` | `getOrderDateBeat(date)` | Converts selected order date into standard weekday code (`MON`, `TUE`, etc.). |
| `SalesOrderComponent` | `confirmRouteBeat(msg)` | Displays modal popup prompting the user to confirm out-of-beat/route orders. |
| `SalesOrderComponent` | `confirmCreateCustomerGeo()` | Displays modal popup offering to auto-register store coordinates from current device GPS. |
| `SalesOrderComponent` | `generateOrder()` | Validates form and line items, then submits order payload to ERP API. |
| `PortalService` | `getCustomerGeoByCompanyAndCustomer()` | Queries customer coordinates, route code, and beat days from the database. |
| `PortalService` | `calculateCustomerGeoDistance()` | Calculates physical distance in meters between salesman and store. |
| `PortalService` | `createSalesmanCustomerVisit()` | Automatically logs visit record (`Order` vs `No Order`) with GPS coordinates and order amount. |
| `GeocodingService` | `getCurrentLocation()` | Interacts with browser geolocation API to retrieve device coordinates with accuracy. |

---

## 7. API List Table

| Method | HTTP | Endpoint | Request Payload | Response | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `checkGeoAccess` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'ACCESS_LIST', filter: { ROLE_ID, COMPID } }` | List of `AccessId` privileges | Fetch security privileges (`GEO_TRACK`, `OUT_RANGE_ORD`, `OUT_BEAT_ORD`). |
| `getCustomerGeo` | `POST` | `/api/customer/geo/byCompanyAndCustomer` | `{ companyCode, customerCode }` | `{ data: { latitude, longitude, routeCode, beatCode } }` | Fetch reference store GPS coordinates and beat assignments. |
| `calculateDistance` | `POST` | `/api/customer/geo/calculate-distance` | `{ lat1, lon1, lat2, lon2 }` | `{ status: 'SUCCESS', distance_km: 0.0072 }` | Compute physical distance in meters between salesman and store. |
| `createCustomerGeo` | `POST` | `/api/customer/geo/create` | `{ companyCode, customerCode, latitude, longitude, address, ... }` | `{ status: 'SUCCESS' }` | Auto-register customer coordinates in database from device GPS. |
| `generateOrder` | `POST` | `/api/sales/quotation/create` | Full Order JSON (Header, Customer, Items, Totals, GPS coordinates) | `{ status: 'SUCCESS', quoteId, quoteRef }` | Generate ERP Sales Order document. |
| `createSalesmanCustomerVisit` | `POST` | `/api/salesman/customer/visit/save` | `{ salesmanCode, customerCode, visitDate, visitTime, flexField03: 'Order', ... }` | Success confirmation | Log visit record in manager audit tables. |
| `sendLocationLog` | `POST` | `/api/map-api-log/save` | `{ wmalCompId, wmalUserId, wmalLatitude, wmalLongitude, flexField02: 'Order' }` | Success confirmation | Update background telemetry map audit history. |

---

## 8. Business Rules Summary

1. **50-Meter Geofence Threshold:** The salesman must be within 50 meters of the customer's pinned store coordinates.
2. **`OUT_RANGE_ORD` Privilege Override:** If distance $> 50\text{m}$, order booking is locked unless the salesman's role contains `OUT_RANGE_ORD` in its privileges list.
3. **Beat Day Matching:** Customer's scheduled beat days (`customerGeo.data.beatCode`) must match today's weekday code (`getOrderDateBeat()`). Otherwise, order booking is blocked unless the user has `OUT_BEAT_ORD`.
4. **Route Matching:** Customer's assigned route must match the salesman's assigned route (`localStorage.getItem('Route')`). Otherwise, blocked unless user has `OUT_ROUTE_ORD`.
5. **Credit Control Enforcement:** If `creditControlYN === 'Y'`, the order's `netTotal` cannot exceed the customer's available credit balance (`conBalanceAmt`).
6. **Automatic Visit Audit:** Every successful order booking automatically creates an entry in `SALESMAN_CUSTOMER_VISIT_LIST` marked with `flexField08: 'INSIDE'` or `'OUTSIDE'` based on the geofence calculation.

---

## 9. Concepts You Need to Learn

1. **Browser Geolocation Permissions & High Accuracy Mode:** How `navigator.geolocation.getCurrentPosition` handles GPS satellite locks vs. cellular triangulation.
2. **Reverse Geocoding via Google Maps API:** How $(\text{Lat}, \text{Lng})$ pairs are translated into physical formatted street addresses.
3. **Promise-Wrapped Modal Dialogs in Angular:** How to suspend synchronous execution flow and await user confirmation from a Bootstrap modal template.
4. **Decoupled Audit Telemetry:** How transaction creation triggers parallel background telemetry logs to keep manager maps synchronized.

---

## 10. Things That Are Still Unclear (Unknowns / Server-Side Logic)

1. **Backend Distance Formula Details:** The specific threshold constants and elevation adjustments used in `POST /api/customer/geo/calculate-distance` reside on the backend server.
2. **Credit Rule Checking Stored Procedures:** The exact financial validation procedures (`creditRuleCheck()`, `checkCreditRuleOrion()`) query database views (`BALANCE_AMT`, `OSAMT`) that depend on the ERP accounting engine.
3. **Dynamic Form Field Configuration (`leftColumnData`):** How specific form fields are dynamically shown, hidden, or re-ordered across different companies via `getHeaderFormFields()` is driven by company-specific master database tables.



<!-- 





































 -->

 Listed directory salesman-tracker
Viewed salesman-tracker.component.ts:150-250
Viewed salesman-tracker.component.ts:251-450
Viewed salesman-tracker.component.ts:451-650
Viewed salesman-tracker.component.ts:720-800
Viewed salesman-tracker.component.ts:801-900
Viewed salesman-tracker.component.ts:901-1050
Viewed salesman-tracker.component.ts:1051-1250
Viewed salesman-tracker.component.ts:1270-1350
Viewed salesman-tracker.component.ts:1351-1397

# Complete Runtime Flow: Salesman Route Tracker & Map Routing

---

## 1. Simple Flow Explanation

1. **Route Selection & Loading:** When the salesman opens the [SalesmanTrackerComponent](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts) screen, SFA fetches the salesman's assigned routes (`ROUTE_FOR_USER`) and descriptive route names (`ROUTE_MASTER`).
2. **Parallel Outlets & Visits Fetch:** Upon selecting a route, SFA queries in parallel:
   * Today's scheduled customer shops on the route (`CUSTOMER_GEO_DETAILS_DATED`).
   * Today's actual logged visits and check-ins (`SALESMAN_CUSTOMER_VISIT_LIST`).
3. **Nearest-Neighbor Sequence Optimization:** SFA merges visited and unvisited shops. It orders visited shops chronologically by visit timestamp, and then dynamically sequences all remaining unvisited shops using a **Nearest-Neighbor Traveling Salesperson (TSP) algorithm** starting from the salesman's current GPS device location (`loadCurrentSalesmanPosition()`).
4. **Google Map & Animated Path Rendering:** SFA initializes the Google Maps JavaScript API, rendering:
   * 🔵 **Blue 'S' Pin:** Salesman's current GPS location.
   * 🟢 **Green Numbered Pins:** Visited customer stores.
   * 🟡 **Yellow Numbered Pins:** Scheduled, unvisited customer stores.
   * 🟢 **'O' Numbered Pins:** Unplanned, out-of-route visits ("Outside Bookings").
   * 🟢 **Animated Progressive Polyline:** An animated directional green path with arrow symbols linking all stops in sequence.
5. **Field Actions (Check-In, Remarks, Order Creation):**
   * Clicking a pin or list card opens an InfoWindow showing store details, visit status, and order values.
   * Salesmen can click **Remarks** to update visit notes (`updateSalesmanCustomerVisit`).
   * Clicking **Create Order** opens a modal to select the transaction type (e.g. `SO`, `INV`) and seamlessly transitions to [SalesOrderComponent](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts) pre-loaded with the selected customer.

---

## 2. ASCII Flow Diagram

```
+----------------------------------------------------------------------------------------------------+
|                                    1. INITIALIZATION & ROUTE LOADING                               |
|  Salesman navigates to /master/salesman-tracker                                                    |
|  --> ngOnInit() -> fetchRoutesForUser()                                                            |
|  --> forkJoin({ userRoutes: ROUTE_FOR_USER, routeMaster: ROUTE_MASTER })                           |
|  --> Sets default selectedRoute = routesList[0]                                                    |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                2. FETCH SHOPS & VISITS IN PARALLEL                                 |
|  fetchCustomerGeoDetails()                                                                         |
|  --> forkJoin({                                                                                    |
|        routeCustomers: POST /api/common/dynamic/landing/list (CUSTOMER_GEO_DETAILS_DATED),         |
|        visits: POST /api/common/dynamic/landing/list (SALESMAN_CUSTOMER_VISIT_LIST)                |
|      })                                                                                            |
|  --> Separates: In-Route Customers vs. Outside Bookings                                            |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                3. NEAREST-NEIGHBOR ROUTE SEQUENCING                               |
|  buildNearestVisitSequence() -> orderLocationsByNearest(locations, currentSalesmanPosition)        |
|  1. Visited stops are sorted chronologically by visitTime.                                         |
|  2. Unvisited stops are iteratively sequenced by finding the minimum Haversine distance from     |
|     the salesman's current GPS coordinate (or the last visited shop).                              |
|  3. Distance from previous stop (in Km) is computed for every stop.                                |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                4. GOOGLE MAP & MARKER RENDERING                                    |
|  applyLocationsToMap()                                                                             |
|  --> Places custom SVG pins:                                                                       |
|      - Blue 'S': Salesman current position                                                         |
|      - Green (1..N): Visited customer stops                                                        |
|      - Yellow (1..N): Unvisited pending stops                                                      |
|      - Green 'O1..ON': Outside booking visits                                                      |
|  --> Progressive Polyline Animation: Draws animated green path with arrow symbols                  |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                     5. FIELD SALES ACTIONS                                         |
|  - View Store Info Window: Clicking map marker or left card highlights stop & pans camera          |
|  - Save Remarks: Opens modal -> PortalService.updateSalesmanCustomerVisit(visitId, { flexField06 })|
|  - Create Order: Opens modal -> Selects Order Type (SO/INV) -> Navigates to /sales-order           |
+----------------------------------------------------------------------------------------------------+
```

---

## 3. Step-by-Step Runtime Flow Trace

```
User Action
  ↓
HTML (Template)
  ↓
Component
  ↓
Method
  ↓
Service
  ↓
API Endpoint
  ↓
Request Payload
  ↓
Backend Response
  ↓
State / Data Update
  ↓
HTML (Template View)
  ↓
UI Feedback
  ↓
Navigation
```

---

### Step 1: Screen Entry & Route Master Loading

* **Exact File Path:** [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts#L234-L324)
* **Class Name:** `SalesmanTrackerComponent`
* **Method Name:** `ngOnInit()` $\rightarrow$ `fetchRoutesForUser()`
* **What Happens:**
  1. Component initializes `this.selectedDate = this.getTodayDate()`.
  2. Reads user session from `localStorage.getItem('SOLIDS_DATA')`.
  3. Executes `fetchRoutesForUser()` which runs a `forkJoin` requesting the user's assigned routes and generic master route names.
* **Why It Happens:** Salesmen may be assigned to multiple routes; the system must load their route dropdown before querying specific customer outlets.
* **Service:** [PortalService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts)
* **API Endpoints:**
  1. `POST {SOLIDS_MASTER_URL}/api/common/dynamic/landing/list` (`subcatgid: 'ROUTE_FOR_USER'`)
  2. `GET {SOLIDS_MASTER_URL}/api/common/active/genericMaster/list?catgId=ROUTE_MASTER`
* **Request Payload (ROUTE_FOR_USER):**
  ```json
  {
    "ascordesc": "",
    "filter": {
      "USERID": "USR001"
    },
    "fromno": "",
    "orderbycol": "",
    "search": "",
    "subcatgid": "ROUTE_FOR_USER",
    "tono": ""
  }
  ```
* **State / Data Update:**
  * `this.routesList = this.applyRouteNames(normalizedRoutes)`
  * `this.selectedRoute = this.routesList[0]`
* **What It Calls Next:** `this.fetchCustomerGeoDetails()`.

---

### Step 2: Fetch Scheduled Route Outlets & Actual Visits in Parallel

* **Exact File Path:** [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts#L393-L500)
* **Class Name:** `SalesmanTrackerComponent`
* **Method Name:** `fetchCustomerGeoDetails()`
* **What Happens:**
  * SFA constructs two parallel requests using `forkJoin`:
    1. **`routeCustomerReq` (`CUSTOMER_GEO_DETAILS_DATED`):** Queries all scheduled shops on the selected route for today's date.
    2. **`visitReq` (`SALESMAN_CUSTOMER_VISIT_LIST`):** Queries all visits logged by this salesman for today's date.
* **API Endpoints:**
  * `POST {SOLIDS_MASTER_URL}/api/common/dynamic/landing/list`
* **Request Payloads:**
  ```json
  // Request 1: Route Customers
  {
    "subcatgid": "CUSTOMER_GEO_DETAILS_DATED",
    "filter": {
      "ROUTECODE": "ROUTE-01",
      "COMPCODE": "COMP01",
      "DATED": "2026-09-02"
    }
  }

  // Request 2: Logged Visits
  {
    "subcatgid": "SALESMAN_CUSTOMER_VISIT_LIST",
    "filter": {
      "COMPCODE": "COMP01",
      "SALESMANCODE": "USR001",
      "VISITDATE": "2026-09-02"
    }
  }
  ```
* **Backend Responses:**
  * `routeCustomers`: Array of customer stores containing `customerCode`, `customerName`, `latitude`, `longitude`, `address`.
  * `visits`: Array of logged visits containing `salesmanLatitude`, `salesmanLongitude`, `visitTime`, `flexField03` (`Order` / `No Order`), `flexField04` (Order Value), `flexField06` (Remarks).
* **Data Transformation:**
  1. `mergeCustomerVisits()`: Merges visit timestamps, order values, and remarks into matching customer objects.
  2. Outside visits detection: Any logged visit whose `customerCode` is not in the assigned route customer list is pushed to `this.outsideBookings`.
* **What It Calls Next:** `this.buildNearestVisitSequence()`.

---

### Step 3: Nearest-Neighbor TSP Sequencing Algorithm

* **Exact File Path:** [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts#L751-L836)
* **Class Name:** `SalesmanTrackerComponent`
* **Method Name:** `orderLocationsByNearest(locations, startPoint)`
* **Algorithm Execution:**
  1. **Partitioning:** Separates stops into `visited` and `unvisited`.
  2. **Visited Sorting:** Sorts visited shops ascending by `visitTime` (`visited.sort(...)`).
  3. **Visited Chain:** Appends all visited shops to `ordered[]`, assigning sequential stop numbers `sequenceNo: 1..K` and calculating distance from the previous visited shop.
  4. **Unvisited Proximity Loop:**
     * Sets `currentPoint = last visited stop` (or `currentSalesmanPosition` if no shop has been visited yet).
     * While `remainingUnvisited.length > 0`:
       * Iterates through remaining unvisited stops and computes the Haversine distance `calculateDistanceKm()` from `currentPoint`.
       * Identifies the closest stop with minimum distance (`nearestDistance`).
       * Removes (`splice`) the nearest stop from `remainingUnvisited` and appends it to `ordered[]`.
       * Updates `currentPoint = nextLocation`.
* **Haversine Math:**
  $$d = 2R \cdot \arcsin\left(\sqrt{\sin^2\left(\frac{\Delta\text{lat}}{2}\right) + \cos(\text{lat}_1)\cos(\text{lat}_2)\sin^2\left(\frac{\Delta\text{lon}}{2}\right)}\right)$$
  *(where $R = 6371\text{ km}$)*
* **State / Data Update:** Sets `this.customerLocations = ordered`.
* **What It Calls Next:** `this.applyLocationsToMap()`.

---

### Step 4: Device GPS Position Acquisition

* **Exact File Path:** [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts#L588-L616)
* **Class Name:** `SalesmanTrackerComponent`
* **Method Name:** `loadCurrentSalesmanPosition()`
* **What Happens:**
  * Calls `navigator.geolocation.getCurrentPosition({ enableHighAccuracy: true, timeout: 10000 })`.
  * Sets `this.currentSalesmanPosition = { latitude: pos.coords.latitude, longitude: pos.coords.longitude }`.
  * Re-evaluates nearest unvisited sequencing starting from the newly acquired live GPS coordinate.

---

### Step 5: Google Maps Initialization & Custom Marker Placement

* **Exact File Path:** [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts#L926-L1056)
* **Class Name:** `SalesmanTrackerComponent`
* **Method Name:** `applyLocationsToMap()`
* **Execution Details:**
  1. Runs inside `ngZone.runOutsideAngular()` for performance optimization.
  2. Clears existing map markers (`this.clearMap()`).
  3. **Salesman Marker:** If `this.currentSalesmanPosition` exists, creates a Blue pin with label `'S'`.
  4. **Customer Outlets Markers:** Iterates through `this.customerLocations`:
     * If `customer.visited === true` $\rightarrow$ Places **Green Marker** with number `(index + 1)`.
     * If `customer.visited === false` $\rightarrow$ Places **Yellow Marker** with number `(index + 1)`.
  5. **Outside Booking Markers:** Iterates through `this.outsideBookings`:
     * Places **Green Marker** with label `'O1'`, `'O2'`, etc.
  6. Attaches click listener `addMapMarkerClick()` to open the store InfoWindow upon user tap.
  7. Calls `this.googleMap.fitBounds(bounds)` to automatically zoom and center the map view to encompass all route pins.

---

### Step 6: Progressive Animated Polyline Path Drawing

* **Exact File Path:** [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts#L1059-L1128)
* **Class Name:** `SalesmanTrackerComponent`
* **Method Name:** `applyLocationsToMap()` (Polyline section)
* **What Happens:**
  1. Instantiates `google.maps.Polyline` with `strokeColor: '#007b04ff'`, `strokeWeight: 6`, and `SymbolPath.FORWARD_CLOSED_ARROW` icons repeated every 100px.
  2. Generates 30 interpolated sub-coordinates per route segment (`totalStepsPerSegment = 30`).
  3. Runs an animation interval (`setInterval` every 30ms) pushing coordinates progressively into `animatedPath` (`google.maps.MVCArray`).
* **UI Visual:** The salesman sees a live animated green path drawing dynamically from stop to stop across the map with directional navigation arrows.

---

### Step 7: Customer Card Click & InfoWindow Interaction

* **Exact File Path:** [src/app/masters/salesman-tracker/salesman-tracker.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.html#L120-L240) and [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts#L1193-L1209)
* **Class Name:** `SalesmanTrackerComponent`
* **Method Name:** `onCustomerClick(customer, index)` $\rightarrow$ `openMarkerInfoWindow(marker)`
* **User Action:** Salesman clicks a customer card in the left route sidebar or clicks a map pin.
* **What Happens:**
  1. Sets `this.activeCustomer = customer`.
  2. Pans Google Map camera to the marker coordinates: `this.googleMap.panTo(marker.getPosition())`.
  3. Opens `google.maps.InfoWindow` displaying:
     * Customer name and address.
     * Distance from previous stop in Kilometers.
     * Visit status badge: `"Visited - Order Placed"`, `"Visited - No Order"`, or `"Pending Visit"`.
     * Booked order value (if order was booked) or reason code (e.g. `"Shop Closed"`).
     * Action buttons: **Create Order**, **Remarks**, **View Order**.

---

### Step 8: Update Customer Visit Remarks API

* **Exact File Path:** [src/app/masters/salesman-tracker/salesman-tracker.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.html#L315-L350) and [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts#L160-L190)
* **Class Name:** `SalesmanTrackerComponent`
* **Method Name:** `saveRemarks()`
* **User Action:** Salesman opens the Remarks modal on a visited shop, edits notes, and clicks **Save**.
* **Service:** [PortalService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts) $\rightarrow$ `updateSalesmanCustomerVisit()`
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/salesman/customer/visit/update?id={remarksVisitId}`
* **Request Payload:**
  ```json
  {
    "flexField06": "Shop owner requested follow-up on Thursday morning regarding new product catalog."
  }
  ```
* **Backend Response:**
  ```json
  {
    "status": "SUCCESS",
    "message": "Visit updated successfully"
  }
  ```
* **State / Data Update:** Updates `this.selectedCustomerForRemarks.flexField06` and closes the modal.
* **UI Feedback:** Toast success: *"Remarks updated successfully."*.

---

### Step 9: Launch Order Creation & Transition to Sales Order

* **Exact File Path:** [src/app/masters/salesman-tracker/salesman-tracker.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.html#L355-L410) and [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts#L209-L232)
* **Class Name:** `SalesmanTrackerComponent`
* **Method Name:** `submitCreateOrder()`
* **User Action:** Salesman clicks **Create Order** on a customer card, selects transaction type (e.g. `Sales Order - SO`), and clicks **Proceed**.
* **Execution & Navigation:**
  1. Stores active customer in browser storage:
     ```typescript
     localStorage.setItem('currentCustomer', JSON.stringify(this.selectedCustomerForOrder));
     localStorage.setItem('trackerSelectedRoute', this.getRouteCode(this.selectedRoute));
     ```
  2. Angular Router navigates to the Sales Order creation screen:
     ```typescript
     this.router.navigate(['/sales-order'], {
       queryParams: {
         order_type: this.selectedTxnType.order_type,
         erp_txn_code: this.selectedTxnType.erp_txn_code,
         erp_txn_name: this.selectedTxnType.erp_txn_name
       }
     });
     ```
* **Result:** [SalesOrderComponent](file:///d:/Ramesh/SH_FE/sh_fe/src/app/sales-order/sales-order.component.ts) opens pre-hydrated with the selected customer and route.

---

## 4. Important Technical Concepts

### 4.1 Angular Concepts
* **NgZone Run Outside Angular (`runOutsideAngular`):**
  * Drawing 30+ marker overlays and running progressive polyline interval timers at 30ms causes massive change detection overhead. Executing inside `this.ngZone.runOutsideAngular(() => ...)` completely isolates high-frequency animation loops from Angular's digest cycle.
* **HostListener Document Clicks:**
  * `@HostListener('document:click') closeRouteDropdown()` automatically dismisses custom route selection dropdowns when clicking anywhere outside.
* **Template Element References (`#mapContainer`, `#infoWindowContent`):**
  * `@ViewChild('mapContainer')` attaches Google Maps directly into the Angular DOM element reference; `@ViewChild('infoWindowContent')` provides the HTML snippet injected into `google.maps.InfoWindow`.

### 4.2 TypeScript & Algorithms
* **Greedy Nearest-Neighbor TSP Algorithm:**
  * Computes $O(N^2)$ distance comparisons to find the shortest continuous path across all retail outlets starting from the current GPS location.
* **SVG Data URI Generator:**
  * Dynamically generates animated radar ping SVG markers with customized stroke, gradient fills, and stop numbers without relying on external image asset files:
    ```typescript
    getCustomMarkerIcon(colorType: 'green' | 'yellow' | 'blue'): { url: string }
    ```

### 4.3 RxJS Concepts
* **forkJoin for Coordinated Multi-Source Queries:**
  * Executes `routeCustomers` and `visits` queries simultaneously, completing only when both database queries resolve to construct the unified route model.
* **of() Operator for Cached Observables:**
  * `const masterObs$ = this.routeMasterList.length > 0 ? of(this.routeMasterList) : this.portalService.getActiveGeneric(...)` — Prevents redundant HTTP calls if master route names are already in memory.

---

## 5. Important Files

| File Path | Role / Purpose |
| :--- | :--- |
| [src/app/masters/salesman-tracker/salesman-tracker.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.ts) | Primary route tracker component controller; executes route loading, nearest-neighbor sequencing, marker rendering, and visit updates. |
| [src/app/masters/salesman-tracker/salesman-tracker.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.html) | UI template containing route selectors, summary statistics bar, customer stops sidebar, map viewport, and action modals. |
| [src/app/masters/salesman-tracker/salesman-tracker.component.scss](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-tracker/salesman-tracker.component.scss) | Styling for responsive split-screen view, pin pulse animations, and mobile card layouts. |
| [src/app/service/portal.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts) | Backend service handling queries for `ROUTE_FOR_USER`, `CUSTOMER_GEO_DETAILS_DATED`, `SALESMAN_CUSTOMER_VISIT_LIST`, and visit updates. |
| [src/app/service/geocoding.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/geocoding.service.ts) | Geolocation service loading Google Maps JavaScript library modules (`maps`, `marker`, `core`). |

---

## 6. Important Methods

| Class | Method | Purpose |
| :--- | :--- | :--- |
| `SalesmanTrackerComponent` | `fetchRoutesForUser()` | Loads routes assigned to the salesman and resolves descriptive route names. |
| `SalesmanTrackerComponent` | `fetchCustomerGeoDetails()` | Retrieves scheduled shops and logged visits in parallel via `forkJoin`. |
| `SalesmanTrackerComponent` | `orderLocationsByNearest(locations, startPoint)` | Executes Nearest-Neighbor TSP optimization to sequence pending stops. |
| `SalesmanTrackerComponent` | `applyLocationsToMap()` | Places color-coded SVG markers, fits bounds, and draws animated path polylines. |
| `SalesmanTrackerComponent` | `saveRemarks()` | Submits updated visit notes and salesman feedback to the database. |
| `SalesmanTrackerComponent` | `submitCreateOrder()` | Stores selected customer in `localStorage` and navigates to the Sales Order screen. |
| `SalesmanTrackerComponent` | `calculateDistanceKm(lat1, lon1, lat2, lon2)` | Computes Haversine great-circle distance in kilometers between two GPS coordinates. |
| `PortalService` | `updateSalesmanCustomerVisit(id, payload)` | Updates visit record fields (`flexField06` remarks) in the database. |

---

## 7. API List Table

| Method | HTTP | Endpoint | Request Payload | Response | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `fetchRoutesForUser` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'ROUTE_FOR_USER', filter: { USERID } }` | List of route codes assigned to salesman | Retrieve salesman's daily assigned routes. |
| `routeMaster` | `GET` | `/api/common/active/genericMaster/list?catgId=ROUTE_MASTER` | None | Generic master list (`value`, `valueDesc`) | Resolve route display names. |
| `routeCustomers` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'CUSTOMER_GEO_DETAILS_DATED', filter: { ROUTECODE, COMPCODE, DATED } }` | Array of customer outlets with GPS coordinates | Fetch scheduled shops for selected route & date. |
| `visitList` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'SALESMAN_CUSTOMER_VISIT_LIST', filter: { COMPCODE, SALESMANCODE, VISITDATE } }` | Array of visit logs, check-in times, order amounts | Fetch actual visits made today. |
| `saveRemarks` | `POST` | `/api/salesman/customer/visit/update?id={id}` | `{ flexField06: "..." }` | Success response | Save visit remarks / notes entered by salesman. |
| `getAccessibleTransList` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_ACCESS_TXN', filter: { TXNCODE: 'ALL' } }` | List of allowed transaction codes | Populate "Create Order" transaction dropdown modal. |

---

## 8. Business Rules Summary

1. **Route Partitioning & Sorting:**
   * Visited customer stops are strictly sorted by actual chronological check-in timestamp (`visitTime`).
   * Unvisited customer stops are dynamically ordered by geographic proximity to the salesman's current live GPS position.
2. **Outside Bookings Isolation:**
   * Any visit logged for a customer not present in the assigned route's scheduled list is categorized as an "Outside Booking" (`outsideBookings`) and marked with an `'O'` pin prefix.
3. **Color-Coded Status Pins:**
   * 🟢 **Green Pin:** Visited customer (regardless of whether an order was booked or no-order check-in).
   * 🟡 **Yellow Pin:** Planned customer outlet that has not yet been visited today.
   * 🔵 **Blue Pin:** Salesman's current GPS location.
4. **Summary Metrics Calculation:**
   * Route metrics dynamically calculate: Visited Count, Planned Count, Route Orders Value, Outside Orders Value, and Total Combined Orders Value.
5. **Session Pre-Hydration:**
   * Launching an order from the tracker stores `trackerSelectedRoute` and `currentCustomer` in `localStorage` to ensure the Sales Order form automatically binds to the chosen store.

---

## 9. Concepts You Need to Learn

1. **Traveling Salesperson Problem (TSP) Nearest-Neighbor Heuristic:** How greedy distance evaluation creates efficient daily driving routes for field reps.
2. **Google Maps SDK `MVCArray` & Dynamic Polylines:** How `google.maps.MVCArray` enables real-time coordinate streaming for progressive route animations.
3. **Data URI SVGs in Google Maps Markers:** Embedding scalable XML vector strings directly into `google.maps.Marker.icon.url` for lightweight custom pins.
4. **`NgZone.runOutsideAngular` Performance Optimization:** Bypassing Angular's zone change detection during high-frequency map animation timers.

---

## 10. Things That Are Still Unclear (Unknowns / Server-Side Logic)

1. **Beat Filtering in `CUSTOMER_GEO_DETAILS_DATED`:** The backend stored procedure behind `CUSTOMER_GEO_DETAILS_DATED` evaluates beat calendar rules on the database server before returning the active customer rows.
2. **GPS Accuracy Thresholds in Visit Creation:** Whether the server rejects visits submitted with poor GPS accuracy ($>100\text{m}$) or flags them internally in supervisor audit views.
3. **Dynamic Polyline Turn-by-Turn Directions:** The current polyline draws straight-line geodesic segments between stops; whether Google Directions API (road network routing) is enabled depends on the client's Google Cloud billing quotas.




<!-- 








































 -->







 Listed directory salesman-performance-dashboard
Viewed salesman-performance-dashboard.component.ts:1-150
Viewed salesman-performance-dashboard.component.ts:151-350
Viewed salesman-performance-dashboard.component.ts:351-550
Viewed salesman-performance-dashboard.component.ts:551-750
Viewed salesman-performance-dashboard.component.ts:751-850
Viewed salesman-performance-dashboard.component.ts:870-950
Viewed salesman-performance-dashboard.component.ts:980-1050
Viewed salesman-performance-dashboard.component.ts:1050-1100
Viewed salesman-performance-dashboard.component.ts:1101-1200
Viewed salesman-performance-dashboard.component.ts:1201-1320
Viewed salesman-performance-dashboard.component.ts:1321-1420
Viewed salesman-performance-dashboard.component.html:1-120

# Complete Runtime Flow: Salesman Performance Dashboard & Analytics

---

## 1. Simple Flow Explanation

1. **Manager & Role Discovery:** When a supervisor opens [SalesmanPerformanceDashboardComponent](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts), SFA checks the user's role (`SOLIDS_ROLEID`, `ALQ_ACCESS`). If the user is a Sales Manager, the manager dropdown locks to their profile; if an Admin/Superuser (`ALQ_ACCESS === 'ALL'`), it unlocks the ability to select any manager or choose `ALL`.
2. **Cascading Dropdowns (Manager $\rightarrow$ Salesmen):** Selecting a Manager queries all assigned sales reps via `GEO_DSH_SALMAN_LIST`. SFA auto-selects all salesmen by default and computes `kpis.totalSalesmen`.
3. **Multi-Source Parallel Aggregation (`forkJoin`):** Clicking **Search** dispatches 6 parallel SQL aggregation queries:
   * **`GEO_SALES_PERF_1`:** Total planned store visits across the selected date range.
   * **`GEO_SALES_PERF_2` (Order):** Total orders placed, partitioned into inside-geofence vs. outside-geofence.
   * **`GEO_SALES_PERF_2` (No Order):** Productive vs. non-productive check-ins (e.g. Shop Closed).
   * **`GEO_SALES_PERF_3`:** Route-level breakdown table (Outlets, Inside Visits, Outside Orders, Total Value).
   * **`GEO_SALES_PERF_4`:** Individual Salesman performance summary grid.
   * **`GEO_SALES_PERF_6`:** Customer-level performance ranking grid.
4. **Interactive Drilldown Modals (Timeline & Customer Detail):**
   * Double-clicking a salesman row opens a chronological visit timeline modal (`GEO_SALES_PERF_5`) showing exact check-in times, geofence compliance (`INSIDE` vs `OUTSIDE`), customer codes, order types, and booked revenue.
   * Clicking a customer opens a customer stop history modal (`GEO_SALES_PERF_7`).
   * Clicking an order reference directly navigates to the read-only ERP document view `/sales-order/view/:voucherId`.
5. **Multi-Format Reporting:** Generates Excel spreadsheets (`XXlList`) for the overall dashboard, route grid, salesman grid, or individual salesman timelines, as well as binary PDF report downloads (`downloadPDFReport`).

---

## 2. ASCII Flow Diagram

```
+----------------------------------------------------------------------------------------------------+
|                                    1. FILTERS & ROLE RESOLUTION                                    |
|  ngOnInit() -> loadSalesManagers() [GEO_DSH_MNG_LIST]                                              |
|  Manager Selected -> onManagerChange() -> loadSalesmen [GEO_DSH_SALMAN_LIST]                       |
|  Date Range Selected -> [fromDate, toDate]                                                        |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                2. PARALLEL AGGREGATION PIPELINE                                    |
|  User Clicks 'Search' -> searchDashboard()                                                         |
|  --> forkJoin({                                                                                    |
|        planned:      POST /api/common/dynamic/landing/list [GEO_SALES_PERF_1],                     |
|        orderData:    POST /api/common/dynamic/landing/list [GEO_SALES_PERF_2 (Order)],             |
|        noOrderData:  POST /api/common/dynamic/landing/list [GEO_SALES_PERF_2 (No Order)],          |
|        gridData:     POST /api/common/dynamic/landing/list [GEO_SALES_PERF_3 (Route Breakdown)],   |
|        salesmanPerf: POST /api/common/dynamic/landing/list [GEO_SALES_PERF_4 (Salesman Summary)],  |
|        customerPerf: POST /api/common/dynamic/landing/list [GEO_SALES_PERF_6 (Customer Summary)]   |
|      })                                                                                            |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                  3. KPI CARDS & DATA GRIDS RENDERING                               |
|  - Top KPI Metrics: Planned Visits, Visited Count, Total Orders Placed, Total Booked Revenue      |
|  - Card 3 & 4: Inside-Geofence vs Outside-Geofence Order Counts and Revenue                        |
|  - Table 1: Route Performance Grid (Outlets, In-Visits, Out-Visits, No-Orders, Value)              |
|  - Table 2: Salesman Performance Summary Grid (Scheduled, Visited %, Order Value)                 |
|  - Table 3: Customer Performance Summary Grid (Order Count, No Order Count, Total Value)          |
+----------------------------------------------------------------------------------------------------+
                                                  |
                         +------------------------+------------------------+
                         |                                                 |
                         v                                                 v
+--------------------------------------------------+  +---------------------------------------------+
|        4. TIMELINE DRILLDOWN MODAL               |  |           5. EXPORT & ERP ROUTING           |
| User clicks Salesman Row:                        |  | User clicks Excel / PDF Button:             |
| -> openSalesmanDetails(row)                      |  | -> PortalService.XXlList(reqObj)            |
| -> Query GEO_SALES_PERF_5                        |  | -> Streams application/vnd.openxml blob    |
| -> Renders chronological stop-by-stop audit      |  | User clicks Order Reference link:           |
|    timeline with check-in time & INSIDE/OUTSIDE  |  | -> router.navigate(['/sales-order/view',    |
|                                                  |  |                     voucherId])             |
+--------------------------------------------------+  +---------------------------------------------+
```

---

## 3. Step-by-Step Runtime Flow Trace

```
User Action
  ↓
HTML (Template)
  ↓
Component
  ↓
Method
  ↓
Service
  ↓
API Endpoint
  ↓
Request Payload
  ↓
Backend Response
  ↓
State / Data Update
  ↓
HTML (Template View)
  ↓
UI Feedback
  ↓
Navigation
```

---

### Step 1: Component Initialization & Manager List Fetch

* **Exact File Path:** [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts#L186-L239)
* **Class Name:** `SalesmanPerformanceDashboardComponent`
* **Method Name:** `ngOnInit()` $\rightarrow$ `loadSalesManagers()`
* **What Happens:**
  1. Component parses `localStorage.getItem('SOLIDS_DATA')` and `localStorage.getItem('ALQ_ACCESS')`.
  2. Sets default date range to today's UTC start and end: `[todayUTC, todayUTC]`.
  3. Dispatches `loadSalesManagers()` requesting the company's sales manager list.
* **Why It Happens:** Populates the supervisor dropdown and locks the selector if the logged-in user is a field manager.
* **Service:** [PortalService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts)
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/common/dynamic/landing/list`
* **Request Payload:**
  ```json
  {
    "ascordesc": "DESC",
    "filter": {
      "COMPCODE": "COMP01"
    },
    "fromno": "",
    "orderbycol": "",
    "search": "",
    "subcatgid": "GEO_DSH_MNG_LIST",
    "tono": ""
  }
  ```
* **State / Data Update:**
  * If `ALQ_ACCESS === 'ALL'`, prepends `{ code: 'ALL', name: 'ALL' }` to `managerList`.
  * Invokes `checkUserRoleAndSetDefault()`: sets `selectedManager` and triggers `onManagerChange()`.

---

### Step 2: Cascading Salesmen Dropdown Fetch

* **Exact File Path:** [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts#L352-L421)
* **Class Name:** `SalesmanPerformanceDashboardComponent`
* **Method Name:** `onManagerChange()`
* **What Happens:**
  1. Clears downstream salesmen and beats arrays.
  2. Resolves selected manager role IDs (`ROLEIDS: "MGR-01|MGR-02"` or single role ID).
  3. Queries `GEO_DSH_SALMAN_LIST` to retrieve all salesmen reporting to the selected manager(s).
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/common/dynamic/landing/list`
* **Request Payload:**
  ```json
  {
    "ascordesc": "DESC",
    "filter": {
      "ROLEIDS": "MGR-01"
    },
    "subcatgid": "GEO_DSH_SALMAN_LIST"
  }
  ```
* **State / Data Update:**
  * `this.salesmenList = res.map(s => ({ code: s.code, name: s.name }))`
  * `this.selectedSalesmen = [...this.salesmenList]` (auto-selects all by default).
  * Automatically invokes `this.searchDashboard()`.

---

### Step 3: Dispatching 6-Way Parallel Aggregation Queries

* **Exact File Path:** [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.html#L98-L101) and [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts#L445-L580)
* **Class Name:** `SalesmanPerformanceDashboardComponent`
* **Method Name:** `searchDashboard()`
* **User Action:** Manager selects date range, tweaks salesmen multiselect, and clicks `<button (click)="searchDashboard()"> Search </button>`.
* **Execution:** SFA formats date range into `yyyy-MM-dd` UTC strings, formats SQL `IN ('...')` strings for `ROLEID` and `USERID`, and triggers `forkJoin`:
  1. **`req1` (`GEO_SALES_PERF_1`):** Total planned visits.
  2. **`req2_order` (`GEO_SALES_PERF_2`):** Total orders (`FLEXFIELD3: "Order"`).
  3. **`req2_noorder` (`GEO_SALES_PERF_2`):** Total no-orders (`FLEXFIELD3: "No Order"`).
  4. **`req3` (`GEO_SALES_PERF_3`):** Route performance grid.
  5. **`req4` (`GEO_SALES_PERF_4`):** Salesman summary grid.
  6. **`req6` (`GEO_SALES_PERF_6`):** Customer summary grid.
* **API Endpoints:** `POST {SOLIDS_MASTER_URL}/api/common/dynamic/landing/list`

---

### Step 4: KPI Aggregation & Geofence Metric Calculation

* **Exact File Path:** [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts#L581-L675)
* **Class Name:** `SalesmanPerformanceDashboardComponent`
* **Method Name:** `searchDashboard()` (Response subscriber)
* **Backend Responses & Math:**
  1. `plannedCount = Number(res.planned[0].totalVisitCount || 0)`
  2. `insideOrderCount = Number(res.orderData[0].insideCount || 0)`
  3. `outsideOrderCount = Number(res.orderData[0].outsideCount || 0)`
  4. `insideOrderValue = Number(res.orderData[0].insideOrderValue || 0)`
  5. `outsideOrderValue = Number(res.orderData[0].outsideOrderValue || 0)`
  6. `visitedCount = insideOrderCount + outsideOrderCount + insideNoOrderCount + outsideNoOrderCount`
* **State / Data Update:**
  ```typescript
  this.kpis = {
    totalSalesmen: this.selectedSalesmen.length,
    plannedVisits: plannedCount,
    visitedCount: visitedCount,
    visitedPercentage: plannedCount > 0 ? (visitedCount / plannedCount) * 100 : 0,
    totalOrders: insideOrderCount + outsideOrderCount,
    orderValue: insideOrderValue + outsideOrderValue,
    formattedOrderValue: this.changeAmtBcDeci2(insideOrderValue + outsideOrderValue)
  };
  ```
* **UI Update:** The top 4 KPI metric cards and Card 3/4 geofence breakdown boxes immediately update on screen.

---

### Step 5: Data Grids Population (Route, Salesman, Customer)

* **Exact File Path:** [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.html#L200-L380) and [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts#L636-L675)
* **Class Name:** `SalesmanPerformanceDashboardComponent`
* **What Happens:**
  1. **Route Table (`routePerformanceData`):** Binds columns `route`, `outlets`, `insiteVisit`, `insideOrders`, `outsideVisit`, `outsideOrders`, `noOrders`, `value`.
  2. **Salesman Table (`salesmanPerformanceData`):** Binds columns `salesmanName`, `managerName`, `schedule`, `planned`, `visited`, `orders`, `total`.
  3. **Customer Table (`customerPerformanceData`):** Binds columns `customerCode`, `customerName`, `totalOrderCount`, `noOrderCount`, `totalOrderValue`.
* **UI Feedback:** All 3 tables support live client-side filtering via search input boxes (`routeSearchText`, `salesmanSearchText`, `customerSearchText`).

---

### Step 6: Salesman Timeline Drilldown Modal

* **Exact File Path:** [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.html#L400-L520) and [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts#L684-L738)
* **Class Name:** `SalesmanPerformanceDashboardComponent`
* **Method Name:** `openSalesmanDetails(row)`
* **User Action:** Manager clicks or double-clicks a salesman row in Table 2.
* **What Happens:**
  1. Sets `this.showDetailsModal = true` and `this.detailsLoadingYN = true`.
  2. Dispatches `GEO_SALES_PERF_5` query filtered by `USERID: salesmanCode` and selected dates.
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/common/dynamic/landing/list`
* **Request Payload:**
  ```json
  {
    "subcatgid": "GEO_SALES_PERF_5",
    "filter": {
      "FROMDATE": "2026-09-01",
      "TODATE": "2026-09-02",
      "USERID": "USR001",
      "COMPCODE": "COMP01"
    }
  }
  ```
* **Backend Response:**
  ```json
  [
    {
      "routeCode": "ROUTE-01",
      "customerCode": "CUST-1002",
      "customerName": "Al Maya Supermarket",
      "visitTime": "10:15:30",
      "inout": "INSIDE",
      "visitStatus": "Order Placed",
      "flexField01": "SO",
      "flexField02": "SON2026-00088",
      "flexField04": 472.5,
      "flexField06": "Follow up next week",
      "flexField07": "SO,Sale Order,114088"
    }
  ]
  ```
* **UI Modal:** Displays a chronological audit list with badges for `INSIDE` (green) vs `OUTSIDE` (red), order reference links, and booked revenue amounts.

---

### Step 7: ERP Document Navigation from Timeline

* **Exact File Path:** [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts#L1401-L1417)
* **Class Name:** `SalesmanPerformanceDashboardComponent`
* **Method Name:** `navigateToOrderView(customer)`
* **User Action:** Manager clicks the blue document reference link `#SON2026-00088` inside the timeline modal.
* **Execution & Navigation:**
  1. Parses `flexField07: "SO,Sale Order,114088"` into `erpTxnCode`, `erpTxnName`, and voucher ID `agId`.
  2. Navigates directly to the finalized ERP sales order screen:
     ```typescript
     this.router.navigate(['/sales-order/view', '114088'], {
       queryParams: {
         order_type: 'SO',
         erp_txn_code: 'SO',
         erp_txn_name: 'Sale Order'
       }
     });
     ```

---

### Step 8: Multi-Table Excel Export Streaming

* **Exact File Path:** [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts#L749-L840)
* **Class Name:** `SalesmanPerformanceDashboardComponent`
* **Method Name:** `exportRoutePerformanceExcel()` / `exportSalesmanPerformanceExcel()`
* **User Action:** Manager clicks the green **Excel** export button on any grid header.
* **Service:** [PortalService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts) $\rightarrow$ `XXlList(reqObj)`
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/common/dynamic/landing/list`
* **Request Payload (`GEO_SALES_PERF_XL_3`):**
  ```json
  {
    "subcatgid": "GEO_SALES_PERF_XL_3",
    "excelName": "GEO_SALES_PERF_XL_3_5",
    "filter": {
      "FROMDATE": "2026-09-01",
      "TODATE": "2026-09-02",
      "ROLEID": "MGR-01",
      "USERID": "USR001",
      "COMPCODE": "COMP01"
    }
  }
  ```
* **Execution:**
  1. Receives raw binary byte array (`responseType: 'blob'`).
  2. Instantiates `Blob` with MIME `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`.
  3. Creates temporary download anchor `<a>` element, triggers browser download `Route_Performance_2026-09-01_to_2026-09-02.xlsx`, and immediately revokes the object URL via `window.URL.revokeObjectURL(url)`.
* **UI Feedback:** Toast success: *"Excel downloaded successfully."*.

---

## 4. Important Technical Concepts

### 4.1 Angular Concepts
* **Reactive Filter Getters (Template Optimization):**
  * Instead of evaluating filtering logic inside template loops, the component exposes pure TypeScript getters (`filteredRoutePerformanceData`, `filteredSalesmanPerformanceData`, `filteredDetailsData`). This prevents change detection recalculations during scroll and mouse events.
* **Bootstrap DateRangePicker Integration (`bsDaterangepicker`):**
  * Two-way model binding `[(ngModel)]="selectedDateRange"` with UTC timezone safety prevents day-shift bugs across different timezones.
* **NgMultiSelectDropdown (`ng-multiselect-dropdown`):**
  * Handles select-all, item search, and selection limit tags with customized callback handlers (`onSalesmanSelect()`).

### 4.2 TypeScript & RxJS
* **takeUntil Component Lifecycle Management:**
  * Uses `private destroy$ = new Subject<void>()` and `.pipe(takeUntil(this.destroy$))` on every HTTP observable. When the manager navigates away, `ngOnDestroy()` completes `destroy$`, aborting all 6 pending aggregation requests instantly.
* **forkJoin Parallel Multi-Query Synchronization:**
  * Coordinates 6 distinct API requests into a single reactive execution block, preventing UI flickering and state inconsistency.

### 4.3 Data & Number Formatting
* **Decimal & Currency Localization:**
  * `changeAmtBcDeci2()` wraps `toLocaleString('en-IN', { minimumFractionDigits: 2 })` to render standardized formatted Indian/Gulf numbering systems (e.g. `1,25,450.00`).
* **Binary File Stream Download:**
  * Creating dynamic `Blob` URLs and simulating programmatic `.click()` events allows secure in-memory file downloads without exposing public server file URLs.

---

## 5. Important Files

| File Path | Role / Purpose |
| :--- | :--- |
| [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.ts) | Core component controller; coordinates manager filters, 6-way `forkJoin` aggregation, KPI calculations, and Excel exports. |
| [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.html](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.html) | UI template with date pickers, multiselect dropdowns, 4 top KPI cards, 3 data tables, and 2 drilldown modals. |
| [src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.scss](file:///d:/Ramesh/SH_FE/sh_fe/src/app/masters/salesman-performance-dashboard/salesman-performance-dashboard.component.scss) | Visual styling for KPI metric gradients, table hover states, modal overlays, and status badges. |
| [src/app/service/portal.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts) | Backend service managing queries for `GEO_DSH_MNG_LIST`, `GEO_SALES_PERF_1..8`, and Excel binary downloads (`XXlList`). |

---

## 6. Important Methods

| Class | Method | Purpose |
| :--- | :--- | :--- |
| `SalesmanPerformanceDashboardComponent` | `loadSalesManagers()` | Fetches list of sales managers and enforces role-based lock. |
| `SalesmanPerformanceDashboardComponent` | `onManagerChange()` | Fetches salesmen reporting to selected manager(s) and auto-selects all. |
| `SalesmanPerformanceDashboardComponent` | `searchDashboard()` | Dispatches 6-way `forkJoin` aggregation and computes executive KPIs. |
| `SalesmanPerformanceDashboardComponent` | `openSalesmanDetails(row)` | Opens salesman visit timeline modal and queries `GEO_SALES_PERF_5`. |
| `SalesmanPerformanceDashboardComponent` | `openCustomerDetails(row)` | Opens customer stop history modal and queries `GEO_SALES_PERF_7`. |
| `SalesmanPerformanceDashboardComponent` | `showCurrencyBreakdown()` | Queries multi-currency revenue breakdown via `GEO_SALES_PERF_8`. |
| `SalesmanPerformanceDashboardComponent` | `exportRoutePerformanceExcel()` | Streams route grid Excel spreadsheet (`GEO_SALES_PERF_XL_3`). |
| `SalesmanPerformanceDashboardComponent` | `exportSalesmanPerformanceExcel()` | Streams salesman grid Excel spreadsheet (`GEO_SALES_PERF_XL_4`). |
| `SalesmanPerformanceDashboardComponent` | `navigateToOrderView(customer)` | Navigates directly to the finalized ERP sales order view screen. |

---

## 7. API List Table

| Method | HTTP | Endpoint | Request Payload | Response | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `loadSalesManagers` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_DSH_MNG_LIST', filter: { COMPCODE } }` | Array of manager objects (`code`, `name`) | Populate Sales Manager filter dropdown. |
| `onManagerChange` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_DSH_SALMAN_LIST', filter: { ROLEIDS } }` | Array of salesman objects (`code`, `name`) | Populate Salesmen multiselect dropdown. |
| `search (Planned)` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_SALES_PERF_1', filter: { FROMDATE, TODATE, ROLEID, USERID } }` | `[{ totalVisitCount }]` | Fetch total scheduled store visits. |
| `search (Orders)` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_SALES_PERF_2', filter: { FLEXFIELD3: 'Order' } }` | `[{ insideCount, outsideCount, insideOrderValue, ... }]` | Compute geofenced order counts and revenue. |
| `search (No Orders)` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_SALES_PERF_2', filter: { FLEXFIELD3: 'No Order' } }` | `[{ insideCount, outsideCount }]` | Compute productive vs non-productive visits. |
| `search (Routes)` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_SALES_PERF_3' }` | Array of route performance summaries | Populate Route Performance table. |
| `search (Salesmen)` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_SALES_PERF_4' }` | Array of salesman performance metrics | Populate Salesman Performance table. |
| `search (Customers)` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_SALES_PERF_6' }` | Array of customer performance metrics | Populate Customer Performance table. |
| `salesmanTimeline` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_SALES_PERF_5', filter: { USERID: salesmanCode } }` | Array of chronological visit logs with GPS status | Populate Salesman Detail timeline modal. |
| `customerTimeline` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_SALES_PERF_7', filter: { CUSTCODE } }` | Array of visits made to this specific customer | Populate Customer Detail modal. |
| `currencyBreakdown` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_SALES_PERF_8' }` | Array of revenue sums grouped by currency | Populate Multi-Currency modal. |
| `excelExport` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'GEO_SALES_PERF_XL_*' }` | Binary Excel (`.xlsx`) stream | Download Excel spreadsheets. |

---

## 8. Business Rules Summary

1. **Role Access Restriction:**
   * If a logged-in user has a Manager role, they can only inspect salesmen assigned under their hierarchy.
   * Only users with `ALQ_ACCESS === 'ALL'` can switch across different managers or select `ALL`.
2. **Geofence Compliance Partitioning:**
   * Orders booked within the 50m geofence are aggregated into `insideOrderValue` / `insideCount`.
   * Orders booked beyond 50m via manager override are segregated into `outsideOrderValue` / `outsideCount` to give executives visibility over off-site bookings.
3. **Productive Visit Ratio:**
   * A visit with `flexField03 === 'Order'` is classified as productive; visits with `flexField03 === 'No Order'` are non-productive check-ins.
4. **Multi-Currency Normalization:**
   * Main dashboard cards aggregate revenue in the company base currency (`SOLIDS_CURR_CODE`). If foreign currency orders exist, executives can open the Currency Breakdown modal (`GEO_SALES_PERF_8`) to see exact amounts in USD, EUR, AED, etc.

---

## 9. Concepts You Need to Learn

1. **RxJS `forkJoin` Synchronization Pattern:** Handling simultaneous database aggregation queries with unified completion.
2. **Memory Leak Prevention via `takeUntil` Subject:** Ensuring long-running analytics queries cancel cleanly when leaving the screen.
3. **Pure TypeScript Getters for Template Filtering:** Reducing CPU usage and eliminating change detection lag during heavy grid rendering.
4. **Browser Blob Object URL Lifecycle:** Generating in-memory links with `URL.createObjectURL(blob)` and cleaning up memory with `URL.revokeObjectURL(url)`.

---

## 10. Things That Are Still Unclear (Unknowns / Server-Side Logic)

1. **SQL View Definitions for `GEO_SALES_PERF_1` through `8`:** The complex multi-table SQL joins and currency exchange rate conversion formulas run on the database server.
2. **Role Hierarchy Mapping Rules:** How manager-to-salesman hierarchies are defined in the backend user master tables (`GEO_DSH_MNG_LIST`).
3. **Database Indexing on `SALESMAN_CUSTOMER_VISIT_LIST`:** High-speed execution of date-range filters relies on backend database indexing on `VISITDATE` and `SALESMANCODE`.
<!-- 








































 -->

 Viewed app.component.ts:150-230
Viewed app.component.ts:320-550
Viewed app.component.ts:551-650
Viewed app.component.ts:651-720
Viewed app.component.ts:270-320
Searched for "NavigationEnd"
Searched for "router.events"
Searched for "loadGeoTrackingIntervalAndInit"
Searched for "initBackgroundGeoTracking"
Viewed app.component.ts:80-149
Viewed app.component.ts:230-270
Searched for "onRouteChange"
Viewed app.component.ts:1300-1400

# Complete Runtime Flow: Background Salesman Geo-Tracking & Telemetry Engine

---

## 1. Simple Flow Explanation

1. **Startup & Configuration Check:** When a user logs in, [AppComponent](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts) checks whether background tracking is enabled for the company (`GEO_TRACK_ENABLE_YN`) and loads the configured polling heartbeat interval (`GEO_TRACK_INTERVAL`).
2. **Role Authorization Gate:** It verifies if the user's role possesses the `GEO_TRACK` security privilege in `ACCESS_LIST`.
3. **Persistent Device Identity:** It reads or generates a persistent device UUID (`SOLIDS_DEVICE_ID` formatted as `DEV_WEB_UUID`) to uniquely identify the physical browser hardware across sessions.
4. **Heartbeat Loop Execution:** Every interval tick (e.g. every 1 hour or custom milliseconds), SFA collects in parallel:
   * **Device GPS Coordinates:** Latitude, Longitude, and accuracy radius in meters via [GeocodingService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/geocoding.service.ts).
   * **Reverse-Geocoded Physical Address:** Google Maps Geocoding API resolves coordinates to a formatted street address.
   * **Battery Telemetry:** Battery level % via `navigator.getBattery()`.
   * **Public IP Address:** Real-time client IP via `https://api.ipify.org?format=json`.
5. **Telemetry Logging API:** Dispatches the composite payload to `POST /api/map-api-log/save` with category `Regular`. If GPS fails, an error record is logged (`sendLocationErrorLog`).
6. **Anti-Drain Idle Timeout Protection:** If the salesman stays inactive on the same screen/route for a duration exceeding $2 \times \text{Interval}$ (`routeTimeoutMultiplier = 2`), SFA automatically pauses background tracking (`pauseBackgroundGeoTracking()`) to protect device battery life.

---

## 2. ASCII Flow Diagram

```
+----------------------------------------------------------------------------------------------------+
|                                    1. APPLICATION STARTUP & CONFIG                                 |
|  AppComponent.ngOnInit()                                                                           |
|  --> PortalService.getActiveGeneric('GEO_TRACK_ENABLE_YN') -> Checks if 'Y' for COMPID             |
|  --> PortalService.getActiveGeneric('GEO_TRACK_INTERVAL')  -> Loads interval duration (ms)         |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                      2. ROLE ACCESS VERIFICATION                                   |
|  AppComponent.checkGeoTrackingAccess(roleId, compId)                                               |
|  --> POST /api/common/dynamic/landing/list [ACCESS_LIST]                                           |
|  --> Has 'GEO_TRACK' in AccessId?                                                                  |
|      ├── [NO]  --> Halt tracking.                                                                  |
|      └── [YES] --> Start window.setInterval(tick, intervalDuration)                                |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                    3. HARDWARE TELEMETRY COLLECTION                                |
|  Every Interval Tick -> sendLocationLog(sessionId)                                                 |
|  ├── GPS Coordinates: GeocodingService.getCurrentLocation() [Lat, Lng, Accuracy]                   |
|  ├── Reverse Geocode: GeocodingService.getAddressFromCoords(lat, lng) [Formatted Street Address]   |
|  ├── Battery Level:   navigator.getBattery() -> Math.round(level * 100)%                           |
|  ├── Device UUID:     getOrCreateDeviceId() -> 'DEV_WEB_XXXXXXXX-XXXX-4XXX-...'                   |
|  └── Public IP:       GET https://api.ipify.org?format=json                                        |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                      4. POST TELEMETRY TO BACKEND                                  |
|  POST /api/map-api-log/save                                                                        |
|  --> wmalCompId, wmalUserId, wmalDeviceId, wmalLatitude, wmalLongitude,                             |
|      wmalLocationAccuracy, wmalAddress, wmalBatteryPercentage, wmalIp, wmalFlexField02: 'Regular'  |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+----------------------------------------------------------------------------------------------------+
|                                5. ANTI-DRAIN IDLE TIMEOUT MONITOR                                  |
|  handleTrackingCycleCompleted(sessionId)                                                           |
|  --> Increments trackingCallCount++                                                                |
|  --> If trackingCallCount >= routeTimeoutMultiplier (e.g. 2 ticks without route change):           |
|      --> pauseBackgroundGeoTracking() [Clears interval to save battery]                            |
|  --> User navigates / changes route -> Reset timer & restart tracking loop                         |
+----------------------------------------------------------------------------------------------------+
```

---

## 3. Step-by-Step Runtime Flow Trace

```
Trigger / Event
  ↓
Component
  ↓
Method
  ↓
Service
  ↓
Hardware / Browser API
  ↓
API Endpoint
  ↓
Request Payload
  ↓
Backend Response
  ↓
State / Data Update
  ↓
Anti-Drain Protection
  ↓
Lifecycle Cleanup
```

---

### Step 1: Startup Configuration & Interval Discovery

* **Exact File Path:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L132-L207)
* **Class Name:** `AppComponent`
* **Method Name:** `ngOnInit()` $\rightarrow$ `loadGeoTrackingIntervalAndInit()`
* **What Happens:**
  1. `AppComponent.ngOnInit()` detects active login session `this.SOLIDS_DATA`.
  2. Calls `loadGeoTrackingIntervalAndInit()` which queries `GEO_TRACK_ENABLE_YN` for the company.
  3. If enabled (`valueDesc === 'Y'`), it queries `GEO_TRACK_INTERVAL` to retrieve the heartbeat frequency in milliseconds (defaults to 1 hour if not specified).
* **Service:** [PortalService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts) (`this.Roots`)
* **API Endpoints:**
  1. `GET {SOLIDS_MASTER_URL}/api/common/active/genericMaster/list?catgId=GEO_TRACK_ENABLE_YN`
  2. `GET {SOLIDS_MASTER_URL}/api/common/active/genericMaster/list?catgId=GEO_TRACK_INTERVAL`
* **Backend Responses:**
  ```json
  [
    { "value": "COMP01", "valueDesc": "Y" }
  ]
  [
    { "value": "COMP01", "valueDesc": "3600000" }
  ]
  ```
* **State / Data Update:** Sets `this.canEnableYN = true` and `this.geoTrackingIntervalDuration = 3600000`.
* **What It Calls Next:** `this.initBackgroundGeoTracking()`.

---

### Step 2: Role Permission Check (`ACCESS_LIST`)

* **Exact File Path:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L367-L394)
* **Class Name:** `AppComponent`
* **Method Name:** `checkGeoTrackingAccess(roleId, compId)`
* **What Happens:**
  1. Evaluates whether the user's role is granted background tracking rights.
  2. Dispatches `subcatgid: 'ACCESS_LIST'` query for `SOLIDS_ROLEID` and `SOLIDS_COMPID`.
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/common/dynamic/landing/list`
* **Request Payload:**
  ```json
  {
    "ascordesc": "DESC",
    "filter": {
      "ROLE_ID": "ROLE_SALESMAN",
      "COMPID": "COMP01"
    },
    "subcatgid": "ACCESS_LIST"
  }
  ```
* **Backend Response:**
  ```json
  [
    { "AccessId": "GEO_TRACK" },
    { "AccessId": "OUT_RANGE_ORD" }
  ]
  ```
* **Permission Gate:** If `accessList.some(a => a.AccessId === 'GEO_TRACK')` is `true`, tracking proceeds; otherwise, execution terminates silently.

---

### Step 3: Device UUID Resolution

* **Exact File Path:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L396-L439)
* **Class Name:** `AppComponent`
* **Method Name:** `getOrCreateDeviceId()`
* **What Happens:**
  1. Checks `localStorage.getItem('SOLIDS_DEVICE_ID')`.
  2. If absent, calls `generateDeviceUUID()` using cryptographic entropy (`crypto.randomUUID()` or `crypto.getRandomValues()`).
  3. Formats and persists the hardware token:
     ```typescript
     deviceId = 'DEV_WEB_' + this.generateDeviceUUID().toUpperCase();
     localStorage.setItem('SOLIDS_DEVICE_ID', deviceId);
     ```
* **Why It Happens:** Gives managers a consistent, tamper-proof hardware audit trail across browser refreshes and multi-tab sessions.

---

### Step 4: Heartbeat Interval Activation

* **Exact File Path:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L322-L342)
* **Class Name:** `AppComponent`
* **Method Name:** `initBackgroundGeoTracking()`
* **What Happens:**
  1. Clears any existing interval (`this.clearGeoInterval()`).
  2. Resets `this.trackingCallCount = 0`.
  3. Assigns a new `trackingSessionId = ++this.trackingSessionId` to invalidate any obsolete async operations.
  4. Starts the background heartbeat timer:
     ```typescript
     this.backgroundGeoTrackIntervalId = window.setInterval(() => {
       if (sessionId !== this.trackingSessionId) {
         this.clearGeoInterval();
         return;
       }
       this.sendLocationLog(sessionId);
     }, this.geoTrackingIntervalDuration);
     ```

---

### Step 5: Hardware & Geolocation Data Capture

* **Exact File Path:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L441-L502) and [src/app/service/geocoding.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/geocoding.service.ts)
* **Class Name:** `AppComponent`
* **Method Name:** `sendLocationLog(sessionId)`
* **Telemetry Data Points Collected:**
  1. **GPS Coordinates & Accuracy:**
     * Calls `GeocodingService.getCurrentLocation()` $\rightarrow$ `navigator.geolocation.getCurrentPosition({ enableHighAccuracy: true, timeout: 10000 })`.
     * Extracts `lat`, `lng`, and `accuracy: "12m"`.
  2. **Reverse Geocoded Street Address:**
     * Calls `GeocodingService.getAddressFromCoords(lat, lng)`.
     * Interacts with Google Maps Geocoder API to obtain formatted address string (e.g. `"Sheikh Zayed Rd, Dubai, UAE"`).
  3. **Battery Percentage:**
     * Calls `navigator.getBattery()`.
     * Computes `batteryPercent = Math.round(battery.level * 100) + '%'`.
  4. **UTC Timestamp:**
     * Constructs ISO-8601 UTC string: `trackingDate: "2026-09-02"`, `trackingHour: 10`, `utcDateTime: "2026-09-02T10:15:30Z"`.

---

### Step 6: Public IP Resolution

* **Exact File Path:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L549-L558)
* **Class Name:** `AppComponent`
* **Method Name:** `sendLocationLog()` (IP lookup)
* **API Endpoint:** `GET https://api.ipify.org?format=json`
* **Response:**
  ```json
  {
    "ip": "86.98.142.21"
  }
  ```

---

### Step 7: Saving Location Telemetry Log API

* **Exact File Path:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L503-L547)
* **Class Name:** `AppComponent`
* **Method Name:** `sendLocationLog()` $\rightarrow$ HTTP POST
* **API Endpoint:** `POST {SOLIDS_MASTER_URL}/api/map-api-log/save`
* **Request Payload:**
  ```json
  {
    "wmalCompId": "COMP01",
    "wmalUserId": "USR001",
    "wmalDeviceId": "DEV_WEB_A1B2C3D4-E5F6-4789-8012-3456789ABCDE",
    "wmalTrackingDate": "2026-09-02",
    "wmalTrackingHour": 10,
    "wmalTime": "2026-09-02T10:15:30Z",
    "wmalLatitude": 25.204849,
    "wmalLongitude": 55.270782,
    "wmalLocationAccuracy": "12m",
    "wmalAddress": "Al Fahidi St, Bur Dubai, UAE",
    "wmalApiUrl": "https://maps.googleapis.com/maps/api/geocode/json",
    "wmalApiResponseJson": "{\"status\":\"OK\",\"results\":[{\"formatted_address\":\"Al Fahidi St, Bur Dubai, UAE\"}]}",
    "wmalError": null,
    "wmalIp": "86.98.142.21",
    "wmalBatteryPercentage": "84%",
    "wmalTrackingSource": "GPS",
    "wmalFlexField01": "",
    "wmalFlexField02": "Regular",
    "wmalFlexField03": "",
    "wmalFlexField04": "",
    "wmalFlexField05": "",
    "wmalFlexField06": ""
  }
  ```
* **Backend Response:**
  ```json
  {
    "status": "SUCCESS",
    "message": "Location log saved successfully"
  }
  ```
* **What It Calls Next:** `this.handleTrackingCycleCompleted(sessionId)`.

---

### Step 8: Error Recovery & Geolocation Error Logging

* **Exact File Path:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L566-L650)
* **Class Name:** `AppComponent`
* **Method Name:** `sendLocationErrorLog(error, sessionId)`
* **Condition:** Triggered if device GPS is turned off or browser permission is denied (`PERMISSION_DENIED`, `POSITION_UNAVAILABLE`, `TIMEOUT`).
* **What Happens:**
  * SFA still sends a telemetry record with `wmalLatitude: 0`, `wmalLongitude: 0`, but records the exact hardware error message in `wmalError: "User denied Geolocation"` along with battery % and IP address.
* **Why It Happens:** Prevents salesmen from evading tracking by turning off location services undetected; managers receive immediate audit proof of location disabling.

---

### Step 9: Anti-Drain Idle Timeout Protection

* **Exact File Path:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L652-L668)
* **Class Name:** `AppComponent`
* **Method Name:** `handleTrackingCycleCompleted(sessionId)`
* **Execution Details:**
  1. Increments `this.trackingCallCount++`.
  2. Evaluates:
     ```typescript
     if (this.trackingCallCount >= this.routeTimeoutMultiplier) {
       console.log(`[Geo Tracking] Target of 2 intervals reached. Pausing timer.`);
       this.pauseBackgroundGeoTracking();
     }
     ```
  3. When `trackingCallCount >= 2` (meaning 2 consecutive intervals elapsed without the user navigating or taking new actions), SFA halts the timer.
* **Battery Protection Result:** Background GPS polling pauses when the device is sitting idle on a desk, saving mobile battery.

---

### Step 10: Lifecycle Teardown on Logout & Route Navigation

* **Exact File Path:** [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L670-L687) & [L1368-L1385](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts#L1368-L1385)
* **Class Name:** `AppComponent`
* **Method Name:** `clearBackgroundGeoTracking()` and `logoutAction()`
* **What Happens:**
  1. When the user logs out or the component destroys, SFA calls `clearBackgroundGeoTracking()`.
  2. Clears the `window.setInterval` handle (`clearInterval`).
  3. Resets `this.trackingSessionId`, `this.trackingCallCount = 0`, and unregisters background references in [PortalService](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts).

---

## 4. Important Technical Concepts

### 4.1 Browser & Web Hardware APIs
* **HTML5 Geolocation API (`navigator.geolocation`):**
  * Requests hardware GPS satellite lock with high-accuracy fallback.
* **Battery Status API (`navigator.getBattery`):**
  * Modern Web API returning a `BatteryManager` promise with `level` (0.0 to 1.0) and charging status.
* **Web Cryptography API (`crypto.randomUUID`, `crypto.getRandomValues`):**
  * High-entropy cryptographic random number generation for RFC 4122 compliant device identifiers.

### 4.2 Angular & State Architecture
* **Global Singleton App Root (`AppComponent`):**
  * Hosting the background tracking loop at the root component level ensures tracking runs continuously across all feature routes without interruption.
* **Session Invalidation Tokens (`trackingSessionId`):**
  * An incremental integer counter invalidates in-flight asynchronous promises (e.g. slow reverse geocode responses) when the user switches accounts or triggers a route change.

---

## 5. Important Files

| File Path | Role / Purpose |
| :--- | :--- |
| [src/app/app.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/app.component.ts) | Root component controller; manages background tracking intervals, battery extraction, and anti-drain idle monitors. |
| [src/app/service/geocoding.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/geocoding.service.ts) | Hardware abstraction layer for browser GPS acquisition and Google Maps reverse geocoding. |
| [src/app/service/portal.service.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/service/portal.service.ts) | Master service managing `ACCESS_LIST`, `GEO_TRACK_ENABLE_YN`, and background interval handles. |
| [src/app/report-geo-api/geo-api-report.component.ts](file:///d:/Ramesh/SH_FE/sh_fe/src/app/report-geo-api/geo-api-report.component.ts) | Manager audit report screen displaying all historical records created by this telemetry engine. |

---

## 6. Important Methods

| Class | Method | Purpose |
| :--- | :--- | :--- |
| `AppComponent` | `loadGeoTrackingIntervalAndInit()` | Queries `GEO_TRACK_ENABLE_YN` and `GEO_TRACK_INTERVAL` configs. |
| `AppComponent` | `checkGeoTrackingAccess(roleId, compId)` | Verifies if user's role has `GEO_TRACK` privilege. |
| `AppComponent` | `initBackgroundGeoTracking()` | Starts the periodic `window.setInterval` heartbeat loop. |
| `AppComponent` | `sendLocationLog(sessionId)` | Collects GPS, reverse geocoded address, battery %, and public IP, then posts to API. |
| `AppComponent` | `sendLocationErrorLog(error, sessionId)` | Submits telemetry error logs when GPS hardware is disabled by the user. |
| `AppComponent` | `handleTrackingCycleCompleted(sessionId)` | Increments tick counter and halts timer when idle limit ($2\times$) is reached. |
| `AppComponent` | `getOrCreateDeviceId()` | Generates and persists stable device UUID `DEV_WEB_UUID`. |
| `GeocodingService` | `getCurrentLocation()` | Captures current device latitude, longitude, and accuracy radius. |
| `GeocodingService` | `getAddressFromCoords(lat, lng)` | Translates GPS coordinates into formatted street address via Google Maps. |

---

## 7. API List Table

| Method | HTTP | Endpoint | Request Payload | Response | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `checkTrackingEnabled` | `GET` | `/api/common/active/genericMaster/list?catgId=GEO_TRACK_ENABLE_YN` | None | `[{ value: COMPID, valueDesc: 'Y' }]` | Check if company enabled tracking. |
| `getIntervalDuration` | `GET` | `/api/common/active/genericMaster/list?catgId=GEO_TRACK_INTERVAL` | None | `[{ value: COMPID, valueDesc: '3600000' }]` | Fetch heartbeat polling duration in ms. |
| `checkRoleAccess` | `POST` | `/api/common/dynamic/landing/list` | `{ subcatgid: 'ACCESS_LIST', filter: { ROLE_ID, COMPID } }` | Array of privileges (`GEO_TRACK`) | Verify user role tracking authorization. |
| `getPublicIp` | `GET` | `https://api.ipify.org?format=json` | None | `{ ip: "86.98.142.21" }` | Retrieve client public IP address. |
| `reverseGeocode` | `GET` | Google Maps Geocoding API | `{ latlng: 'lat,lng' }` | `{ results: [{ formatted_address }] }` | Resolve physical street address. |
| `saveTelemetryLog` | `POST` | `/api/map-api-log/save` | Full Telemetry JSON (GPS, Battery %, Address, UUID, IP) | `{ status: 'SUCCESS' }` | Persist telemetry ping in audit database. |

---

## 8. Business Rules Summary

1. **Company Enablement Rule:** Tracking will not start unless `GEO_TRACK_ENABLE_YN` is `'Y'` for the company.
2. **Role Authorization Rule:** Tracking only activates for roles containing `GEO_TRACK` in `ACCESS_LIST`.
3. **Anti-Drain Idle Threshold:** If no route navigation occurs for $2 \times \text{Interval}$, tracking is suspended (`routeTimeoutMultiplier = 2`).
4. **Hardware Failure Logging:** If GPS capture fails or permission is denied, SFA logs a record with `wmalLatitude: 0`, `wmalLongitude: 0`, and the failure reason in `wmalError`.
5. **Persistent Device Identity:** `SOLIDS_DEVICE_ID` is stored in `localStorage` to distinguish different laptops/tablets used by the same salesman.

---

## 9. Concepts You Need to Learn

1. **HTML5 Battery Status API:** How browser apps monitor device battery percentages.
2. **Web Cryptography API (`window.crypto`):** Generating standard RFC UUIDs in modern browsers without third-party npm libraries.
3. **Session Invalidation Tokens in Timers:** Preventing race conditions and memory leaks when asynchronous timers cross route/session boundaries.
4. **Anti-Drain Battery Optimization:** Implementing multiplier-based idle timeouts to balance field telemetry resolution with mobile device battery conservation.

---

## 10. Things That Are Still Unclear (Unknowns / Server-Side Logic)

1. **Database Retention & Purging Policies for `map-api-log`:** How long raw telemetry heartbeats are retained in the database before automated archival or truncation.
2. **Server-Side Battery Alert Triggers:** Whether low battery levels ($<15\%$) trigger automated alert notifications to supervisors in the manager dashboard.