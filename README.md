# Customer Health Scoring Workflow (n8n)

## What it does

An n8n workflow that pulls two independent data signals for a set of customer accounts (usage/engagement data and support ticket data), merges them per account, and uses Claude to synthesize a health score (0-100), a risk classification (Healthy / Watch / At-Risk), and a one-to-two sentence reasoning explanation. Results are posted to Slack as formatted alerts.

This is a public rebuild of a general, industry-recognized workflow pattern (composite customer health scoring, used across CS platforms like Gainsight, ChurnZero, Vitally, and others), built independently on personal infrastructure using mock data. **It is not a reproduction of any former employer's systems, data, code, or IP, and does not imply continued access to any such environment.**

## Why this project

My other two portfolio projects (Finnhub MCP Connector, Support Ticket Triage Agent) are developer-flavored: TypeScript, SDKs, tool-use loops. This one deliberately proves a different, adjacent skill: configuring and connecting existing SaaS systems together using a low-code orchestration tool, which is closer to what an implementation consultant or AI deployment role actually involves day to day.

## Architecture

5 nodes, one AI step, one output:

1. **Manual Trigger**
2. **Google Sheets: Get Row(s)** (usage/engagement mock data): runs in parallel with node 3
3. **Google Sheets: Get Row(s)** (support ticket mock data): runs in parallel with node 2
4. **Merge** (Combine mode, matched on `account_name`): joins the two parallel streams into one object per account
5. **HTTP Request → Claude API** (`claude-sonnet-5`): one call per account, synthesizes health score + risk flag + reasoning
6. **Code node**: strips markdown code-fence wrapping and parses Claude's response into clean JSON
7. **HTTP Request → Slack incoming webhook**: posts a formatted alert per account

(Numbered as 7 steps above for clarity; counts as 5 *functional* nodes against the original scope target, since Merge and the Code node are structural/parsing steps rather than data-source or AI nodes.)

## Data

Both data sources are mock CSV files with 8 fictional accounts, deliberately varied to test the AI step's reasoning rather than just its formatting:

- **Acme Robotics / Ironclad Manufacturing**: control cases where usage and ticket signals agree (clearly healthy / clearly at-risk)
- **Vertex Analytics**: sparse, low-activity account. Tests whether the model distinguishes "new and quiet" from "disengaged and quiet" (it doesn't have enough information to fully separate these, and treats low activity as risk regardless; see Limitations)
- **Cascade Foods**: the deliberate conflict case. Strong, rising usage paired with rising ticket volume and severity. This is the project's core test of whether the AI step actually weighs both signals or anchors on the more favorable one.

## Honest framing

- Built with AI assistance (Claude, Claude Code); not claimed as unaided, from-scratch engineering.
- Demo-scale only: mock CSV data, not connected to any real customer data, not a production deployment.
- If asked directly whether this was deployed in production: no.
- OAuth access is scoped to the minimum required (Sheets read-only equivalent), not blanket Google Drive access. See Limitations for the trade-off this creates.

## Documented findings

**1. Markdown fence-wrapping (recurring).** Despite an explicit "return ONLY valid JSON" instruction, Claude's response was wrapped in a ` ```json ` code fence in the large majority of calls across all three test runs. A parsing step (strip fence, trim, `JSON.parse`) is required downstream; this cannot be assumed away by prompt wording alone.

**2. Item-level formatting inconsistency within a single execution.** In one of three runs, 3 of 8 accounts returned raw unfenced JSON accompanied by an extended-thinking block, while the other 5 in the same execution returned fenced JSON with no thinking block. Same prompt template, same batch, same model call structure. The model's choice of whether to "think" and how to format its final answer varied per item, not just per run. This is a distinct and arguably more production-relevant finding than run-to-run variance: a fixed prompt does not guarantee a fixed response shape even within one batch.

**3. Score drift with stable categorical classification.** Across 3 full runs (24 account-evaluations), the numeric health score varied by up to 6 points for the same account and same input data (Blue Ridge Logistics: 84 → 78 → 78). The categorical risk classification (Healthy / Watch / At-Risk) was 100% consistent across all 24 evaluations. No account ever crossed a category boundary between runs. Practical read: the exact score is not reproducible, but the decision the score is meant to drive is stable.

**4. Conflict-signal handling (Cascade Foods).** In all 3 runs, the account with strong-but-improving usage paired with rising ticket severity was classified "Watch" with reasoning that explicitly named the tension between the two signals, rather than averaging them silently or anchoring on the more favorable metric. Direct evidence the model is weighing inputs, not just pattern-matching to the friendlier number.

**5. OAuth least-privilege trade-off.** Scoping the Google OAuth credential to Sheets access only (rather than requesting Drive metadata access) is the correct security practice, but it breaks n8n's "From list" spreadsheet picker (`403: Request had insufficient authentication scopes`). Workaround: reference spreadsheets directly by document ID and sheet `gid` rather than browsing. A real, specific trade-off between tightly-scoped security and UI convenience.

**6. OAuth token lifecycle (testing-mode apps).** Google OAuth clients left in "Testing" publish status issue refresh tokens that expire after 7 days of the app remaining unverified/unpublished. Mid-project, the credential required manual re-authorization after a multi-day gap between work sessions. Expected behavior, not a bug, but a real operational detail worth understanding before assuming a credential will keep working indefinitely.

## Limitations (explicit, not hidden)

- **Structured signals only.** This workflow scores accounts using two structured data sources (usage counters, ticket counts/severity). Current CS-industry practice increasingly treats qualitative/conversational signal capture (call transcripts, survey free-text, sentiment) as the next-generation layer beyond structured telemetry, which reportedly hits a real accuracy ceiling on its own. This project intentionally does not attempt that layer. It demonstrates the orchestration/integration pattern, not a state-of-the-art scoring model.
- **Sparse-data accounts are not clearly distinguished from disengaged accounts.** Vertex Analytics (a plausible "new account, not yet ramped" case) was scored identically to a genuinely disengaged account in every run. The model does not currently ask for or infer account tenure/context that would resolve this ambiguity.
- **3 runs is a small sample** for the score-variance claim above; the pattern is consistent but not exhaustively validated.
- **Not connected to any real CRM, support system, or production data source.** Mock CSV data only.

## Setup (for a stranger to reproduce)

1. Self-host n8n locally: `npx n8n` (no install required) or via Docker. Requires Node.js v18+.
2. Create two Google Sheets from the CSVs in this repo (`Mock_Usage_Data.csv`, `Mock_Ticket_Data.csv`).
3. Set up a Google Cloud OAuth client (Web application type) with redirect URI `http://localhost:5678/rest/oauth2-credential/callback`, and connect it in n8n's Google Sheets node credential.
4. Get an Anthropic API key from console.anthropic.com; add it to an n8n Header Auth credential (`x-api-key` header) for the HTTP Request node calling Claude.
5. Create a Slack incoming webhook (api.slack.com/apps → your app → Incoming Webhooks) and paste the URL into the final HTTP Request node.
6. Import `workflow.json` from this repo into n8n, wire the credentials above into the corresponding nodes, and execute.

## Stack

n8n (Community Edition, self-hosted), Google Sheets API, Claude API (`claude-sonnet-5`), Slack Incoming Webhooks.
