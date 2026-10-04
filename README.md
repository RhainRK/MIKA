<p align="center"> <picture> <source media="(prefers-color-scheme: dark)" srcset="docs/img/mika-logo-dark.svg"> <img src="docs/img/mika-logo-light.svg" alt="MIKA" width="220"> </picture> </p>
MIKA
A sanctions and risk monitor for a real investment portfolio. It tells you what you're exposed to, what changed since you last looked, and what would happen if someone new got sanctioned.

MIKA checks your holdings against the official US (OFAC) and UK sanctions lists. It follows ownership through real corporate parent data, so it catches companies that are blocked because of who owns them, even when their own name is on no list. It reads the news, SEC filings, Reddit and X for events about the companies you hold. Every flag points back to the exact source row that caused it.

It runs on your own machine. Nothing about your portfolio leaves it unless you switch a source on.

The Exposure page: the moon shows the share of portfolio value that needs attention, the panel below lists what changed since the last run, and the detail pane traces a holding back to a sanctioned owner

The questions it answers
Am I exposed right now? Each holding is marked Blocked, Needs review or Clear, with the reasons in plain words. On the Exposure page, the lit part of the moon is exactly the share of portfolio value that needs attention. A clean portfolio is a new moon. (The name comes from mikazuki, the Japanese word for a crescent moon.)

What changed since last time? Every run is saved and compared with the one before. MIKA lists holdings that got worse or better and new list entries that touch your holdings. It also catches flags that picked up new evidence. If OFAC lists a company overnight, the next run shows that company and every holding it owns 50% or more of, in one place.

What if? mika whatif "Company" reruns the screen as if that party had just been sanctioned. It uses the same rules as the real run, so you see which holdings would fall with it and how much value is at stake. Nothing is saved.

Should we buy this? mika check "Company" screens a company before it enters the portfolio. It returns a nonzero exit code when the answer is not Clear, so it can sit in a script.

Why ownership matters
Sanctions reach further than the names on a list. Under the US Treasury's OFAC 50 Percent Rule, a company is blocked when sanctioned parties own 50% or more of it in total, directly or through other blocked companies. Take an example from MIKA's test scenarios (every name in them is made up):

listed person Orsik Velmarov owns 30% of Vorell Energy Partners,
listed company Quintor Halvane Trading owns another 25%,
and Vorell owns 70% of Drevik Drilling.
Neither Vorell nor Drevik is on a list, so a simple name check clears both. But 30% plus 25% is 55%, so Vorell is blocked, and because Vorell is blocked, so is Drevik. MIKA flags both and draws the chain that explains why.

Quick start
You need Python 3.13 or newer. The dashboard comes pre-built.

python -m venv .venv
.venv/Scripts/pip install -e ".[server]"        # macOS/Linux: .venv/bin/pip
1. Import your portfolio. Export your holdings as a CSV:

issuer,identifier,market_value_usd
Apple Inc.,HWUPKR0MPOU8FGXBT394,1250000
Example Shipping Group,,400000
The identifier column is optional. It can be an LEI (MIKA checks its ISO 17442 check digits), a ticker or an ISIN.

mika portfolio import my-portfolio.csv
2. Choose which sources may see your holdings. Copy .env.example to .env, then for example:

MIKA_ONLINE_SOURCES=gleif,sec,gdelt
MIKA_CONTACT_EMAIL=you@example.com      # the SEC asks for a contact address
3. Run it, then open the dashboard.

mika monitor                                     # one run
mika watch --every 6h                            # or keep watching
mika serve                                       # prints a one-time password
Open http://127.0.0.1:8000 and sign in as mika. Windows step-by-step instructions are in WINDOWS_SETUP.md.

