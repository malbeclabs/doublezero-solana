# Revenue Distribution Program

The Revenue Distribution program is a smart contract that channels revenue from
DoubleZero users to network contributors. Like any smart contract, it functions
as a state machine—here, one that manages accounting to enforce the 2Z
tokenomics.

Rewards to contributors are not distributed continuously but at discrete
intervals, aligned with the start of new epochs on the DoubleZero Ledger
(approximately every 48 hours). At each epoch, the system calculates both the
debt owed and the performance of network contributors.

For more information, please read the following articles for the concepts behind
the core mechanism of this smart contract:

- [A Primer to the 2Z Token]
- [Rewards to Network Contributors]
- [Value and Prices for Solana Validators]
- [Integrity in the Rewards Model]

## Guides

- [Lifecycle of a Distribution]

[A Primer to the 2Z Token]: https://doublezero.xyz/journal/a-primer-to-the-2z-token
[Lifecycle of a Distribution]: LIFECYCLE_OF_A_DISTRIBUTION.md
[Rewards to Network Contributors]: https://doublezero.xyz/journal/rewards-to-network-contributors
[Value and Prices for Solana Validators]: https://doublezero.xyz/journal/value-and-prices-for-solana-validators
[Integrity in the Rewards Model]: https://doublezero.xyz/journal/integrity-in-the-rewards-model