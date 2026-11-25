# Okta JML Automation (Joiner / Mover / Leaver)

HR-feed-driven identity lifecycle automation for Okta, written in Python. A CSV from the
HR system is the source of truth. The script reconciles Okta against it using the Okta
Users API, and **group rules** turn the `department` attribute into role-based access.

| Event      | HR feed says                                  | Script does in Okta                                                                 |
|------------|-----------------------------------------------|--------------------------------------------------------------------------------------|
| **Joiner** | `active`, no Okta user exists                 | Create user + activate (no email sent)                                              |
| **Mover**  | `active`, name/department/title/manager changed | Partial profile update. Group rules re-evaluate, so access follows the new department |
| **Leaver, stage 1** | `terminated`, Okta user `ACTIVE`     | **Suspend** + revoke sessions and OAuth tokens (user locked out, assignments kept) |
| **Leaver, stage 2** | `terminated`, suspended for >= N days | **Deactivate** (deprovisions assigned apps). Default grace period is 7 days        |
| **Leaver, stage 3** | optional, deprovisioned >= M days    | **Delete** permanently (off unless `--delete-after-days` is set)                   |

## Why it is built this way

- **Idempotent and stateless.** The leaver clock is Okta's own `statusChanged` timestamp, so there is no
  local state file and re-running the script never double-acts.
- **Suspend before deprovision.** Immediate lockout, with a grace window to catch HR mistakes or
  rescinded terminations, then a clean deprovision of downstream apps.
- **Bad-feed guard.** If one run would take more than `--max-leavers` (default 10) leaver actions, it
  aborts. A truncated or corrupted HR file therefore cannot mass-terminate the company.
- **Dry-run first.** `--dry-run` prints the exact plan and changes nothing.
- **Audit trail.** Every action (including dry-runs) is appended to `jml_audit.jsonl`.
- **Safe lookups.** Employee IDs and emails are validated before they are used in Okta search
  filters. Search is eventually consistent, so the script falls back to the strongly consistent
  `profile.login` filter to avoid creating a user twice.
- **Review queue.** Rehires (HR `active` but Okta `DEPROVISIONED`) and active-but-suspended users are
  flagged `REVIEW` rather than being changed automatically.
- **Rate limits.** HTTP 429 is handled using Okta's `X-Rate-Limit-Reset` header.

## Files

| File                  | Purpose                                                              |
|-----------------------|----------------------------------------------------------------------|
| `jml.py`              | The automation (live Okta mode and `--mock` mode share one code path) |
| `hr_feed_day0.csv`    | Day 0 feed: 3 new hires (Dana, Priya, Tomas)                         |
| `hr_feed_day1.csv`    | Day 1 feed: Priya moves Clinical to IT, Tomas is terminated          |
| `requirements.txt`    | `requests` (only needed for live mode)                               |
| `screenshots/`        | Evidence screenshots referenced below                                |

The feeds are deliberately small. A free Okta Integrator org is capped at 10 active users, so a
3-user lab leaves room for existing users.

### CSV schema

```
employee_id,first_name,last_name,email,department,title,manager_email,status,termination_date
```

`status` is `active` or `terminated`. `termination_date` (YYYY-MM-DD) is optional, and a future
date defers the leaver flow until that day. `employee_id` is stored in the Okta `employeeNumber`
attribute and is the join key. `email` is used as the Okta login.

---

## Part A: Okta setup (done in the Okta Admin Console)

### API access

Create an API token (**Security > API > Tokens**) and export it:

```bash
export OKTA_ORG_URL="https://<your-org>.okta.com"
export OKTA_API_TOKEN="<token>"
```

A token inherits the permissions of the admin who creates it. A lab org can use Super Admin. In
production, use a dedicated service account with a custom admin role scoped to user create, update
and lifecycle actions, or an OAuth 2.0 service app with the `okta.users.manage` scope.

### Role groups and group rules (role-based access)

| Group             | Group rule name         | Expression                    |
|-------------------|-------------------------|-------------------------------|
| `JML-Role-IT`       | `JML Rule - IT`           | `user.department == "IT"`       |
| `JML-Role-Finance`  | `JML Rule - Finance`      | `user.department == "Finance"`  |
| `JML-Role-Clinical` | `JML Rule - Clinical`     | `user.department == "Clinical"` |

Each group would normally be assigned to its apps and policies. When the script changes a user's
`department`, the group rule moves the user between role groups automatically, with no separate
access-removal step. Rule-assigned membership is also removed when the rule stops matching.
Okta evaluates a rule once against all users when it is activated, then again on each profile update.

#### Screenshot Example 1: Role groups created in Okta

![Example 1 - Role groups](screenshots/example-1-role-groups.jpg)

#### Screenshot Example 2: Group rule definition (`department == "IT"` assigns `JML-Role-IT`)

![Example 2 - Group rule definition](screenshots/example-2-group-rule-definition.jpg)

#### Screenshot Example 3: All three group rules in `Active` state

![Example 3 - Group rules active](screenshots/example-3-group-rules-active.jpg)

---

## Usage

```bash
pip install -r requirements.txt

# See what would happen
python3 jml.py --csv hr_feed_day0.csv --dry-run

# Day 0: joiners
python3 jml.py --csv hr_feed_day0.csv --active-on-create

# Day 1: mover + leaver stage 1
python3 jml.py --csv hr_feed_day1.csv

# Day 8+: leaver stage 2 (any run after the grace period)
python3 jml.py --csv hr_feed_day1.csv
```

