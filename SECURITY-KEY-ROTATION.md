# EdgeVantage — Secret & Key Rotation Runbook

**Audience:** the maintainer (you).
**Purpose:** step-by-step instructions for rotating each secret EdgeVantage depends on, so a rotation is never a "think about it for an hour" task.

---

## Secrets in scope

| Name | Sensitivity | Rotation pain | Rotation frequency |
|---|---|---|---|
| `TOKEN_ENCRYPTION_KEY` | Highest — encrypts every stored Notion token | High (see below) | Only if compromised |
| `SESSION_SECRET` | High — HMAC key for sessions | Low (everyone logs out once) | Annually or on suspicion |
| `NOTION_OAUTH_CLIENT_SECRET` | Medium — lets anyone impersonate the integration | Medium (requires Notion console change) | Annually or on suspicion |
| `NOTION_OAUTH_CLIENT_ID` | Low — public anyway | N/A | N/A |

All live in Cloudflare Workers secrets on the `edgetrack-api` worker.

---

## When to rotate

Rotate **immediately** if any of these happen:
- You accidentally committed a secret to git (even if reverted — git history is forever)
- A laptop or environment that had a secret in a file is lost/stolen
- You see unexpected KV activity or unrecognised sessions in logs
- A third-party service in your dependency chain announces a breach
- You start sharing repo access with someone new and want a clean slate

Rotate **routinely** at least once a year for `SESSION_SECRET` and `NOTION_OAUTH_CLIENT_SECRET`. Document the rotation date somewhere (back of a notebook, whatever).

**Do not** rotate `TOKEN_ENCRYPTION_KEY` unless you have to — see its section below.

---

## Rotating `SESSION_SECRET`

**Impact:** All logged-in users are logged out the moment you deploy. They'll need to log in again via OAuth or token. Their data is untouched.

1. Generate a new random 32+ byte string:
   ```
   openssl rand -hex 32
   ```
   (or use any password manager's generator with 64+ character length)

2. Update the secret in Cloudflare:
   - Dashboard → Workers & Pages → `edgetrack-api` → Settings → Variables and Secrets
   - Find `SESSION_SECRET` → click its menu → Edit
   - Paste the new value → Save → Deploy

3. Done. No code change needed. Sessions signed with the old secret will fail verification and get treated as logged-out.

4. Log the rotation date somewhere you'll remember to check next year.

---

## Rotating `NOTION_OAUTH_CLIENT_SECRET`

**Impact:** Until the new secret is deployed to Cloudflare, `/auth/notion/callback` will fail for all new OAuth attempts. Existing users already connected are unaffected — their access tokens continue to work.

1. Go to https://www.notion.so/profile/integrations → Public integrations → EdgeVantage.

2. Scroll to "OAuth Client Secret" → click "Reset" / "Regenerate" (name varies with Notion's UI).

3. **Copy the new secret immediately.** Notion may not show it again.

4. In Cloudflare:
   - Workers & Pages → `edgetrack-api` → Settings → Variables and Secrets
   - Find `NOTION_OAUTH_CLIENT_SECRET` → Edit → paste the new value → Deploy

5. Test OAuth end-to-end in an incognito window before assuming it worked.

6. Log the rotation date.

---

## Rotating `TOKEN_ENCRYPTION_KEY`

**This is a hard one. Read fully before starting.**

The encryption key is used to decrypt every stored Notion access token. If you rotate it naively, every existing user's token becomes undecryptable garbage — effectively logging them all out and forcing a full reconnect.

There are three approaches. Pick the one that matches your situation.

### Option A — Emergency rotation (key is compromised)

If you believe the old key is exposed, you want *all* old tokens to become unusable *immediately*, even at the cost of forcing every user to reconnect.

1. Generate a new key:
   ```
   openssl rand -hex 32
   ```

2. Update `TOKEN_ENCRYPTION_KEY` in Cloudflare → Deploy.

3. At this moment, every user's stored token is garbage. Their next dashboard load will fail with a decryption error.

4. (Optional but recommended) Send all users an email / announcement: "EdgeVantage rotated an encryption key after a security review. Please reconnect at [URL] — your mapping and preferences are preserved."

5. The old revoked Notion tokens remain valid on Notion's side until Notion naturally expires them (or until users revoke them manually). If the key compromise is also a token compromise, you should:
   - Mass-revoke all Notion tokens (requires calling Notion's API for each stored user — see "Mass token revocation" below), OR
   - Accept that tokens in attacker hands remain valid until Notion's own expiry

### Option B — Planned rotation with graceful migration

You want to rotate preemptively (annual hygiene) without forcing anyone to reconnect. This requires temporary code changes.

The idea: run both keys in parallel for a migration window, re-encrypt tokens lazily on next read, then drop the old key.

1. Introduce a second secret: `TOKEN_ENCRYPTION_KEY_NEW`. Deploy it alongside the existing one.

2. Modify `encryptToken()` in the worker to always encrypt with the *new* key, prefixing the ciphertext with a version marker (e.g. `v2:<iv>:<ciphertext>`).

3. Modify `decryptToken()` to:
   - If the ciphertext starts with `v2:`, decrypt with the new key.
   - Otherwise, decrypt with the old key, then re-encrypt with the new key and write it back to KV.

4. Deploy. Over the following weeks, as users log in, their tokens are silently re-encrypted.

5. After some window (say 90 days, matching your user-record TTL), all active users have migrated. Remove the old key handling from the code, rename `TOKEN_ENCRYPTION_KEY_NEW` → `TOKEN_ENCRYPTION_KEY`, deploy.

Not trivial. Budget half a day.

### Option C — Don't rotate it

For a personal project at your current scale, **not rotating is genuinely defensible** provided:
- The key was generated securely in the first place (long random value)
- It lives only in Cloudflare secrets, never in a file, never in git
- You trust Cloudflare's access controls to your worker

Many real production systems go years without rotating encryption keys for this reason. Rotation is insurance against *your* operational mistakes (leaked value, lost device with a copy), not against cryptographic weakness — AES-256-GCM does not become weaker over time.

---

## Mass token revocation

If you need to revoke every user's Notion access (e.g. after a suspected breach):

There is no bulk endpoint — each revocation is a per-user action on Notion's side. The realistic options are:

1. **Ask users to revoke themselves.** Send a message pointing everyone to `notion.so/profile/integrations` → remove EdgeVantage. They do this in seconds.

2. **Invalidate all sessions server-side.** Purge every `session:*` key from KV (via Cloudflare dashboard or `wrangler kv:key list` / `wrangler kv:key delete`). This doesn't revoke the Notion tokens themselves, but it does force every user to log in again via OAuth, which after re-connecting gives them a fresh token — and the old stored token, whatever it's worth, becomes irrelevant for EdgeVantage's purposes.

3. **Delete all user records.** Nuclear — purge every `user:*` key. Forces everyone to fully reconnect. Rarely the right answer.

---

## Secrets hygiene checklist

Review this quarterly:

- [ ] Cloudflare account has 2FA enabled
- [ ] No secret has ever been committed to git (search history with `git log --all -p | grep -iE "SECRET|KEY|ntn_|secret_"`)
- [ ] The maintainer's laptop has full-disk encryption
- [ ] Password manager is used for all generated secrets (not stored in browser history or chat logs)
- [ ] GitHub account has 2FA enabled
- [ ] Notion account has 2FA enabled (protects the integration config)
- [ ] There is no `.env` file with real secrets anywhere on disk

---

_Last reviewed: 2026-04-18._
