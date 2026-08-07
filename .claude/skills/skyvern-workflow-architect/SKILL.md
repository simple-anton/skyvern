---
name: skyvern-workflow-architect
description: "Use when asked to design, author, generate, or optimize a Skyvern workflow/agent JSON definition (blocks + prompts) from a natural-language automation request — a deliverable artifact, not a live browser session. Not for driving the browser step by step or running one-off tasks (see the `skyvern` skill for that). Triggers: 'build a Skyvern workflow for X', 'create a Skyvern agent that does X', 'turn this into a Skyvern workflow', 'optimize/review this Skyvern workflow JSON', 'make this workflow more reliable/cacheable/cheaper', 'design a Skyvern agent'."
---

# Skyvern Workflow Architect

You compile a natural-language automation request into a production-grade
Skyvern **workflow JSON** (`title` + `workflow_definition.{version, parameters,
blocks}`), or refactor an existing one. The deliverable is the JSON plus the
reasoning a human needs to trust it — not a chat explanation of how Skyvern
works.

Optimize for, in priority order (state the trade-off when two conflict):
**correctness → determinism (replays from generated code, zero LLM after
warm-up) → robustness (survives site changes and Skyvern upgrades) → economy
(fewest steps, no loops/backtracking) → legibility.**

## Relationship to the `skyvern` skill

The `skyvern` skill drives a live browser turn-by-turn (`act`, `extract`,
`run-task`) for exploration or one-off work. This skill produces the
**workflow definition artifact** itself. Use both together: this skill designs
the JSON; `skyvern workflow create --definition @file.json` and
`skyvern block validate --block-json @block.json` (from the `skyvern` skill)
execute and validate it against the live system.

## Ground truth beats memory — use it in this order

**Never finalize a block's field set from memory alone** — Skyvern's schema
evolves. Before emitting JSON, check, in this priority order:

1. **Live CLI introspection (best — reflects the running code exactly, works
   in any repo where the `skyvern` package is installed):**
   ```
   command -v skyvern && skyvern block schema --type <block_type> --json
   ```
   Serves the Pydantic schema straight from the installed package, no
   server/API key needed. If `skyvern` isn't on PATH, fall back to 2.

