A small team of AI agents built with [Hermes Agent](https://hermes-agent.nousresearch.com/docs/) (Nous Research).
Each agent is a separate Hermes **profile** with its own Telegram bot. They work together in one Telegram group by @mentioning each other.

| Agent | Telegram bot | Job | Tools |
| --- | --- | --- | --- |
| Coordinator | `@MDL_ALTER_BOT` | Talks to the human, splits the request into tasks, sends them to the specialists, merges the results, writes the final answer | `todo`, `memory`, `session_search` — no web, no terminal |
| Researcher | `@mdl_researcher_bot` | Searches the web, returns a short summary (≤150 words) with links | `web`, `todo`, `memory` |
| Coder | `@mdl_coder_bot` | Writes Python, **runs it**, returns the code and the real output (and plots) | `terminal`, `file`, `todo`, `memory` |

Use case: a study helper for the MDL course — explain an ML concept and demonstrate it with working code.

## Example flow

```
Human:        @MDL_ALTER_BOT explain L1 vs L2 regularization and show it on sklearn
Coordinator:  @mdl_researcher_bot TASK #1: Explain L1 vs L2 regularization (sparsity, when to use). ≤150 words + links
Coordinator:  @mdl_coder_bot TASK #1: Fit Lasso and Ridge (alpha=1.0) on sklearn's diabetes dataset, print R² and number of zero coefficients
Researcher:   @MDL_ALTER_BOT RESULT #1  <summary>  Sources: ...
Coordinator:  Got research for #1, waiting for code.          ← no @mention → nobody is woken up
Coder:        @MDL_ALTER_BOT RESULT #1  <code>  Output: ...
Coordinator:  <final answer for the human, no bot mentions>  ✅ Done #1   ← chain ends
```

## Repository layout

```
coordinator/SOUL.md       identity + hand-off protocol of the coordinator
coordinator/config.yaml   its Hermes config (model, toolsets, Telegram gating)
researcher/SOUL.md
researcher/config.yaml
coder/SOUL.md
coder/config.yaml
.env.example              which secrets each profile needs (real .env files are NOT in the repo)
```

`config.yaml` files are full Hermes configs. The lines changed from the default are:

| Setting | Value | Why |
| --- | --- | --- |
| `platform_toolsets.telegram` | different per agent (table above) | each agent can only do its own job |
| `group_sessions_per_user` | `false` | one shared session per group, so the coordinator still remembers the question when the specialists' results arrive (with `true` every sender, including other bots, gets its own session) |
| `telegram.require_mention` | `true` | ignore group chatter unless the bot is @mentioned or replied to |
| `telegram.exclusive_bot_mentions` | `true` | if a message mentions bots, only those bots react |
| `telegram.bots_require_mention` | `true` | a message from another bot triggers this bot **only** if it explicitly @mentions it (a quote-reply is not enough) |
| `display.show_reasoning` | `false` | keep the group readable |
| `terminal.cwd` (coder only) | `C:/Users/user/mdl-coder-work` | coder writes and runs scripts in its own folder |

## Setup

Tested on Windows 10 with a recent Hermes Agent. Hermes data lives in `%LOCALAPPDATA%\hermes` on Windows (`~/.hermes` on Linux/macOS).

### 1. Telegram bots

1. In [@BotFather](https://t.me/BotFather) create three bots with `/newbot`.
2. For **each** bot:
   - `/mybots` → bot → *Bot Settings* → *Group Privacy* → **Turn off**;
   - BotFather **Mini App** → bot → enable **Bot-to-Bot Communication Mode** (without it Telegram does not deliver one bot's messages to another).
3. Create a group, add yourself and the three bots (add them **after** step 2 — Telegram applies privacy settings when a bot joins).
4. Get your user id from [@userinfobot](https://t.me/userinfobot) and the group id (negative number) from Telegram Web (`web.telegram.org/a/#-…`).

### 2. Hermes profiles

```bat
hermes profile create coordinator --clone
hermes profile create researcher --clone
hermes profile create coder --clone
```

Copy files from this repo into the profiles:

```bat
for %a in (coordinator researcher coder) do (
  copy %a\SOUL.md      "%LOCALAPPDATA%\hermes\profiles\%a\SOUL.md"
  copy %a\config.yaml  "%LOCALAPPDATA%\hermes\profiles\%a\config.yaml"
)
mkdir C:\Users\user\mdl-coder-work\outputs
```

If your bots have different usernames, replace them in the three `SOUL.md` files.

### 3. Secrets

Append to each `%LOCALAPPDATA%\hermes\profiles\<agent>\.env` (see `.env.example`):

```
OPENROUTER_API_KEY=...            # can be the same for all three
TELEGRAM_BOT_TOKEN=...            # DIFFERENT for each agent
TELEGRAM_ALLOWED_USERS=<your user id>
TELEGRAM_GROUP_ALLOWED_CHATS=<group id>
TELEGRAM_ALLOW_BOTS=mentions
```

Make sure the default profile (`%LOCALAPPDATA%\hermes\.env`) does **not** use any of these bot tokens.

### 4. Run

Hermes runs **one host gateway** that serves every profile:

```bat
hermes gateway            :: keep this window open (or: hermes gateway install && hermes gateway start)
hermes gateway status     :: in another window — coordinator, researcher, coder should be running
```

Coder needs Python with `numpy pandas scikit-learn matplotlib` installed.

### 5. Use

In the group: `@MDL_ALTER_BOT <your question>`.
Logs: `%LOCALAPPDATA%\hermes\profiles\<agent>\logs\gateway.log`.

## How it works

**Why three agents.** Each agent gets a narrow prompt and only the tools for its job. The coordinator literally cannot search or run code, so it must delegate; the coder cannot browse, so its answer is based on code it actually ran; the researcher cannot run code. Hand-offs are visible in the chat, so it is easy to see which step failed. One agent with all tools would be cheaper and faster (one context, no hand-offs), but its prompt would be bigger, nothing would force it to verify by running code, and the work would not be visible.

**Hand-off and completion.** The protocol lives in `SOUL.md`. The coordinator sends `@bot TASK #n: …` with everything the specialist needs (specialists did not read the human's message). Specialists answer with exactly one message `@MDL_ALTER_BOT RESULT #n …` or `ERROR #n …`. The coordinator tracks open tasks with its `todo` tool; a task is finished when every specialist it called has returned RESULT/ERROR for that number, then it posts the final answer ending with `✅ Done #n`.

**Why bots don't reply to each other forever** (several layers):
1. Telegram: bots see other bots' messages only with Bot-to-Bot mode enabled.
2. Hermes gating: `TELEGRAM_ALLOW_BOTS=mentions` + `bots_require_mention: true` — a bot message wakes this bot only if it explicitly @mentions it; `exclusive_bot_mentions` — only the mentioned bot wakes.
3. Protocol in `SOUL.md`: specialists send exactly one message per task and mention only the coordinator; the coordinator's status lines and final answer contain no bot mentions; max 2 follow-ups per task.
4. Hermes `bot_loop_guard` (on by default): after 20 bot messages in one chat within 5 minutes, bot messages are dropped for 10 minutes.

**Model.** All three agents use `nvidia/nemotron-3-ultra-550b-a55b:free` via OpenRouter: free, large mixture-of-experts model, 1M context, good at tool calling. Downsides: free tier is rate-limited and sometimes unavailable, and three agents share one key. A smaller model tends to break the protocol — forget the `TASK #n`/`RESULT #n` format, mention the wrong bot (or no bot, so the chain stalls), or make up a result instead of calling a tool.

**SOUL.md vs skills vs tools.** `SOUL.md` is the first part of the system prompt: who the agent is, its rules and message format. Tools (`platform_toolsets`) define what the agent *can* do — removing a tool is a hard guarantee. Skills are reusable step-by-step procedures the agent loads when needed. A rule in `SOUL.md` is only a request the model may ignore; a missing tool cannot be used.

**Memory.** Every profile has its own storage under `profiles/<agent>/`: `memories/MEMORY.md` and `memories/USER.md` (short notes added to every new session's prompt), `state.db` (all past sessions and messages, searchable with `session_search`), and the current group session. Profiles do not share memory with each other.

**Gateway.** One Hermes process (the host gateway) polls all three bots and routes each update to the matching profile. Two gateways on the same token would steal each other's updates, so Hermes refuses to start a second one.

## Known failures and limits

- **Provider unavailable / rate limit** on the free model — Hermes retries up to 5 times, the task stalls meanwhile. Fix: paid model or a fallback provider.
- **Specialist forgets the @mention** → the coordinator is never woken and the chain stops. Fix: stricter format in `SOUL.md`, stronger model.
- **Coordinator answers by itself** instead of delegating → tools removed from the coordinator.
- **Bot messages not delivered** → Bot-to-Bot mode or privacy setting not applied; remove and re-add the bot to the group.
- Coder runs code directly on the host machine (`terminal.backend: local`); a Docker backend would isolate it.

<!-- Add a screenshot of a real failure here before the defense. -->
