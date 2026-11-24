`tl;dr just download the HTML page and open it.`


# local-ai
Make native local AI as easy to use as an ordinary chat website.  
Developer handoff · Standalone HTML proof of concept · 24 November 2026

## 1. Project purpose

Make native local AI as easy to use as an ordinary chat website. A non-technical user opens `local_ai.html`, runs one setup command only if necessary, and starts chatting. The application chooses and prepares the AI automatically.

**The primary success metric is ease of use on the way to the first real AI response, not the number of features.**

### Non-negotiable product decisions

- Native llama.cpp is the chosen inference engine. No WebGPU, WebAssembly, browser-side inference, cloud inference or cloud fallback.
- The browser owns the UI and orchestrates the native engine through HTTP. It does not execute the model itself.
- The user does not select, download, load, unload or delete models manually. There is no model management UI, settings page, developer dashboard, account system or extra onboarding confirmation screen in this prototype.
- Keep llama.cpp defaults; the only agreed server configuration is port `9931`. Reuse a compatible running instance rather than restarting or reinstalling it. Do not add custom authentication, proxies or configuration to this MVP.
- Automatically advance based on real server state. Never simulate progress, readiness or AI responses.
- Technical details belong in the implementation and browser console, not the main interface.

## 2. Deliverable and how to run it

| File | Purpose |
|---|---|
| `local_ai.html` | Complete application: HTML, CSS and vanilla JavaScript in one file |
| `README.md` | This developer handoff |

Open `local_ai.html` directly in Chrome on a Mac. No npm install, bundler, framework, application backend or API key is required. If llama.cpp is already serving on `http://localhost:9931`, the app should connect automatically. Otherwise follow the two setup instructions on screen, keep Terminal open and return to the page. No refresh or Continue button is needed.

If opening via `file://` is blocked by the browser, developers can serve the file from its containing directory:

```bash
python3 -m http.server 8080
```

Open `http://localhost:8080/local_ai.html`. This is a troubleshooting fallback, not a normal user step. Local HTTP serving does not itself guarantee cross-origin access; browser permissions and llama.cpp's response headers still apply.

## 3. Architecture and responsibilities

```text
local_ai.html (browser)
  Platform detection / onboarding / chat / state orchestration
       | HTTP fetch + server-sent events (SSE)
       v
http://localhost:9931 (native llama.cpp)
  Model inventory / downloads / loading / inference
       | 
       v
Local disk + native CPU/GPU memory and execution

Terminal command (only if engine unavailable)
  Install if missing -> start native server -> browser detects it
```

Model downloads require internet access when files are missing. Inference and conversation requests go to the local server, not a cloud AI API. The current page does not persist chat history to disk; history is kept in JavaScript memory and resets on refresh. Model files and runtime state belong to llama.cpp, so the browser can resume preparation by inspecting the server.

### Browser limitation

A normal website cannot install native executables or launch Terminal commands, even with a one-time browser permission. Copying the command is the only browser-assisted native setup step. The user must paste and execute it. Later, a packaged native installer/companion (macOS app/pkg, Windows EXE/MSI) can replace that step without changing the basic UI or inference architecture. This packaging is future work, not part of the current HTML.

## 4. User experience specification

### Screen A — Detect / setup

Immediately identify the OS using browser platform/user-agent hints. Display the detected OS, not an invented laptop model, CPU, RAM or compatibility guarantee. Probe `/health` automatically.

If the engine is unavailable, show a short setup screen:

1. Open Terminal (on Mac: Command + Space, type Terminal, Enter).
2. Copy the setup command, paste it into Terminal, press Enter.

One prominent **Copy setup command** button. Keep the long command behind **View command**, accessible before execution. Keep troubleshooting behind **Having trouble?**. Tell the user to leave Terminal open and return to the page. Do not expose technical explanations by default. Keep polling automatically.

If the engine is already available, skip the setup screen. A failed browser request does not prove the engine is uninstalled; it may be stopped or inaccessible from the browser.

### Screen B — Automatic preparation

Show four simple stages: Connect to your computer → Download your AI → Prepare your AI → Say hello. Show genuine server-reported progress when available; otherwise use an indeterminate waiting indicator. No manual buttons for the model lifecycle.

### Screen C — Chat

Only enter chat after a successful real greeting. Display that greeting as the first assistant message. Subsequent messages stream from the native server. Enter sends, Shift+Enter adds a line. Keep the UI minimal.

