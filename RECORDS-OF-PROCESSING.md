# EdgeVantage — Records of Processing Activities (RoPA)

**Required by:** UK GDPR Article 30.
**Maintained by:** the EdgeVantage maintainer.
**Scope:** all personal data processed by EdgeVantage in its operation as a trading analytics dashboard.

_Last reviewed: 2026-04-18._

---

## 1. Controller

- **Name:** EdgeVantage (operated by the individual maintainer of the project)
- **Contact:** https://github.com/harry528282/EdgeVantage/issues
- **Role:** Data Controller for all data described below. EdgeVantage is a solo-maintained project; no DPO is designated (not required at this scale).

---

## 2. Categories of data subjects

- Users who connect a Notion workspace to EdgeVantage, via either:
  - OAuth 2.0 ("Connect with Notion" flow), or
  - Internal integration token (legacy paste-in flow)
- No other categories (no employees, no suppliers, no children known to be directly targeted).

---

## 3. Categories of personal data processed

### 3.1 Credentials & identifiers
- Notion OAuth access token (or internal integration token) — encrypted at rest (AES-256-GCM).
- Notion workspace ID (UUID).
- Notion workspace name (display string, e.g. "Harry's Space").

### 3.2 Configuration
- Database ID selected by the user (UUID).
- Database name (display string).
- Field mapping — user-defined mapping from Notion column names to EdgeVantage field roles (date, R value, pair, etc.). Does not contain data values, only column names.

### 3.3 Preferences
- UI preferences: theme, panel layout choices, tile order. Small JSON blob, under 8 KB.

### 3.4 Session
- Session ID (random 32-byte hex string, signed with HMAC).
- Timestamp of session creation and last seen.

### 3.5 NOT processed
- Trade data itself is not stored. Rows are fetched from Notion on demand, rendered in the user's browser, and discarded on tab close.
- Email addresses.
- Names (beyond the workspace display name the user set in Notion).
- Payment information.
- Device information, IP addresses (beyond transient request metadata handled by Cloudflare).
- Behavioural/analytics data.

---

## 4. Purposes of processing

| Data category | Purpose | Legal basis (UK GDPR Art. 6) |
|---|---|---|
| Credentials (access token) | Authenticate with Notion on behalf of the user to fetch their trade journal | Art. 6(1)(b) — contract (provision of the service the user requested) |
| Workspace ID/name | Identify the user's account in EdgeVantage's storage; display in UI | Art. 6(1)(b) — contract |
| Database ID/name | Scope Notion API calls to the correct database | Art. 6(1)(b) — contract |
| Field mapping | Translate Notion column names to EdgeVantage field roles | Art. 6(1)(b) — contract |
| UI preferences | Persist the user's interface choices across sessions | Art. 6(1)(b) — contract |
| Session ID | Keep the user logged in between page loads | Art. 6(1)(b) — contract |

All processing is for the purpose of operating the service the user explicitly signed up for. No data is used for marketing, profiling, or training any model.

---

## 5. Recipients / transfers

### 5.1 Third parties
- **Cloudflare, Inc.** — hosts the worker and KV store. Subprocessor. Located in the US with presence in the UK/EU. Cloudflare has its own GDPR-compliance framework and publishes a DPA.
- **Notion Labs, Inc.** — the user's trade journal lives there; EdgeVantage reads from it on behalf of the user. Unavoidable (the whole point of EdgeVantage). Located in the US. Notion has its own GDPR-compliance framework.
- **GitHub, Inc.** — hosts the static frontend (privacy policy, terms, onboarding, dashboard HTML). Processes no personal data; merely serves static files.

### 5.2 No other recipients
- No third-party analytics.
- No third-party advertising.
- No marketing partners.
- No data brokers.
- No law enforcement disclosure has occurred; if one is ever compelled by court order, it will be noted here.