Commands
Command	What it does
mika portfolio import FILE	Loads your holdings and reports any rows it skipped and why.
mika monitor [--offline]	Refreshes the lists, looks up your holdings, screens them, collects signals and records the run.
mika watch --every 6h	Runs the monitor on a schedule (15 minutes to 7 days). A failed run is reported and retried.
mika changes	Shows what changed since the previous run.
mika whatif "A" ["B" ...]	Screens the portfolio as if those parties were sanctioned.
mika check "Company" [--identifier LEI]	Screens one company before you buy it.
mika feeds check	Confirms each enabled source is reachable.
mika serve	Starts the local dashboard.
mika audit-verify	Checks that the audit chain has not been edited.
Signals: news, SEC filings and posts ranked by risk, each listing the words that flagged it

Data sources
Source	Used for	Needs	Limits MIKA enforces	Terms
OFAC SDN list	Sanctions	nothing	follows only OFAC's signed S3 redirect	US Government, public domain
UK Sanctions List	Sanctions	nothing	120 MB cap, streamed	Open Government Licence v3.0 (attribution shown)
GLEIF	LEIs and parent companies	gleif enabled	about 1 request/s, 7-day cache	CC0; no implied endorsement
SEC EDGAR	Tickers and 8-K filings	sec enabled and a contact email	fewer than 10 requests/s	SEC fair-access policy
GDELT	News	gdelt enabled	1 request per 5.5 s	Free with citation (shown)
Reddit	Posts	reddit enabled, an approved app	1 request/s, no authors stored, 48 h retention	Reddit Data API terms; approval required
X	Posts	x enabled and a bearer token	per-run post budget, no authors stored, 48 h retention	Pay per post read
Downloading a public sanctions list reveals nothing about you, so those downloads always run. Every other source would learn which companies you hold, so each one stays off until you list it. The API details behind these choices were checked against each provider's documentation and live responses, and are written up in docs/research/real-data-sources.md.

How it works
flowchart LR
    P[Your holdings CSV] --> E[Enrich<br/>LEI, parents, tickers]
    G[GLEIF] -. opt-in .-> E
    S1[SEC EDGAR] -. opt-in .-> E
    O[OFAC SDN] --> W[Watchlist<br/>merged, versioned]
    U[UK list] --> W
    E --> R[Screen<br/>names + 50% rule]
    W --> R
    R --> REP[Exposure report]
    R --> H[Run history<br/>changes since last run]
    R --> C[Cases citing source rows]
    N[GDELT news] -. opt-in .-> SIG[Signals<br/>relevance + risk categories]
    F[SEC 8-K] -. opt-in .-> SIG
    RD[Reddit / X] -. opt-in .-> SIG
    R -- flagged holdings first --> SIG
    SIG --> REP
    REP --> L[Audit chain]
Every request goes through one gate. A single HTTP client does all outbound traffic. It only talks to each source's allowlisted hosts over HTTPS and refuses redirects to any host the source hasn't declared. It caps response size, including after decompression, and spaces requests to respect rate limits. Its errors never include response bodies, since those can contain tokens.
Name matching built for sanctions data. Names are Unicode-normalised and stripped of legal suffixes like "LLC" and "PLC". Words are compared with Jaro-Winkler similarity, which copes with different transliterations, and reversed name order is recognised. Candidates are ranked by how rare their shared words are, which keeps the screen fast at scale.
Honest ownership. GLEIF records which company consolidates which. That shows control, not a percentage. MIKA labels those links "controlled by a blocked parent through accounting consolidation (ownership % not disclosed)" instead of inventing a number. You can add exact stakes in data/ownership-overrides.csv using the company names you already use.
Scenarios reuse the real engine. A what-if is a copy of the list with your hypothetical entries added, screened by the same code as a real run. There is no second rulebook to drift out of sync.
Relevance before risk. A social post must name the company in full, use a multi-word name or use a $TICKER cashtag, so "apple tree" never counts as Apple Inc. News queries use the exact legal name as a phrase, and filings are matched by the company's SEC ID.
Results
Measured on this build (Python 3.13, Windows 11). The screening scenarios are synthetic. They prove the rules behave correctly; they don't measure real-world accuracy.