### Failure behavior

Show a short error and Try again during initialization. During chat, restore the failed prompt and display the error. Do not claim the AI is ready if inference failed. Console errors support developer diagnosis; no separate developer panel is required.

## 5. Native setup — macOS first

The exact Unix command currently embedded in the HTML is:

```sh
if command -v llama-server >/dev/null 2>&1; then llama-server --port 9931; elif command -v llama >/dev/null 2>&1; then llama serve --port 9931; elif [ -x "$HOME/.llama-app/llama" ]; then "$HOME/.llama-app/llama" serve --port 9931; else curl -LsSf https://llama.app/install.sh | sh && "$HOME/.llama-app/llama" serve --port 9931; fi
```

This checks existing executables before installing anything; uses the llama CLI installer path when necessary; starts the engine in the foreground. It does not download the starter model itself. The browser handles model preparation. The installer URL and installed CLI path are external dependencies: verify them against the current official release before distribution. Users can inspect the command before running it.

Developers who already use Homebrew can instead run:

```bash
(command -v llama-server || brew install llama.cpp) && llama-server --port 9931
```

Do not specify `-m` or `-hf` for this router-mode test. Leave the server terminal running; Ctrl+C stops the foreground server. If the browser is already connected, do not run another server on the same port. The demo is not an automatic login service and does not install a background daemon.

Windows has a PowerShell/winget setup branch and Linux shares the Unix branch in the HTML. These are not yet validated on real target systems. Mobile and unknown OS show desktop instructions rather than a guessed install command. macOS is the current acceptance-test platform.

## 6. Automatic model initialization contract

Starter model: `ggml-org/Qwen3.5-0.8B-GGUF:Q4_0` (small 0.8B model, approximately 563 MB of model weights; supporting files can increase download size).

1. Wait until `GET /health` reports `status: ok`.
2. Read `GET /models`; find the exact starter ID (router mode).
3. If absent/incomplete, request download via `POST /models`. If downloading, wait instead of starting another download. If already cached, skip download.
4. Watch `/models/sse` for progress and poll `/models` until the starter is available. A `success: true` acknowledgment is not completion.
5. Request `POST /models/load` if needed; if loading, wait. If already loaded, skip loading.
6. Verify runtime status via `/models` and send a real non-streaming greeting using the explicit model ID. The current code also accepts `sleeping` as an existing model state and relies on inference to verify readiness.
7. Require non-empty `choices[0].message.content` before opening chat.

Existing single-model server compatibility: if the router inventory endpoint is unsupported (recognized error) or returns models without runtime status, the current HTML uses the first ID from `/v1/models` and sends its greeting to that existing model rather than restarting the server. In router mode it deliberately chooses the starter model, even if another model is already loaded; llama.cpp defaults determine eviction/resource behavior. No model picker is exposed.

Do not omit `model` in router chat requests. This was tested on the user's server and returned HTTP 400: `model name is missing from the request`.

The initial prompt is `Hello! Greet me briefly in one sentence.` The actual response becomes the chat welcome message. The hidden greeting prompt and its response are retained in the conversation history for subsequent turns.

## 7. Developer curl reference

These commands are for developer testing and troubleshooting only. Ordinary users must not perform them. Run in a separate terminal while the native server is active. `jq` is optional for raw API use but required for the examples below (`brew install jq` if needed).

```bash
BASE='http://localhost:9931'
MODEL='ggml-org/Qwen3.5-0.8B-GGUF:Q4_0'
PAYLOAD=$(jq -n --arg model "$MODEL" '{model:$model}')
```

### A. Detect server and inspect models

```bash
curl -sS --max-time 3 "$BASE/health" | jq
curl -sS "$BASE/models" | jq
```

Expected health: `{"status":"ok"}`. This does not mean a model is loaded. For a compact list:

```bash
curl -sS "$BASE/models" | jq '.data[] | {id,status:.status.value}'
```

### B. Download starter model and wait

Open a separate terminal to monitor events before starting the download:

```bash
curl -N http://localhost:9931/models/sse
```

In the original terminal:

```bash
curl -sS "$BASE/models" -H 'Content-Type: application/json' \
  -d "$PAYLOAD" | jq
```

