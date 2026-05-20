# 18 — Engagement and community

How to be visible, how to ask, and how to keep up with Coral. Short and pragmatic — the goal is to get you from passive reader to recognized voice in three months.

## Maintainers

Coral is a small project by headcount. From `git shortlog -sn --since="2 years ago"`, the active contributor set over the past two years is:

| Name | Commits (last 2y) |
|---|---|
| Kevin Ge | 11 |
| Walaa Eldin Moustafa | 7 |
| Jiefan Li | 7 |
| Yogesh Kothari | 4 |
| Ruolin Fan | 3 |
| Junhao Liu | 3 |
| Aastha Agrrawal | 3 |
| C. H. Afzal | 2 |

Walaa Eldin Moustafa is the original author and remains the project's technical lead; he authored most of the public talks listed in `docs/talks/`. Kevin Ge, Jiefan Li, and Aastha Agrrawal are the most active reviewers in the current period. Over a longer window (three years), Aastha Agrrawal, Jiefan Li, and Kevin Ge each have 12–25 commits, indicating sustained ownership across the codebase.

Coral is maintained almost entirely by LinkedIn engineers in the US. Practical implication: reviews tend to be fast during US Pacific business hours, slow on weekends and US holidays. If you open a PR Friday evening UTC, expect a Monday review at the earliest.

To get the current set yourself: `git shortlog -sn --since="6 months ago"` is the most reliable signal for "who is reviewing PRs right now."

## Slack

The public Coral Slack workspace invite is in the repo's `README.md` (look for the Slack section near the top). Use it for:

- Design questions before opening a PR. "I'm planning to add a `LeftPadOperatorTransformer` to coral-trino — does that fit?" gets you a yes/no in an hour.
- Diagnosing build failures when CI fails for non-obvious reasons.
- Connecting with the maintainers who reviewed your previous PR.

Don't use it for:

- Bug reports — file an issue instead.
- Sensitive vulnerabilities — `CONTRIBUTING.md` directs these to `security@linkedin.com` with PGP, not Slack.

Slack threads are not searchable from outside the workspace, so anything you discover that other contributors will need belongs in an issue or a study-guide chapter, not just a Slack thread.

## CONTRIBUTING.md

Read `CONTRIBUTING.md` at the repo root before your first PR. The whole file is short — under 30 lines — and contains:

- The Contribution Agreement (the CLA — submitting code is implicit acceptance).
- The security disclosure address.
- Four tips for PR acceptance: test new features, pass `./gradlew clean build`, include a failing test in bug fix PRs, open an issue first for non-trivial changes.

Tip 4 ("open an issue first") is the one that gets violated most often. A two-line issue saves you from writing 500 lines of code that the maintainers will redirect.

## Release cadence

Coral uses [Shipkit](https://github.com/shipkit/shipkit) to auto-generate release notes from merged PR titles. Practical implications:

- **Write good PR titles.** "Fix bug" becomes a release note that says "Fix bug" — and that's what users see when they decide whether to upgrade. "Coral-Trino: Fix substring start index for negative indices" tells a downstream user exactly what changed.
- **Use the `[Coral-Module]` prefix** when the change is module-scoped — scan recent commit titles in `git log --oneline -20` to see the convention. `[Coral-Trino]`, `[Coral-Hive]`, `[Coral-Schema]` are the common prefixes.
- **Releases land roughly weekly** when there are merges. Check the [Releases page](https://github.com/linkedin/coral/releases) on GitHub. If you've shipped a feature and need it consumed by a downstream service, the wait is usually under a week.

The release version follows semver as documented in the `README.md` — major for class removals, minor for method removals/renames, patch otherwise.

## Issue tracker

GitHub Issues at `linkedin/coral/issues`. Many issues are tagged by module. The active design conversations as of this writing center on:

- **#575** — `CoralCatalog` migration off the deprecated `HiveMetastoreClient`. This is the biggest in-flight architectural change; reading the thread teaches you how the maintainers think about long migrations.
- **#604, #600** — `MergeCoralSchemaWithAvro` and the Iceberg schema-generation work. Active week-over-week.

Issues are also the right place to file documentation gaps. If a module's behavior surprised you, that's an issue ("`HiveTypeSystem` rounds DECIMAL precision in this case — is that intended?"), not a Slack message. Documented confusions become the next contributor's onboarding wins.

## A short engagement playbook

Five concrete ways to move from observer to contributor:

- **Open an issue documenting a confusion you hit.** Not a complaint — a question or a documentation request. "I expected `coral-spark` to emit X for this view but it emitted Y; here's the minimal repro." Maintainers love these because they surface real ambiguity.
- **Comment thoughtfully on others' PRs.** You don't need approval rights to add value. "Did you intend this transformer to fire before `XTransformer`? Reading the order in `SqlCallTransformers.of(...)` it seems like it might not." That's a review comment a maintainer will value — and it teaches you the codebase faster than reading docs.
- **Attend the talks.** `docs/talks/coral-viewshift.pdf` (Databricks Data + AI Summit 2025) and `docs/talks/coral-incremental-view-maintenance.pdf` (Iceberg Summit 2024) are the most current. They're the maintainers' own narrative of where the project is going.
- **Volunteer to review a small PR.** Ask in Slack: "Could I take a first pass on PR #N as a learning exercise?" Most maintainers will say yes. Your review may not block the PR, but the maintainer will look at your comments before merging.
- **Write the chapter that wasn't there.** This study guide exists because the codebase was hard to onboard onto. If you find a topic that needs deeper treatment, propose it. The cycle of read → confused → research → write is exactly how senior contributors are built.

The single best signal of "I'm a Coral contributor" is being cc'd on PRs in your area without asking. That tends to happen around the 3-5 merged PR mark, assuming the PRs were thoughtful and the review interactions were professional.

## Read next

- Chapter 17 — first contributions (the concrete PR ideas that get you started).
- Chapter 16 — PR review companion (so when you do volunteer to review, you have a checklist).
