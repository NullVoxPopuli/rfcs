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

# Move off the release train to a simpler release model

## Summary

Ember's "release train" today bundles two separate things:

1. **A six-week release cadence.** A new minor ships on a steady, predictable clock.
2. **A multi-branch promotion pipeline.** Code is promoted from the default branch through dedicated `beta` and `release` branches (plus LTS branches), shepherded by a small number of people who manage each release.

**This RFC keeps (1) and replaces (2).** The six-week cadence is the part that works, and it stays exactly as it is. What goes away is the extra release branches and the bespoke per-cycle process.

Concretely:

- A single long-lived branch: **`main`**. No `beta` or `release` branches.
- Stable releases are cut from `main` by [`release-plan`][release-plan] on the same six-week schedule, with the `npm publish` gated behind a protected [GitHub deployment environment][gh-environments] — a required reviewer approves before it publishes.
- The `beta` channel collapses into a single **`alpha`**, published nightly from `main` the way Embroider and Glint *used to* publish their prereleases. `ember-source@beta` consumers move to `ember-source@alpha`.

SemVer, the six-week cadence, the deprecation policy, LTS, and the major-version process from RFC [#0830][rfc-830] are all **unchanged**. Only the branch structure and the publishing mechanics change.

[release-plan]: https://github.com/embroider-build/release-plan
[gh-environments]: https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment
[rfc-830]: https://github.com/emberjs/rfcs/blob/master/text/0830-evolving-embers-major-version-process.md

## Motivation

The expensive part of the release train is not the cadence — it's the machinery around it, and the time it takes maintainers with limited availability to manage it all.

### Releases depend on too few people

There is no formal release-manager rotation. In practice the release depends on essentially one person per project — Katie for `ember.js`, Chris for `ember-cli`. That is a real bus-factor and burnout risk: when that person is unavailable, the release slips.

The biggest win of moving off the train is that releasing stops being specialized knowledge. With [`release-plan`][release-plan], **any maintainer can trivially cut a release**: they review the release-preview PR and approve a deployment. No npm keys on anyone's machine, no checklist, no tribal knowledge.

### Extra release branches are overhead

Keeping `beta`, `release`, and the LTS branches alive alongside the default branch means ongoing backporting and branch bookkeeping that exists only to service the shape of the train. A single `main` removes all of it.

### release-plan automates the mechanical work

Computing the next version number and assembling the changelog are exactly the mechanical, error-prone steps that should not be done by hand — and this is an important point: **`release-plan` does this for us**, deterministically, from the labels and titles of the PRs that were merged. The Ember ecosystem has already standardized on `release-plan` for almost every addon; the framework does not need a bespoke process that is strictly more work than the tooling everyone else already trusts.

### Keep the cadence

The six-week cadence is predictable, well-understood, and well-loved. It is *not* what makes the train expensive, so this proposal deliberately keeps it. We are moving off the *branch-and-process* model, not the calendar.

## Detailed design

### Branching model

A single long-lived branch: **`main`**. All pull requests merge here. `main` is both where work integrates and the source of every release. There are no `beta` or `release` branches, and none of the backport machinery that keeping multiple release branches requires.

### Per-PR release metadata

`release-plan` is label-driven — this is the one way it works. Every PR is labeled with one of its labels:

- `breaking` → major impact
- `enhancement` → minor impact
- `bug` → patch impact
- `documentation`, `internal` → recorded, no release on their own

The changelog entry for a release *is* the set of merged PR titles (editable later by editing the title), so there is nothing extra to author. `release-plan` reads the labels and titles of everything merged since the last release to compute the next version and assemble the changelog. The judgment a maintainer used to apply by reading the diff is captured by the label on each PR at the time it lands.

### Releasing

The cadence is unchanged; the *mechanics* are `release-plan`'s defaults:

1. `release-plan` keeps a **release-preview PR** up to date — it bumps the version in `package.json`, edits `CHANGELOG.md`, and records the plan, from the labels and titles merged since the last release.
2. On the scheduled six-week release date, a maintainer merges that preview PR, which triggers the publish workflow. (The schedule is the rhythm; nothing forces a release between scheduled dates, and a date can still be held or moved as it is today.)
3. The publish job targets a **protected GitHub Environment** (e.g. `npm-publish`) with a *required reviewers* rule. The npm token / trusted-publishing identity is scoped to that environment, so nothing can publish until a required reviewer clicks **Approve** on the pending deployment. Maintainers never need npm keys locally.
4. On approval, the CI job runs `release-plan publish`: it tags, pushes, and publishes to npm (with provenance via OIDC trusted publishing).

Across a cycle, `release-plan` collapses everything merged into a *single* version bump — highest impact wins, so six weeks of `enhancement` PRs yields one minor, exactly as one stable minor per cycle does today. `@alpha` publishes the in-progress version nightly (e.g. `6.5.0-alpha.N`); the scheduled stable cut publishes the finalized version (`6.5.0`) as `@latest`. The stable release is just a snapshot of `main` at the scheduled date — the same thing promoting `release` from `beta` produced before.

The approval click is the entire residual ceremony: no manual version edit, no manual changelog, no manual publish, and no branch to cut or promote. Because the gate is a GitHub Environment, the existing GitHub permission and audit model applies — who may approve and the record of who approved what are standard repo configuration rather than tribal knowledge.

### Channels

- **Stable (`latest`)** — published from `main` through the gated environment, on the six-week cadence, as above.
- **`alpha`** — published nightly from `main`, the way Embroider and Glint *used to* publish their prereleases. This is the bleeding-edge stream, and it gives `main` a turnkey npm dist-tag (`ember-source@alpha`).
- **`beta`** — removed. The dedicated `beta` branch and the `ember-source@beta` dist-tag go away; its consumers move to `@alpha`. A change that warrants extra baking can still be merged early and exercised via `@alpha` before the next scheduled stable release.

### Deprecations, majors, and LTS

Because the six-week cadence is retained, the policies layered on top of it are unaffected:

- **Deprecations** are still introduced as SemVer-minor and removed in majors, on the same deprecation-freeze schedule.
- **Majors** are not a manual process either. A PR is labeled `breaking`, and because RFC #0830 puts a major on the normal cadence — one major roughly every twelve 6-week minors, after the `M.10` deprecation freeze — `breaking` PRs only merge in that window. So `release-plan`'s label-driven bump produces the major on schedule, with no manual version work. The major-version process is unchanged; it simply executes via `release-plan` from `main`.
- **LTS** continues as today.

In other words, this RFC changes *how* a release is cut and *which branches exist*, not *when* releases happen or *what compatibility they guarantee*.

### Lockstep across packages

`ember-source` and `ember-cli` continue to release in lockstep — but lockstep here is a *timing* property, not a coupling. Each repo releases independently via `release-plan` (its natural per-repo mode), and the releases don't need to know about each other. Because both cut stable on the same six-week date, their versions stay aligned. There is no cross-repo coordination step in the workflow; the shared cadence is what keeps them in step.

### What is removed

- The `beta` and `release` branches, and the `ember-source@beta` dist-tag.
- The backport machinery for keeping multiple release branches alive.
- The bespoke per-cycle release process: version bumps, changelog assembly, and cut/promote steps now handled by `release-plan`.

### What is kept

- **The six-week release cadence**, unchanged.
- SemVer and every existing compatibility guarantee.
- The deprecation policy, LTS, and the major-version process (RFC #0830).
- Steering control over majors and over what ships.
- A deliberate human gate before each publish.

## How we teach this

**Contributors** keep doing what they already do in virtually every addon: label each PR with a `release-plan` label. The PR title becomes the changelog entry.

**Maintainers** gain the ability to release. The role shrinks from "shepherd the branches and the cut" to "review the release-preview PR and approve the deployment" — something any maintainer can do, from anywhere, without npm keys.

**Consumers of `beta`** are the audience whose workflow moves. Anyone who today depends on `ember-source@beta` — `ember-try` scenarios, addon CI matrices that test against upcoming Ember, people bisecting regressions — switches to `ember-source@alpha`. The default `ember-try` / blueprint scenarios should be updated to point at `@alpha`.

Documentation work:

- Rewrite the website Releases page: `main` is the release line, the six-week cadence is unchanged, the prerelease stream is `@alpha` (published nightly from `main`), and `beta` no longer exists.
- Update the contributor guide with the labeling workflow.
- Document the environment, who can approve, and the approval procedure.
- A migration/announcement blog post focused on `@beta` → `@alpha`.

## Drawbacks

- **Removing the `@beta` dist-tag moves some consumers.** `ember-try` scenarios and addon CI matrices pinned to `ember-source@beta` have to switch to `@alpha`. This is a one-time, mechanical migration rather than a loss of capability, but it is still ecosystem-wide churn that needs coordinating.
- **Collapsing `beta` into `alpha` removes a soak stage.** Today `beta` is a distinct checkpoint between the default branch and stable. Folding it into `@alpha` means one prerelease stream, not two; changes get less differentiated baking before a scheduled stable release.
- **Per-PR labeling discipline.** A wrong label yields a wrong bump. `release-plan` makes the bump deterministic, but the label is human-supplied.
- **Publish authority.** The people who can approve the protected environment are the same active folks who cut releases today — informal, no change — but it does mean the npm publish is only as locked-down as that environment's protection rules and those accounts' security (2FA / OIDC).
- **Tooling dependency.** The framework's release process becomes coupled to `release-plan` — a small, community-owned tool, but a new dependency.
- **Cultural change.** `beta` is a long-standing, load-bearing part of Ember's testing culture and infrastructure; removing it is not only a mechanical change.

## Alternatives

- **Keep the train as-is (status quo).** Pays the recurring branch-management cost and the bus-factor risk indefinitely.
- **Keep `beta` as a second prerelease stream.** Retain a `@beta` dist-tag alongside `@alpha`. This proposal drops `beta` for a single `@alpha` stream; the alternative would preserve the extra soak stage at the cost of a second prerelease tag, and could be revisited if a single `@alpha` proves too coarse for downstream CI.
- **Use `changesets` instead of `release-plan`.** Functionally similar; `release-plan` is preferred because the Ember ecosystem has standardized on it.
- **Also drop the cadence (fully continuous releases).** A more radical model where every merge can release. Explicitly **not** proposed here — the six-week cadence is retained deliberately because it works.

## Unresolved questions

- **`@alpha` version scheme.** The exact prerelease format and the `release-plan` config that produces it (e.g. `semverIncrementAs` / `semverIncrementTag`) so nightly `@alpha` builds version sensibly ahead of the next stable.
- **`ember-try` and ecosystem CI migration.** The mechanics of moving the default `ember-try` / blueprint scenarios off `@beta` onto `@alpha`, and helping the ecosystem follow.
