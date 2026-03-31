# OKR Tracking & Reporting Pipeline — Build Plan

## Overview

An OKR (Objectives and Key Results) tracking pipeline that automates the full cycle of organizational goal management: defining company and team objectives with measurable key results, tracking progress through periodic check-ins, calculating confidence scores and completion percentages, generating weekly progress digests for team leads, and producing quarterly review reports with trend analysis and retrospective insights. At-risk objectives automatically get escalated as GitHub issues.

This example demonstrates AO as a strategic planning and organizational alignment tool — every company that uses OKRs runs this cycle. Uses memory MCP for persistent cross-quarter tracking, GitHub MCP for issue creation on at-risk items, and scheduled workflows for weekly/quarterly cadences.

## Agents

| Agent | Model | Role |
|---|---|---|
| **objective-definer** | claude-sonnet-4-6 | Parses OKR definition files, validates structure (objectives have 2-5 key results each, key results are measurable), normalizes into canonical JSON format |
| **progress-tracker** | claude-haiku-4-5 | Reads check-in data, calculates completion percentages per key result, rolls up to objective-level progress, flags stale check-ins |
| **confidence-scorer** | claude-sonnet-4-6 | Analyzes progress trajectories, calculates confidence scores (0.0-1.0), classifies objectives as on-track/at-risk/off-track/achieved, uses sequential-thinking for nuanced assessment |
| **digest-writer** | claude-sonnet-4-6 | Generates weekly progress digests with highlights, blockers, and action items per team; writes at-risk alerts |
| **quarterly-reviewer** | claude-opus-4-6 | Produces comprehensive quarterly review reports with trend analysis, retrospective insights, cross-quarter comparisons, and strategic recommendations |
| **okr-reviewer** | claude-sonnet-4-6 | Quality gate — reviews outputs for accuracy, completeness, actionability; catches score miscalculations or missing context |

## Phase Pipeline

```
1. parse-objectives       (objective-definer)       → validate and normalize OKR definitions
2. track-progress         (progress-tracker)        → calculate completion % from check-in data
3. score-confidence       (confidence-scorer)       → assign confidence scores, classify status
4. review-scores          (okr-reviewer)            → validate scoring accuracy and completeness
   └─ on "rework" → back to score-confidence (scoring methodology issues)
   └─ on "fail" → back to track-progress (data integrity problems)
5. write-digest           (digest-writer)           → generate weekly progress digest
6. escalate-at-risk       (command phase)           → create GitHub issues for off-track objectives
7. review-digest          (okr-reviewer)            → quality gate on digest output
   └─ on "rework" → back to write-digest (tone, completeness, or accuracy issues)
```

For the quarterly workflow:
```
1. parse-objectives       (objective-definer)       → validate and normalize OKR definitions
2. track-progress         (progress-tracker)        → calculate final completion %
3. score-confidence       (confidence-scorer)       → final confidence scores
4. quarterly-review       (quarterly-reviewer)      → comprehensive quarter review with trends
5. review-quarterly       (okr-reviewer)            → quality gate on quarterly report
   └─ on "rework" → back to quarterly-review (depth, accuracy, or strategic framing)
6. archive-quarter        (command phase)           → archive quarterly data, reset for next quarter
```

## Decision Contracts

### review-scores
- **advance** → confidence scores are accurate and well-calibrated — proceed to digest
- **rework** → back to score-confidence (scoring seems off: too aggressive, too conservative, or missing context)
- **fail** → back to track-progress (underlying data has integrity issues that need recalculation)

Required fields: `verdict`, `reasoning`, `calibration_assessment` (well-calibrated/too-optimistic/too-pessimistic), `accuracy_score` (1-10), `issues`

### review-digest
- **advance** → digest is clear, accurate, and actionable — ready to ship
- **rework** → back to write-digest (missing context, unclear action items, or tone issues)

Required fields: `verdict`, `reasoning`, `clarity_score` (1-10), `actionability_score` (1-10), `issues`

### review-quarterly
- **advance** → quarterly report is comprehensive and strategically sound
- **rework** → back to quarterly-review (insufficient depth, missing trends, or weak recommendations)

