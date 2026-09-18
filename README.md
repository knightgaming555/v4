# V4 cache refresher (public runner)

This repository exists for one reason: to run the private `V4-GUCAPI` cache
refresh job on GitHub's **free public-repo runners**, so the private repo's
Actions quota is never touched.

It contains no application code. The workflow checks the private repo out with
a read-only token, installs its dependencies, and runs its refresh script.

**The only file that matters is `.github/workflows/refresh-cache.yml`.**
This README is just the setup notes.

---

## 1. What the job does

| Cron (UTC) | Command | Covers |
|---|---|---|
| `0 * * * *` (hourly) | `refresh_cache.py fast` | guc_data, grades, attendance, exam_seats, cms_notifications |
| `0 1 * * *` (daily) | `refresh_cache.py slow` | schedule, cms_courses, global cms_content |

Both write to the production Redis the API reads from. Nothing is served from
this repo; it is a scheduler plus a credential holder.

---

## 2. Setup (about 10 minutes)

1. **Create the repo** — public, on your second account. Empty is fine.
2. **Add the two files**: `.github/workflows/refresh-cache.yml` (required) and
   this `README.md` (optional). Push to the **default branch** — scheduled
   workflows only arm themselves from the default branch.
3. **Add the secrets** — repo → *Settings* → *Secrets and variables* → *Actions*.
   See the table in §3.
4. **Add the variable** `SOURCE_REPO` on the **Variables** tab (same screen).
   Use the `owner/name` form, e.g. `yourname/V4-GUCAPI`.
5. **Dry run it** — *Actions* → *Refresh Cache Sections* → *Run workflow*, set
   `username` to one real account and leave the two YES flags alone. That runs
   the hourly `fast` set for a single user. Confirm it goes green before
   trusting the cron.
6. **Turn off the old cron** in the private repo — see §5. Until you do, you are
   still spending the quota you are trying to save.

---

## 3. Secrets and variables

### Secrets (encrypted; never logged)

| Name | What it is | Where to get it |
|---|---|---|
| `SOURCE_REPO_PAT` | Fine-grained PAT that lets this repo read the private one | GitHub → *your second account* → Settings → Developer settings → Personal access tokens → **Fine-grained tokens** → Generate new token. Repository access: **Only select repositories** → `V4-GUCAPI`. Permissions → Repository permissions → **Contents: Read-only**. Nothing else. Set an expiry and a calendar reminder. |
| `REDIS_URL` | Production Redis connection string | Upstash console → your database → *Connect* → the `rediss://…` URL. **Must be the same value the Vercel project uses**, or the refresher will warm a different cache than the API reads. |
| `ENCRYPTION_KEY` | Fernet key that decrypts stored user credentials | The value already in your Vercel project's environment variables. **Must match it exactly** — a different key means the refresh cannot decrypt any stored password and every user silently fails. |

### Variables (plain text; visible to anyone who can see the repo)

| Name | Value |
|---|---|
| `SOURCE_REPO` | `owner/V4-GUCAPI` — the private repo to run |

Nothing else is needed. `VERIFY_SSL` and `LOG_LEVEL` are set inline in the
workflow, and `ADMIN_SECRET` / `CACHE_REFRESH_SECRET` are not used by the
refresh script at all.

---

## 4. Verifying it works

- The run log must end with a JSON summary from `refresh_cache.py` showing
  `"failed": 0` (or only expected failures) and a sane `redis_reads` /
  `redis_writes` count.
- `GET /api/admin/analytics` on the API should show `X-Cache-Status` mostly
  `hit` during the hours after a run.
- With `username` filled in and `refresh_schedule`/`refresh_cms` set to `YES`,
  a manual dispatch touches the slow kinds for one user — useful when a single
  student reports stale data.

---

## 5. Turn off the old cron (do this)

The private repo's `.github/workflows/refresh-all-cache.yml` still has the same
two `cron:` lines. Delete them and keep `workflow_dispatch:` so the job stays
available for manual runs:

```yaml
on:
  # schedule:            <- remove these two lines
  #   - cron: "0 * * * *"
  #   - cron: "0 1 * * *"
  workflow_dispatch:
```

Otherwise both crons fire and you keep paying the minutes.

---

## 6. Known caveats

- **GitHub disables scheduled workflows after 60 days of repository
  inactivity.** This repo never gets a commit, so the cron can silently stop.
  Fix: make a trivial commit (or push any change) every couple of months, or
  ask for a monthly heartbeat job that commits a timestamp.
- **Cron is not punctual.** GitHub's scheduler can run several minutes late
  under load. The TTLs were designed with that slack in mind.
- **The PAT expires.** When it does, the run fails at the checkout step with a
  clear auth error. Rotate it and update the secret.
- **`workflow_dispatch` is open to any writer on this repo.** There are no
  other writers, so keep it that way — do not add collaborators.