`{"success":true}` means accepted. Watch download events (`download_progress`, `download_finished`, `download_failed`) and inventory status. Wait for the download to finish. Ctrl+C in the event-monitor terminal closes the monitor, not the native server.

### C. Verify the exact model is available

```bash
curl -sS "$BASE/models" | jq --arg m "$MODEL" \
  '.data[] | select(.id==$m)'
```

A completed cached model normally reports `unloaded`, or is already loading/loaded. An absent entry, `downloading`, `incomplete` or failed status is not evidence of availability. Do not load a partial download.

### D. Load and verify

```bash
curl -sS "$BASE/models/load" -H 'Content-Type: application/json' \
  -d "$PAYLOAD" | jq
```

Wait for the runtime status; press Ctrl+C once it reports loaded:

```bash
while true; do
  curl -sS "$BASE/models" | jq --arg m "$MODEL" \
    '.data[] | select(.id==$m) | {id,status}'
  sleep 2
done
```

Skip the load request if already running. A successful load acknowledgment alone is not proof of readiness.

### E. Send first message

```bash
curl -sS "$BASE/v1/chat/completions" \
  -H 'Content-Type: application/json' \
  -d "$(jq -n --arg model "$MODEL" '{model:$model,messages:[{role:"user",content:"Hello! Greet me briefly in one sentence."}],chat_template_kwargs:{enable_thinking:false},max_tokens:100,stream:false}')" | jq
```

Check the response's model and `choices[0].message.content`. For raw text, use `jq -r '.choices[0].message.content'`. The thinking option is model-template-specific and used for the starter in the HTML.

### F. Test streaming

```bash
curl -sS -N "$BASE/v1/chat/completions" \
  -H 'Content-Type: application/json' \
  -d "$(jq -n --arg model "$MODEL" '{model:$model,messages:[{role:"user",content:"Tell me a short story."}],chat_template_kwargs:{enable_thinking:false},max_tokens:512,stream:true}')"
```

Expect SSE `data:` frames containing `choices[0].delta.content`, followed by a finish reason and/or `[DONE]`.

### G. Discover currently loaded models

```bash
curl -sS "$BASE/models" | jq -r \
  '.data[] | select(.status.value=="loaded") | .id'
```

This is useful for diagnostics. Zero results means no loaded model; multiple results mean the router has more than one running. The HTML uses its internally selected ID, not a user-facing model chooser.

### H. Optional developer cleanup — unload and delete

Not part of user onboarding or the UI. Do not automatically remove existing user models. Only run deletion for a disposable test model you intentionally want to remove.

```bash
curl -sS "$BASE/models/unload" -H 'Content-Type: application/json' \
  -d "$PAYLOAD" | jq
```

Verify `unloaded` using step D's status query. Then, if deletion is supported on the tested build:

```bash
curl -sS -G -X DELETE "$BASE/models" --data-urlencode "model=$MODEL" | jq
curl -sS "$BASE/models" | jq --arg m "$MODEL" \
  '.data[] | select(.id==$m)'
```

No matching entry indicates removal from the inventory. Shared supporting files may remain on disk. Unload/delete endpoints are retained here for the team's technical reference, not as a request to implement model management.

## 8. API and source code map

| Method | Endpoint | Used by HTML |
|---|---|---|
| GET | `/health` | Automatic detection |
| GET | `/models` | Router inventory, download and load verification |
| POST | `/models` | Download starter |
| GET | `/models/sse` | Optional progress event stream |
| POST | `/models/load` | Load starter |
| GET | `/v1/models` | Existing single-model server compatibility |
| POST | `/v1/chat/completions` | Greeting and streaming chat |
| POST | `/models/unload` | Developer reference only |
| DELETE | `/models?model=ID` | Developer reference only |

All HTTP calls use the browser's Fetch API; EventSource handles model events. The HTML is the reference implementation, with no build step.

| Symbol in HTML | Responsibility |
|---|---|
| `BASE`, `STARTER`, `GREETING` | Agreed endpoint, model and initial prompt |
| `detectPlatform`, `unixCommand`, `windowsCommand` | OS hints and native bootstrap instructions |
| `screen`, `step`, `percent` | Minimal onboarding UI and reported progress |
| `api`, `health`, `listModels` | JSON HTTP wrapper, readiness and inventory |
| `startEvents`, `closeEvents` | SSE progress; polling remains authoritative |
| `waitDownload`, `waitLoaded` | State verification and timeouts |
| `boot`, `prepare`, `greet` | Automatic sequential setup, reuse and greeting |
| `completionBody`, `streamChat`, `sendMessage` | Explicit model ID, chat SSE parser and history |