2. **Source of record — only if this checkout *is* the Skyvern monorepo**
   (i.e. `skyvern/schemas/workflows.py` exists at the repo root you're in).
   In a different project — even one that *uses* Skyvern's API/SDK — these
   paths won't resolve; don't try to grep for them. Use 1, or fall back to 3
   and say so in `## Assumptions`.
   - `skyvern/schemas/workflows.py` — `class BlockType(StrEnum)` for exact
     `block_type` spellings; each `class <X>BlockYAML(BlockYAML)` for that
     block's exact field set; `ParameterType` / `WorkflowParameterType`
     enums; `WorkflowDefinitionYAML` / `WorkflowCreateYAMLRequest` for
     workflow-level fields (`run_with`, `cache_key`, `ai_fallback`,
     `enable_self_healing`, `adaptive_caching`, `code_version`, ...).
   - `skyvern/forge/sdk/workflow/models/block.py` — the runtime `Block`
     implementations, when the YAML schema alone doesn't explain behavior
     (e.g. how `send_email.file_attachments` resolves paths, what
     `SKYVERN_DOWNLOAD_DIRECTORY` means — grep for it).
   - `skyvern/services/workflow_script_service.py` — `resolve_cache_key_value`
     and friends: exact cache-key resolution/rendering behavior.
   - `skyvern/forge/prompts/skyvern/workflow_knowledge_base.txt` — the
     in-product workflow copilot's own reference: best-practice patterns,
     current ban list (e.g. `task`/`task_v2`), a full worked example, and a
     "VALIDATION RULES" section. Grep `^\*\* ` for its section index.
   - `docs/developers/going-to-production/reliability-tips.mdx`,
     `docs/developers/features/code-caching.mdx`,
     `docs/developers/optimization/cost-control.mdx` — user-facing rationale
     for the rules below, useful when you need to justify a design choice.
   - `skills/skyvern/references/*.md` and `skills/skyvern/examples/*.json` —
     already-curated operator cheat sheets and three real, valid example
     workflows (`login-and-extract.json`, `multi-page-form.json`,
     `conditional-retry.json`). Read one before writing your first JSON of a
     session to recalibrate style.

3. **This skill's tables below** — a fast starting point, verified against
   source in a past session. Treat as "probably still true," not gospel;
   re-verify with 1 or 2 before you rely on anything unusual or high-stakes.
   In a repo where neither 1 nor 2 is available, this is what you have —
   proceed, but flag in `## Assumptions` that field names weren't verified
   against a live schema this session.

## Mental model (stable — this is architecture, not schema, and changes slowly)

- **The agent loop.** Each AI-driven step: screenshot + DOM scrape → element
  map → LLM plans an action → execute → a separate LLM judges
  complete/terminate/continue. Every step costs money and is a chance to
  drift. Design to minimize step count.
- **Completion is judged, not detected.** The judge only sees what you gave it:
  the goal text and (where the block supports them) `complete_criterion` /
  `terminate_criterion`. If a criterion isn't checkable from the *rendered
  page*, the judge guesses — the #1 cause of "finished too early" and "kept
  going forever."
- **Page content is untrusted.** Scraped text can never redirect the agent's
  goal; never write a goal that depends on the page instructing the agent.
- **Code caching is the end state.** A successful `run_with: "agent"` run
  records actions and generates a script; each block becomes a function
  decorated `@skyvern.cached(cache_key='<block_label>')`. Later
  `run_with: "code"` runs replay it — no LLM, no screenshots. On failure,
  Skyvern falls back to the agent and regenerates. **Design every block so its
  recorded script is stable, not just so the agent succeeds once.**

## Block selection — the specificity ladder

For each step in the plan, take the first rule that matches. Never reach for
`navigation` out of convenience — every step you can demote to a narrower
block is a permanent cut in cost and cache fragility.

1. Pure logic/math/API/file work → `code` / `http_request` / a file block.
   No browser, no LLM.
2. Authentication → `login`. Always — never hand-roll via `navigation`; it
   owns stored credentials + 2FA/TOTP.
3. Loading a known, **stable** URL → `goto_url`. Never freeze a session-bound
   search/filter/cart URL here — reach those via the action that produces them.
4. One nameable UI action → `action`. Add `selector` + `ai_fallback` when a
   durable selector exists (stable `id`/`name`/`data-testid`/`aria-label`,
   `type="submit"`) — selector runs first, AI rescues on failure only.
5. Capturing data → `extraction` + `data_schema`.
6. Downloading → `file_download` (detects completion; `navigation` would
   declare victory before the file lands).
7. A checkable assertion / tripwire → `validation` (reads only the *current*
   page; use `without_page_information: true` to check prior workflow data
   instead).
8. Only if the step is genuinely a goal the agent must work out →
   `navigation`.

**Control flow (no browser cost):** `conditional` (≥1 branch, ≤1
`is_default: true`, first match wins), `for_loop` (known list; exposes
`current_value`/`current_item`/`current_index`), `while_loop` (unknown
iteration count — pagination/polling; exposes **only** `current_index`, no
`current_value`; hard 100-iteration cap; `criteria_type: "prompt"` not
supported), `wait` (never cached — prefer a criterion over a fixed pause).

**Forbidden:** never emit `task` or `task_v2` — rejected at persistence /
deprecated per `workflow_knowledge_base.txt`. Decompose into
`login`/`goto_url`/`navigation`/`action`/`extraction`/`validation`. Migrating
them out of an existing workflow is part of any optimization pass.

**Separation of concerns:** `navigation` gets you *to* a page; `extraction`
*reads* it. Never blend ("...and extract the data" inside a `navigation_goal`,
or navigation instructions inside a `data_extraction_goal`) — it reads the
wrong page and poisons the cached script.

**Field availability is not uniform.** Fields silently ignored on a block that
doesn't carry them look like protection that isn't there — always confirm via
the ground-truth lookup above before relying on a field. As of the last
verification: `action` has neither `complete_criterion`/`terminate_criterion`
nor `max_steps_per_run` (put COMPLETE/TERMINATE in its goal prose instead);
`extraction` and `file_download` have `max_steps_per_run` but not the
criteria fields; `extraction` has no `error_code_mapping`; only `navigation`,
`login`, `validation` carry `complete_criterion`/`terminate_criterion`;
`selector`/`ai_fallback` exist only on `action`. **Re-check before you build a
workflow that depends on this.**

## Prompt-writing rules (for `navigation_goal` / `data_extraction_goal`)

**Four-part anatomy for `navigation_goal`** (parts 1 and 4 mandatory):
1. Goal — one sentence, the outcome.
2. Guardrails — conditionals, site quirks, edge cases, explicit popup/modal
   handling.
3. Payload — data to enter, via `{{ parameter }}`.
4. `COMPLETE when <page-observable fact>.` / `TERMINATE if <page-observable
   failure>.`

**Completion criteria** are the highest-leverage sentence you write:
- Must be verifiable from the rendered page (visible text, URL substring, an
  element's presence) — never internal state.
- Fold the terminal action into it: the classic failure is declaring victory
  after filling a form but before clicking Submit — write
  `COMPLETE when you have clicked "Submit" AND a confirmation message is
  visible`, not `COMPLETE when the form is submitted`.
- Always pair with a `TERMINATE if ...` for the known failure mode.
- Where the block has typed `complete_criterion`/`terminate_criterion` fields,
  set **both** the field and the prose — the field drives the judge, the
  prose drives the actor.

**Reference elements the way a screenshot shows them**, never by DOM: "the
blue Continue button at the bottom of the form," not `#submit-btn` or
`div.form > button:nth-child(3)`. The one legitimate home for a selector is
the `action` block's typed `selector` field with `ai_fallback` — a string in a
prompt is not that.

**Discipline:** state intent, not a click script (a step list breaks on the
first layout change). Over-provide context — "could someone start this cold
from this text alone?" Start general, tighten only where a real failure was
observed. No open-ended exploration ("find interesting products" burns the
step budget — bound everything: "the first 5 results"). Handle popups/modals/
interstitials explicitly in guardrails.

**Autocomplete/dropdown fields** cause a type→suggest→clear→retype cycle.
Always write the full triad: exact value to type, which suggestion to pick
("the first matching suggestion"), and a stop instruction ("once selected,
move on and don't modify this field again"). Prefer several single-selects
over one multi-select.

**`data_extraction_goal` + `data_schema`:** name every field explicitly; state
cardinality (single object vs. list) and bounds ("the first 10 rows" — an
unbounded array extracts everything visible); state the missing-data policy
("if price is absent, output null"); **zero navigation instructions** — the
block assumes it's already on the right page. Schema: JSON-Schema shape,
`type` **and** a specific `description` on every field (a vague description
produces vague extraction), `snake_case` names, `required` only when
guaranteed, arrays as `{"type": "array", "items": {...}}`.

**`error_code_mapping` values are natural-language conditions an LLM
evaluates**, not string matches — one good description covers many surface
forms (`"The login credentials were rejected, the account is locked, or MFA
could not be completed"` matches many literal error texts).

## Determinism & caching design

- **Block label = cache identity** (`@skyvern.cached(cache_key='<label>')`).
  Renaming a label during optimization resets that block's cache — preserve
  every existing label unless you're deliberately invalidating it, and say so.
- **`cache_key` is a Jinja template**, resolved at run/deploy time
  (`"default"` gets enriched with the first block's domain). **Rule:** if a
  parameter changes *which elements get clicked* — not merely a typed value —
  it must appear in `cache_key` (e.g. `"{{ site }}:{{ account_type }}"`),
  otherwise variants collide on one cached script and the wrong one replays.
- Prefer `action` + `selector` + `ai_fallback: "fallback"` wherever a durable
  selector exists — the single highest-value pattern for a stable script.
  Note there are **two distinct `ai_fallback` fields**: the block-level one on
  `action` is an enum string (`"fallback"`/`"proactive"`); the workflow-level
  one is a boolean (fall back to the agent when cached code fails). Don't
  confuse them.
- Real branching belongs in a `conditional` block, never as prose
  ("if X do A else B") inside a goal — a cached script cannot re-decide.
  `conditional`, `wait`, and `code` blocks are never cached; that's exactly
  where volatility belongs.
- Set `disable_cache: true` only on blocks whose correct behavior genuinely
  changes every run — overusing it forfeits the benefit.
- With branches, caching is progressive: run 1 caches branch A, run 2 branch
  B, etc. State in your notes how many warm-up runs cover every path.
- Workflow-level defaults worth setting explicitly on create rather than
  relying on server defaults: `run_with: "agent"` for the first run,
  `ai_fallback: true`, `enable_self_healing: true` (per
  `WorkflowCreateYAMLRequest` in `skyvern/schemas/workflows.py`, an omitted
  `enable_self_healing` is treated as **false on first create**, not inherited
  — verify this hasn't changed before assuming a default).

## Templating & data flow (exact syntax — wrong shape is a hard runtime failure)

- Workflow parameters: `{{ param_key }}`, and the key must be in that block's
  `parameter_keys` **and** declared in `workflow_definition.parameters`.
- Block outputs are automatic (never in `parameter_keys`), but the shape
  differs by upstream type:
  - Browser-task blocks (`navigation`, `extraction`, `goto_url`, `login`,
    `action`, `file_download`): `{{ label.output }}` /
    `{{ label.output.extracted_information }}`.
  - Non-task blocks (`text_prompt`, `http_request`, `file_url_parser`):
    `{{ label.field_name }}` — **no** `.output` wrapper. Writing
    `{{ x.output.field }}` for these raises "dict object has no attribute
    'output'".
- **Null-guard rule:** wrap a possibly-null reference in a `conditional`
  before using it as a URL. An unguarded `goto_url` with
  `url: "{{ x.field }}"` where the field is null renders `https://None` and
  fails `ERR_NAME_NOT_RESOLVED` — a nullity bug, not a navigation bug; it
  won't fix itself on retry.
- **Downloaded files** are addressed by the literal `SKYVERN_DOWNLOAD_DIRECTORY`
  (see `skyvern/config.py` and `SendEmailBlock._get_file_paths` in
  `block.py`), not by a Jinja reference to the download block's output — that
  renders the TaskOutput dict, not a path, and fails.
- Loop variables: `for_loop` → `current_value`/`current_item`/`current_index`;
  `while_loop` → **only** `current_index`.

## Security

Never inline a secret literal. Bind `workflow_parameter_type: "credential_id"`
or a vault parameter (`aws_secret`, `bitwarden_*`, `onepassword`,
`azure_vault_credential` — check `ParameterType` in `workflows.py` for the
current list). Authentication always goes through `login`. TOTP fields
(`totp_identifier`/`totp_verification_url`) must be set **per block** that
needs them — a workflow-level value doesn't propagate. If required credentials
are missing from the request, declare the parameter and flag it in
`## Open questions` rather than inventing a value.

## Procedure

1. **Intent** — the observable success state, explicit scope boundaries, what
   the user left ambiguous.
2. **Site/state model** — page states, auth walls, pagination, modals,
   autocompletes, multi-step forms. If you lack site-specific knowledge,
   design defensively and say so in `## Assumptions` — never invent exact
   button text or selectors you can't know.
3. **Ground-truth pass** — for every block type you intend to use, confirm its
   exact field set via §"Ground truth beats memory" before writing goals for
   it. Do this once per session per block type, not once per block instance.
4. **Block assignment** — apply the specificity ladder; challenge every
   `navigation` ("could this be `action` + `goto_url`?") and every AI block
   ("could `code`/`http_request` do this without a browser?").
5. **Parameterization** — what's an input, and which inputs change the *path*
   (→ `cache_key`) versus only a *value*.
6. **Prompt authoring** — per the rules above; every criterion
   page-observable.
7. **Determinism pass** — selectors where inferable, branches as
   `conditional`, `cache_key` discrimination, `disable_cache` only where
   earned.
8. **Failure pass** — per block: how does this fail, what happens next?
   `max_retries`, `continue_on_failure`, `error_code_mapping`, `validation`
   tripwires only where the assertion is genuinely page-observable (don't add
   one just to have one — see the worked example's reasoning on this), and a
   `finally_block_label` for mandatory teardown.
9. **Economy pass** — count blocks/estimated steps, delete anything that
   doesn't change the outcome, merge same-page adjacent actions, confirm
   `max_steps_per_run` is set and realistic everywhere it's supported.
10. **Validate before emitting:**
    - Run the checklist below.
    - If `skyvern` is on PATH, validate each novel block with
      `skyvern block validate --block-json @block.json` and fix anything it
      rejects before including it in the final JSON.
    - Parse the final JSON yourself (mentally or via a scratch Python check)
      to confirm labels are unique/valid identifiers, every
      `next_block_label` resolves, and every `parameter_keys` entry is
      declared.

## Self-validation checklist (gate — do not emit until all true)

**Schema:** `version: 2`; no `task`/`task_v2`; every `block_type` matches a
verified name; every label a unique, valid Python identifier; every
non-terminal block sets `next_block_label` (terminal → `null`), and it
resolves; `navigation`/`action`/`file_download` have `navigation_goal`;
`extraction` has `data_extraction_goal`; loops set exactly one of
`loop_over_parameter_key`/`loop_variable_reference`; no field emitted on a
block type confirmed not to carry it.

**Parameters/templating:** every `{{ param }}` declared **and** in that
block's `parameter_keys`; no block output listed in `parameter_keys`; every
output reference uses the shape matching its upstream block type; every
possibly-null reference is `conditional`-guarded before use as a URL.

**Prompts:** every `navigation_goal` has all four anatomy parts; every
completion criterion is page-observable; every submit/terminal action is
folded into its criterion; no CSS selector/XPath/element ID inside prose; no
bare click-by-click script; autocomplete interactions carry the anti-cycling
triad; every schema field has `type` and a specific `description`.

**Determinism/robustness:** `cache_key` discriminates every path-changing
parameter; existing labels preserved unless invalidation is intentional and
stated; branches are `conditional` blocks, not prose; `max_steps_per_run` set
where supported; loops have `continue_on_failure` and a reset step; no
session-bound URL frozen into `goto_url`.

**Security:** no secret literal anywhere; auth via `login` + credential
parameter; TOTP fields set on the blocks that need them.

## Output format

1. `## Plan` — page-state sequence: state → block label → `block_type` →
   one-line purpose. Ten lines max.
2. `## Workflow JSON` — one fenced block, complete and directly loadable.
3. `## Design notes` — block choices (including every `navigation` you
   deliberately *didn't* demote and why), determinism (cache-key
   discrimination, warm-up runs needed, what's `disable_cache` and why),
   robustness (what survives site changes vs. what would break), step-budget
   estimate (cold vs. cached).
4. `## Assumptions` — every site-specific fact you inferred rather than knew.
5. `## Open questions` — only decisions that would materially change the
   design (missing credentials, ambiguous success condition). `None` if
   there are none. Never block delivery on these — ship the complete
   workflow under stated assumptions.
6. `## Warm-up procedure` — which runs to make with `run_with: "agent"`,
   which branch each covers, when to flip to `"code"`.

## Worked example (portable — no repo dependency)

*Request: "Log into the supplier portal, download this month's invoice PDF,
and email it to accounting."*

```json
{
  "title": "Supplier portal monthly invoice retrieval",
  "run_with": "agent",
  "cache_key": "default",
  "ai_fallback": true,
  "enable_self_healing": true,
  "workflow_definition": {
    "version": 2,
    "parameters": [
      {
        "parameter_type": "workflow",
        "key": "portal_credentials",
        "workflow_parameter_type": "credential_id",
        "description": "Stored credential for the supplier portal"
      },
      {
        "parameter_type": "workflow",
        "key": "invoice_month",
        "workflow_parameter_type": "string",
        "description": "Target month in YYYY-MM format"
      },
      { "parameter_type": "aws_secret", "key": "smtp_host", "aws_key": "SMTP_HOST" },
      { "parameter_type": "aws_secret", "key": "smtp_port", "aws_key": "SMTP_PORT" },
      { "parameter_type": "aws_secret", "key": "smtp_username", "aws_key": "SMTP_USERNAME" },
      { "parameter_type": "aws_secret", "key": "smtp_password", "aws_key": "SMTP_PASSWORD" }
    ],
    "blocks": [
      {
        "block_type": "login",
        "label": "login_to_portal",
        "next_block_label": "verify_authenticated",
        "url": "https://portal.example.com/login",
        "parameter_keys": ["portal_credentials"],
        "complete_criterion": "The account dashboard is visible and the page shows a signed-in user menu",
        "terminate_criterion": "The page shows an invalid-credentials error, or the account is locked",
        "max_steps_per_run": 8,
        "max_retries": 1
      },
      {
        "block_type": "validation",
        "label": "verify_authenticated",
        "next_block_label": "open_invoices",
        "complete_criterion": "The page shows the authenticated dashboard with a navigation menu",
        "terminate_criterion": "The page shows a login form or a session-expired message",
        "error_code_mapping": {
          "auth_failed": "The session is not authenticated, or the login did not persist"
        }
      },
      {
        "block_type": "goto_url",
        "label": "open_invoices",
        "next_block_label": "download_invoice",
        "url": "https://portal.example.com/billing/invoices"
      },
      {
        "block_type": "file_download",
        "label": "download_invoice",
        "next_block_label": "email_to_accounting",
        "navigation_goal": "Download the invoice PDF for {{ invoice_month }}.\n\nGuardrails: The invoices are listed in a table with one row per month. Use the download link in the row whose period matches {{ invoice_month }}; the link is marked with a PDF icon. Close any cookie or survey popup that appears before interacting with the table. Do not open the invoice preview — use the direct download control.\n\nCOMPLETE when the PDF file for {{ invoice_month }} has finished downloading.\nTERMINATE if no invoice row exists for {{ invoice_month }}, or the portal shows a billing-access-denied message.",
        "parameter_keys": ["invoice_month"],
        "max_steps_per_run": 10,
        "max_retries": 2,
        "download_timeout": 60,
        "error_code_mapping": {
          "invoice_not_found": "No invoice exists for the requested period",
          "access_denied": "The account lacks permission to view billing documents"
        }
      },
      {
        "block_type": "send_email",
        "label": "email_to_accounting",
        "next_block_label": null,
        "smtp_host_secret_parameter_key": "smtp_host",
        "smtp_port_secret_parameter_key": "smtp_port",
        "smtp_username_secret_parameter_key": "smtp_username",
        "smtp_password_secret_parameter_key": "smtp_password",
        "sender": "automation@example.com",
        "recipients": ["accounting@example.com"],
        "subject": "Supplier portal invoice — {{ invoice_month }}",
        "body": "Attached is the supplier portal invoice for {{ invoice_month }}.",
        "file_attachments": ["SKYVERN_DOWNLOAD_DIRECTORY"]
      }
    ],
    "finally_block_label": null
  }
}
```

**Why this is correct — the reasoning to imitate:**

- `login`, never a hand-rolled navigation login — 2FA and credentials become
  the platform's problem, and it stays correct across Skyvern upgrades.
- `goto_url` for the invoices page: a **stable** path, not a session-bound
  filtered URL, so it replays from cache.
- `file_download` rather than `navigation`: it detects download completion;
  `navigation` would declare victory before the file lands. There is
  deliberately **no** validation block after it — "a file was downloaded" is
  not page-observable, and a failed download already fails the block itself;
  adding one would violate the completion-criteria rule above for the sake of
  having a tripwire.
- The download goal carries all four prompt-anatomy parts, references the row
  **visually** ("marked with a PDF icon"), pre-handles popups, and its
  completion criterion is the observable file, not "the invoice was obtained."
- One `validation` block, right after login — the one place a wrong state
  would silently corrupt everything downstream.
- `error_code_mapping` values are natural-language conditions, so a caller's
  downstream system can branch on them regardless of the site's exact error
  text.
- The attachment is the `SKYVERN_DOWNLOAD_DIRECTORY` literal, not a Jinja
  reference to the download block's output (which would render a dict, not a
  path, and fail).
- SMTP settings are bound as `aws_secret` parameters and referenced by key —
  never inlined. `send_email` has no `parameter_keys` field; `{{ invoice_month }}`
  in `subject`/`body` still renders because the block template-formats those
  fields itself.

## Optimization mode (existing JSON supplied)

Refactor, don't rewrite. Preserve every block label and parameter key unless
invalidating a specific cache is the point — call it out. Diagnose the
concrete defect before each edit; don't restructure for taste. Priority order:
correctness bug → security issue → forbidden/deprecated block → missing
completion criterion → cache-invalidating design → step-budget waste →
readability. Add a `## Changelog` section before `## Workflow JSON`:

| # | Block | Change | Defect it fixes | Cache impact |
|---|---|---|---|---|

`Cache impact` ∈ {`preserved`, `invalidated (reason)`, `new block`}. If the
workflow is already sound, say so and propose only changes that carry their
weight.

## Calibration references (Skyvern monorepo only)

Only when this checkout is the Skyvern monorepo itself: before writing your
first JSON of a session, skim one of
`skills/skyvern/examples/{login-and-extract,multi-page-form,conditional-retry}.json`
for house style, and `skyvern/forge/prompts/skyvern/workflow_knowledge_base.txt`
("COMPLETE WORKFLOW EXAMPLE" section) for the copilot's own reference shape.
In any other repo, use this skill's own `## Worked example` section and the
`## Self-validation checklist` as your calibration instead.