Required fields: `verdict`, `reasoning`, `depth_score` (1-10), `strategic_quality` (1-10), `issues`

## MCP Servers

- **filesystem** — read OKR definition files, check-in data, write all output artifacts
- **sequential-thinking** — structured reasoning for confidence scoring (trajectory analysis, risk assessment)
- **memory** — persist OKR history across quarters: prior scores, trends, board feedback, strategic priorities
- **github** (gh-cli-mcp) — create issues for at-risk/off-track objectives, link to relevant context

## Data Files

| File | Purpose | Written By | Read By |
|---|---|---|---|
| `data/objectives/company-q1-2026.yaml` | Company-level OKRs for Q1 2026 | Seed data | objective-definer |
| `data/objectives/engineering-q1-2026.yaml` | Engineering team OKRs | Seed data | objective-definer |
| `data/objectives/sales-q1-2026.yaml` | Sales team OKRs | Seed data | objective-definer |
| `data/objectives/product-q1-2026.yaml` | Product team OKRs | Seed data | objective-definer |
| `data/objectives/marketing-q1-2026.yaml` | Marketing team OKRs | Seed data | objective-definer |
| `data/checkins/week-10.yaml` | Weekly check-in data (week of Mar 2) | Seed data | progress-tracker |
| `data/checkins/week-11.yaml` | Weekly check-in data (week of Mar 9) | Seed data | progress-tracker |
| `data/checkins/week-12.yaml` | Weekly check-in data (week of Mar 16) | Seed data | progress-tracker |
| `data/checkins/week-13.yaml` | Weekly check-in data (week of Mar 23) | Seed data | progress-tracker |
| `data/okr-definitions.json` | Normalized, validated OKR definitions | objective-definer | progress-tracker, confidence-scorer, digest-writer, quarterly-reviewer |
| `data/progress-snapshot.json` | Current completion percentages per KR and objective | progress-tracker | confidence-scorer, digest-writer |
| `data/stale-checkins.json` | List of KRs with no recent check-in data | progress-tracker | digest-writer, okr-reviewer |
| `output/confidence-scores.json` | Confidence scores and classifications per objective | confidence-scorer | digest-writer, quarterly-reviewer |
| `output/score-review.json` | Review verdict on confidence scores | okr-reviewer | confidence-scorer (on rework) |
| `output/weekly-digest.md` | Weekly progress digest for team leads | digest-writer | okr-reviewer |
| `output/at-risk-objectives.json` | List of objectives to escalate as GitHub issues | confidence-scorer | escalate-at-risk command phase |
| `output/digest-review.json` | Review verdict on weekly digest | okr-reviewer | digest-writer (on rework) |
| `output/quarterly-report.md` | Comprehensive quarterly review report | quarterly-reviewer | okr-reviewer |
| `output/quarterly-review.json` | Review verdict on quarterly report | okr-reviewer | quarterly-reviewer (on rework) |

## OKR Definition Schema (YAML seed data)

```yaml
# data/objectives/engineering-q1-2026.yaml
team: engineering
owner: "VP Engineering"
quarter: "Q1-2026"
objectives:
  - id: ENG-O1
    title: "Ship v3.0 platform with zero critical bugs"
    owner: "Director of Engineering"
    key_results:
      - id: ENG-KR1
        description: "Release v3.0 to production by March 15"
        metric: release_date
        target: "2026-03-15"
        unit: date
      - id: ENG-KR2
        description: "Achieve <2 P1 incidents per month post-launch"
        metric: p1_incidents_monthly
        target: 2
        unit: count
        direction: below
      - id: ENG-KR3
        description: "Maintain 99.95% uptime throughout quarter"
        metric: uptime_pct
        target: 99.95
        unit: percent
        direction: above
```

## Check-in Schema (YAML seed data)

