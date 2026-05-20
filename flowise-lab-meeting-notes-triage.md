# Flowise Workshop — Lab: Meeting Notes Triage Agent

_End-to-end flow: trigger (we're faking this with a form) → summarise → policy check → draft → notify_ (this would be the logical thing to do using teams/slack/come sort of notifcation)

## What you will build

A Flowise Agentflow that, given a meeting-notes email, summarises it, checks the content against a small policy knowledge base, and drafts a follow-up email to the team.

### The five logical stages

1. Trigger: detect an incoming email containing the words "meeting notes"
2. Summarise: extract key points, decisions, action items, attendees
3. Policy check: retrieve relevant policy snippets and flag inconsistencies
4. Draft: compose a follow-up email and place it in Gmail Drafts
5. Notify: (this would be the logical next step but we can't do it today) alert the user (Slack, email, or chat) that a draft is ready

## Important: how triggering really works

Flowise itself can't poll (ping/check) Gmail for new emails. Its Start node accepts input from either the chat interface or a form. To trigger on an incoming Gmail message in production, you would need to put a thin polling layer in front of Flowise (n8n, Make, or Zapier) that "watches" Gmail or whatever email provider you use and calls the Flowise prediction API when a match is found to launch the flow.

For this workshop, we will simulate the trigger using a Form Input on the Start node. This lets you focus on the agent logic in the time we have. The final section of this document shows exactly how to wire up the real Gmail trigger via n8n once you are back at your desk.

> **Dream Architecture in one line (if we had all the tooling and no blockers)**
>
> Gmail → n8n (polls every 1 min, filters by subject) → HTTP POST to Flowise /api/v1/prediction/{id} → Flowise agentflow runs → Gmail draft created → Slack message sent.

## Prerequisites

- Flowise account (free tier is fine)
- OpenAI API key configured in Flowise credentials
- Three sample policy documents (provided — see Appendix A at the bottom of this file)
- A sample meeting-notes email body (provided — see Appendix B)

> **Model choice**
>
> Use gpt-4o-mini for the summary and policy-check steps to keep costs low. Use gpt-4o for the email drafting step where tone and structure matter most. Do not default to the most expensive model for everything.

## Part 1 — Build the policy knowledge base (25 min)

Before we build the agent, the knowledge base needs to exist. The agent will retrieve from this in step 3 of the flow. Flowise's Document Store guides you through five steps: pick a loader, configure a splitter, preview, process, then upsert. We'll use Pinecone as the vector store — free tier, no credit card required.

> **Heads-up: you'll need your own Pinecone account**
>
> Pinecone free orgs are per-email. Everyone in the room will need to sign up individually with their own email. Sign-up is fast (no credit card) - just put that you're a hobby dev or exploring/experimenting. You're using low/no code tools with flowise.

### 1.1 Create a Pinecone account and index

1. Go to [app.pinecone.io](https://app.pinecone.io) and sign up (use a personal email — workshop only)
2. Once logged in, click **+ Create index**
3. **Index Name**: `meeting-policies`
4. **Configuration**: choose **Custom settings** (not the suggested embedding model option)
5. **Dimensions**: `1536` (this matches text-embedding-3-small — the embedding model we'll use in Flowise)
6. **Metric**: `cosine`
7. **Capacity mode**: Serverless
8. **Cloud provider**: AWS, **Region**: `us-east-1` (the only option on the free tier)
9. Click **Create index**. It takes ~30 seconds to provision.

### 1.2 Get your Pinecone API key - Copy it straight away and save it somewhere!!!!

1. In the Pinecone console left sidebar, click **API Keys**
2. Copy the default API key (it starts with `pcsk_...`) — you'll paste it into Flowise shortly

> **Keep this key handy**
>
> You'll need it in step 1.7. If you close the console, you can always come back to API Keys to re-copy it.

### 1.3 Create a Flowise Document Store

1. In the Flowise sidebar, click **Document Stores**
2. Click **+ Add New**
3. Name it: `meeting-policies`
4. Description: `Policy documents that meeting notes are checked against`
5. Click **Add**

### 1.4 Select a Document Loader and add the files

You should have the three .txt policy files ready (`travel-expense-policy.txt`, `decision-authority-policy.txt`, `vendor-engagement-policy.txt`). If not, grab them from the apendix at the bottom of this file (or from your instructor).

1. Click into the `meeting-policies` store you just created
2. Click **+ Add Document Loader**
3. From the loader list, select **Text File**
4. Upload all three .txt files into the **Txt File** field
5. Leave the metadata field empty (we are not filtering by metadata in this workshop)

### 1.5 Configure the Text Splitter

On the same configuration page, scroll down to the Text Splitter section.

1. Click **Select Text Splitter** → **Recursive Character Text Splitter**
2. **Chunk Size**: `2000`
3. **Chunk Overlap**: `200`
4. Click **Preview Chunks** — each of the three policy documents should appear as a single, coherent chunk
5. Click **Save**

> **Why chunk size 2000, not something smaller**
>
> The three policy files are short — well under 2000 characters each. A chunk size of 2000 keeps each whole policy document in one piece. This matters because the policies use repeating clause numbers (2.1, 2.2) across all three files: a small chunk size slices a policy mid-document and stitches fragments from different policies together, producing incoherent chunks where clauses are attributed to the wrong policy. The compliance check downstream then works against garbled rules. Small chunks are the right choice for long documents (annual reports, regulations) where one chunk could never hold the whole thing — but for a handful of short, self-contained policy documents, one-policy-per-chunk is correct. This trade-off — chunk to fit the document's natural structure, not to a fixed number — is one of the most important practical lessons in RAG. Demonstrate it: set chunk size to 300, preview, and show the room how the clauses fragment.

### 1.6 Process the data

Saving the loader takes you back to the document store overview. The loader will show its chunks but they are not yet in a vector store.

1. Click the loader row to open it
2. Click **Process** at the top right
3. Wait for the chunks to appear in the list — with chunk size 2000 you should see roughly 3 chunks, one per policy document

### 1.7 Configure embeddings and vector store

Now go back to the document store overview (breadcrumb at the top).

1. Click your store and then click the options list and and click **Upsert Chunks**
2. You will see three cards: **Select Embeddings**, **Vector Store** and **Record Manager**

#### Select Embeddings card

1. Click the **Select Embeddings** card
2. Choose **OpenAI Embeddings**
3. **Connect Credential**: select your existing OpenAI credential (or click Create New and paste your OpenAI API key)
4. **Model Name**: `text-embedding-3-small`
5. Leave all other fields at defaults
6. Click the back arrow / Save

> **Dimension match is critical**
>
> text-embedding-3-small produces 1536-dimensional vectors. Your Pinecone index was created with 1536 dimensions. These must match exactly — if they don't, upsert will fail with a dimension mismatch error. If you accidentally created the index with a different dimension count, delete it in the Pinecone console and recreate it with 1536.

#### Select Vector Store card

1. Click the **Select Vector Store** card
2. Choose **Pinecone**
3. **Connect Credential** → **Create New**
4. **Credential Name**: `pinecone-workshop` (or anything memorable)
5. **Pinecone API Key**: paste the `pcsk_...` key you copied earlier (you can also add this to the credentials as we did with the open AI API key if you want to be consistent)
6. Click **Add** to save the credential
7. **Pinecone Index**: `meeting-policies` (must exactly match the index name you created in Pinecone)
8. **Pinecone Namespace**: leave blank (namespaces let you partition data within an index — not needed here)
9. **Top K**: set this to `4`. This is where retrieval depth is set — the Retriever node in Part 2 has no Top K field of its own, it inherits this value. With only ~3 policy chunks in the index, a Top K of 4 guarantees every policy is retrieved on every run, which is what you want for a compliance check. Leave Search Type and other advanced fields at defaults.
10. Click the back arrow / Save

#### Select Record Manager card

Skip this — leave it empty. Record Managers are useful for production when you re-upsert frequently and want to avoid duplicates, but unnecessary here.

### 1.8 Upsert

1. Click **Upsert** at the top right
2. Wait for the success message showing the number of vectors added (should be ~3, matching your chunk count)
3. **Verify in Pinecone**: go back to the Pinecone console → click your `meeting-policies` index → you should see the vector count populate within a minute
4. Back in Flowise, click **Retrieval Query** to test
5. Try the query: _"What is the limit for sole-source vendor procurement?"_ — you should see the relevant vendor-engagement-policy chunk in the top results

> **Troubleshooting upsert errors**
>
> **Dimension mismatch**: delete and recreate the Pinecone index with dimensions = 1536.
>
> **Index not found**: check the index name in Flowise exactly matches the one in Pinecone (case-sensitive, no leading/trailing spaces).
>
> **Invalid API key**: re-copy from the Pinecone console — keys can be regenerated and you may have an old one.
>
> **Quota exceeded**: very unlikely for a workshop, but if it happens, the free tier allows 5 indexes per org. Delete unused indexes in the Pinecone console.

> **Pinecone index "not ready" error**
>
> Pinecone serverless indexes can take 30–60 seconds to become queryable after creation. If you try to upsert immediately after creating the index, it may fail. Wait until the index shows as "Ready" in the Pinecone console before going back to Flowise.

## Part 2 — Build the Agentflow (90 min)

We will build this as a single Agentflow V2 with six nodes wired in sequence. Each node does one job. Build it one node at a time and test after each addition — do not wire everything up and try to debug at the end.

### 2.1 Create the flow

1. Sidebar → **Agentflows** → **+ Add New** → **Agentflow V2**
2. Name it: `meeting-notes-triage`

### 2.2 Configure the Start node

A Start node is already on the canvas. Click it to configure.

1. **Input Type**: Form Input
2. **Form Title**: `Meeting Notes Triage`
3. **Form Description**: `Paste the body of a meeting-notes email below`
4. Under **Form Input Types**, add a field: **Type** = String, **Label** = `Email body`, **Variable Name** = `emailBody`
5. Add a second field: **Type** = String, **Label** = `Sender name`, **Variable Name** = `senderName`

> **How form fields are referenced later**
>
> Form field values are available to every downstream node as `{{ $form.emailBody }}` and `{{ $form.senderName }}` — using the Variable Name you set above.

### 2.3 Node 1 — Summarise the email

1. From the canvas **+** button, add an **LLM** node. Connect Start → LLM
2. Rename the node to `Summarise Email` (click the node name to edit it — do this for every node, it makes variable references readable later)
3. **Model**: ChatOpenAI, model name `gpt-4o-mini`
4. Expand the model's additional parameters and set **Temperature** to `0.2`

#### Add the System message

In the **Messages** section, the node starts with one message row. Set its **Role** to **System** and paste this into the content:

```
You are a meeting-notes analyser. Given the body of an email containing meeting notes, produce a structured summary.

Return your answer using exactly these headed sections, in this order:

MEETING TITLE:
DATE:
ATTENDEES:
KEY DISCUSSION POINTS:
DECISIONS:
ACTION ITEMS:
OPEN QUESTIONS:
MONETARY AMOUNTS:
VENDOR MENTIONS:

For ACTION ITEMS, put each item on its own line as: owner — task — deadline.
For MONETARY AMOUNTS, list each sum mentioned and what it relates to.
For VENDOR MENTIONS, list any third-party vendors, suppliers or external companies named.
If a section has no content, write "None" under that heading.

Be precise. Do not invent details — only include what is actually in the email.
```

#### Set the Input Message

Scroll down to the **Input Message** field (this is the most-recent user message appended to the conversation). Enter:

```
Email from: {{ $form.senderName }}

Meeting notes:
{{ $form.emailBody }}
```

Type `{{` in the field to open the variable picker and select `$form.senderName` and `$form.emailBody` rather than typing them by hand — the picker guarantees the exact reference.

> **Why temperature 0.2**
>
> Extraction tasks reward determinism. You want the same input to produce the same output every time so downstream nodes can rely on it. Save the higher temperatures for the email-drafting step where you want some natural phrasing variation.

### 2.4 Node 2 — Policy retrieval

This node queries the knowledge base for policy snippets relevant to the meeting content.

1. Add a **Retriever** node. Connect Summarise Email → Retriever
2. Rename it to `Retrieve Policies`
3. **Knowledge (Document Stores)**: select `meeting-policies`
4. **Output Format**: `Text`
5. **Retriever Query**: see below

In the **Retriever Query** field, type `{{` and select the output of the Summarise Email node from the picker. The retriever needs a single text query, so we feed it the whole summary — the decisions, monetary amounts and vendor mentions in it are what drive the semantic match against the policy chunks. The field should end up containing something like:

```
{{ summariseEmail }} or whatever flowise has for the output of that step (probably llmflow_0 or something like that)
```

### 2.5 Node 3 — Policy check

1. Add an **LLM** node. Connect Retrieve Policies → LLM
2. Rename it to `Policy Check`
3. **Model**: ChatOpenAI, `gpt-4o-mini`, **Temperature** `0.1`

#### System message

Set the first message Role to **System** and paste:

```
You are a compliance reviewer. You will be given (a) a meeting summary and (b) excerpts from company policy documents.

Identify any inconsistencies, policy breaches, or items requiring escalation. Be specific, and for each issue name the policy it is in tension with.

Only flag what the retrieved policy excerpts actually say. Do not invent or assume policy text that is not in the excerpts. If the excerpts do not support a concern, do not raise it.

Return your answer using exactly these headed sections:

OVERALL STATUS:
(one of: clear, flags-found, blocking-issues)

FLAGS:
(each issue on its own line, formatted as: SEVERITY — issue — policy reference — recommended action. SEVERITY is one of info, warning, blocker. If there are no issues, write "None".)

NOTES FOR DRAFT:
(a short plain-language note for whoever drafts the follow-up email, summarising what needs confirming)
```

#### Input Message

In the **Input Message** field, build the following — use the `{{` picker to insert each node output:

```
MEETING SUMMARY:
{{ summariseEmail }} (again, this will be what flowise has assigned the variable name liee llmflow_0 or something similar)

RETRIEVED POLICY EXCERPTS:
{{ retrievePolicies }} (same thing as above - this will be the second llm nodes output)
```

> **Why "do not invent policy text" matters**
>
> The most common failure mode in RAG-based compliance work is the model hallucinating policy language that sounds plausible. The system message above pushes the model to ground each flag in retrieved text. Test it by running an email about something not covered by the policies — a correct implementation returns OVERALL STATUS: clear.

### 2.6 Node 4 — Draft the follow-up email

1. Add an **LLM** node. Connect Policy Check → LLM
2. Rename it to `Draft Follow-up`
3. **Model**: ChatOpenAI, `gpt-4o`, **Temperature** `0.6`

#### System message

```
You are drafting a follow-up email on behalf of the meeting organiser to the team members who attended.

The email should:
- Open with a brief thank-you for attending
- Recap the key decisions in 2-3 bullet points
- List action items clearly, with owners and deadlines
- If any policy flags were raised, add a "Points to confirm" section that raises them tactfully — do not accuse anyone, just flag for confirmation
- Close with clear next steps

Tone: professional, concise, suitable for a consulting team. No filler. Do not open with "I hope this email finds you well."

Return your answer using exactly these headed sections:

SUBJECT:
RECIPIENTS:
(comma-separated attendee names, excluding the sender)
BODY:
(the full email body, ready to send)
```

#### Input Message

```
SENDER (drafting on their behalf): {{ $form.senderName }}

MEETING SUMMARY:
{{ summariseEmail }}

POLICY CHECK RESULT:
{{ policyCheck }}
```

## Part 3 — Test the flow (30 min)

### 3.1 Clean test

Run the flow with the sample email from Appendix B (the basic one). Expected behaviour:

- Summarise Email returns a summary with headed sections — attendees, decisions, action items, etc.
- Retrieve Policies pulls back all ~3 policy chunks, each a complete, coherent policy document
- Policy Check returns OVERALL STATUS: clear, or one minor info flag
- Draft Follow-up produces a coherent email
- Slack/chat notification fires

### 3.2 Flagged test

Now run the second sample from Appendix B (the one mentioning a £15,000 sole-source vendor decision and a Friday-afternoon flight in business class). Expected behaviour:

- Policy Check returns OVERALL STATUS: flags-found with at least 2 flags
- Flagged severity should include at least one "warning" or "blocker"
- Draft email contains a "Points to confirm" section referencing the issues tactfully
- Slack notification shows non-zero flag count

### 3.3 Inspect the execution trace

After each run, you can inspect the input and output of each individual node by inspecting in the chat and clicking the expansion icon on each step. Take a look at the I/Os of each node and trace the data through the flow.

**Consider:**

- What each node received as input — was the variable interpolation correct?
  If something doesn't work - you can use this trace method to find where the flow breaks.

> **This is the most important habit**
>
> Reading traces is 80% of debugging agent flows. Get into the habit now. When you ship something to a client and it misbehaves, the trace is the only place that tells you what actually happened — model logs alone will not.

## Appendix A — Sample policy documents

Save each as a separate .txt file.

### `travel-expense-policy.txt`

```
POLICY: Travel and Expense Reimbursement

1. Flight bookings
1.1 All domestic flights must be booked in economy class regardless of duration.
1.2 International flights over 6 hours may be booked in premium economy with line-manager approval.
1.3 Business class is only permitted for flights over 10 hours and requires Partner-level approval.
1.4 Bookings should be made at least 14 days in advance where possible.

2. Accommodation
2.1 Maximum nightly rate: £200 in UK regional cities, £300 in London, £350 international.
2.2 Anything above these caps requires written approval from the client engagement Partner.

3. Meals
3.1 Daily meal allowance: £50 UK, £70 international.
3.2 Client entertainment must be pre-approved and logged separately.

4. Approval workflow
4.1 All travel must be booked through the corporate travel system.
4.2 Reimbursements outside this system will not be processed without exceptional justification.
```

### `decision-authority-policy.txt`

```
POLICY: Decision-Making Authority Matrix

1. Spend authority
1.1 Up to £5,000: Senior Manager
1.2 £5,000 to £25,000: Director, with documented business case
1.3 £25,000 to £100,000: Partner, with written approval recorded in the engagement file
1.4 Above £100,000: Engagement Partner plus Practice Lead, joint sign-off required

2. Vendor selection
2.1 Single-source procurement is only permitted where (a) sole supplier of required capability, or (b) below £10,000 total contract value.
2.2 For all other vendor engagements, at least three quotes must be obtained.
2.3 Any sole-source decision above £10,000 requires written justification and Partner approval.

3. Client commitments
3.1 Verbal commitments to clients regarding scope changes are not binding until confirmed in writing by the engagement Partner.
3.2 Action owners assigned in client meetings must be confirmed via follow-up email within 24 hours.
```

### `vendor-engagement-policy.txt`

```
POLICY: Third-Party Vendor Engagement

1. Onboarding requirements
1.1 All new vendors must complete a due diligence questionnaire before any contract is signed.
1.2 Information security review is required for any vendor that will process client data.
1.3 GDPR data processing addendum required for vendors handling EU personal data.

2. Engagement timing
2.1 Vendor engagements should not commence until contracts are fully executed.
2.2 Work performed before contract execution is at the firm's own risk and is not reimbursable.

3. Conflict of interest
3.1 Any personal relationship (familial, financial, or material professional) between a staff member and a vendor representative must be declared.
3.2 Failure to declare may result in disciplinary action.

4. Performance review
4.1 Vendors on engagements over £25,000 must be formally reviewed quarterly.
4.2 Underperformance must be documented with the vendor's account manager.
```

## Appendix B — Sample meeting-notes emails

### Sample 1 — Clean (should pass policy check)

```
Subject: Meeting notes — Project Helios kick-off, 14 May

Hi all,

Quick recap from this morning's kick-off:

Attendees: Sarah Chen (PM), James Okafor (lead engineer), Priya Patel (design), Tom Reynolds (client lead)

Key points discussed:
- Project scope confirmed: 6-week discovery, 12-week build, soft launch end Q3
- Tech stack agreed: existing client AWS environment, no new vendor procurement needed
- Workshop schedule locked in for the next 3 Tuesdays, all in client offices in Birmingham

Decisions made:
- Sarah will lead client comms going forward
- James to draft the technical architecture doc by end of next week
- Priya to share initial design directions Friday
- Internal review meeting set for 28 May at 2pm

Action items:
- James: architecture doc by 22 May
- Priya: design directions by 17 May
- Sarah: weekly client update template by 16 May

Open questions:
- Need confirmation from client on data residency requirements (Tom to chase)

Thanks all,
Sarah
```

### Sample 2 — Should trigger policy flags

```
Subject: Meeting notes — Acme retainer planning

Hi team,

Notes from today's planning session.

Attendees: Mark Foster (engagement director), Lisa Wang (senior manager), Daniel Brooks (analyst), Jenny Holloway (analyst)

Discussion:
- Acme have asked us to bring in an external specialist for the data migration piece
- Mark spoke to DataStream Ltd this morning and is happy to go ahead with them directly — no need for other quotes given the timeline pressure
- Contract value will be around £15,000 for a 4-week engagement
- Mark mentioned his brother-in-law works there but it's not a problem
- We agreed to start work next Monday so DataStream can hit the ground running — contracts can follow

Decisions:
- DataStream Ltd selected as the data migration vendor
- Work starts Monday 19 May
- Lisa to brief the Acme team Friday

Travel:
- Mark and Lisa to fly Birmingham to Edinburgh next Thursday for the client steering committee
- Mark has booked business class given the early start (6am Friday return)
- Hotel: the Balmoral, two nights, £420/night

Action items:
- Lisa: brief Acme by Friday
- Daniel and Jenny: prep migration runbook by Wednesday
- Mark: countersign the DataStream paperwork when it comes through

Cheers,
Mark
```

> **What the flagged sample should surface**
>
> Expected flags from a correctly-built flow:
>
> - Vendor selection: £15,000 sole-source without three quotes (vendor policy 2.2 and decision policy 2.2)
> - Conflict of interest: undeclared family relationship with vendor (vendor policy 3.1)
> - Work commencing before contract execution (vendor policy 2.1)
> - Business class flight under 10 hours (travel policy 1.3)
> - Hotel at £420/night exceeds the London cap of £300 (travel policy 2.1) — though Edinburgh, not London, so this may or may not flag depending on retrieval quality. Useful discussion point.

## Appendix C — Troubleshooting

### A downstream node ignores an upstream result (most common bug)

Symptom: a node clearly produced the right output — you can see it in the trace — but the node that consumes it behaves as if that output never arrived. In this lab the classic case is the Policy Check node correctly returning OVERALL STATUS: flags-found with several flags, yet the drafted email contains no "Points to confirm" section.

Cause: Agentflow V2 assigns node IDs (`llmAgentflow_0`, `llmAgentflow_1`, `llmAgentflow_2`, `retrieverAgentflow_0`, and so on) in the order the nodes were **created**, not the order they appear left-to-right on the canvas. If you built nodes out of order, or deleted and re-added one, the ID numbers no longer line up with the visual flow. So `{{ llmAgentflow_1 }}` may look like "the second node" but actually point somewhere else entirely. The reference still resolves — it just silently pulls the wrong node's output.

Fix: open the consuming node, delete the suspect `{{ ... }}` reference, type `{{` to reopen the variable picker, and select the entry by its **node name** (e.g. "Policy Check"), not by guessing the ID. Verify every variable reference in the flow this way. Always pick by node name; never type an ID by hand. Renaming every node early (as Part 2 instructs) is what makes the picker readable enough to do this reliably.

### "Variable did not resolve" or empty interpolation

Always insert variables with the `{{` picker rather than typing them by hand — the picker lists exactly what is available at that point in the flow and inserts the correct reference. If a reference still resolves empty, check that the node it points to actually runs before the node consuming it (follow the connections on the canvas), and that you renamed each node as instructed — a node referenced before it has executed returns nothing.

### Policy breaches are not being flagged

Work through the trace node by node — the break is always at one of three points. (1) Retrieve Policies output: does it contain actual policy clause text? If empty, check the Pinecone index has a non-zero vector count and the Retriever points at the meeting-policies store. (2) Policy Check input: does it contain BOTH the meeting summary AND the retrieved policy excerpts? If one is missing, a variable reference is wrong — see the first entry in this appendix. (3) Policy Check output: if it says "clear" despite breaches, the policy text reaching it is likely garbled — see the next entry on chunking.

### Retrieved policy chunks are garbled or clauses are mixed between documents

Symptom: retrieved excerpts show clause numbers from different policies welded together (e.g. an "Approval workflow" heading followed by vendor-selection clauses). Cause: a character-based splitter cuts on character count, not on document structure, so it slices mid-policy and stitches unrelated fragments into one chunk. Because clause numbering (2.1, 2.2) repeats across all three policy files, the result is incoherent. Fix for short policy documents like these: raise the chunk size so each whole policy lands in a single chunk — set Chunk Size to 2000 and Chunk Overlap to 200 in the document loader, then re-process and re-upsert. Verify in the trace that retrieved chunks now contain one complete, coherent policy each.

### The summariser invents content or editorialises

Symptom: the summary contains policy judgements, invented compliance questions, or numbers that do not appear in the email (e.g. a threshold figure substituted for the real contract value). Cause: the extraction prompt is not constraining the model tightly enough. Fix: make the Summarise Email system message strictly verbatim — instruct it to copy figures exactly and never substitute or round them, to list only questions explicitly asked in the email, and to add no analysis. Keep temperature at 0.2. The summariser must extract only; all judgement belongs to the Policy Check node.

### Retrieval returns irrelevant chunks

Check (a) embeddings model is consistent — you cannot mix text-embedding-3-small for upsert and text-embedding-3-large for query; (b) the policy docs actually upserted — go back to the document store and verify the chunk count; (c) the query is meaningful — inspect the rendered query in the trace before it goes to the retriever.

### Retrieval returns irrelevant chunks

Check (a) embeddings model is consistent — you cannot mix text-embedding-3-small for upsert and text-embedding-3-large for query; (b) the policy docs actually upserted — go back to the document store and verify the chunk count; (c) the query is meaningful — print the rendered query before it goes to the retriever.

### LLM node output is not in the expected format

This flow asks each LLM node to return labelled prose sections (SUBJECT, DECISIONS, FLAGS, and so on) via the System message — there is no schema. If a node returns something unstructured, restate the required headings clearly in the System message and keep temperature low (0.1–0.2) for the extraction and policy-check nodes. If you do later need machine-addressable fields — for example to branch on a value with a Condition node — that is when to add the LLM node's **JSON Structured Output** parameter, which lets you define String, Number, Boolean, or Enum keys. It does not support nested arrays of objects, so use newline-separated text inside String keys for list-like data.

# Finally: ask yourself -

`Is this an agent or a workflow?`

```txt
What you've built is a prompt chain (or "workflow"): a fixed, predetermined sequence of LLM calls. Start → Summarise → Retrieve → Check → Draft → Notify. The path never varies. Every email goes through exactly those six steps in exactly that order. The LLM does cognitive work inside each node, but it has no say over what happens next. You, the builder, decided the control flow at design time.
A flow is agentic when the LLM itself decides the control flow at runtime — which tool to call, whether to call one at all, whether to loop, when it's done. The model reasons about the task and picks the next action. That's the dividing line: who's driving the sequence, the developer or the model.
So why didn't we use an Agent node? Because the task doesn't need one, and using one would make it worse. Your flow has a known, fixed procedure. When the steps are always the same, hard-wiring them is the correct engineering choice. An Agent node decides things — and every decision point is a chance to decide wrong, plus added latency, token cost, and non-determinism. For a compliance check, non-determinism is a liability: you want the same email to produce the same six steps every time, fully traceable. You'd actively design out the agency.
This is genuinely the most important lesson in the workshop, and it cuts against the hype: most useful "AI agent" builds are workflows, not agents. Reach for an Agent node only when you can't predetermine the steps:

Workflow (what you built): "Summarise this, check it, draft a reply." Steps known in advance.
Agent (genuinely needs one): "Here's a client query — figure out whether you need the policy KB, the CRM, a web search, or all three, in whatever order, and stop when you can answer." Steps unknowable until the model sees the input.
```
