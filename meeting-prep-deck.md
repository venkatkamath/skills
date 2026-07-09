Generate a Meeting Prep for a financial advisor by combining the
advisor's interaction data held in database with
fresh **public web research**, and render it in html format. Email the final content using gmail tool

**Usage:** `/meeting-prep-deck [contact name]`
- Example: `/meeting-prep-deck Abbey Henderson`
- Example: `/meeting-prep-deck Marcus Ellington`

triggred by keywords - meeting prep

---

Parse the arguments: `$ARGUMENTS`

Extract:
- **Contact name** — first argument


---


## Data access ( using database)

The advisor data lives in **two linked tables** reached through your n8n data tool (Supabase / Postgres):

- an **advisor-profile** table — one row per advisor, holding the CRM snapshot facts (AUM, tier,
  target-account flag, relationship length, approved strategies, last in-person date, firm, title, location); and
- an **interaction-history** table — one row per interaction (calls, emails, commitments, tasks, opportunities).

The two tables join on the advisor's **contact identifier**.

> **Do not hardcode or assume column names.** At run time, read the **live schema and the column
> descriptions (comments)** from the database, and let those descriptions tell you which field holds
> what. That is the whole reason the columns are documented in the database — it keeps this skill in
> sync as the schema evolves and lets the agent map a natural-language request to the right fields
> without editing this file.

Discover the schema whichever way your tooling prefers — the Supabase/Postgres node's schema
introspection, the auto-generated REST (PostgREST) description, or a catalog query that returns each
column alongside its comment. (If the data instead lives in a spreadsheet, apply the same principle:
read the header row and any notes; never assume positions or names.)


---

## Phase 1 — Resolve the advisor and pull the data

1. Match the **contact name** from the arguments to the advisor-profile table; retrieve that advisor's
   profile row and their contact identifier.
2. Pull **all interaction-history rows** for that identifier (the tables are linked on it).
3. Using the field descriptions you just discovered, **segment the interactions** into the output sections:
   - **Recent interactions** — the advisor's most recent call/email touchpoints.
   - **Open commitments** — promised follow-ups or deliverables that are not yet done.
   - **Risk flags** — incomplete or overdue items, plus any pattern of repeatedly broken commitments.
   - **Open pipeline** — the advisor's opportunities, each with its stage and dollar amount.
   Let the schema's own descriptions tell you which field carries the interaction's category, the
   originating record type, the stage/status, the amount, the fund, and the owner — don't rely on names.
4. Derive the roll-ups: **pipeline total** (sum of the opportunity amounts), **open-pipeline count**,
   **approved strategies** (from the profile, else the distinct funds seen across the interactions),
   **last in-person**, and a **broken-commitment count**.

If the advisor-profile row is missing a snapshot fact (e.g. tier or relationship length), derive what
you can and mark the rest as a clearly-labelled placeholder — never invent a value.

---

## Phase 2 — Public research (web search)

Run these searches in order using the contact name and firm name. Collect only what is verifiable and
recent (~2 years, except founding/background facts). Discard speculation and unsourced AI summaries.
Every fact used on the Public Intel / Who She Is slides **must** carry an inline source.

1. **Advisor background & bio** — `"[contact name]" "[firm name]"` → LinkedIn, firm team page, speaker bios, awards. Extract: years in role, prior firms, credentials (CFP, CFA, CAIA, RLP, CAP, AEP), quoted views.
2. **Firm regulatory profile (SEC ADV)** — `site:adviserinfo.sec.gov "[firm name]"` or `"[firm name]" Form ADV AUM`. Extract: reported AUM, # accounts, % discretionary, client types, disclosures. **This is the authoritative firm-size figure** — show it next to the CRM AUM on the snapshot slide when they differ.
3. **Recent firm news** — `"[firm name]" news [current year]` → mergers, hires, new funds, affiliations, regulatory items. Flag anything that explains a shift in the CRM data.


Consolidate into a `PUBLIC INTEL` set and a `SOURCES` list. If a search returns nothing useful, skip it silently.

---

## Phase 3 — Assemble the brief content

Build these blocks. Data-derived blocks come from Phase 1; profile + web blocks from the profile row and
Phase 2; synthesized blocks are your analysis of the combined data.

- **Relationship Snapshot** (profile + web): AUM (CRM figure vs SEC ADV), tier, target-account flag, relationship length, approved strategies, last in-person, open-pipeline total & count.
- **Who She Is** (web + profile): 3–5 lines — firm type, city, book focus, growth story, what they respond to.
- **Last 3 Interactions** (data): the three most recent touchpoints — date, channel, what happened, any commitment and its status.
- **Open Commitments** (data): every not-yet-done follow-up — item, committed/asked date, status. Overdue first.
- **Bring to the Meeting** (synthesized): for each open commitment, the material or answer that closes it.
- **Recommended Approach** (synthesized): lead by closing the open loops, then bridge to the funds using the advisor's own words; name the live pipeline decision.
- **Public Intel** (web): credentials, regulatory AUM, firm affiliation/news, a direct quote, media — each with a source.
- **Risk Flags** (data + synthesized): the incomplete/overdue items plus the broken-commitment pattern.
- **Open Pipeline** (data): each opportunity — name, stage, amount, close date + a total.
- **Sources** (web): the list of URLs/titles actually used.

---



## Rules

- **Discover, don't assume.** Read the table and column descriptions from the database at run time; never
  hardcode column names in this skill. If a field you expect isn't there, adapt from the schema you see.
- **Two sources of truth:** the linked tables are authoritative for interactions, commitments, tasks, and
  pipeline; web research is authoritative for public intel and firm size (SEC ADV). Keep them distinct —
  the snapshot must show CRM AUM and SEC ADV AUM side by side when they differ.
- **Cite every public fact inline** on the Public Intel / Who She Is slides (e.g. "per SEC ADV",
  "per firm site", "per [podcast] interview"). No unverifiable claims.
- **Never fabricate** identifiers, AUM, tier, dates, names, or quotes. If a snapshot fact is absent from
  both the data and the web, show a clearly-marked placeholder — do not guess.
- **Nulls are expected** in the interaction data — a missing amount, owner, or identifier is normal;
  don't invent a value to fill it.
- If the advisor has a recent public quote, use their **exact words** in the Recommended Approach / Call Opener.
- Do not add any preamble. Build the content in html format and email using gmail tool, then stop.
