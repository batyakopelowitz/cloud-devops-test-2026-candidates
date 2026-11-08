## Problem 5 : Git controls

**Risk**

- **direct push to `main`** - code reaches `main` with no review and no tests.
- **any push deploys** - an unreviewed commit goes straight to production, nobody decides when to release, and we can't tell what is in production.
- **urgent fixes** - without a defined path, people bypass the controls exactly when the risk is highest.

**Fix**

Workflow:

1. Every change starts from a ticket (Azure Boards work item). The branch and the PR are named by the ticket id, e.g. `AB-123-fix-something`.
2. Open a PR into `main`. Required checks: unit tests, E2E tests (Docker + Testcontainers on the agent, as in the Validate stage of Problem 4) and a security scan. Without them, the PR can't be merged.
3. After the checks pass, another developer must review and approve.
4. Merge (squash) into `main`. Nobody pushes to `main` directly. **A merge does not deploy.**
5. Release is a separate step: the pipeline builds the image by commit hash and waits at the `production` environment until an authorized approver releases it (manually or on a schedule). The environment history shows the commits and work items in each release.

**Branch policies** on `main` and the `release/` prefix:

| Policy | Setting |
| --- | --- |
| Build validation | PR pipeline (unit, E2E, security scan), **Required** |
| Require a minimum number of reviewers | 1; "Allow requestors to approve their own changes" off; reset approvals when new changes are pushed |
| Check for linked work items | Required |
| Limit merge types | Squash merge only |

**Repository permissions:** Contributors - *Deny* "Force push" and "Bypass policies".

**Pipeline controls** on the `production` environment: **Approvals** (release approvers group, not their own run) and **Branch control** (`refs/heads/main`, `refs/heads/release/*`). The pipeline `trigger` includes both branches.

**Urgent fix** - passes all the controls, but skips the queue (commits merged to `main` and not released yet):

1. Take the commit hash running in production from the last `deploy-record` (Problem 4).
2. Create `release/AB-456` from that hash, so it has no unreleased changes.
3. Fix on `hotfix/AB-456-fix-something`, PR into `release/AB-456`, the same checks and review apply.
4. The pipeline builds from `release/AB-456`, a release approver approves `production`.
5. Cherry-pick the fix into `main` with a PR, so the next regular release does not bring the bug back.

If the last release broke production, roll back first (Problem 4), then fix. **Emergency bypass** only when the normal path is blocked: two named admins, recorded in the audit log, reviewed in a PR afterwards.

**Safe implementation**

1. Create the PR build pipeline and run it on a test branch until it is green.
2. Enable the branch policies (build validation *Optional* for a day, then *Required*) and remove force push and bypass from Contributors.
3. Add the Approvals and Branch control checks to the `production` environment.

**Verification** (proposed, not tested)

- `git push origin main` is rejected.
- A PR with a failing test or no second approval can't be completed.
- A merge to `main` waits for approval and does not deploy; a run from a feature branch stops at Branch control.

**Assumptions**

- Tickets are Azure Boards work items; any other developer can review, production is approved by the release approvers group.