```yaml
# data/checkins/week-13.yaml
week: 13
date_range: "2026-03-23 to 2026-03-29"
checkins:
  - kr_id: ENG-KR1
    current_value: "shipped"
    notes: "v3.0 released March 14, one day early"
    updated_by: "Director of Engineering"
  - kr_id: ENG-KR2
    current_value: 1
    notes: "Only 1 P1 incident in March so far"
    updated_by: "SRE Lead"
  - kr_id: ENG-KR3
    current_value: 99.97
    notes: "Brief blip on March 20, recovered in 4 minutes"
    updated_by: "SRE Lead"
```

## Confidence Score Schema

```json
{
  "objective_id": "ENG-O1",
  "title": "Ship v3.0 platform with zero critical bugs",
  "confidence": 0.85,
  "classification": "on-track",
  "completion_pct": 78,
  "key_results": [
    {
      "kr_id": "ENG-KR1",
      "completion_pct": 100,
      "confidence": 1.0,
      "trajectory": "achieved"
    }
  ],
  "reasoning": "2 of 3 KRs on track or achieved. KR3 uptime at 99.97% with 3 days remaining.",
  "risks": ["Minor P1 risk if weekend deploy goes wrong"],
  "weeks_remaining": 0
}
```

## Schedules

| Schedule | Cron | Workflow | Purpose |
|---|---|---|---|
| weekly-digest | `0 9 * * 1` (Monday 9am) | okr-weekly | Parse → Track → Score → Review → Digest → Escalate → Review |
| quarterly-review | `0 9 1 1,4,7,10 *` (1st of quarter months) | okr-quarterly | Parse → Track → Score → Quarterly Review → Review → Archive |

## Workflow Routing

### okr-weekly (primary workflow)
```
parse-objectives → track-progress → score-confidence → review-scores
  ├─ advance → write-digest → escalate-at-risk → review-digest
  │                                                  ├─ advance → done
  │                                                  └─ rework → write-digest (max 3)
  ├─ rework → score-confidence (max 3)
  └─ fail → track-progress (max 2)
```

### okr-quarterly
```
parse-objectives → track-progress → score-confidence → quarterly-review → review-quarterly
  ├─ advance → archive-quarter → done
  └─ rework → quarterly-review (max 3)
```

## Command Phases

### escalate-at-risk
```bash
# Read at-risk objectives and create GitHub issues
jq -r '.[] | select(.classification == "off-track" or .classification == "at-risk") | .objective_id + ": " + .title' output/at-risk-objectives.json
```
Uses gh CLI via the github MCP server instead — the digest-writer agent will create issues for at-risk items using the github MCP tools.

### archive-quarter
```bash
# Create timestamped archive of quarterly data
mkdir -p archive/Q1-2026
cp -r data/checkins/ archive/Q1-2026/checkins/
cp data/okr-definitions.json archive/Q1-2026/
cp data/progress-snapshot.json archive/Q1-2026/
cp output/confidence-scores.json archive/Q1-2026/
cp output/quarterly-report.md archive/Q1-2026/
cp output/weekly-digest.md archive/Q1-2026/last-weekly-digest.md
```

## Company Context (Seed Data)

Series B SaaS company (~150 employees), same context as board-deck-generator:
- 5 teams: Engineering, Sales, Product, Marketing, People (People OKRs optional, 4 teams minimum)
- Q1 2026 quarter, with 4 weeks of check-in data (weeks 10-13)
- Mix of on-track, at-risk, and achieved objectives for realistic scoring
- At least one off-track objective to demonstrate escalation flow

## Memory MCP Usage

The memory server persists between runs. Agents store:
- `okr_scores_<quarter>`: Confidence scores snapshot per quarter for trend comparison
- `at_risk_history`: Track which objectives were flagged and whether they recovered
- `team_velocity_<team>`: How teams typically perform against OKRs (calibration data)
- `quarterly_themes_<quarter>`: Key themes and patterns from quarterly reviews

## README Outline

1. What this does (1 paragraph)
2. Architecture diagram (agent pipeline with arrows)
3. Quick start (ao daemon start)
4. OKR definition format
5. Check-in data format
6. Output files description
7. Schedules (weekly + quarterly)
8. Customization (add teams, change scoring thresholds)
