# RemoteShell AI

Text your Mac through iMessage. A Gemini tool loop researches the web, creates files and presentations, and uses a bounded set of desktop tools. A local dashboard shows progress and recent results.

This is the complete source bundle: Photon gateway, Python agent, Gemini integration, dashboard, original executor/skills, dependency manifests and tests. No separate patches are needed. Credentials and installed dependencies are excluded.

## Already running RemoteShell?
Keep your working folder. This ZIP is a separate complete copy for sharing or GitHub. It does not contain your personal .env files, .venv or node_modules. Your current project can continue running unchanged. To run this copy, configure its environment and install its dependencies first. Stop the old services before starting another copy on the same port.

## Setup for a teammate (macOS)

Prerequisites: Node 24+, Python 3.11, and the signed CuaDriver app for desktop actions. Keynote or PowerPoint is needed to open generated presentations. An OpenRouter account/key and a configured Photon iMessage project are required.

1. Extract the ZIP and open Terminal in this folder.
2. Run `bash scripts/setup_mac.sh`. It uses existing Node/Python installations or installs missing ones through Homebrew. On older macOS, Homebrew can compile dependencies slowly; preinstalling Node 24 is an alternative.
3. Fill in `agent/.env` and `gateway/.env` from their examples. Never commit those files.
   - Agent: OPENROUTER_API_KEY, AGENT_MODEL (default google/gemini-2.5-flash), VISION_MODEL, AGENT_SHARED_SECRET.
   - Gateway: PHOTON_PROJECT_ID, PHOTON_PROJECT_SECRET, ALLOWED_SENDER (your phone number in E.164 format), AGENT_SHARED_SECRET.
   - Generate a secret with `openssl rand -hex 32` and use the SAME value in both files. Keep port and timeout settings matched.
4. Install and launch the signed CuaDriver app using its official installation instructions: https://cua.ai/docs/cua-driver/quickstart . Grant Accessibility and Screen Recording when prompted. The Python dependency alone does not install the signed app.
5. Start the driver:

```bash
export PATH="$HOME/.local/bin:$PATH"
open -n -g -a CuaDriver --args serve
cua-driver check_permissions
```

6. In one terminal, from the project root:

```bash
export PATH="$HOME/.local/bin:$PATH"
.venv/bin/python -m agent.main
```

7. In another terminal, from the project root:

```bash
cd gateway
npm start
```

8. Open the FULL `Dashboard:` link printed by the agent, including `#token=...`. It changes on restart. The bare `/` URL is not a page.
9. Text your provisioned Photon number from ALLOWED_SENDER.

Optional combined startup after configuration: `bash scripts/start.sh`.

## Try it

- Write a study plan and save it as study-plan.md.
- Make it shorter and save a new version as study-plan-short.md.
- Research three recent uses of AI in sports and save a brief with source links as sports.md.
- Create a 3-slide presentation based on that brief.
- Open Calculator.

Text-file inputs and outputs live in `~/Desktop/RemoteShell Workspace`. Put .txt, .md or .csv inputs there. PPTX files are generated on Desktop. See GEMINI.md for tool details and DASHBOARD.md for the monitor.

## Architecture and limits

Photon → Node gateway → authenticated localhost FastAPI agent → Gemini via OpenRouter → validated tools → result/progress back to iMessage and dashboard.

The gateway authorizes one sender, deduplicates messages and queues tasks. Gemini gets at most eight tool calls per task within the existing 120-second timeout. Follow-up memory retains four recent requests for up to an hour in RAM; text `forget` to clear it. `status`, `cancel`, and `help` are handled by the gateway.

Only workspace text files can be read or created; existing files cannot be overwritten. Allowed apps can be opened or observed; generic UI input is limited to Calculator. No arbitrary shell, signed-in browser automation, PDF/Word reading, purchases, deleting files, or third-party messaging. Original Roblox launch code remains in the repository but is not exposed as a Gemini match-launch tool.

Workspace contents selected for reading and task context are sent to the model provider. Web research uses OpenRouter web search and can incur additional charges. App-window observations may also be sent to the model. Dashboard access uses a separate random per-start token, not the shared gateway secret.

## Verification

Local automated tests cover core task behavior and mocked tools; these do not prove full live Mac integration. The developer's Mac has exercised iMessage, Calculator and presentation workflows, but the combined Gemini workflows still require rehearsal on the target Mac. Dependency/API availability depends on accounts and installed versions.

```bash
(cd gateway && npm run check && npm test)
.venv/bin/python -m unittest discover -s agent/tests -v
```

## GitHub

Commit source, README, lockfile and `.env.example` files. Never commit `.env`, .venv, node_modules, backups, personal documents or keys. Share through a private repository and invite teammates as collaborators. Each teammate configures their own environment.
