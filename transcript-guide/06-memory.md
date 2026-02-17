# Memory

## Two Kinds of Memory

1. **Thread-scoped (short-term)** – Conversation and state **within one run or one thread**: messages, tool calls, current triage decision. This is what the **checkpointer** saves so you can pause and resume. It’s “memory for this conversation.”
2. **Across-thread (long-term)** – Information that should **persist across many threads** (e.g. “prefer 30-minute meetings,” “keep replies concise”). That’s what we use the **store** for: the agent can read and update preferences that apply to **all** future runs.

For an email assistant, thread-scoped memory is the current email and the back-and-forth in that thread. Long-term memory is “how this user likes to be triaged, how they like replies, and their calendar preferences.”

---

## The LangGraph Store

The **store** is a key–value style storage built into LangGraph (or provided by the platform). You use it for **long-term** memory.

- **Namespace** – You organize entries by a **namespace** (e.g. a tuple like `("user_preferences", "calendar")` or a string). That way you can have separate “buckets”: triage preferences, response preferences, calendar preferences.
- **Operations** – You **put** (namespace, key, value) and **get** or **search** (namespace, …) to read. Values can be strings or more structured data.
- **Where it runs** – In notebooks you might use an **in-memory** store (e.g. a dict) that disappears when the kernel restarts. For `langgraph dev` locally, the platform often uses a **persistent** store (e.g. pickled on disk). For **hosted** deployment, the store is typically in **Postgres**, so it’s durable and shared.

You **compile** your graph with both a **checkpointer** (thread-scoped) and a **store** (long-term) when you want HITL + memory. The platform can inject these for you in deployment so you don’t have to create them manually in the script.

---

## How We Use Memory in the Assistant

We keep **three** preference categories (each in its own namespace or key):

1. **Triage preferences** – e.g. “when in doubt, notify me” or “ignore newsletters.”
2. **Response preferences** – e.g. “be concise,” “less formal.”
3. **Calendar preferences** – e.g. “prefer 30-minute meetings,” “suggest a single time.”

Each is stored as a **string** (or a short structured blob) that we **overwrite** when we update (simple approach). The agent (and triage) **read** these strings and inject them into the **system prompt** (or context) so the model behaves according to stored preferences.

---

## When Do We Update Memory?

We update memory **only when the user gives feedback** through HITL, not on every run. For example:

- User **edits** a meeting (e.g. 45 → 30 min) → update **calendar preferences** (e.g. “prefer 30-minute meetings”).
- User **edits** a drafted email** (e.g. makes it shorter) → update **response preferences** (e.g. “prefer concise replies”).
- User **ignores** a “notify” email → update **triage preferences** (e.g. “emails like this can be ignored”).
- User chooses **“respond”** on a “notify” item → update **triage preferences** (e.g. “when notified about this type, user may want to respond”).
- User gives **natural-language feedback** (e.g. “make it less formal”) → update the relevant preference (response or calendar) based on which tool it was.

So: **human feedback** (edit, ignore, respond, or free-text) triggers an **update step** that writes back to the store.

---

## How We Update Memory (LLM Step)

We don’t hard-code rules like “if they edited duration then add ‘prefer 30 min’.” Instead we use an **LLM** to interpret the feedback and produce an **updated** preference string:

1. **Inputs** – Current preference string for that namespace (e.g. current “calendar preferences”), plus the **messages** that contain the feedback (e.g. the tool call, the edit, or the user’s text).
2. **Prompt** – Instructions like: “Here is the current preference profile. Here is the user’s feedback (from message history). Reflect on it and output an **updated** profile that incorporates this feedback. Don’t drop existing important points; only add or refine.” You can add reasoning steps or examples.
3. **Structured output** – e.g. `updated_profile: str`, `justification: str` so you can log why the profile changed.
4. **Write** – Save `updated_profile` to the store under that namespace/key, **overwriting** the previous value (in this simple design).

So the **model** generalizes from one edit or one piece of feedback into a reusable preference. You can tune the prompt and the schema (e.g. allow a list of bullet points instead of one big string) later.

---

## Where in the Graph We Update Memory

- **Triage interrupt handler** – When the user chooses “respond” or “ignore” on a “notify” item, call the “update memory” helper for **triage preferences** with the current messages (and maybe a short hint like “user chose to respond” or “user chose to ignore”).
- **Tool interrupt handler** – When the user **edits** or **ignores** or gives **feedback** on write_email or schedule_meeting, call the update-memory helper for **response** or **calendar** preferences with the relevant messages (tool call + edit or feedback). Optionally also update triage when they ignore a tool call (e.g. “don’t auto-respond to this kind of email”).

We **don’t** update memory when the user simply **accepts** a tool call with no changes—there’s no new signal to learn from.

---

## Reading Memory Into the Prompt

In the **LLM node** (and optionally in triage), before building the system prompt:

1. **Read** from the store for the namespaces you care about (e.g. triage, response, calendar).
2. If nothing is there, use **default** strings (e.g. “No specific preferences yet” or a short default policy).
3. **Inject** those strings into the system prompt (e.g. “User preferences: …”) so the model sees them every time.

So the agent and triage always act with the **latest** stored preferences. As the user gives more feedback, the stored strings get updated and the agent behaves more personalized.

---

## Initializing the Store (Defaults)

When the graph first runs for a user (or thread), the store might be empty. You can **initialize** it with default preference strings (e.g. “Prefer 30-minute meetings; 15 minutes acceptable”) so the first run already has something to show and to update. Your “get or default” helper can do: if no value in store for this namespace, write the default and return it.

---

## Viewing Memory in Studio

When running with `langgraph dev`, LangGraph Studio can show a **Memory** (or Store) tab. You can inspect the current keys and values (e.g. the three preference strings). After you give feedback and the graph runs the update step, you’ll see the stored values change. In LangSmith traces you can see the “update memory” node: the prompt, the model output (updated profile + justification), and the write to the store.

---

## Scaling and Alternatives

The course uses a **simple** design: one string per category, overwritten each time. That can work for a small number of preferences but may not scale to hundreds of facts. The transcript mentions:

- **Semantic search** over a **collection** of memories (e.g. store many snippets and retrieve by similarity) instead of one overwritten string.
- **LangMem** and other primitives for more advanced memory schemas and management.

So the idea here is “preferences as strings, updated by an LLM from feedback”; you can later replace that with a richer memory layer (e.g. LangMem or a vector store) while keeping the same “read at start of run, update on feedback” pattern.

---

## Summary

- **Thread-scoped memory** = checkpointer (conversation/state per thread). **Long-term memory** = store (preferences across threads).
- **Store** = namespace + key + value; read at run start, write when user gives feedback.
- **Update** = run an LLM on “current profile + feedback messages” → get updated profile (+ optional justification) → write to store.
- **Use** = inject stored preferences into the system prompt so triage and the agent follow them.
- **When to update** = on edit, ignore, “respond,” or natural-language feedback in the interrupt handler; not on plain “accept.”

Next: [Deployment and Gmail](./07-deployment-and-gmail.md) — local vs hosted deployment, Gmail setup, and connecting to Agent Inbox.