### 5.3 International transfers
Processing may occur in the US (Cloudflare, Notion) and the UK/EU (Cloudflare edge locations). Transfers are covered by Standard Contractual Clauses where required and by the UK–US Data Bridge where applicable. EdgeVantage does not transfer data to any jurisdiction outside these frameworks.

---

## 6. Retention

| Data | Retention | Trigger for deletion |
|---|---|---|
| User record (tokens, mapping, prefs) | 90 days from last seen | Auto-expiry on inactivity, or immediate on user-triggered account deletion |
| Session | 30 days from creation | Auto-expiry, or immediate on logout / account deletion |
| Workspace→session index | 30 days | Auto-expiry |
| OAuth state tokens | 10 minutes | Auto-expiry, or immediate on consumption |
| Rate-limit counters | 60 seconds | Auto-expiry |
| Worker logs | Per Cloudflare defaults (short, days not months) | Automatic |

Users can trigger immediate deletion via Settings → Delete Account in the dashboard, which purges `user:*`, `session:*`, and `wsindex:*` keys for their workspace.

---

## 7. Security measures (Art. 32)

### Technical
- **Encryption in transit:** TLS 1.2+ enforced on all endpoints via Cloudflare.
- **Encryption at rest:** Notion access tokens encrypted with AES-256-GCM using a server-side key (`TOKEN_ENCRYPTION_KEY`) not stored alongside the ciphertext.
- **Session integrity:** Session IDs signed with HMAC-SHA-256 using a server-side secret (`SESSION_SECRET`) to prevent forgery.
- **OAuth CSRF protection:** `state` parameter is cryptographically random, single-use, KV-validated, 10-minute TTL.
- **Scope minimisation:** Notion integration requests only read-content capability; does not request user info or email addresses.
- **Rate limiting:** Per-IP rate limit on `/auth/notion/start` to prevent abuse.
- **Cookies:** All cookies flagged `HttpOnly`, `Secure`, `SameSite=Lax`.

### Organisational
- Maintainer uses 2FA on Cloudflare, GitHub, and Notion accounts.
- Secrets (`SESSION_SECRET`, `TOKEN_ENCRYPTION_KEY`, `NOTION_OAUTH_CLIENT_SECRET`) are held only in Cloudflare Workers secrets, never in source code, never in git history.
- Key rotation procedures are documented in `SECURITY-KEY-ROTATION.md`.
- Incident response procedure documented in `INCIDENT-RESPONSE.md`.
- Public GitHub repository is audited for secret leaks before every push (`git log --all -p | grep -iE "SECRET|KEY|ntn_"`).

---

## 8. Rights of data subjects

Users can exercise their UK GDPR rights via:

- **Access (Art. 15):** Settings → Download My Data, which returns a JSON file containing the full record EdgeVantage holds.
- **Rectification (Art. 16):** Users control their own data in Notion; EdgeVantage reflects whatever is there. Account-level fields (workspace name, etc.) update automatically on next login.
- **Erasure (Art. 17):** Settings → Delete Account. Immediate purge from storage.
- **Restriction (Art. 18):** Users can disconnect (Settings → Disconnect) to halt processing while retaining the record, or delete their account.
- **Portability (Art. 20):** The "Download My Data" export is machine-readable JSON.
- **Object (Art. 21):** No automated decision-making exists; objection is moot. Users who object to any processing can delete their account.
- **Complaint to supervisory authority:** UK users may complain to the Information Commissioner's Office (https://ico.org.uk).

Requests outside the in-app flows can be made via the GitHub issues link above. Response within 30 days per Art. 12(3).

---

## 9. Automated decision-making

None. EdgeVantage does not perform automated decision-making under Art. 22. All analytics displayed are aggregations of data the user provided in their own Notion trade journal, with no inference, scoring, or recommendation.

---

## 10. Review schedule

This record is reviewed:
- On any material change to data processing (new fields collected, new subprocessors, new purpose).
- At minimum annually.

Last reviewed: 2026-04-18.
Next review due: 2027-04-18.
