---
stage: accepted
start-date: 2026-06-05T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
  - steering
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite:
---

# Move off the release train to a simpler, automated release model

## Summary

Retire Ember's calendar-driven "release train" — the fixed six-week minor
cadence, the rotating release-manager ceremony, and the hand-maintained
`canary` / `beta` / `release` channels — and replace it with the same
automated, PR-driven release model the rest of the ecosystem already runs:
[`release-plan`][release-plan].

Concretely:

- Work integrates on a long-lived `develop` branch. Every pull request that
  changes shipped code carries a SemVer impact label and a changelog entry.
- `release-plan` derives the next version and the changelog from that per-PR
  metadata. There is no longer a human deciding "what goes in this release" by
  hand.
- Promoting `develop` to `main` cuts a release. **Every merge can release**,
  but the actual `npm publish` runs inside a GitHub Actions job bound to a
  protected [Deployment environment][gh-environments]. A required reviewer
  approves the deployment before the publish runs. That approval is the only
  manual gate, and it replaces the entire release-manager checklist.
- Official, stable releases go through that same gated `release-plan publish`.
  Pre-release / bleeding-edge builds (today's `canary`) come straight off
  `develop`.

SemVer is unchanged. Ember keeps the same compatibility promises it has always
made; only the *mechanics and cadence* of cutting a release change.

[release-plan]: https://github.com/embroider-build/release-plan
[gh-environments]: https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment

## Motivation

Ember's release train has served the project well for over a decade. But the
train is *expensive to run*, and almost all of that expense is recurring human
effort that has to be staffed forever.

### The train is a standing labor cost

A minor ships every six weeks. Each cycle a release manager (RM) works through a
long, mostly-manual checklist: cut `beta` from `canary`, promote `beta` to
`release`, publish `ember-source`, `ember-cli`, and (historically) `ember-data`
in lockstep, regenerate and proofread the changelog, coordinate the release blog
post, and smoke-test the result. RFC [#0830][rfc-830] layered an analogous
*major* train on top, with deprecation-freeze deadlines pinned to `M.10` and a
new major every `M.12`.

This produces a few chronic problems:

- **It needs a volunteer, on a clock, indefinitely.** When no RM is available
  the release simply slips. The cadence is only as reliable as the rota behind
  it, and that rota is a recurring source of burnout and bus-factor risk.
- **The work is manual and therefore error-prone.** Version bumps, changelog
  assembly, and multi-package lockstep publishing are exactly the kind of
  mechanical, repetitive steps that tooling does more reliably than a human at
  the end of a long checklist.
- **Three live channels cost branch-management and CI overhead.** Maintaining
  `canary`, `beta`, and `release` (plus LTS branches) means cherry-picks,
  back-merges, and CI matrices that exist only to service the train's shape.

### The cadence couples "ready" to "the calendar"

A fixed six-week beat means a finished, reviewed change waits for the next
departure, while an unfinished change feels pressure to make the train. The
calendar, not the readiness of the work, decides when users get it.

### The ecosystem already solved this

Nearly every modern Ember addon — including the ones maintained by the same
people who staff the train — releases with [`release-plan`][release-plan]. It
derives the SemVer bump and the changelog from merged PRs, publishes to npm with
provenance, and needs no scheduled human ceremony. It is battle-tested across
hundreds of packages.

The framework does not need a bespoke, hand-run process that is strictly more
work than the tooling everyone else trusts. **We can get away with
`release-plan`**, and we should.

[rfc-830]: https://github.com/emberjs/rfcs/blob/master/text/0830-evolving-embers-major-version-process.md

## Detailed design

### Branching model

Two long-lived branches:

- **`develop`** — the integration branch. All pull requests merge here. This is
  the source of pre-release (`canary`-equivalent) builds.
- **`main`** — the released line. Whatever is on `main` corresponds to the
  latest published stable version.

A release is "promote `develop` to `main`." In practice this is `release-plan`
opening (and later merging) a release PR; the promotion is what triggers the
publish job.

> The names `develop` / `main` are illustrative. A `main` (integration) /
> `release` (published) split works identically. See
> [Unresolved questions](#unresolved-questions).

### Per-PR release metadata

Every PR that changes shipped code declares:

1. **A SemVer impact** — `patch`, `minor`, or `breaking` (major) — via a label
   (or a changeset file, whichever the team prefers; `release-plan` supports the
   label-driven flow out of the box).
2. **A changelog entry** — the human-readable "what changed," authored as part
   of review rather than reconstructed afterward.

`release-plan` aggregates these across everything merged since the last release
to compute the next version number and assemble the changelog. The decision a
release manager used to make by reading the diff is now made, deterministically,
from metadata that was reviewed when the change landed.

Breaking changes (`breaking` label) still require the same approval they require
today: a `major` is a steering-level decision, not something an automated bump
performs silently. The label only records the *impact*; the gate (below) still
governers *when* it ships.

### Releasing: every merge, gated by a GitHub deployment

The publish itself runs in CI, not on a maintainer's laptop:

1. A merge to `develop` lets `release-plan prepare` compute the pending release
   (version + changelog) and surface it as a release PR.
2. Promoting that to `main` triggers the publish workflow.
3. The workflow's publish job targets a **protected GitHub Environment** (e.g.
   `npm-publish`). The environment has a *required reviewers* protection rule.
   The npm token / trusted-publishing identity is scoped to that environment, so
   nothing can publish until a required reviewer clicks **Approve** on the
   pending deployment.
4. On approval, the job runs `release-plan publish`: it tags, pushes, and
   publishes to npm (with provenance via OIDC trusted publishing).

So "every merge can release" is true *and* safe: the cadence is no longer the
calendar, but a human still consciously approves each stable publish. That
single click is the entire residual ceremony — there is no checklist, no manual
version edit, no manual changelog, no manual `npm publish`.

Because the gate is a GitHub Environment, the existing GitHub permission and
audit model applies: who may approve, the record of who approved what, and
required-reviewer rotation are all standard repo configuration rather than tribal
release-manager knowledge.

### Channels

- **Stable** — published from `main` through the gated environment, as above.
- **Pre-release (`canary`)** — published automatically and unattended from
  `develop` (e.g. `X.Y.Z-canary.N` / `--tag canary`). Users who want the
  bleeding edge keep a continuous stream, now produced by CI on every merge
  rather than by a nightly job against a special branch.
- **`beta`** — the dedicated beta channel goes away as a *standing* concept.
  Its purpose (a soak period before stable) is served by the `canary` stream
  plus the deliberate approval gate. A specific change that warrants extended
  baking can still be shipped under a pre-release tag before promotion; it just
  isn't a permanent, separately-managed branch.

### Deprecations, majors, and LTS

The train coupled three things that this proposal decouples:

- **Deprecations.** Still introduced freely; still SemVer-minor. A deprecation
  may be *removed* in any subsequent major. What changes is that removal is no
  longer pinned to a calendar major (RFC #0830's `M.10` freeze / `M.12` major).
  Removal lands behind a `breaking` label and ships in the next major that
  steering approves.
- **Majors.** No longer calendar-bound. A major ships when breaking work is
  ready *and* steering approves the publish — the gate is the control, not a
  date. "Majors should be rare" remains a *policy* the approvers uphold, not a
  property enforced by an 18-month timer. This supersedes the cadence mechanics
  of RFC #0830 while keeping its intent (rare, predictable-in-spirit majors).
- **LTS.** Decoupled from "every fourth release." LTS becomes a *designation*
  applied to chosen stable releases (and an associated support window), rather
  than a slot mechanically derived from the train's position. Enterprises still
  get a marked, supported line to target; it just isn't generated by the
  cadence.

### Lockstep across packages

Historically `ember-source`, `ember-cli`, and `ember-data` released in lockstep.
`release-plan` operates per repository. This proposal does **not** mandate
dropping lockstep, but it makes lockstep an explicit, opt-in coordination step
rather than a side effect of the train:

- If lockstep is kept, the release workflow coordinates the version across the
  packages at promotion time.
- If lockstep is relaxed, normal SemVer ranges already express cross-package
  compatibility, and each package releases on its own merges.

This is called out as a design consideration rather than decided here; see
[Unresolved questions](#unresolved-questions).

### What is removed

- The six-week minor cadence and the major cadence/`M.10` freeze mechanics.
- The release-manager rotation and its checklist.
- The standing `beta` channel and dedicated channel branches.
- Manual version bumps, manual changelog assembly, and manual `npm publish`.

### What is kept

- SemVer and every existing compatibility guarantee.
- The `canary` stream (now CI-produced on merge).
- A deliberate human gate before each stable publish.
- Steering control over majors and over what ships.
- An LTS line for consumers who need one.

## How we teach this

**Contributors** learn one new habit: every PR that changes shipped code gets a
SemVer-impact label and a changelog entry, authored during review. This is the
same workflow they already follow in virtually every addon, so for most
contributors it is *less* new process than the train, not more.

**Release approvers** (a documented, rotating group with access to the protected
environment) learn that their job is to review the pending release PR — version
and changelog are pre-computed — and approve the deployment. The role shrinks
from "run the checklist" to "review and click approve."

**App and addon authors** need to know that Ember now releases continuously and
that versions remain strict SemVer. The Releases page is rewritten to explain:
releases come out as work lands (not on a six-week clock), "what changed" lives
in the generated changelog, `canary` is still there for early adopters, and
there is still a marked LTS line to target. The "Ember keeps its version number
low / majors are rare" messaging is reframed around *policy and the gate* rather
than the train.

Documentation work:

- Rewrite the website Releases page.
- Update the contributor guide with the label/changelog workflow.
- Document the environment, the approver group, and the approval procedure.
- A migration/announcement blog post explaining the change and the new LTS
  designation.

## Drawbacks

- **Loss of a predictable calendar.** Downstream teams that plan upgrades around
  the six-week beat (and addon authors who time releases against it) lose that
  shared rhythm. The LTS line mitigates this for teams that need predictability.
- **More version numbers.** Releasing per-merge produces more versions than a
  batched six-week train. RFC #0830 deliberately valued *rare* version-number
  growth (especially for majors); a continuous model trades some of that for
  immediacy. Majors stay rare by policy, but minors/patches will be more
  numerous and changelog volume per version drops.
- **Per-PR labeling discipline.** A wrong impact label yields a wrong bump.
  `release-plan` makes the bump deterministic, but the input is still
  human-supplied; mislabeled PRs are a new failure mode the train didn't have.
- **Concentrated publish authority.** Publish power moves to whoever can approve
  the protected environment. That group's security and rotation matter, and a
  too-small group reintroduces a bus-factor problem in a different place.
- **Tooling dependency.** The framework's release process becomes coupled to
  `release-plan`. That is a small, well-understood, community-owned tool, but it
  is a dependency the bespoke process did not have.
- **Cultural change.** The release train is a visible, load-bearing part of
  Ember's identity and contributor culture. Retiring it is not only a mechanical
  change.

## Alternatives

- **Keep the train (status quo).** Pays the recurring labor cost and the
  bus-factor risk indefinitely. This is the thing the RFC argues against.
- **Automate the train without changing cadence.** Script the RM checklist but
  keep the six-week beat, the channels, and the rotation. This removes some toil
  but keeps calendar coupling, channel-branch overhead, and a standing rota —
  i.e. it keeps the parts that are most expensive to staff.
- **Use `changesets` instead of `release-plan`.** Functionally similar
  (PR-derived versions + changelog). `release-plan` is preferred here because it
  is what the Ember ecosystem has already standardized on, so contributors and
  maintainers reuse existing muscle memory and infrastructure.
- **Time-based automation (e.g. a weekly cron release).** A middle ground:
  automated like this proposal, but batched on a timer rather than per-merge.
  Cheaper changelog/version churn, but reintroduces calendar coupling and is
  arguably just a faster, automated train. The gated per-merge model is
  preferred for immediacy; a timer could be layered on later if churn proves a
  problem.

## Unresolved questions

- **Branch names.** `develop` / `main` vs `main` / `release` — pick the pair
  that best fits existing automation and contributor expectations.
- **Lockstep.** Do `ember-source`, `ember-cli`, `ember-data`/WarpDrive keep
  lockstep versioning, or release independently? If lockstep, where does the
  coordination live in the workflow?
- **LTS policy.** Concrete criteria for designating an LTS release and the
  support window attached to it, now that it is not "every fourth release."
- **Relationship to RFC #0830.** Confirm exactly which parts of the major
  process RFC are superseded (the cadence/freeze mechanics) versus retained (the
  intent that majors stay rare and well-communicated).
- **Approver group.** Who can approve the protected environment, how membership
  rotates, and the security posture (2FA, OIDC trusted publishing, audit).
- **Pre-release naming.** Confirm the tag scheme for the continuous channel
  (`canary` vs `alpha`/`next`) and whether any soak/beta tag is retained for
  specific high-risk changes.
