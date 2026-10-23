# Seed Phrases and Shamir Backups

The seed phrase is still the most common wallet backup. It is simple, portable and vendor-independent, and it is a single secret that grants full control to whoever reads it. Shamir secret sharing splits that secret so no single location holds it. This chapter covers BIP 39, SLIP-39 and the practical failure modes of both.

## BIP 39 in brief

BIP 39 encodes 128 to 256 bits of entropy as 12 to 24 words from a 2,048-word list, with a short checksum. The words plus an optional passphrase are stretched with PBKDF2-HMAC-SHA512 (2,048 iterations) into a 512-bit seed, from which BIP 32 derives keys.

| Property | Consequence |
|---|---|
| Checksum is a few bits | catches most typos, not all; not error-correcting |
| Passphrase is unchecked | any passphrase yields a valid wallet; typos create a different, empty wallet |
| PBKDF2 with 2,048 iterations | weak stretching; passphrases must be strong on their own |
| Words are standardised | phrases move between vendors, but so does stolen value |

The passphrase ("25th word") defends against someone who finds the written words. It also enables plausible decoy wallets. But it must be remembered or stored separately, and an unchecked passphrase typo means apparent loss of funds.

## Shamir secret sharing

Shamir's scheme (1979) splits a secret into `n` shares such that any `t` reconstruct it and fewer than `t` reveal nothing. For wallets, SLIP-39 (from SatoshiLabs) standardises this with word-based shares, checksums designed for error detection, and **groups**: for example, "2 of these 3 groups, where group A needs 2 of 3 shares and group B needs 1 of 1".

```text
Group threshold: 2 of 3
  Group 1 (family):    2 of 3 shares
  Group 2 (safe box):  1 of 1 share
  Group 3 (lawyer):    1 of 1 share
```

Important properties and limits:

- **SLIP-39 is not compatible with BIP 39.** A SLIP-39 secret produces a different seed from a BIP 39 phrase of the same entropy. Do not split an existing BIP 39 phrase with SLIP-39 and expect the same wallet unless the tool explicitly supports that conversion.
- **Reconstruction exposes the secret.** Whatever device combines the shares holds the full master secret. Hardware wallets that support SLIP-39 do this on-device.
- **Shares are not independent if stored together.** Two shares in the same safe count as one location for threat purposes.
- **No proactive refresh.** Unlike MPC, plain Shamir shares cannot be refreshed without generating new shares from the reconstructed secret.

Avoid home-made schemes such as "split the 24 words into three groups of 8". Each fragment reveals part of the secret and drastically reduces the remaining search space. Use a vetted implementation of Shamir (SLIP-39, or Blockchain Commons' SSKR) instead.

## Custodial seed backup services

In 2023 Ledger announced Ledger Recover, an optional paid service in which an encrypted backup of the seed is split among several companies, with identity verification for recovery. The announcement prompted wide debate because it showed that device firmware could export the seed in some form at all. The security question for any such service is the same as for social recovery: how many independent parties must be compromised or coerced, and how does identity verification resist impersonation?

## Physical storage

- Metal plates resist fire and water; paper does not.
- Photographs and cloud notes of seed phrases are a frequent theft source; malware searches images and notes for word lists.
- Instructions for heirs should explain what the material is without storing it beside the shares.
- Test recovery periodically on an offline device or a hardware wallet's "check backup" function rather than by typing the phrase into a computer.

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Create and recover a SLIP-39 backup with the reference implementation, using test entropy only.

1. Install the reference tool:

```bash
python3 -m venv v && . v/bin/activate
pip install shamir-mnemonic
```

2. Create a simple 2-of-3 backup and inspect it:

```bash
shamir create 2of3
```

The tool prints the master secret (in hex) and three share mnemonics. Write them to three separate files.

3. Recover using any two shares:

```bash
shamir recover
```

Enter two shares when prompted. The recovered master secret matches.
4. Try recovery with one share only. The tool cannot reconstruct the secret, demonstrating the threshold.
5. Change one word in a share and try again. SLIP-39's checksum rejects the corrupted share.
6. Use `shamir create custom` (see `shamir create --help` for options in your version) to build the two-level group design shown earlier, and test that "two family shares plus the lawyer share" works while "three family shares" alone does not.

## Common pitfalls

- Splitting BIP 39 words into fragments instead of using real secret sharing.
- Storing a threshold of shares in one place or one cloud account.
- Assuming SLIP-39 and BIP 39 produce the same wallet.
- Reassembling shares on an everyday laptop.
- Photographing seed phrases "temporarily".
- Weak passphrases on top of BIP 39's light key stretching.

## Exercises

1. Calculate the remaining search space if an attacker obtains 16 of 24 BIP 39 words in known positions, and compare with a single SLIP-39 share from a 2-of-3 split.
2. Design a SLIP-39 group policy for a family of four with one safe-deposit box, and list which combinations recover and which single events compromise it.
3. Write a script that generates a BIP 39 phrase, derives an address, and confirms that a passphrase typo produces a different address; use only testnet.

## Further reading

- SLIP-0039, Shamir's Secret-Sharing for Mnemonic Codes: <https://github.com/satoshilabs/slips/blob/master/slip-0039.md>
- python-shamir-mnemonic: <https://github.com/trezor/python-shamir-mnemonic>
- Adi Shamir, "How to Share a Secret", Communications of the ACM (1979)
- Blockchain Commons SSKR documentation: <https://github.com/BlockchainCommons>
- BIP 32 and BIP 39 in the BIPs repository
