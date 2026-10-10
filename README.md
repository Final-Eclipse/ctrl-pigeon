# RemoteShell AI

Text your Mac through Photon iMessage. Generate a presentation, open selected apps, launch a configured Roblox experience, and observe target windows without intentionally taking foreground focus.

**Status: implementation and automated tests complete; live macOS/Photon/OpenRouter integration untested here.** This is a constrained hackathon demo build, not a general unrestricted desktop agent. Generic clicks, typing and shortcuts are limited to Calculator. Keynote/PowerPoint use file generation; Roblox uses its deep link. If your chosen Roblox experience needs an in-game Play click, configure ROBLOX_PLAY_LABEL for one bounded background click with AX or vision grounding. Do not pitch automatic match entry until that experience has been rehearsed.

## API contracts checked before implementation

Photon `spectrum-ts` and `@spectrum-ts/imessage` 12.10.1: `await Spectrum({projectId, projectSecret, providers:[imessage.config()]})`; consume `[space,message]` from `app.messages`; `space.send(text)`; `space.send(attachment(buffer,{mimeType:'image/png'}))`; `app.stop()`. Adapter accepts inbound iMessage text only, from direct messages, and rejects outbound echoes. Its sender and space shapes were checked against installed package declarations and source. Project ID/secret replace the brief's speculative PHOTON_API_KEY.

Cua uses the documented daemon CLI: `cua-driver call TOOL JSON`. `list_windows` selects an exact app; `get_window_state` provides elements; input uses an exact window target and `delivery_mode:"background"`. Every call uses the same `remoteshell` session. The Python binding package exists and is pinned, but this implementation deliberately talks to the daemon CLI so the signed CuaDriver app owns macOS permissions. The CLI and signed daemon must be compatible with each other.

