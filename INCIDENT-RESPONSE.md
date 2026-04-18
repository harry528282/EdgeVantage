# EdgeVantage — Incident Response Plan

**Audience:** the EdgeVantage maintainer.
**Purpose:** a written plan for handling security or privacy incidents, so that if something happens the maintainer is not making it up under pressure. Satisfies UK GDPR Art. 33 and 34 expectations.

_Last reviewed: 2026-04-18._

---

## What counts as an "incident"

A security or privacy incident is anything that affects — or might affect — the confidentiality, integrity, or availability of personal data EdgeVantage holds. Examples:

- A secret (encryption key, session secret, OAuth client secret) is leaked or suspected leaked.
- A Cloudflare or Notion credential with admin access to EdgeVantage is compromised.
- A vulnerability is discovered (by the maintainer, a user, or a researcher) that could have allowed unauthorised data access.
- An unusual pattern of activity is observed in logs (mass failed logins, KV anomalies, unexpected session creation).
- A third-party dependency (Cloudflare, Notion, GitHub) announces a breach that may affect EdgeVantage.
- A user reports their account is behaving in ways that suggest another person has access.
- A bug is shipped that causes data to be exposed, misdirected, or deleted.

Not every incident is a **reportable breach** under GDPR (see "When to notify" below). But every incident should be investigated.

---

## The 4 steps

### Step 1 — Detect and triage (first 60 minutes)

Goals: stop the bleeding, understand scope.

1. **Write down what you know.** Timestamp, what triggered your concern, what you observed. Keep a running log through the whole incident — dates, actions, decisions. This log is evidence if you later need to justify your response.

2. **Stop active harm if possible.** If the incident is live (someone is actively exploiting something), your first job is to close the door:
   - If a secret is leaked: rotate it immediately (see `SECURITY-KEY-ROTATION.md`).
   - If a specific user's account is compromised: invalidate their session (`session:*` key deletion in Cloudflare KV dashboard) and email them.
   - If a bug is actively exposing data: deploy a worker update that disables the affected endpoint, even if it breaks the product temporarily. Availability loss is preferable to confidentiality loss.
   - If the worker itself is misbehaving: the nuclear option is to scale the worker to zero invocations or remove its DNS route, cutting off all traffic.

3. **Assess the scope.** What data is potentially affected? How many users? What is the worst-case assumption? Don't minimise; err on the side of pessimism here.

### Step 2 — Contain and investigate (first 24 hours)

Goals: close remaining gaps, understand root cause.

1. Rotate any secret or credential that might have been exposed, even if you're not sure. Cheap insurance.

2. Review Cloudflare logs (workers → logs / live tail). Look for unusual patterns around the time of the incident.

3. Review Cloudflare Audit Logs (account settings → audit log) for unexpected admin actions.

4. Review KV contents if relevant (Cloudflare KV dashboard → namespaces → `EDGETRACK_USERS`). Look for session keys you don't recognise, user records with suspicious patterns.

5. Identify **root cause** in your log: was it a leaked secret, a code bug, a compromised credential, a third-party failure? Write it down.

6. Identify all **affected users** if any.

### Step 3 — Assess whether to notify (within 72 hours)

UK GDPR Art. 33: if the incident is likely to **result in a risk to the rights and freedoms of natural persons**, the Information Commissioner's Office (ICO) must be notified within 72 hours of becoming aware.

Use this flow:

```
Is personal data affected?
├─ NO  → Log it. Done. No external notification.
└─ YES → Is there risk of adverse effect to users?
         ├─ NO (low impact, well-contained, quick recovery)
         │   → Log it. Internal only. Consider telling users anyway if trust is at stake.
         └─ YES → NOTIFY THE ICO within 72 hours.
                  Is the risk HIGH (e.g. plaintext credentials exposed, fraud risk)?
                  └─ YES → Also notify affected USERS directly (Art. 34).
```

