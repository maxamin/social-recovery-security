# Glossary

- **Seed phrase:** BIP 39 word encoding of wallet entropy, stretched into a master seed.
- **Passphrase (25th word):** optional extra input to BIP 39 seed derivation; any value yields a valid wallet.
- **Shamir secret sharing:** splitting a secret into n shares so any t reconstruct it and fewer reveal nothing.
- **SLIP-39:** word-based Shamir backup standard with groups and checksums.
- **Guardian:** party whose approval is required to recover a smart account.
- **Recovery delay:** waiting period between recovery approval and execution, during which it can be cancelled.
- **Cancellation authority:** the party allowed to stop a pending recovery.
- **Griefing:** repeatedly starting or resetting recovery to prevent legitimate completion.
- **Cloud-encrypted backup:** key material encrypted and stored in the user's cloud account.
- **KDF:** key derivation function, such as PBKDF2, scrypt or Argon2id, that turns a password into a key.
- **Recovery phishing:** social engineering that tricks users into revealing recovery secrets or approving malicious recovery.
- **SIM swap:** taking over a phone number to intercept SMS codes and calls.
- **Refresh (timelock):** spending coins to yourself to restart a relative timelock on a recovery path.
