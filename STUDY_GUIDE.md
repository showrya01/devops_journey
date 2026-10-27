# Study guide for the follow-up call (plain English)

Do NOT put this file in your GitHub repo. It is for you only.

On the call they will ask: "Why did you do X?" You do not need to know Python. You need to know the reason behind each decision. Learn the 10 ideas below. Say each one in your own words, out loud, until it feels natural.

---

## The 10 ideas

### 1. Why we hash passwords (and why scrypt)
- **Plain version:** We never save the real password. We save a scrambled version that cannot be un-scrambled. When you log in, we scramble what you typed and compare.
- **Why scrypt:** It is deliberately slow and uses a lot of memory. A normal fast scramble (MD5, SHA-256) lets an attacker try billions of guesses per second. scrypt makes each guess expensive.
- **Say it like this:** "I used scrypt because it is slow and memory-hard, so offline guessing is costly. A fast hash like SHA-256 would let attackers try billions of passwords per second."

### 2. What a salt is
- **Plain version:** A random value added to each password before scrambling. Every user gets a different one.
- **Why:** Two people with the same password get different scrambles, and attackers cannot use ready-made lookup tables.
- **Say it like this:** "Each user has a unique random salt, so identical passwords produce different hashes and precomputed tables are useless."

### 3. Opaque token vs JWT (the choice I made)
- **Plain version:** After login the server gives you a long random string, like a coat-check ticket. The server keeps a list of valid tickets. A JWT is instead a signed note that carries your identity inside it.
- **Why I chose the ticket:** I can cancel one instantly by deleting it from the list. No secret signing key exists that could be stolen to fake everyone's login.
- **Trade-off (be honest):** The server does a database lookup on every request. JWTs avoid that, but they are harder to revoke.
- **Say it like this:** "I chose opaque server-side tokens because they can be revoked immediately and there is no signing key to leak. The cost is one database lookup per request, which is fine here."

### 4. Why we store a HASH of the token
- **Plain version:** The database holds a scrambled copy of each token, not the token itself.
- **Why:** If someone steals the database, they get scrambled copies and cannot log in as anyone.
- **Follow-up they may ask:** "Why is a fast hash okay for the token but not for passwords?" Answer: "The token is 256 bits of pure randomness, so it cannot be guessed even with a fast hash. Passwords are chosen by humans and are guessable, so they need a slow hash."

### 5. The access check (the most important one)
- **Plain version:** Being logged in is not enough. On every read, the server asks: "Do you own this record, or are you an admin?" If no, you get nothing.
- **Where:** `can_read_record` in `app/auth/access.py`. It is one line: admin OR owner.
- **Say it like this:** "Authentication says who you are. Authorisation says what you may touch. I check ownership on every single record read, not just whether the user is logged in."

### 6. Why 404 and not 403
- **Plain version:** If someone asks for a record that is not theirs, we answer "not found", the same answer as for a record that truly does not exist.
- **Why:** A "403 forbidden" would confirm the record exists. That leaks information. Same answer both ways means attackers learn nothing.
- **Say it like this:** "Returning the same 404 for missing and not-yours stops attackers from discovering which ids exist."

### 7. Why UUIDs for record ids
- **Plain version:** Ids are long random strings, not 1, 2, 3.
- **Why:** Nobody can guess the next one. But be honest: this is a bonus layer. The real protection is the access check in idea 5.
- **Say it like this:** "UUIDs are defence in depth. If the access check were ever wrong, ids still could not be guessed easily, but I do not rely on that."

### 8. How we stop users becoming admin
- **Plain version:** The server decides roles, never the user. Registration always makes a normal user. If the request even contains the word "role", it is rejected with 403. No route exists to change roles. The first admin is created by running a script on the server.
- **Say it like this:** "The role is assigned server-side only. Registration hard-codes 'user', any role field is rejected, there is no endpoint to edit roles, and admins are created by a server-side script."

