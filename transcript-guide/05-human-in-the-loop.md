# Human-in-the-Loop (HITL)

## Why Human-in-the-Loop?

For sensitive actions (e.g. sending an email or scheduling a meeting), you often don’t want the agent to act without approval. **Human-in-the-loop (HITL)** means the graph **pauses** at certain points, shows the human something (e.g. a proposed email or a question), and **resumes** only after the human responds (approve, edit, ignore, or give feedback).

In this course, HITL is implemented with LangGraph **interrupts** and an interface called **Agent Inbox** that displays interrupted threads and sends the user’s choices back to the graph.

---

## Where We Add Interrupts

Two main places:

1. **After triage: “notify”** – When the router decides “notify the user” (instead of respond or ignore), we don’t just end. We **interrupt** and show the email in Agent Inbox so the user can choose: “respond after all” or “ignore.”
2. **Before sensitive tool runs** – Before executing tools like **write_email** or **schedule_meeting**, we **interrupt** and show the proposed arguments (e.g. subject and body of the email, or meeting time and duration). The user can then **accept**, **edit**, **ignore**, or give **feedback** in natural language.

So: one interrupt for “notify” triage, and one interrupt per sensitive tool call we want to gate.

---

## The Interrupt Payload (Request Object)

When you call `interrupt(...)` in a node, you pass a **payload**. That payload is what Agent Inbox (or your own UI) uses to render the request and the actions the user can take.

A typical shape (conceptually):

- **action** – A short title (e.g. “email_assistant_notify” or the tool name like “write_email”). Shown as the main label in the inbox.
- **config** – Options the user is allowed: e.g. `["ignore", "respond"]` for notify, or `["ignore", "respond", "edit", "accept"]` for write_email. These map to buttons or actions in the UI.
- **description** – The content to show (e.g. the email body, or a summary of the tool arguments). This is what the user reads before deciding.
- (Optional) **tool call name and arguments** – For tool interrupts, so the UI can show “this is what the agent wants to do” and, for “edit,” pre-fill the fields.

So: **you** define this structure when building the interrupt; **Agent Inbox** (or your front end) renders it and sends back a **response** object when the user acts.

---

## Response Object (What the User Sends Back)

When the user interacts with the inbox, the graph expects a **response** object. A simple format used in the course is a dict with:

- **type** – Kind of action: e.g. `"accept"`, `"edit"`, `"ignore"`, `"response"` (natural-language feedback).
- **args** – The payload: for `"edit"`, the **edited** tool arguments; for `"response"`, the **text** the user wrote.

Your interrupt-handler node reads this and then:

- **accept** → Run the tool with the original arguments; add a tool message; continue.
- **edit** → **Update the tool call in message history** with the edited args, then run the tool with those args, then add a tool message. (See below why we edit the message too.)
- **ignore** → Don’t run the tool; end the thread (or go to end).
- **response** → Append the user’s text as a message (e.g. as a “tool” or user message indicating feedback) so the **agent** can see it and, on the next turn, adjust its behavior (e.g. call the tool again with different args).

---

## Why Edit the Tool Call in Message History?

If the user **edits** the tool arguments (e.g. changes meeting duration from 45 to 30 minutes), you should:

1. **Update the tool call** in the message list so it contains the **edited** arguments (e.g. 30 minutes).
2. **Run the tool** with those same edited arguments.
3. **Add the tool message** with the result.

If you only ran the tool with 30 minutes but left the message history showing a 45-minute tool call, the **next** time the model sees that history it would be inconsistent (“I said 45 but the result says 30”) and can get confused. So: **keep message history consistent** with what was actually executed by editing the tool call in state when the user edits.

---

## Triage Interrupt Handler (Notify)

When triage decides “notify”:

1. You go to a **triage_interrupt_handler** node (instead of ending).
2. That node calls **interrupt** with a request object: action like “email_assistant_notify,” config `["ignore", "respond"]`, description = the email (or summary).
3. When the user responds:
   - **respond** → You add a message like “the user wants to respond after all” (and maybe the email) to state, then route to the **response agent**.
   - **ignore** → You go to **end**.

So the human can “upgrade” a “notify” to “respond” or confirm “ignore.”

---

## Tool Interrupt Handler (Write Email, Schedule Meeting, Question)

For tools like **write_email** and **schedule_meeting**:

1. After the LLM outputs a tool call, you don’t run the tool immediately. You go to an **interrupt handler** node.
2. The handler builds a request object (action = tool name, config = accept / edit / ignore / response, description = email or meeting details, plus tool args).
3. It calls **interrupt** and waits.
4. When the user responds:
   - **accept** → Run tool with current args; add tool message; continue (e.g. back to agent or to “done” logic).
   - **edit** → Replace the tool call’s args in state with the user’s edited args; run tool with those args; add tool message; continue.
   - **ignore** → Don’t run; go to end.
   - **response** → Add user’s text as a message so the agent can see it and possibly call the tool again (or call “done”).

For a **question** tool (agent asks the user something), you might only offer “respond” and “ignore” (no “edit”/“accept” of a tool call). The user’s “respond” is then the answer you inject into the conversation so the agent can continue.

---

## Which Tools Get Interrupted?

You maintain a list of “sensitive” tool names (e.g. `write_email`, `schedule_meeting`, `question`). In the tool-handling logic:

- If the tool is **in** that list → go to the **interrupt handler** (which formats the request and calls `interrupt`).
- If not (e.g. `check_calendar_availability`, `done`) → run the tool as usual and continue.

So only the tools you care about gating go through the human.

---

## Agent Inbox and Local Deployment

- **Local deployment** – When you run `langgraph dev`, threads (and their state) are stored **locally** (e.g. in a `langgraph_api` directory or similar). Interrupted threads are just saved state waiting for a resume.
- **Agent Inbox** – A UI that **lists** those interrupted threads (reading from the local server or, in production, from the hosted API). When you open one, it shows the **request** you built (action, description, config). Buttons map to **response** types (accept, edit, ignore, response). When you click one, Inbox sends a **resume** request to the graph with the appropriate **command** (e.g. `resume` with the response object). The graph then continues from the interrupt with that payload.

So: **Inbox** = viewer for interrupted threads + way to send “resume with this user response.” No special protocol—just “resume” + the same response shape your interrupt handler expects.

---

## Using the SDK to Resume (Same as Inbox)

In Python you can do the same as Inbox: after `invoke` returns an interrupt, you call `invoke` again with a **command** object that says “resume” and passes the response (e.g. `{"type": "accept"}` or `{"type": "edit", "args": {...}}`). Your interrupt-handler node receives that and branches accordingly. So you can test HITL from code or from the Inbox UI; the graph logic is the same.

---

## Summary

- **HITL** = pause at chosen points (after “notify,” before sensitive tools), show the user a request, resume with their response.
- **Interrupt payload** = request object (action, config, description, maybe tool args) that your UI (e.g. Agent Inbox) renders.
- **Response** = object with `type` (accept / edit / ignore / response) and `args` (edited args or feedback text). Handler uses this to run the tool, edit the tool call in state, end, or add feedback for the agent.
- **Edit** = update the tool call in message history to the edited args, then run the tool, so history stays consistent.
- **Agent Inbox** = interface to interrupted threads; it sends resume + response to your graph.

Next: [Memory](./06-memory.md) — how the agent remembers preferences across threads using the store.
