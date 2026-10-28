# Vaultline: Secure File-Sharing Platform (Spec)

**Status:** draft v1, written before coding. Update this file whenever a decision changes, and note why.

## 1. What it is

Vaultline lets a registered user upload a file, store it encrypted, and share it through a link that is hard to guess, expires, can be limited to one download, and can be revoked. Every access attempt is logged.

**Why I'm building it:** to practise building a Java backend where security decisions are part of the design, and to test the result by attacking my own app.

## 2. Scope

**In scope:** registration and login, encrypted upload and download, share links (expiry, download limit, optional password, revoke), access log, rate limiting and lockout, a simple web UI, Docker setup, tests.

**Out of scope (for v1):** email verification, payments, virus scanning, file previews, multiple storage regions, mobile app.

## 3. Users and roles

| Role | Can do |
|---|---|
| USER | Upload, list, download and delete own files; create, view and revoke own share links; see own access log |
| ADMIN | See all access logs; lock or unlock users. Cannot read file contents |
| Anonymous (link holder) | Download one file via a valid link, nothing else |

## 4. Tech stack

Java 17, Spring Boot 3 (Maven), Spring Security, Spring Data JPA/Hibernate, MySQL 8, Docker Compose, JUnit 5 and Mockito, a simple React frontend (fallback: Thymeleaf pages if React slows the project down).

## 5. Key design decisions

1. **Passwords:** stored only as BCrypt hashes. Never logged, never returned by the API.
2. **Login:** JWT access tokens with a short lifetime. After 5 wrong passwords an account locks for 15 minutes.
3. **File encryption:** each file is encrypted with AES-256-GCM using a fresh random IV. The master key is read from an environment variable, never from the code or the repo. *Known limitation:* a real system would use a managed key service (for example AWS KMS) with per-file data keys.
4. **Share links:** the token is 32 random bytes from `SecureRandom`, URL-safe encoded. **Only the SHA-256 hash of the token is stored**, so a leaked database does not leak working links.
5. **One-time links:** the download counter is updated with a single atomic SQL statement (`UPDATE ... SET download_count = download_count + 1 WHERE token_hash = ? AND download_count < max_downloads AND revoked = false AND expires_at > now()`). If zero rows change, the request is refused. This prevents two simultaneous requests from both succeeding.
6. **Stored files:** saved under a random UUID name, never the user's filename. Original name is kept in the database and sanitised before use.
7. **Authorisation:** a user who requests a file they don't own gets `404`, not `403`, so file IDs cannot be probed.
8. **Limits:** max file size 10 MB, allow-list of file types, rate limit on login and on link access (per IP).

## 6. Data model

| Table | Main columns |
|---|---|
| `users` | id, email (unique), password_hash, role, failed_attempts, locked_until, created_at |
| `files` | id, owner_id, original_name, stored_name (UUID), size_bytes, content_type, iv, created_at |
| `share_links` | id, file_id, token_hash (unique), password_hash (nullable), expires_at, max_downloads, download_count, revoked, created_at |
| `access_log` | id, link_id (nullable), user_id (nullable), event_type, ip, user_agent, success, reason, created_at |

## 7. API contract (v1)

| Method and path | Auth | Purpose | Main responses |
|---|---|---|---|
| `POST /api/auth/register` | none | Create account | 201, 400, 409 |
| `POST /api/auth/login` | none | Get JWT | 200, 401, 423 (locked) |
| `POST /api/files` | USER | Upload (multipart) | 201, 400, 413, 415 |
| `GET /api/files` | USER | List own files | 200 |
| `GET /api/files/{id}` | USER | Download own file | 200, 404 |
| `DELETE /api/files/{id}` | USER | Delete file and its links | 204, 404 |
| `POST /api/files/{id}/links` | USER | Create link (expiry, max downloads, optional password) | 201, 400, 404 |
| `DELETE /api/links/{id}` | USER | Revoke link | 204, 404 |
| `GET /api/links/{id}/log` | USER | Access history of own link | 200, 404 |
| `GET /s/{token}` | none | Download via link (password in header if required) | 200, 401, 404, 410 (expired or used up), 429 |
| `GET /api/admin/log` | ADMIN | All access logs | 200, 403 |

Errors use one JSON shape: `{ "error": "CODE", "message": "..." }`. Expired, revoked and unknown links all return the same `404` body, so an attacker cannot tell them apart.

## 8. Edge cases to handle

- Two simultaneous requests on a one-time link: exactly one succeeds.
- Link expires while a download is in progress: the download that already started finishes; new ones are refused.
- User deletes a file that has active links: links stop working immediately.
- Upload interrupted halfway: no partial file is left on disk or in the database.
- Same file uploaded twice: stored as two separate files.
- Wrong link password: counted as a failed attempt and logged; repeated failures are rate limited.
- Filename with path characters (`../../x`) or unusual unicode: sanitised.
- Oversized upload and disallowed type: rejected before the full file is read where possible.

## 9. Test plan

**Automated (JUnit 5 and Mockito):** password hashing and lockout counter; token generation (length, uniqueness); expiry and download-limit rules; encryption round trip (decrypt gives the original bytes); authorisation (user B cannot read user A's file); concurrent one-time download test.

**Manual attack checklist.** Fill the last two columns as I test, including the attacks that worked:

| Attack | Expected defence | Result | Fix made |
|---|---|---|---|
| Guess passwords repeatedly | Lockout after 5 tries | | |
| Guess or iterate share tokens | Long random token, rate limit, uniform 404 | | |
| Open another user's file by changing the ID | 404 | | |
| Upload `../../evil` as filename | Sanitised, random stored name | | |
| Upload oversized or disallowed file | 413 or 415 | | |
| Reuse a one-time link twice at once | One success | | |
| Read stored file straight from disk | Unreadable without the key | | |
| Read the database for passwords or tokens | Only hashes present | | |

## 10. Milestones

| Stage | Done when |
|---|---|
| Week 1 | Register and login work; password hashes in the database; lockout works |
| Week 2 | Upload and download work; file on disk is unreadable |
| Week 3 | Share links with expiry, one-time use, password and revoke; access log written |
| Week 4 | Rate limits and limits in place; attack table completed |
| Week 5 | Simple UI covers every feature |
| Week 6 | One-command Docker start; tests passing; README and 2-minute demo finished |

## 11. Definition of done

A stranger can clone the repo, run one command, and complete the full flow: register, upload, share, download via link, revoke. The attack table is filled in honestly. The README explains the design decisions above and one real problem I hit and how I fixed it.
