# Cloud Backups, MPC Recovery Shares and Recovery Phishing

Many consumer wallets avoid seed phrases entirely. They encrypt a key or key share and store it in the user's cloud account, or split control between the user's device and a provider using MPC. These designs reduce loss, but they tie wallet security to cloud accounts, passwords and provider processes. Meanwhile, attackers have learned that the easiest way to steal a wallet is to convince the owner to "recover" it for them. This chapter covers both sides.

## Cloud-encrypted backups

A typical design:

1. The wallet generates a key (or an MPC share) on the device.
2. It encrypts the key with a key derived from a user password, or with a key protected by the platform (for example a passkey or the platform keychain).
3. It uploads the ciphertext to the user's cloud storage (iCloud, Google Drive) or to the provider.
4. On a new device, the user signs into the cloud and supplies the password or passkey to decrypt.

| Design choice | Risk |
|---|---|
| Password-derived key with weak KDF | offline guessing once the ciphertext leaks |
| Password-derived key with strong KDF (Argon2id, scrypt) | still weak passwords fall to guessing |
| Platform-protected key (keychain, passkey PRF) | security reduces to the platform account and its recovery |
| Provider-held decryption key | provider insiders, breaches and legal requests |
| Ciphertext in user's cloud | cloud account takeover gives the attacker unlimited offline attempts |

The core point: once an attacker has the ciphertext, the password is the only barrier, and they can try guesses at their own speed. Use a memory-hard KDF with parameters measured on the slowest supported device, and require passwords that resist offline attack, or avoid password-only protection.

The WebAuthn PRF extension lets a passkey derive a secret that can encrypt a backup. That makes the backup as strong as the passkey account, which is good against phishing but ties recovery to the platform's own account-recovery process. See **passkeys-webauthn-web3**.

## MPC recovery shares

MPC wallets split the key into shares, for example one on the device, one with the provider, and one recovery share in the user's cloud or with a third party. Recovery combines a threshold of shares into a new set, ideally without ever reconstructing the full key.

Questions to ask:

- Does any recovery path reconstruct the full key on one machine?
- Can the provider plus the cloud-stored share meet the threshold without the user? If so, a provider breach plus cloud takeover is a full compromise.
- How are shares refreshed after a device is lost, so the lost device's share becomes useless?
- What identity checks does the provider apply before helping with recovery, and can they be socially engineered?

See **mpc-wallet-security** for protocol-level issues.

## Recovery phishing

Attackers impersonate recovery because users expect it to be confusing.

- **Fake "validate", "sync" or "rectify" sites** ask for the seed phrase to fix a supposed problem. No legitimate service needs your seed phrase.
- **Fake support** in Discord, Telegram or X replies to users asking for help, then directs them to such sites or to screen-share sessions.
- **Fake wallet apps** in app stores or search ads that ask users to "restore" an existing wallet.
- **SIM swap and email takeover** to pass the provider's identity checks or receive reset links. The FBI and FTC have published repeated warnings about SIM swapping.
- **Malicious guardian enrolment:** convincing a user to add the attacker as a guardian or recovery address, then waiting.
- **Coercion and "wrench attacks"** against people known to hold crypto.

Defensive design:

- Never display or request a seed phrase in a web page; restore only inside the wallet app or hardware device.
- Make recovery initiation conspicuous: notifications on all devices, delays, and plain-language warnings.
- Do not use SMS as a sole recovery factor.
- Treat guardian and recovery-address changes as high-risk, delayed actions.

## Using AI for phishing detection

Classifiers and language models can flag phishing pages and support impersonators. They are useful as triage: the model proposes "likely phishing", and a deterministic check (domain allowlists, certificate and registration data, presence of a seed-phrase input field) or a human analyst confirms before a takedown request. Report false positives and negatives to improve the rules.

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Measure how KDF choice affects offline guessing against an encrypted backup. Use test data only.

1. Install `argon2-cffi` and use Python's built-in `hashlib.pbkdf2_hmac` and `hashlib.scrypt`.
2. Time one derivation each:

```python
import time, hashlib
from argon2.low_level import hash_secret_raw, Type
pw, salt = b"Summer2024!", b"0123456789abcdef"
def t(f):
    s = time.perf_counter(); f(); return time.perf_counter() - s
print("pbkdf2 2048", t(lambda: hashlib.pbkdf2_hmac("sha512", pw, salt, 2048)))
print("scrypt n=2^15", t(lambda: hashlib.scrypt(pw, salt=salt, n=2**15, r=8, p=1, maxmem=2**26)))
print("argon2id 64MiB", t(lambda: hash_secret_raw(pw, salt, time_cost=3, memory_cost=65536, parallelism=1, hash_len=32, type=Type.ID)))
```

3. Using a public wordlist of 10,000 common passwords, estimate how long a single CPU core needs to try them all at each setting. Note that attackers use GPUs, where PBKDF2 is much faster relative to memory-hard functions.
4. Encrypt a test key with a key from each KDF (for example with `cryptography`'s AES-GCM), then run the wordlist against it with your own script to confirm a common password is found. The script is the checker: it either recovers the key or does not.

## Common pitfalls

- Low KDF parameters chosen for app-start speed.
- Backups whose only protection is the cloud account.
- MPC recovery paths where provider plus cloud share suffice without the user.
- SMS-only recovery.
- Instant guardian or recovery-address changes.
- Support channels that ask users to share screens during recovery.

## Exercises

1. Tune Argon2id parameters so derivation takes about 0.5 seconds on a low-end phone (or a throttled VM) and document the memory cost.
2. Map an MPC wallet's documented recovery flow and list every combination of parties that can reach the threshold.
3. Collect five public recovery-phishing warnings from wallet vendors and extract the common lures into a checklist for a support team.

## Further reading

- RFC 9106, Argon2 Memory-Hard Function: <https://www.rfc-editor.org/rfc/rfc9106>
- OWASP Password Storage Cheat Sheet: <https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html>
- W3C WebAuthn Level 3, PRF extension: <https://www.w3.org/TR/webauthn-3/>
- FBI IC3 public service announcements on SIM swapping: <https://www.ic3.gov>
- See **mpc-wallet-security**, **passkeys-webauthn-web3** and **browser-extension-wallet-malware**
