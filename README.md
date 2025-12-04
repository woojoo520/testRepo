# testRepo

Got it—you want more architecture, less code. Let’s treat this as a design note you could share with teammates.

---

## 1. Purpose of the Slack + Claude Agent design

We’re building a **Slack-native assistant** whose “brain” is a Claude agent, not just raw LLM calls.

Key behaviors:

* In a given channel:

  1. **Any new root message** (first message of a thread) automatically gets a bot reply in a thread.
  2. **Any reply in that thread that @mentions the bot** also gets a reply.
* **One Slack thread ≈ one Claude agent session**.
* The agent:

  * Remembers what happened earlier in the thread.
  * Can see references to images users upload (we pass metadata / URLs and, optionally, OCR or vision-derived text).
  * Streams responses back to Slack using Slack’s streaming APIs.

So Slack is just the UI surface. The “agentic” behavior — remembering, reasoning, tool use — lives in the Claude Agent SDK.

---

## 2. Conceptual architecture

### 2.1 Three-layer view

You can think of the system as three layers:

1. **Channel / UI layer (Slack)**

   * Receives user messages, uploads, @mentions.
   * Displays streaming responses in threads.

2. **Adapter / Orchestration layer (Slack bot service)**

   * Runs with Slack Bolt (async) + Slack Web API.
   * Responsibilities:

     * Map Slack concepts → agent concepts:

       * `channel + thread_ts` → `session_id`
       * `message event` → “user turn”
       * attached files → “image / file context descriptors”
     * Enforce behavior rules:

       * When to start a new session.
       * When to route to an existing session.
       * When the bot should stay silent.
     * Manage session lifecycle (creation / TTL / eviction).
     * Call the Claude Agent SDK and relay streaming output back to Slack.

3. **Agent layer (Claude Agent SDK)**

   * Each Slack thread is backed by a `ClaudeSDKClient` instance.
   * This instance:

     * Tracks the internal conversation state with Claude.
     * Can access tools / MCP servers (for code, repos, APIs, etc.).
     * Produces streaming text deltas which are forwarded to Slack.

The important design choice:

> The bot service is intentionally “dumb glue” plus routing. **All the interesting reasoning lives in the agent.**

---

## 3. Conversation model: “one thread = one agent session”

### 3.1 Why bind sessions to threads?

Mapping **Slack thread → agent session** gives you:

* **Clear boundaries**
  Each thread is its own topic / mini-project. No accidental context bleed across threads.

* **Natural UX**
  Slack users already treat thread = “this topic”. You’re just attaching a persistent assistant to that topic.

* **Control over memory scope**
  When a thread becomes quiet, you can safely drop its session without risking impact on others.

### 3.2 Trigger rules (when does the agent respond?)

We enforce two simple rules:

1. **New thread → always respond**

   * A user posts a new message in a monitored channel, not part of any existing thread.
   * We treat that as “start a conversation with the bot about this message”.
   * The bot replies in a thread under that message and creates a new agent session.

2. **Existing thread, @bot → respond**

   * Inside an existing thread, if the message text includes `<@BOT_USER_ID>`, we route it to the agent.
   * We strip the mention and treat the remaining text as the user query.
   * The response should @mention the asker (so multi-user threads still feel personal).

All other messages in the thread are **just human conversation**. You can choose:

* Minimal design: ignore them entirely for the agent.
* Better design (what you’re aiming for): **still add them into the “session context”**, so the agent can see the full thread when it’s next called, even if it only responds on explicit triggers.

---

## 4. Session & context strategy

### 4.1 Session identity and TTL

A session key is derived as:

> `session_id = "<channel_id>:<thread_ts>"`

For that `session_id` you maintain:

* A `ClaudeSDKClient` instance.
* A small, structured context log (your own data structure), e.g.:

```text
[
  {role: "user", user: "U123", text: "How do I debug X?"},
  {role: "assistant", text: "You can do A/B/C."},
  {role: "user", user: "U234", text: "Here is a screenshot", image_url: "..."},
  ...
]
```

You then use a TTL-based in-memory cache (e.g. `TTLCache`) so that:

* If **no new events** hit this thread for 4 hours:

  * The entry automatically disappears from memory.
  * Next time someone talks in that thread, a new Claude agent is created with a fresh session.
  * If you want, you can later add a “rehydration” phase using Slack history.

This gives you:

* **Bounded memory usage** (idle sessions fall out of the cache).
* A config knob: `SESSION_TTL` is tunable per environment.

### 4.2 Who stores the “real” conversational state?

There are two layers of “memory”:

1. **Claude Agent’s internal state**

   * The Claude Agent SDK’s `ClaudeSDKClient` object tracks prior exchanges for that session. You don’t need to rebuild the full prompt every turn; you mostly just send incremental user turns.

2. **Your own context log**

   * A compact log of what happened in Slack: who said what, when, with which images.
   * Used for:

     * Debugging (what exactly did the agent see?).
     * Rehydrating sessions if you later add persistence or history fetch.
     * Analytics.

In v1 you can keep the log simple and short (last N turns). The agent still has its own extended context through the SDK; you only need the log for your own operations.

---

## 5. Handling images in the agent design

Slack can attach files (including images) to messages. On the design level:

1. **Detection**

   * For each message event, check if `files` exist.
   * For each file with `mimetype` starting with `image/`, treat it as an image attachment.