### 9. Brute-force lockout (and its weakness)
- **Plain version:** After 5 wrong passwords, that account is locked for 15 minutes. The 6th attempt gets a 429 "too many requests".
- **Why per account, not per IP:** Attackers can switch IP addresses easily, but they cannot change which account they are attacking.
- **The weakness (say this before they ask):** An attacker can lock someone else's account on purpose by typing wrong passwords. I documented that, and the fix would be gradual delays instead of a hard lock, plus extra IP-based limits.
- **Say it like this:** "Lockout is per account as required. The trade-off is that an attacker can deliberately lock out a victim, so in production I would add progressive delays and IP rate limiting."

### 10. If the user table leaked, what could an attacker get?
They care about this one most. Memorise the structure:
- **They get:** usernames, roles, and password hashes with salts.
- **They cannot get:** the real passwords directly, or working login tokens (we only store hashes of those), or forged logins (no signing key exists).
- **What they CAN do:** guess passwords offline against the hashes. Weak passwords like "password123" would probably be cracked. scrypt makes it slow but not impossible.
- **Honest extras:** they learn who the admin is. If records are in the same stolen database, record contents are readable because they are not encrypted.
- **Say it like this:** "They cannot read passwords or hijack sessions, but they can try offline guessing, so weak passwords are at risk. And if the records table leaked too, those are plaintext, which is why encryption at rest would be my next improvement."

---

## Smaller things they might ask

**What is SQL injection and how did you stop it?**
Attackers type SQL commands into a text box hoping the database runs them. I never glue user input into a query. I use `?` placeholders, so the input is always treated as data. My tests send things like `' OR '1'='1` and nothing happens.

**Why did you return 400 and not crash on bad input?**
Every field is checked for presence, type and length. Bad input gets a clean 400 message. The server also has a catch-all so that real errors show a generic message, never a stack trace that could reveal how the code works.

**Why is there a maximum password length?**
Hashing is slow on purpose. Without a limit, an attacker could send a 10 MB password and make the server burn CPU. 128 characters is plenty.

**Why a timing trick for unknown usernames?**
If unknown usernames returned instantly but real ones took time to hash, an attacker could tell which usernames exist by measuring speed. For unknown names I hash a dummy password so the time looks the same.

**Where are secrets kept?**
In environment variables, loaded from a `.env` file that is git-ignored. The repo only has `.env.example` with fake values.

**What would you improve with more time?**
Logout/revocation endpoint, per-IP rate limiting, an audit log of failed logins and denied reads, encryption of records at rest, HTTPS enforcement, breached-password checking.

**You used AI. How much of this is yours?**
Be honest: "I used AI to help write the code, and I spent time understanding each decision. Here is why I chose X over Y." Then explain. They said AI is allowed. What they test is whether you understand it.

---

## Practice routine (about 2 hours)

1. Read the ten ideas once.
2. Cover the "Say it like this" lines and explain each idea out loud from memory.
3. Open `README.md` and read the threat model section. It is your written answer. Make sure you agree with every sentence, and change anything that does not sound like you.
4. Run the project once (see the README) and run the tests. Seeing 68 tests pass and being able to say "these tests cover access control and lockout" helps a lot.
5. Skim `app/auth/access.py`, `app/routes/auth_routes.py` and `app/routes/records_routes.py`. They are short. You do not need to understand every line, but you should be able to point at where the access check and the lockout live.

## Where things are, if they ask you to show them

| Question | File |
|---|---|
| Password hashing | `app/auth/passwords.py` |
| Session tokens | `app/auth/sessions.py` |
| "Is the user logged in?" and "may they read this?" | `app/auth/access.py` |
| Lockout and 429 | `app/routes/auth_routes.py` (login function) |
| Role can't be set | `app/validation.py` and `app/routes/auth_routes.py` (register function) |
| Parameterised SQL | `app/storage/users.py`, `app/storage/records.py` |
| Tests | `tests/` folder |
