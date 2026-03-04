# Polivote Roadmap

> Campaign intelligence and operations platform for voter/donor targeting and conversion, precinct-level modeling and vote counting, and election night result modeling.

## Phase 1: Define Features
Write detailed descriptions of every feature we want. Focus on what the software should do from a campaign operator's perspective. No implementation details yet.

**Status:** In progress

## Phase 2: Architecture & Tech Decisions
For each feature, evaluate the best technologies, patterns, and approaches. Make stack decisions, choose a database, define the data model, settle on auth strategy, deployment plan, etc.

**Status:** Not started

## Phase 3: Implementation Plan
Break the architecture into an ordered build plan — what gets built first, dependencies between components, milestones, and acceptance criteria for each piece.

**Status:** Not started

## Phase 4: Build
Execute the plan. Implement features incrementally, commit working code at each step, keep the test suite green throughout.

**Status:** Not started

## Phase 5: Test & Harden
End-to-end testing, security audit, load testing, accessibility review. Verify correctness of all vote-counting and modeling logic. Fix everything found.

**Status:** Not started

---

## Feature Set

### Pillar 1: Voter & Donor Targeting and Conversion

**Voter File Management**
- Import and normalize voter files (state-provided, purchased lists, CSV/API)
- Merge/deduplicate records across sources
- Enrich with demographics, vote history, consumer data overlays
- Tag and segment voters by any combination of attributes

**Voter Scoring & Targeting**
- Support score — likelihood of supporting your candidate (0–100)
- Turnout score — likelihood of actually voting (0–100)
- Persuadability score — how movable an undecided voter is
- Custom composite scores (e.g., "high support + low turnout" = GOTV target)
- Filter and build target universes from scores + demographics

**Donor Management & Targeting**
- Donor database with contribution history, amounts, frequency
- Donor scoring — likelihood of giving, predicted capacity
- Lapsed donor identification and re-engagement targeting
- Prospect identification from voter file (high-support, high-income overlays)
- FEC/campaign finance compliance tracking and export

**Outreach & Conversion**
- Contact history log — every door knock, phone call, text, email, mail piece
- Multi-channel campaign builder (door, phone, text, email, mail)
- Script management for phone/door canvassers
- Response capture — supporter ID, issue priorities, volunteer interest, donation ask result
- A/B testing for messaging effectiveness
- Conversion funnel tracking — uncontacted → contacted → ID'd → committed → voted/donated

**Canvassing Operations**
- Turf cutting — assign geographic walk/call lists to volunteers
- Mobile-friendly canvass app — walk list, scripts, GPS, response entry
- Real-time canvass progress dashboard
- Automatic re-queue of not-home doors

**Volunteer Management**
- Volunteer sign-up and onboarding
- Shift scheduling for phone banks, door knocks, events
- Performance tracking (doors knocked, calls made, conversions)
- Automated reminders and follow-ups

### Pillar 2: Precinct-Level Modeling & Vote Counting

**Precinct Data Model**
- Full precinct/ward/district hierarchy for the target jurisdiction
- Historical election results by precinct (multiple cycles)
- Demographic profiles per precinct
- Registration stats (party, active/inactive, new registrations)

**Predictive Modeling**
- Precinct-level vote share projections based on historical trends + current data
- Regression models: what predicts performance in each precinct?
- Scenario modeling — "what if turnout is X% higher in these precincts?"
- Benchmark comparisons — current cycle vs. past cycles at same point in time

**Vote Goal Setting**
- Top-down goal: "we need X votes to win" → allocate targets by precinct
- Bottom-up goal: sum precinct projections → expected total
- Gap analysis — where are we short? where is there upside?
- Resource allocation recommendations — where to spend time/money for max ROI

**Live Vote Counting (Election Day)**
- Precinct-by-precinct vote entry as results come in
- Real-time running totals with percentage reporting
- Comparison to projections — ahead/behind per precinct
- Outstanding vote estimates — how many votes are left and where

### Pillar 3: Election Night Result Modeling

**Real-Time Result Ingestion**
- Manual precinct result entry by poll watchers/staff
- Batch import of partial results (county feeds, AP/SoS scraping if available)
- Distinguish early vote, Election Day, provisional, mail-in batches

**Projection Engine**
- As results come in, project final outcome using reported + modeled unreported precincts
- Confidence intervals — not just a point estimate, but a range
- Flag surprising results — precincts significantly over/underperforming model
- "Needle"-style live probability tracker (win/loss probability updating in real time)

**Decision Support Dashboard**
- Election night war room view — map + table + projection
- Call status per race — safe, likely, lean, toss-up
- Outstanding vote analysis — "X ballots remain in Y precincts, candidate needs Z% to win"
- Scenario explorer — "what if remaining mail ballots break 60/40?"

**Post-Election Analysis**
- Final results vs. projections — model accuracy report
- Precinct-level swing analysis — where did we gain/lose vs. last cycle?
- Demographic performance breakdown
- Exportable reports for campaign post-mortem

---

## Open Questions (Phase 1)
- What level of government is the primary target? (municipal, county, state, federal — affects data complexity)
- Single-campaign tool or multi-campaign/party platform?
- Self-hosted or SaaS? Or both?
- What states/jurisdictions first? (voter file formats vary wildly)
- Real-time election night data sources — manual entry only, or integrate with state/county feeds?
- Mobile app for canvassers, or mobile-responsive web?