| Flag                    | Meaning                                                                   |
|-------------------------|---------------------------------------------------------------------------|
| `--dry-run`             | Print the plan, change nothing                                            |
| `--grace-days N`        | Days suspended before deactivation (default 7)                            |
| `--delete-after-days M` | Permanently delete deprovisioned users after M days (default: never)      |
| `--max-leavers N`       | Abort if a run would take more than N leaver actions (default 10)         |
| `--active-on-create`    | **Lab only.** Create users `ACTIVE` with a random password nobody sees, so suspend can be demonstrated. An admin must reset the password before anyone could sign in. |
| `--mock` / `--as-of`    | Run against a local JSON tenant and simulate the passage of time          |

Without `--active-on-create`, new users land in `PROVISIONED` (awaiting first sign-in). Okta only
allows suspending `ACTIVE` users, so a terminated user who never signed in is deactivated
immediately. There is nothing to suspend and no access to protect.

---

## Part B: Verified in mock mode (no Okta org needed)

Mock mode runs the identical reconcile logic against a local JSON tenant that enforces the same
lifecycle rules (for example, only `ACTIVE` users can be suspended). `--as-of` simulates the passage of
time, so the 7-day grace period can be tested in seconds.

```bash
S="--mock --mock-state /tmp/mock.json --audit-log /tmp/audit.jsonl"
python3 jml.py --csv hr_feed_day0.csv $S --active-on-create --as-of 2026-11-24   # joiners
python3 jml.py --csv hr_feed_day1.csv $S --as-of 2026-11-25                      # mover + suspend
python3 jml.py --csv hr_feed_day1.csv $S --as-of 2026-11-27                      # WAIT (in grace)
python3 jml.py --csv hr_feed_day1.csv $S --as-of 2026-12-02                      # deactivate
python3 jml.py --csv hr_feed_day1.csv $S --as-of 2026-12-02                      # nothing to do
```

Also verified: a run that exceeds `--max-leavers` aborts with a non-zero exit, and a feed with an
invalid `status` value is rejected before anything is touched.

#### Screenshot Example 4: Day 0. Dry-run, then three joiners created

![Example 4 - Mock Day 0 joiners](screenshots/example-4-mock-day0-joiners.png)

#### Screenshot Example 5: Day 1. Mover updated and leaver suspended (stage 1)

![Example 5 - Mock Day 1 mover and leaver stage 1](screenshots/example-5-mock-day1-mover-leaver-stage1.png)

#### Screenshot Example 6: Grace period, deactivation (stage 2), and an idempotent re-run

![Example 6 - Mock grace period and deactivation](screenshots/example-6-mock-grace-and-deactivate.png)

---

## Part C: Live run against the Okta org

The script is run from a machine with network access to the Okta org, using the API token from Part A.
The screenshots below are captured from the Admin Console after each run.

_The screenshots in this part are added after each live run. Until then, each heading marks where the
screenshot goes._

#### Screenshot Example 7: After Day 0. Three joiners in Directory > People

<!-- ![Example 7 - Joiners](screenshots/example-7-joiners.jpg) -->
_Screenshot to be added after the live Day 0 run._

#### Screenshot Example 8: `JML-Role-IT` and `JML-Role-Clinical` membership populated by the group rules

<!-- ![Example 8 - Group membership from rules](screenshots/example-8-group-membership.jpg) -->
_Screenshot to be added after the live Day 0 run._

#### Screenshot Example 9: After Day 1. Priya's department is `IT` and she has moved to `JML-Role-IT`

<!-- ![Example 9 - Mover](screenshots/example-9-mover.jpg) -->
_Screenshot to be added after the live Day 1 run._

#### Screenshot Example 10: After Day 1. Tomas is `Suspended` (leaver stage 1)

<!-- ![Example 10 - Leaver suspended](screenshots/example-10-leaver-suspended.jpg) -->
_Screenshot to be added after the live Day 1 run._

#### Screenshot Example 11: After the grace period. Tomas is `Deactivated` (leaver stage 2)

<!-- ![Example 11 - Leaver deactivated](screenshots/example-11-leaver-deactivated.jpg) -->
_Screenshot to be added after the grace-period run._

#### Screenshot Example 12: System Log showing the lifecycle events (`user.lifecycle.suspend`, `user.lifecycle.deactivate`)

<!-- ![Example 12 - System Log](screenshots/example-12-system-log.jpg) -->
_Screenshot to be added after the grace-period run._

---

## Running it on a schedule

Run daily (cron, or a scheduled GitHub Actions workflow) with the token stored as a secret. The grace
period is time-based, so no extra scheduling logic is needed. Pipe `jml_audit.jsonl` into your SIEM
or archive it as a run artifact.

## Known limits and next steps

- The HR feed is a CSV. A production version would read from the HRIS API (Workday, UKG) or use
  Okta's native HR-as-a-source integration.
- Rehire handling is flag-only (`REVIEW`). Reactivating a deprovisioned user is a policy decision.
- Manager is stored as the manager's email. Map it to `managerId` if your org chart needs it.
- Swap the SSWS token for an OAuth 2.0 service app (private-key JWT) with least-privilege scopes.
- Add an Okta Workflows or Slack notification for each leaver stage.
- Screenshots from the Admin Console show the admin's name and org ID in the top-right corner. Blur or crop
  them before publishing the repo.