Examples:
- Worker downtime, no data accessed → not reportable.
- Encrypted tokens exposed but key still secret → low-risk, probably not reportable, log and rotate.
- Encryption key exposed → HIGH-risk. Rotate all keys. Notify ICO. Notify users.
- One user's session token stolen via cross-site attack → affects one user; notify them directly; report to ICO if the impact is severe.
- A bug caused all users to be logged in as user X for 5 minutes → rights breach. Report.

**If in doubt, over-report.** The ICO would much rather be told about something small than not told about something big.

#### How to notify the ICO
- Form: https://ico.org.uk/for-organisations/report-a-breach/
- Prepared info: what happened, when, what data, how many users, what you did, what you'll do next, your contact details.

#### How to notify affected users
- Plain language.
- What happened. What data may be affected.
- What they should do (e.g. "revoke EdgeVantage's access in Notion, rotate your Notion login password, watch for phishing").
- How they can contact you for more info.

### Step 4 — Remediate and review (weeks after)

Goals: make recurrence less likely.

1. Fix the root cause, not just the symptom. Deploy the fix. Verify.

2. Update this plan or `SECURITY-KEY-ROTATION.md` with any lessons learned.

3. If the incident was severe, consider whether process changes are needed:
   - More frequent secret rotation?
   - Additional monitoring?
   - Change to the architecture to reduce blast radius?

4. If an ICO report was filed, prepare the follow-up communication they may request.

---

## Severity quick reference

| Scenario | Severity | Report to ICO? | Notify users? | Rotate secrets? |
|---|---|---|---|---|
| Worker deployed a bug, caused 500s, rolled back quickly, no data disclosed | Low | No | No | No |
| A user reports their account shows someone else's trades | High | Yes (accidental disclosure) | Yes (the affected user(s)) | Depends on cause |
| You accidentally pushed a `.env` with secrets to a public repo | Medium-High | Probably (exposure of credentials) | Maybe, if encryption key affected | Yes |
| Cloudflare or Notion announce they had a security incident affecting their platform | Depends on their advisory | Depends | Depends | If credentials involved |
| An attacker appears to have been enumerating OAuth state tokens | Low | No | No | No (rate limit already in place) |
| You left your laptop in a taxi and it had a password manager copy of the encryption key | Medium-High | Depends on device encryption status | Maybe | Yes, immediately |
| Your GitHub account was compromised | Medium — no secrets there, but attacker could push malicious code | Not alone | Depends if code was modified | Rotate GitHub token, review commits |
| Your Cloudflare account was compromised | Critical | Yes | Yes | Rotate everything; secrets were accessible |
| A user reports finding a vulnerability (responsible disclosure) | Depends on severity of vuln | Depends on whether it was exploited | Depends | Depends |

---

## Pre-incident preparation (do these once, review quarterly)

- [ ] You know how to reach Cloudflare support (they have a status page + support in dashboard)
- [ ] You have a contact method for your maintainer email / GitHub issues that you check regularly
- [ ] The ICO reporting URL is saved in a password manager or bookmarks (not buried in an email)
- [ ] This plan is saved somewhere you can access even if GitHub is down (local copy, password manager, printed)
- [ ] 2FA is on all critical accounts
- [ ] The `SECURITY-KEY-ROTATION.md` runbook is up to date
- [ ] The `RECORDS-OF-PROCESSING.md` is up to date so you know exactly what data is at stake in any incident

---

## Notes

EdgeVantage is a single-maintainer personal project. There is no "security team" to escalate to. If the maintainer is unavailable during an incident (on holiday, ill, etc.), the in-app disconnect / delete-account functions remain available for users, and the worker will continue operating unless it hits an outage. There is no automated monitoring that would trigger a page; incidents are detected by the maintainer or reported by users.

This is acceptable at current scale (few users, no revenue, data stakes proportionate). If EdgeVantage scales meaningfully, re-evaluate: automated alerting, a public incident page, or a designated backup contact.

---

_Last reviewed: 2026-04-18._
_Next review due: 2026-07-18 (quarterly)._
