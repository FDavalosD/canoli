# Email capture agent: reading a Member's Gmail once a day

Research for GitHub issue #6. Retrieved 2026-11-23. Primary sources only; anything not confirmed on a primary page is marked **unverified**.

Scope: a once-a-day job reads one Member's consumer @gmail.com inbox, decides which emails need action from that Member, and does Capture into that Member's GTD inbox. It only captures. It never replies, labels, archives, clarifies or acts.

## Answer

- **Scope:** request `gmail.readonly` and nothing else. It is a *restricted* scope, but so is `gmail.metadata`, and metadata cannot use the `q` search parameter or read bodies, so it gives no verification advantage and makes classification worse ([scopes](https://developers.google.com/workspace/gmail/api/auth/scopes), [messages.list](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/list)).
- **OAuth project, first choice:** each Member creates their own Cloud project in **Personal mode**. Google says Personal mode apps are restricted to the creator's account, "do not require verification" and show no unverified-app screen ([13464323](https://support.google.com/cloud/answer/13464323)). One project per Member also enforces spouse privacy structurally. **Unverified:** whether Personal mode allows restricted Gmail scopes and whether its refresh tokens are long-lived. Prove both in a spike first.
- **Fallback:** one Household project, External user type, **published "In production" but unverified**, used by 2 people (the "fewer than 100 users" limited-audience case). Both Members click through the unverified-app warning once. This avoids the 7-day refresh-token expiry that applies to Testing status ([oauth2](https://developers.google.com/identity/protocols/oauth2), [15549945](https://support.google.com/cloud/answer/15549945)).
- **Avoid long-term:** leaving the project in **Testing**. It works today with both Members as test users, but every refresh token dies after 7 days, so each Member must re-consent weekly.
- **Job:** Cloud Run job, triggered daily by Cloud Scheduler. Refresh tokens live in Secret Manager. This fits inside free tiers: 3 Scheduler jobs per billing account, 240,000 vCPU-seconds per month for Cloud Run jobs, 6 secret versions ([Scheduler pricing](https://cloud.google.com/scheduler/pricing), [Run pricing](https://cloud.google.com/run/pricing), [free tier](https://docs.cloud.google.com/free/docs/free-cloud-features)).
- **Classifier:** Claude Haiku 5.5 (`claude-haiku-5-5`) via the Message Batches API, with a JSON verdict from structured outputs. Estimated **~$0.45–$1.35/month** for both Members (batch), versus ~$9–$27/month on Sonnet 5.5 batch (see §5).
- **Top risks:** (1) a `gmail.readonly` refresh token is full read access to a Member's whole mailbox, so it must be stored and scoped like a password; (2) prompt injection from email text, mitigated by giving the model no tools and a closed output schema; (3) one Member's emails leaking into the other Member's view.

## 1. Gmail API scopes and calls

Scope classification, from the [Gmail scopes page](https://developers.google.com/workspace/gmail/api/auth/scopes):

| Scope | What it grants | Class |
|---|---|---|
| `gmail.readonly` | "View your email messages and settings." | Restricted |
| `gmail.metadata` | "labels and headers, but not the email body" | Restricted |
| `gmail.modify`, `gmail.compose`, `gmail.insert`, `gmail.settings.basic`, `https://mail.google.com/` | write/send/delete | Restricted |
| `gmail.send` | send as user | Sensitive |
| `gmail.labels` | label settings | Non-sensitive |

Google says to "choose the most narrowly focused scope possible" (same page). No non-sensitive or sensitive scope can read a message, so any read path is restricted.

**Why not `gmail.metadata`:** `messages.list` says of `q`: "Parameter cannot be used when accessing the api using the gmail.metadata scope" ([messages.list](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/list)). `format=full` and `format=raw` likewise "cannot be used" with metadata ([Format enum](https://developers.google.com/workspace/gmail/api/reference/rest/v1/Format)). The scope is still restricted, so it saves no review burden and leaves the classifier with headers only (no body).

**Calls a daily job needs:**

1. `users.messages.list` with `q`, e.g. `newer_than:1d` or `after:<epoch-seconds>`. Returns only `id` and `threadId`, pages via `nextPageToken`, `maxResults` default 100, max 500 ([messages.list](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/list)). Dates in queries are "interpreted as midnight on that date in the PST timezone", so pass epoch seconds for precision ([filtering guide](https://developers.google.com/workspace/gmail/api/guides/filtering)). Operators `newer_than:`/`after:`/`category:` are documented in [Gmail search help](https://support.google.com/mail/answer/7190). Restricting to the inbox (e.g. `labelIds=INBOX`) is **unverified** in this pass.
2. `users.messages.get` with `format=metadata` plus `metadataHeaders` (From, To, Subject, Date, List-Unsubscribe) for a cheap pre-filter, then `format=full` only for candidates. `metadata` "Returns only email message ID, labels, and email headers" ([Format](https://developers.google.com/workspace/gmail/api/reference/rest/v1/Format)).
3. Optional: `users.history.list` from a stored `startHistoryId` for incremental sync. History is "typically available for at least one week"; a stale ID returns HTTP 404 and the client must fall back to a full sync ([sync guide](https://developers.google.com/workspace/gmail/api/guides/sync)). For a daily job, a date-window `messages.list` plus a stored "last processed message id/internalDate" is simpler. Use history only as an optimisation.

**Quota:** 6,000 units per user per minute. `messages.list` = 5, `messages.get` = 20 (60 for full/raw), `history.list` = 2, `watch` = 100 ([quota](https://developers.google.com/workspace/gmail/api/reference/quota)). 150 emails × 60 units = 9,000 units, so throttle to stay under 6,000/min or use metadata first.

## 2. OAuth app verification

- Restricted-scope apps must pass verification and "an annual security assessment from a Google empanelled group of security assessors" ([13464321](https://support.google.com/cloud/answer/13464321)). The assessment is CASA (App Defense Alliance), tiered AL1/AL2, revalidated yearly, and issues a Letter of Validation ([13465431](https://support.google.com/cloud/answer/13465431)). Cost and who pays: **unverified** (not on these pages).
- **When verification is not needed** ([13464323](https://support.google.com/cloud/answer/13464323)):
  - *Personal mode*: "restricted exclusively to the creator's Google Account, do not require verification and do not display the 'unverified app' warning screen." It is "a one-time selection": the project cannot be shared or converted to External/Internal. Set up from `console.cloud.google.com/auth/overview/personal`.
  - *Limited audience*: "fewer than 100 users … can continue using the app without going through verification". Users see the unverified warning and can click through. The cap is 100 new users over the project's lifetime and cannot be reset ([15549945](https://support.google.com/cloud/answer/15549945), [7454865](https://support.google.com/cloud/answer/7454865)).
  - *Development/testing/staging*: "not subject to verification".
  - *Internal* (Workspace/Cloud Identity org only): does not apply to consumer @gmail.com.
- **Testing status:** "limited to up to 100 test users listed in the OAuth consent screen". Each test user must be added there, and they see the unverified warning before granting ([15549945](https://support.google.com/cloud/answer/15549945)).
- **Gap:** none of these pages explicitly says whether the annual CASA assessment is waived for an unverified limited-audience app using restricted scopes. The verification exemption implies it, because CASA is "the final step of the restricted scopes review process" ([13465431](https://support.google.com/cloud/answer/13465431)). Treat it as **likely but unverified**.

## 3. Refresh token expiry

From [Using OAuth 2.0 → Refresh token expiration](https://developers.google.com/identity/protocols/oauth2):

- A project with External user type and "Testing" publishing status "is issued a refresh token expiring in 7 days", unless it requests only name/email/profile (`openid`, `userinfo.email`, `userinfo.profile`). [15549945](https://support.google.com/cloud/answer/15549945) says the same: test-user authorizations expire seven days after consent, "that token will also expire."
- Other invalidation causes: the user revoked access; "not been used for six months"; "The user changed passwords and the refresh token contains Gmail scopes"; the account exceeded the max live tokens; time-based access expired; admin restrictions or session length (Workspace only).
- "a limit of 100 refresh tokens per Google Account per OAuth 2.0 client ID". Past it, "creating a new refresh token automatically invalidates the oldest refresh token without warning."

Implications:

- **Testing:** each Member re-consents every ≤7 days. The job must detect `invalid_grant`, mark the Member's capture as paused, and prompt re-consent in the iPhone app. Expect missed days.
- **In production, unverified (limited audience):** the 7-day rule is tied to Testing only, so tokens persist until revoked, password change, or 6 months unused. A daily job prevents the last.
- **Password change** kills the token in every mode, because Gmail scopes are involved. The app needs a re-consent path regardless.
- **Personal mode:** expiry behaviour is not stated on the pages read (**unverified**).

## 4. Scheduling, hosting and token storage

| Option | Fit | Free tier / caveats (primary source) |
|---|---|---|
| **Cloud Run job + Cloud Scheduler** (recommended) | Same Google project as OAuth client; IAM-scoped access to Secret Manager | Jobs free tier: first 240,000 vCPU-s and 450,000 GiB-s/month (us-central1 basis) ([Run pricing](https://cloud.google.com/run/pricing)); Scheduler "3 jobs per month free" per billing account, then $0.10/job/31 days ([Scheduler pricing](https://cloud.google.com/scheduler/pricing)) |
| GitHub Actions `schedule` | Easy, no cloud account | "can be delayed during periods of high loads"; min interval 5 min; default branch only; in public repos "automatically disabled when no repository activity has occurred in 60 days" ([events](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)). Tokens would sit in repo secrets; a public repo is a bad home for this |
| Claude Managed Agents scheduled deployment | Cron + IANA timezone, vaults for secrets | **Beta** (`managed-agents-2026-04-01`); jitter up to 15% of interval (max 9 min); missed triggers not backfilled; not ZDR-eligible, "transcripts persist until you delete them" ([scheduled deployments](https://platform.claude.com/docs/en/managed-agents/scheduled-deployments)); $0.08/session-hour plus tokens; **no Batch discount** ([pricing](https://platform.claude.com/docs/en/about-claude/pricing)). Overkill: the job is a fixed pipeline, not an agent loop |
| Small always-on box | Full control | GCP free tier includes one e2-micro VM/month in us-west1, us-central1 or us-east1 ([free tier](https://docs.cloud.google.com/free/docs/free-cloud-features)); you own patching and secret storage |

**Token storage:** Secret Manager, one secret per Member (e.g. `gmail-refresh-token-<member>`), readable only by that job's service account. Free tier: 6 active secret versions and 10,000 access operations per month ([free tier](https://docs.cloud.google.com/free/docs/free-cloud-features)). Two Members × one active version fits easily, but destroy old versions on each re-consent (weekly in Testing).

**Push (note only):** `users.watch` + Pub/Sub is Google's recommended trigger for partial sync ([sync guide](https://developers.google.com/workspace/gmail/api/guides/sync)). It needs a Pub/Sub topic, renewal of the watch and an always-reachable handler. For once-a-day Capture, polling is simpler.

## 5. Claude classification and cost

**Shape:** one request per email: system prompt (rules for what "needs action from this Member"), user turn = headers + plain-text body truncated to ~1,500 tokens. Output via `output_config.format` with `type: "json_schema"` ([structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)), e.g. `{needs_action: boolean, title: string, source_message_id: string}` with `additionalProperties: false`. The verdict only decides *whether* to Capture and gives a short title plus a link back to the email. It never decides what the item means; clarifying stays with the Member.

**Batch:** 50% off input and output; most batches finish "within 1 hour", expire if not done in 24h; results kept 29 days; max 100,000 requests/256 MB per batch ([batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)). Batch is not ZDR-eligible ("29-day retention", same page).

**Prices: the pricing page matches the caller's figures** ([pricing](https://platform.claude.com/docs/en/about-claude/pricing), [models overview](https://platform.claude.com/docs/en/models/overview)):

| Model | Input | Output | Cache hit | Batch in / out |
|---|---|---|---|---|
| Haiku 5.5 `claude-haiku-5-5` (≤100K-token prompts) | $0.10 | $0.50 | $0.01 | $0.05 / $0.25 |
| Haiku 5.5 (>100K-token prompts) | $0.50 | $2.50 | $0.05 | $0.25 / $1.25 |
| Sonnet 5.5 `claude-sonnet-5-5` | $2 | $10 | $0.10 | $1 / $5 |
| Opus 5.5 `claude-opus-5-5` | $4 | $20 | $0.20 | $2 / $10 |

(per 1M tokens)

**Worked estimate.** 2 Members × 50–150 emails = **100–300 requests/day**. Per request: 1,000 system + 1,500 email = 2,500 input tokens, 100 output tokens. Caching is ignored in the base numbers.

- Haiku standard: 2,500 × $0.10/1M = $0.00025, plus 100 × $0.50/1M = $0.00005, so **$0.0003/email**.
  - 100/day → $0.03/day → **$0.90/month** (30 d); 300/day → $0.09/day → **$2.70/month**.
- Haiku batch (×0.5): **$0.00015/email**, so $0.015–$0.045/day → **$0.45–$1.35/month**.
- Sonnet standard: 2,500 × $2/1M = $0.005, plus 100 × $10/1M = $0.001, so **$0.006/email**.
  - $0.60–$1.80/day → **$18–$54/month**.
- Sonnet batch: **$0.003/email**, so $0.30–$0.90/day → **$9–$27/month**.

**Caching:** the 1K system prompt clears the 512-token minimum for Haiku 5.5 and Sonnet 5.5 ([prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)). A cache hit cuts those 1,000 tokens from $0.0001 to $0.00001 on Haiku, about 30% of input cost. In batches, hits are "best-effort" (30–98%), and the docs suggest the 1-hour cache (2× write) ([batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)). At this volume caching saves cents per month and is optional.

**Caveat on output tokens:** the models overview lists Haiku 5.5 thinking as "Adaptive" with default effort `medium` ([models overview](https://platform.claude.com/docs/en/models/overview)). Thinking tokens bill as output, so real output may exceed 100 tokens. Set a low effort. Even 1,000 output tokens/email on Haiku batch is ~$0.0004/email (~$3.60/month at 300/day). Whether thinking can be fully disabled on Haiku 5.5 is **unverified**.

## 6. Privacy risks

- **Data sent to Anthropic:** under the Commercial Terms, Anthropic "may not train models on Customer Content from Services", and the customer "owns its Outputs" ([commercial terms](https://www.anthropic.com/legal/commercial-terms)). API inputs/outputs are deleted "within 30 days" by default. Exceptions: flagged Usage Policy violations (up to 2 years), legal requirements, or other agreements ([retention](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)). Batch results persist 29 days. Managed Agents transcripts persist until deleted (§4).
- **Google Limited Use:** data from restricted scopes may be used only for "providing or improving user-facing features". Transfers are barred except for user-facing features with consent, security, law, or M&A. No humans may read it without consent. No ads, no creditworthiness ([API Services User Data Policy](https://developers.google.com/terms/api-services-user-data-policy)). The Workspace policy also prohibits using the data to "create, train, or improve" a general AI model and says developers must protect against prompt injection ([Workspace API user data policy](https://developers.google.com/workspace/workspace-api-user-data-developer-policy)). Sending email content to Claude for a Capture feature the Member sees is a user-facing use, and Anthropic does not train on it. This reading is our interpretation, not a Google ruling.
- **Token risk:** a `gmail.readonly` refresh token gives read access to the entire mailbox, not just the last day. Keep it in Secret Manager only (never in the iPhone app's backend DB, logs, or a repo). Give each Member's secret its own IAM binding. Give each Member a revoke button that also calls Google's revoke endpoint.
- **Prompt injection:** email bodies are attacker-controlled. Mitigations by design: the model gets no tools; the OAuth scope cannot send or modify; output is a closed JSON schema (boolean plus short title plus message id). Treat the title as untrusted text: render it plainly and never as a link or instruction. The worst realistic outcome is a junk or misleading inbox item, or a missed one.
- **Spouse privacy:** Fernando's emails must create items only in Fernando's inbox, and likewise for Chloe. Enforce this by keying the token, the job run and the destination inbox on the same Member ID. Never batch one Member's emails into the other's request or results. Item titles must never surface in Household-shared views. Separate Personal-mode projects per Member (§2) make this structural.
- **Minimisation:** pre-filter on headers (`format=metadata`) and skip obvious bulk mail before fetching bodies. Strip HTML, quoted replies and signatures, and truncate. Never persist bodies. Store only the Gmail message id plus the captured title. Do not log prompts.

## Open questions for the Members

1. Personal-mode projects per Member (no warning; feasibility unverified) vs one Household project published unverified (one-time warning) vs Testing (weekly re-consent)?
2. Who owns the Google Cloud billing account, and are you fine adding a card to it, even at ~$0/month?
3. What counts as "needs action from a Member"? Bills, school, appointments, replies owed? Should newsletters and receipts be excluded by rule, before Claude?
4. Does Capture link back to the Gmail message, or copy only a title? A link exposes nothing extra; a body snippet would.
5. Is ~30-day retention at Anthropic (29 days for batch) acceptable for your email content, or do you want headers-only classification?
6. What time of day should the job run, and in which time zone?
7. When a token dies (password change, revoke), how should the app nag the Member to re-consent?

## Sources

- https://developers.google.com/workspace/gmail/api/auth/scopes
- https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/list
- https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/get
- https://developers.google.com/workspace/gmail/api/reference/rest/v1/Format
- https://developers.google.com/workspace/gmail/api/guides/sync
- https://developers.google.com/workspace/gmail/api/guides/filtering
- https://developers.google.com/workspace/gmail/api/reference/quota
- https://support.google.com/mail/answer/7190
- https://developers.google.com/identity/protocols/oauth2
- https://support.google.com/cloud/answer/13464321
- https://support.google.com/cloud/answer/13464323
- https://support.google.com/cloud/answer/13465431
- https://support.google.com/cloud/answer/15549945
- https://support.google.com/cloud/answer/7454865
- https://developers.google.com/terms/api-services-user-data-policy
- https://developers.google.com/workspace/workspace-api-user-data-developer-policy
- https://cloud.google.com/run/pricing
- https://cloud.google.com/scheduler/pricing
- https://docs.cloud.google.com/free/docs/free-cloud-features
- https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows
- https://platform.claude.com/docs/en/about-claude/pricing
- https://platform.claude.com/docs/en/models/overview
- https://platform.claude.com/docs/en/build-with-claude/batch-processing
- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- https://platform.claude.com/docs/en/managed-agents/scheduled-deployments
- https://www.anthropic.com/legal/commercial-terms
- https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data
