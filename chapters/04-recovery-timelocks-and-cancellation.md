# Recovery Timelocks, Cancellation and Races

A delay turns recovery from an instant takeover into a contest the legitimate owner can win. But a delay only helps if the owner notices, has a way to respond, and wins the resulting race. This chapter looks at delay design on smart accounts and on Bitcoin, and at the races that decide outcomes.

## What the delay buys

Without a delay, whoever meets the recovery threshold owns the account immediately. With a delay `D`:

- the owner has `D` to notice and cancel a malicious recovery;
- guardians have `D` to notice a malicious guardian change or owner action, if the design allows them to intervene;
- the legitimate owner, if truly locked out, waits `D` for access.

| Delay | Owner reaction window | Cost for honest recovery | Typical use |
|---|---|---|---|
| hours | too short for a holiday or hospital stay | low | low-value hot accounts |
| 2–7 days | reasonable for active users with notifications | moderate | consumer smart wallets |
| weeks to months | covers long absences | high | savings, inheritance, cold storage |

Pick the delay from how long the owner might realistically be unreachable, not from how impatient a recovering user might be.

## Who can cancel

| Cancel authority | Defends against | Fails when |
|---|---|---|
| Owner key only | malicious guardians | owner key is stolen (thief cancels honest recovery) |
| Guardian threshold only | stolen owner key | guardians are the attackers |
| Either | most single-party attacks | both sides active: endless cancel/restart loop |
| Owner key, but guardian supermajority overrides | stolen key with honest guardians | supermajority collusion |

There is no universally right answer. The design must say which party wins when owner key and guardians disagree, and the user should understand it.

## Notification

A delay is useless if nobody hears about the recovery. Notifications should go through channels the attacker is unlikely to control at the same time:

- in-app push to all registered devices;
- email and messaging to addresses set long before the recovery;
- on-chain events watched by a monitoring service the user chose.

Remember that the attacker initiating recovery may already control the user's email or phone (that is often why they can attempt recovery at all). Use several channels and do not let recovery itself change notification settings.

## Griefing and liveness

Short-term griefing: a single guardian, or the thief with the owner key, repeatedly starts and cancels recoveries so legitimate recovery never completes. Defences include rate limits on new recovery attempts, requiring a threshold to start (not just one guardian), and letting a larger threshold override cancellations.

Long-term liveness: guardians lose keys, change phones, or die. Prompt users to review guardians periodically, and treat a guardian set where fewer than `threshold` guardians have been active recently as degraded.

## Timelocked recovery on Bitcoin

Bitcoin wallets can encode recovery in script. Wallets such as Liana (from Wizardsardine) use miniscript descriptors in which the primary key spends at any time and a recovery key (or set of keys) spends only after a relative timelock:

```text
wsh(or_d(pk(PRIMARY),and_v(v:pkh(RECOVERY),older(52560))))
```

`older(52560)` is about one year of blocks. Two properties follow:

- **Each coin's timelock starts at its confirmation.** Coins the owner never moves eventually become spendable by the recovery key. The owner must "refresh" coins by spending them to themselves before the timelock matures.
- **No cancellation transaction exists.** The owner's defence is to move the coins first. That is a race decided by fees and confirmation, not by a contract function.

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Test the cancellation race in Foundry using the contract from chapter 3, and the Bitcoin refresh pattern on regtest.

1. In the Foundry project, write a test where the attacker holds the owner key (simulate with `vm.prank(owner)` from an "attacker" context) and cancels each honest recovery. Show that honest guardians never succeed:

```solidity
function test_stolenKeyBlocksRecovery() public {
    for (uint i; i < 5; i++) {
        vm.prank(g1); acct.approveRecovery(newOwner);
        vm.prank(g2); acct.approveRecovery(newOwner);
        vm.prank(owner); acct.cancel();   // thief with stolen key
    }
    vm.warp(block.timestamp + 30 days);
    vm.expectRevert("not ready");
    acct.execute();
}
```

2. Add a guardian override (for example, 3-of-3 guardians can mark the recovery non-cancellable) and show the test now ends with `newOwner` in control.
3. On Bitcoin regtest, create the descriptor above with a short timelock (`older(20)`), fund it, and confirm the recovery-key spend is rejected by `testmempoolaccept` before 20 blocks and accepted after.
4. Now "refresh": after 15 blocks, spend the coin with the primary key back to a new address of the same descriptor. Mine 10 blocks. The recovery path on the new coin is not yet mature, showing how refreshing resets the clock.

## Common pitfalls

- Delays chosen for convenience rather than for realistic owner absence.
- Notification only through the channel the attacker likely controls.
- Recovery that can change notification settings or guardian sets without delay.
- Not documenting who wins when owner key and guardians disagree.
- Bitcoin timelocked wallets whose owners forget to refresh coins.

## Exercises

1. Implement a rate limit on recovery attempts and show it stops a single-guardian griefing loop in a Foundry test.
2. For the Bitcoin descriptor, write a script that warns 30 days before any UTXO's recovery path matures.
3. Write a one-paragraph user-facing explanation of your design's race rules and test it on someone non-technical.

## Further reading

- Liana wallet: <https://github.com/wizardsardine/liana>
- Miniscript reference: <https://bitcoin.sipa.be/miniscript/>
- BIP 68 and BIP 112 (relative timelocks)
- Foundry cheatcodes reference (`vm.warp`, `vm.prank`): <https://book.getfoundry.sh/cheatcodes/>
- See **account-abstraction-security** and **bitcoin-l2-bitvm-security**