What is measured	Result
Live OFAC sync (SDN and alias files, via the signed redirect)	19,483 entries in 5.7 s
Live GLEIF enrichment	Apple Inc. resolved by name; a subsidiary traced to its parent
Live GDELT query	25 articles parsed and classified
Labelled screening scenarios (aliases, transliterations, 1 to 3-company chains, aggregation, the exact 50% boundary, a 49.9% near-miss, ownership loops)	30 / 30; blocked precision and recall 1.00
Screening at scale: 50,000 entities and 63,925 ownership links	2.3 to 5.0 s end to end (same laptop, idle vs. busy)
Python tests / branch coverage	244 passed / 90%
Browser smoke test: 5 views, 2 themes, 2 widths, plus 8 edge cases including the changes panel, a hostile link and injection text	28 / 28, no console errors
Engineering notes
A few things that went wrong along the way, and what they taught me.

GLEIF's full-text search ranks badly. Apple Inc. isn't in its top 10 full-text results. MIKA now searches by legal name first and only accepts an exact normalised match. A live test caught this, not a unit test.
The UK list outgrew the downloader. It is now about 50 MB with one row per name and address. The old 25 MiB cap failed silently, so the new parser streams the file and groups rows by ID. It also drops aliases that OFSI marks as low quality, as OFSI advises.
A rate limit broke page loads. Splitting the dashboard into ES modules meant about 24 requests per page load, and two quick reloads used up the API budget. Static files are now exempt from that budget (they still need a login), and the browser smoke test guards it.
The first run reported everything as new. With no earlier run to compare against, every holding looked "new". A first run now reports no changes, and a test pins that.
Reproducibility. One test runs entity resolution under six different hash seeds, after a tie in candidate ranking turned out to depend on Python's string hashing.
Where it came from
MIKA started as a local evidence engine: search over your own documents, with every answer cited back to its source and every action written to a tamper-evident log. That part is still here (the Evidence and Cases pages). Version 2.0 pointed it at a concrete problem, sanctions exposure in a portfolio, where citing your sources is a requirement and not a nice extra. Version 2.1 swapped the sample data for real lists, real company data and real news. Version 2.2 made it watch over time and answer "what if".

Security and privacy
Local web console. Only reachable from this machine. It uses a fresh password every time it starts, has a strict Content Security Policy and a read-only API, and only ever reports whether credentials are set, never what they are.
Untrusted text. Text from news and social posts is never treated as instructions. It is scored for prompt injection, shown as plain text, and only http(s) URLs become links.
Secrets. Credentials come from .env, are typed as secrets, and never reach logs, the audit chain, reports or the browser.
See SECURITY.md for the full model and how to report a vulnerability.

Notices
Not advice. MIKA's output is a lead for a qualified person to review. It is not legal, compliance or investment advice, and a Clear result is not a guarantee.
No affiliation. MIKA is an independent project. It is not affiliated with or endorsed by OFAC, the FCDO, GLEIF, the SEC, GDELT, Reddit or X. Data terms and the required attributions are in NOTICE.md.
Fictional test data. Everything under tests/fixtures/ is made up. No real person or organisation is shown as sanctioned.
Repository layout
mika/connectors/   the single allowlisted HTTP client and provenance snapshots
mika/sanctions/    OFAC and UK list parsers and sync
mika/portfolio/    holdings import, LEI validation, GLEIF and SEC enrichment
mika/screening/    matching engine, 50% rule propagation, what-if scenarios, evaluation
mika/signals/      GDELT, SEC 8-K, Reddit and X connectors, relevance, classifier, store
mika/              monitor pipeline, run history, evidence store, cases, audit, API, CLI
dashboard/         strict TypeScript ES modules (no framework, no external scripts)
tests/             pytest suite; fixtures hold the fictional regression data
scripts/           browser smoke test, scale-data generator, release checks
docs/              design notes, ADRs, research, screening rules
Limitations
GLEIF parent links show control, not percentages. Exact stakes need your own data or a licensed ownership dataset.
Owners missing from the data are invisible to the screen, and to a what-if.
News and social relevance is a heuristic. A possible match is a lead, not a finding.
MIKA is built for one person on one machine. Evidence, caches and backups are not encrypted at rest.
License
MIT. See LICENSE.