Sources: [Photon repository](https://github.com/photon-hq/spectrum-ts), [Cua CLI](https://cua.ai/docs/cua-driver/reference/cli/mcp), [window state](https://cua.ai/docs/cua-driver/reference/mcp-tools/window-state), [background input](https://cua.ai/docs/cua-driver/reference/mcp-tools/click), [SDK limitations](https://cua.ai/docs/cua-driver/guides/use-the-sdk).

## Architecture

The Node gateway maintains a Photon stream, normalizes and constant-time compares one allowed phone number, deduplicates the latest 500 message IDs, and runs one task at a time. At most ten tasks are admitted. Unknown senders receive no response; logs retain only their last four digits. Email handles and ambiguous national numbers are rejected instead of guessing an E.164 identity. Group messages and non-text events are ignored.

The FastAPI agent binds to 127.0.0.1. Every task/status/cancel request requires the shared secret. POST /task admits a UUID once, and authenticated GET /task/UUID returns incremental progress. The gateway polls this *local agent*, not the Messages database. A dropped POST can be retried with the same ID without executing again. Receipts survive for one hour in memory, not across process restarts. Unknown outcomes are not automatically resubmitted under a new ID.

Hermes produces strict Pydantic-validated JSON, with one correction opportunity. The same schema supports deterministic skills through `params.skill`; no extra action enum is required. Text UI observations stay with the text planner. Empty trees use a screenshot and the configured vision model, which returns text and coordinates. Screenshots sent to OpenRouter may contain the target window's content.

## Setup on the demo Mac

1. Use macOS 14 or newer. Install Homebrew from [brew.sh](https://brew.sh) if needed. Install Keynote (or Microsoft PowerPoint) and Roblox. Open each once and finish onboarding.
2. Extract this ZIP. Open Terminal in the `remoteshell-ai` folder and run:

```bash
bash scripts/setup_mac.sh
```

This installs Node 24, Python 3.11, a local virtual environment, pinned direct Python dependencies, and locked npm dependencies. To perform those steps individually:

```bash
brew install node@24 python@3.11
export PATH="$(brew --prefix node@24)/bin:$PATH"
"$(brew --prefix python@3.11)/bin/python3.11" -m venv .venv
.venv/bin/python -m pip install -r agent/requirements.txt
(cd gateway && npm ci && npm run check)
cp gateway/.env.example gateway/.env
cp agent/.env.example agent/.env
chmod 600 gateway/.env agent/.env
```

3. Follow the signed macOS app installation instructions in the [Cua quickstart](https://cua.ai/docs/cua-driver/quickstart). The Python dependency alone is not a substitute for the signed app's permission flow. Then run:

```bash
open -n -g -a CuaDriver --args serve
cua-driver permissions grant
cua-driver permissions status
cua-driver status
cua-driver call list_windows '{"app":"Calculator","session":"remoteshell"}'
```

4. In **System Settings → Privacy & Security → Accessibility**, enable **CuaDriver**. In **Privacy & Security → Screen & System Audio Recording** (called **Screen Recording** on older releases), enable **CuaDriver**. These are lists of apps with switches on the right; turn its switch on and accept the relaunch prompt. If running a different permission-owning binary, grant that binary instead. Do not grant microphone access. When permissions are stuck, find the actual installed bundle ID, reset just that application's grant, then grant and relaunch again:

```bash
driver_bundle_id="$(osascript -e 'id of app "CuaDriver"')"
tccutil reset Accessibility "$driver_bundle_id"
tccutil reset ScreenCapture "$driver_bundle_id"
cua-driver permissions grant
```

Do not reset every app's permissions. The read-only AppleScript document-count action may separately prompt for Automation permission to control Keynote; normal deck generation does not use it.

5. Create/sign into a project at [app.photon.codes](https://app.photon.codes). Complete the dashboard's current hosted iMessage number/allocation setup. Copy the project's ID and secret into `gateway/.env`. Use the number assigned by Photon as the iPhone conversation recipient, and your personal phone number as `ALLOWED_SENDER`. This code accepts direct messages only. A dashboard or plan that has not provisioned iMessage will not work just from adding credentials.
6. Generate the shared secret and copy the same output into **both** `.env` files:

```bash
openssl rand -hex 32
```

7. Put your OpenRouter key into `agent/.env`. Keep the Hermes default or choose another text planner available to your account. Set `VISION_MODEL` to an available image-input model from [OpenRouter's model catalog](https://openrouter.ai/models). It cannot be the text-only Hermes model. Set `ROBLOX_PLACE_ID` to a numeric experience place ID you have tested. Match `AGENT_PORT` and `TASK_TIMEOUT_S` between the files. Leave `MAX_RETRIES=2`, `TASK_TIMEOUT_S=120` for rehearsal.
8. Start both services:

```bash
bash scripts/start.sh
```

9. In another terminal:

```bash
curl --fail http://127.0.0.1:8765/health
```

The host is fixed to loopback. If you changed the port in both files, use it here too. `/health` reports agent liveness, not Photon connectivity or successful permissions.

10. From the allowed iPhone number, text the Photon number `help`, then `status`, then `Create a 3-slide presentation on RemoteShell AI and open it`. Text `cancel` to cancel the active task and clear waiting tasks. Cancellation does not undo a file already created or an app already launched.

## Commands and deterministic demo skills

- `Create a 3-slide presentation on RemoteShell AI and open it`: Hermes writes content, python-pptx generates a dark widescreen deck, the executor writes it exclusively to a new `~/Desktop/RemoteShell_TIMESTAMP_ID.pptx`, and opens it using `open -g -a Keynote`. PowerPoint is attempted if Keynote cannot be verified. A title-matching document window must appear before success is reported. A retry reuses the saved file.
- `Launch Roblox and start a match`: opens the configured deep link with `open -g`, then waits up to 45 seconds for a Roblox window. The result explicitly distinguishes window launch from match entry. Rehearse with an experience that joins directly, or set `ROBLOX_PLAY_LABEL` to its exact Play button text. That configured action uses AX first, then vision coordinates when the tree is empty, and attempts only one click. A fresh observation must support gameplay before reporting it; uncertain effects are not replayed.
- `Open Calculator`: background app launch and window presence check.
- `Take a screenshot of Calculator`: writes a target-window PNG and relays it with Photon's attachment helper. Retained screenshots remain on Desktop until the user deletes them. Vision-only temporary screenshots are removed.
- `status`, `cancel`, `help`: handled by the gateway without an LLM call.

## Safety and failure behavior

The executor allows fixed actions rather than an arbitrary shell channel. A denylist is a second layer, not its security foundation. AppleScript is restricted to one fixed read-only Keynote document-count template. Deep links are built from a validated numeric Roblox ID. New files have restricted names, live only on Desktop, and never overwrite existing paths. Generic input is limited to Calculator; numeric text and two non-destructive shortcuts are permitted. Unknown apps, raw scripts, payments, arbitrary messages, terminal commands and unrecognized parameters are refused.

Background delivery is explicit. There is no foreground fallback, activation call, global pointer tool or microphone feature. A self-activating app can still violate the desired user experience despite a background launch request; verify this on the real Mac. On the projector, arrange windows in advance: a newly opened background window may be behind another app. The operator can reveal it manually for the audience, but must say so.

Retry events use exactly `⚠️ Task failed, retrying...`. Generic UI recovery first re-observes and checks whether the action already landed. The next retry asks for one replacement step using the next suitable route. The same two-retry budget applies. Deterministic skills keep checkpoints and deck opening tries the alternative presentation app. No foreground escalation is allowed. An input timeout with an uncertain effect stops rather than blindly typing/clicking again. Each task has a hard 120-second ceiling; no plan executes more than twelve steps.

| Failure | Detection | User message | Recovery |
|---|---|---|---|
| Unknown sender | Normalized constant-time gate | None | Silent drop; redacted log |
| Duplicate iMessage | Last-500-ID cache | None | Ignore replay |
| Photon transport ends | SDK/iterator error | May be undeliverable while disconnected | Reconnect with exponential delay and jitter, cap 30 seconds |
| Agent unavailable | Failed localhost admission | Exact retry text, then offline notice | Retry once after 3 seconds with same task ID |
| Invalid plan JSON | Pydantic validation | Failure if two invalid responses | One validation correction request |
| OpenRouter request fails | HTTP/parse/timeout exception | Exact retry text, then reason | Bounded retry within task deadline |
| Missing element or unverified click | Fresh window observation | Exact retry text | Re-observe, then replan one step; no blind unknown-effect replay |
| Keynote opening fails | No matching document window | Exact retry text | Reuse saved deck; attempt PowerPoint |
| Roblox fails to open | No Roblox window within 45 seconds | Exact retry text, then reason | Poll again without reissuing an acknowledged deep link |
| Task deadline | asyncio timeout | Failure; server still running | Cancel task coroutine and any active subprocess |
| User cancellation | Authenticated cancel endpoint | Cancellation notice | Cancel active task and clear gateway queue |
| Image send fails | Attachment helper throws | Screenshot saved, delivery failed | Keep local file; finish task |
| Reply delivery fails | Photon send throws | May be undeliverable | Log and continue; no guaranteed outbound durable outbox |

Expected exceptions are contained per task. Startup misconfiguration fails fast. Fatal native crashes, process termination, disk exhaustion, and corrupted process state cannot be made impossible. Node's required uncaught-exception handler logs without exiting, but continuing after an unknown process fault is not a production reliability guarantee. Restart and rehearse if that happens. The start script drains the gateway before stopping the agent.

## Pre-demo checklist

- [ ] Mac plugged into power, awake and unlocked; working internet and projector.
- [ ] Cua driver healthy with Accessibility and Screen Recording grants.
- [ ] Photon number provisioned; `help` and `status` reply to the allowed phone.
- [ ] Text from a second phone gets no reply and creates no task.
- [ ] OpenRouter account has credits and both configured models respond.
- [ ] Generate and open a real three-slide deck; dismiss first-run/import dialogs.
- [ ] Roblox logged in; tested place accessible; no update, age gate or purchase gate; rehearse ROBLOX_PLAY_LABEL if needed.
- [ ] Watch physical cursor and focus during both demos; background behavior verified.
- [ ] Try `cancel`; try one bad request, then a valid request to show recovery.
- [ ] Enable Do Not Disturb. Keep an honestly labeled backup recording ready.

## Three-minute run-of-show

**0:00–0:15 — Introduce the scene.** Presenter holds the iPhone. Manager stands nearby. Operator watches the projector and logs. Arrange the Mac desktop so the deck can be visible without bringing its app to the foreground automatically.

**0:15–0:30 — Set up the problem.** Manager: “Great, so you'll walk the client through the deck in five minutes?” Presenter, glancing at the phone: “Absolutely. Give me one second.” Text: “Create a 3-slide presentation on RemoteShell AI and open it.”

**0:30–1:30 — Show actual progress.** Show the acknowledgment on the phone. Presenter: “That text goes through Photon to an agent running on this Mac. It writes the slide content, builds the presentation file, and opens it in the background.” When the verified deck appears: “Deck's ready.” If the operator reveals a covered background window, say: “We're bringing the finished deck into view for you.”

**1:30–2:15 — Roblox.** Text: “Launch Roblox and start a match.” Presenter: “Same thread, different task. The agent launches our configured Roblox experience through a native deep link.” Only after visibly entering the experience say “We're in.” If only the window is verified, say “Roblox is open; the experience is still loading.”

**2:15–3:00 — Explain and finish.** Show `status`. Presenter: “Only this whitelisted phone number can start tasks. The gateway serializes requests, checks duplicates, and reports failures. The execution layer uses background delivery, with deterministic file generation for the deck.” If recovery actually occurs: “It caught an error and is checking the current state before retrying.” Say “replanned” only when logs show a replan. If live services fail, show the labeled backup recording and explain which dependency failed.

## 45-second technical pitch

“RemoteShell AI turns one authorized iMessage thread into a command channel for a Mac. Photon Spectrum provides the messaging stream, so our app doesn't read the local Messages database. The gateway filters one phone number, rejects duplicate messages, and queues tasks. A localhost agent validates Hermes plans and runs a small set of allowed actions. For the presentation, it generates the file directly instead of clicking through slides. Cua Driver handles targeted window observation and background input. We re-observe after UI actions, bound retries, and report uncertain outcomes honestly. The Roblox demo launches a configured experience; the game controls when the match is ready. It's a focused prototype for delegating work without intentionally taking over your mouse.”

## Tests and evidence

Verified in the build environment: npm dependency installation, TypeScript compilation, shell syntax, Python compilation, three Node tests, and three Python test cases. Python tests cover authentication, task deduplication, task cancellation, continued health, schema/command rejection, and actual three-slide PPTX generation with retry reuse. Photon, OpenRouter, app opening, and driver calls are mocked where needed; these tests do not establish live compatibility.

```bash
(cd gateway && npm run check && npm test)
.venv/bin/python -m unittest discover -s agent/tests -v
```

All direct dependency versions were checked in npm or PyPI during this build. The npm lockfile includes transitive versions. Python direct versions are pinned; transitive Python dependencies are resolver-selected. The Cua native wheel and signed macOS app were not installed or run in this build environment.

## Known Risks & Verify List

1. **VERIFY: Cua CLI response envelope on the installed signed release.** `agent/adapters/cua.py` expects JSON with `structuredContent` or a raw object. Run the documented `list_windows` and `get_window_state` calls; inspect `windows`, `elements`, `capture_id` and screenshot output. Docs: https://cua.ai/docs/cua-driver/reference/cli/mcp . No unverified field should be silently interpreted as success.
2. **VERIFY: Roblox deep-link behavior and client version.** See the comment in `agent/skills/roblox.py` and https://create.roblox.com/docs/production/promotion/deeplinking . The deep-link documentation could not be fetched in this session. Window presence does not prove match entry. An experience with an in-game Play gate requires configuring and testing ROBLOX_PLAY_LABEL; the generic Play helper cannot guarantee that every game accepts background input.
3. **VERIFY: launch focus on this Mac.** `open -g` requests background opening, but some apps can self-activate, and native import dialogs can block verification. See https://cua.ai/docs/cua-driver/reference/mcp-tools/apps-and-windows . Do not claim universal cursor/focus isolation.
4. **VERIFY: Photon account allocation and sender fields.** Package APIs compile against 12.10.1; delivery was not tested. `sender.service` must be `iMessage`, `space.type` must be `dm`, and `sender.id` must normalize to the configured phone. No email identity bypass. See https://github.com/photon-hq/spectrum-ts . SDK-internal reconnects may not emit every low-level connection state through this wrapper.
5. **VERIFY: OpenRouter availability and latency.** Hermes is text-only; the vision model must support image input. Model availability, output format support and account credits require a real API call. See https://openrouter.ai/nousresearch/hermes-3-llama-3.1-405b . A 120-second budget takes priority over completing every retry.
6. **Scope differences from the requested general agent:** generic UI input is Calculator-only; arbitrary AppleScript and URL execution are intentionally not offered; Roblox Play requires experience-specific rehearsal; outbound retries/outbox are not durable; task receipts and message IDs are in memory. These are actual limits, not hidden VERIFY stubs.