Current implementation details: polling every 1.5 seconds; health timeout 2.5 seconds; regular API timeout 10 seconds; operation acknowledgment timeout 30 seconds; download wait up to 45 minutes; load wait up to 5 minutes (earlier failure on persistent unloaded/incomplete status); greeting timeout 3 minutes; chat inactivity timeout 3 minutes. Chat uses a 512-token response budget and one request at a time. Greeting uses 100 tokens. History is sent with each chat request. Assistant output is rendered as text, not executable HTML. No browser-side model libraries are included.

## 9. Compatibility and scope boundaries

Use the native server's defaults for this demo. The application assumes a compatible llama.cpp build with the management endpoints described above. An older build may support chat but not router downloads. Do not silently introduce browser or cloud inference to work around missing endpoints. Record the server version when reporting bugs (`llama-server --version`).

A successful curl call does not guarantee browser connectivity: browser local-network permissions, cross-origin rules, `file://` policy, and clipboard restrictions can differ. Follow the browser's permission prompt when shown; use the setup screen's troubleshooting disclosure only when needed. The application cannot reliably distinguish blocked requests from a stopped engine. No additional security layer is in scope for this local prototype; do not intentionally expose its unauthenticated API to the public internet. Revisit distribution and browser-origin access only when preparing a production release.

## 10. Acceptance tests and evidence

**Observed in real use:** The project owner reported that `local_ai.html` works perfectly when llama.cpp is active. Earlier curl output from build `b11960-8642541b2` verified router inventory, explicit model inference, automatic loading of an available LFM3 model and loaded-status discovery. Requests to download/load Gemma returned success acknowledgments, but the attempted inference occurred before Gemma was running; the separate CLI download was manually interrupted. Those results do not independently verify a completed HTTP download or deletion.

**Previously tested in development:** JavaScript syntax and browser logic with mocked API responses (automatic detection, preparation, model reuse, error recovery and streaming). Mock tests are not native inference tests. The current working-connection report is valuable but does not establish every clean-install or platform path.

Team acceptance checklist:

- [ ] Existing router on port 9931: open HTML, no terminal interaction, automatic greeting and chat.
- [ ] Existing single-model server on agreed port: reuse existing model where supported.
- [ ] Fresh Mac: OS recognized, concise setup instructions, copy/paste one command, return without refreshing.
- [ ] Missing starter: automatic download, real wait, verified availability, load, verified readiness, real greeting.
- [ ] Cached starter and already loaded starter: skip unnecessary operations.
- [ ] Refresh mid-download: inspect existing state and continue without duplicate download.
- [ ] Send multiple chat turns; verify streaming and conversation context.
- [ ] Interrupt network/download or stop server: useful error and retry, never fake success.
- [ ] Check that no model-management controls or unnecessary technical setup details appear.
- [ ] Record macOS/browser/llama.cpp versions and elapsed time to first greeting; count required user actions.
- [ ] Test Windows/Linux setup separately before claiming support.

The key usability targets are zero setup actions when the compatible engine is running; one copied terminal command otherwise; zero manual model operations; zero model decisions; and an actual conversation as the final result. Measure elapsed time separately from user effort because first-time download duration depends on network and hardware.

## 11. Next steps — deliberately limited

1. Validate the fresh-Mac onboarding path and the current simplified opening page with non-technical users.
2. Fix only issues preventing easy setup and reliable chat; preserve the working integration.
3. When ready for distribution, replace the terminal step with a native packaged installer/companion. Maintain the same browser-managed onboarding and native inference.

Do not expand this prototype into a model manager or add features without a new product decision.

## 12. References

- llama.cpp project: https://github.com/ggml-org/llama.cpp
- Server/router documentation: https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md
- CLI installer endpoint used by the HTML: https://llama.app/install.sh
- Homebrew formula: https://formulae.brew.sh/formula/llama.cpp
- Starter model repository: https://huggingface.co/ggml-org/Qwen3.5-0.8B-GGUF

The HTML and observed behavior on the target build are the immediate integration reference. Upstream documentation can change; verify API compatibility when upgrading.