2. **Representation to the agent**
   In the minimal version, you don’t do vision in the Slack bot itself. Instead, you:

   * Store metadata into your context log:

     * Uploader user ID
     * File name
     * Slack `url_private` (requires bearer auth)
   * Optionally construct a textual description in the prompt like:

     > “User `<@U123>` uploaded an image: `error_screenshot.png` (Slack URL: …). Please consider that it likely shows the error they mention.”

3. **Later extensions**
   Depending on what Claude Agent SDK exposes and what tools you wire in:

   * Add an OCR/vision tool (external service or MCP server) that:

     * Downloads the Slack image using a bot token,
     * Runs OCR or visual analysis,
     * Returns structured text or description to the agent.
   * Or, once the Agent SDK exposes image content blocks natively, replace the text hack with first-class image inputs.

The design point is: **image events are part of the same thread context**, just another kind of message the agent can reason about.

---

## 6. Why Claude Agent SDK instead of Google ADK for this Slack bot?

You’re basically choosing between:

* A **lightweight, single-/few-agent SDK** (Claude Agent SDK).
* A **full-blown multi-agent orchestration framework** (Google ADK).

For *this* use case, the trade-offs look like this:

### 6.1 Problem shape: one powerful assistant vs many orchestrated agents

* Your Slack bot use case is fundamentally:

  * One UI (Slack),
  * One main assistant per thread,
  * Maybe a handful of tools behind it.

Claude Agent SDK is exactly optimized for that shape:

* It’s built on the same harness as Claude Code — a strong “single agent” that can plan, call tools, iterate, and stream.
* You can extend it with tools / MCP servers as needed, but the mental model stays **one agent talking to one user (or one thread) at a time**.

ADK, by contrast, is built for scenarios like:

* Multi-agent “workforces” with parent–child relationships.
* Agents that delegate to other agents, route tasks, run complex workflows and flows.

You *can* use ADK for a Slack bot, but you’d be bringing a whole multi-agent orchestration platform to solve a problem that so far only needs “one really good assistant”.

### 6.2 Integration and operational overhead

* **Claude Agent SDK**:

  * Drops directly into your existing Python Slack bot service.
  * No need to adopt a new platform — you call the SDK from the same process that listens to Slack.
  * You stay cloud-agnostic; you can run it on your own infra.

* **Google ADK**:
  ADK is intentionally designed to plug into Google’s Vertex AI Agent Builder / Agent Engine story:

  * It’s open-source and model-agnostic, but the “happy path” is Gemini + Vertex AI + Agent Engine.
  * You gain:

    * Standardized workflows,
    * Built-in evaluation frameworks,
    * Managed deployment to Agent Engine with scaling.
  * You also accept:

    * More structure,
    * Typically more moving parts (flows, evaluation, deployment config),
    * Stronger coupling to the GCP stack if you go all-in.

For a single Slack bot you want to iterate on quickly, the additional complexity of ADK is overkill unless:

* Your org already standardizes on ADK for all agent workloads, or
* You know you’ll soon build complex multi-agent flows beyond Slack.

### 6.3 Tooling and ecosystem fit

Claude Agent SDK:

* Very strong at **code-centric workflows and developer productivity**, inherited from Claude Code.
* Deeply integrated with **MCP (Model Context Protocol)**, which is quickly becoming a standard way for LLMs to talk to tools and data sources.
* Great fit if this bot is going to be:

  * A coding buddy,
  * A repo navigator,
  * A glue layer on top of MCP servers (GitHub, Jira, DBs, internal APIs).

Google ADK:

* Comes with strong support for tools and multi-agent coordination, but its sweet spot is building **complex agent systems that may have multiple UIs** (web, contact center, Slack, etc.).
* Excellent choice if your long-term goal is:

  * Many agents collaborating,
  * Heavy evaluation and A/B of agent flows,
  * Centralized governance and observability in GCP.

For your current Slack-first agent, Claude Agent SDK matches the “one main agent per thread” mental model better and requires less infrastructure.

### 6.4 Observability and governance

* ADK shines when you need:

  * Formal evaluation pipelines,
  * Fine-grained tracing of multi-agent flows,
  * A clear path to managed deployment on Vertex AI Agent Engine.

* With Claude Agent SDK:

  * You’re responsible for your own logging, metrics, and tracing.
  * For a Slack bot, that’s often enough: log interactions per thread, track token usage via API usage endpoints, and maybe push basic metrics into Prometheus / Grafana.

Given your goals (Slack bot, internal users, fast iteration), the Claude route is usually the more pragmatic one.

---

## 7. How to explain this choice to others

If you have to justify the design in one paragraph:

> We implement the Slack bot as a thin adapter around Claude Agent SDK. Each Slack thread maps to one Claude agent session, with a TTL-based lifetime and optional context logging. The agent handles reasoning, memory, and tool calls; Slack just provides the UI and events. We chose Claude Agent SDK over Google’s ADK because this problem is “one powerful assistant per thread”, not a complex multi-agent workflow. Claude’s SDK is lighter to integrate, matches our Python stack, and gives us strong coding and MCP tooling out of the box. If, later, we need cross-channel multi-agent orchestration and fully managed evaluation and deployment on GCP, we can revisit ADK or a similar orchestration layer.

If you want, I can turn this into a 1–2 page “design one-pager” with a short problem statement, architecture diagram description, and a crisp “Why Claude Agent over ADK” section tailored for an internal design review.
