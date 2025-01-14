# Protocol levers

The protocol provides a number of settings controllable by the DAO. Modifying each of them requires
the caller to have a specific permission. After deploying the DAO, all permissions belong to the DAO
`Voting` app, which can also manage them. This means that, initially, levers can only be changed by
the DAO voting, and other entities can be allowed to do the same only as a result of the voting.

All existing levers are listed below, grouped by the contract.

### A note on upgradeability

The following contracts are upgradeable by the DAO voting:

- [`Lido`](/contracts/lido)
- [`NodeOperatorsRegistry`](/contracts/node-operators-registry)
- [`LidoOracle`](/contracts/lido-oracle)

Upgradeability is implemented by the Aragon kernel and base contracts. To upgrade an app, one needs
the `dao.APP_MANAGER_ROLE` permission provided by Aragon. All upgradeable contracts use the
[Unstructured Storage pattern] in order to provide stable storage structure across upgrades.

[unstructured storage pattern]: https://blog.openzeppelin.com/upgradeability-using-unstructured-storage

## [Lido](/contracts/lido)

### Burning stETH tokens

- Mutator: `burnShares(address _account, uint256 _sharesAmount)`
  - Permission required: `BURN_ROLE`

DAO members can burn token shares via DAO voting to offset slashings using insurance funds.
E.g. protocol was slashed by 5 Ether; by burning the amount of shares corresponding to 5 stETH
the stakers can be made whole.

Currently, the `BURN_ROLE` assigned only to the
[`SelfOwnedStETHBurner`](/contracts/self-owned-steth-burner) contract and constrained by the
[Aragon OS ACL parameter interpretation](https://hack.aragon.org/docs/aragonos-ref#parameter-interpretation)
feature to allow burn stETH token shares only from the contract's own balance. Moreover, the shares
for burning could be obtained only by a direct `Voting` request.

As a result, to burn stETH token shares, DAO needs to explicitly approve `stETH` transfer from
the `Voting` app contract to the `SelfOwnedStETHBurner` contract and request to burn
the received token shares on the next quorum-reaching oracle report.

### Protocol contracts

The addresses of the oracle, treasury, and isurance fund contracts.

Oracle contract is allowed to make periodical updates of beacon stats by calling `handleOracleReport`.
Treasury contract is used to accumulate the protocol treasury fee.
Insurance fund contract is used to accumulate the protocol insurance fee.

- Mutator: `setProtocolContracts(address _oracle, address _treasury, address _insuranceFund)`
  - Permission required: `MANAGE_PROTOCOL_CONTRACTS_ROLE`
- Accessors:
  - `getOracle() returns (address)`
  - `getTreasury() returns (address)`
  - `getInsuranceFund() returns (address)`

The oracle contract serves as a bridge between ETH 2.0 -> ETH oracle committee members and the rest of the protocol,
implementing quorum between the members. The oracle committee members report balances controlled by the DAO
on the ETH 2.0 side, which can go up because of reward accumulation and can go down due to slashing.

### Fee

The total fee, in basis points (`10000` corresponding to `100%`).

- Mutator: `setFee(uint16)`
  - Permission required: `MANAGE_FEE`
- Accessor: `getFee() returns (uint16)`

The fee is taken on staking rewards and distributed between the treasury, the insurance fund, and
node operators.

### Fee distribution

Controls how the fee is distributed between the treasury, the insurance fund, and node operators.
Each fee component is in basis points; the sum of all components must add up to 1 (`10000` basis points).

- Mutator: `setFeeDistribution(uint16 treasury, uint16 insurance, uint16 operators)`
  - Permission required: `MANAGE_FEE`
- Accessor: `getFeeDistribution() returns (uint16 treasury, uint16 insurance, uint16 operators)`

### ETH 2.0 withdrawal Credentials

Credentials to withdraw ETH on ETH 2.0 side after phase 2 is launched.

- Mutator: `setWithdrawalCredentials(bytes)`
  - Permission required: `MANAGE_WITHDRAWAL_KEY`
- Accessor: `getWithdrawalCredentials() returns (bytes)`

The protocol uses these credentials to register new ETH 2.0 validators.

### Deposit loop iteration limit

Controls how many ETH 2.0 deposits can be made in a single transaction.

- A parameter of the `depositBufferedEther(uint256)` function
- Default value: `16`
- [Scenario test](https://github.com/lidofinance/lido-dao/blob/master/test/scenario/lido_deposit_iteration_limit.js)

When someone calls `depositBufferedEther`, `Lido` tries to register as many ETH 2.0 validators
as it can given the buffered Ether amount. The limit is passed as an argument to this function and
is needed to prevent the transaction from [failing due to the block gas limit], which is possible
if the amount of the buffered Ether becomes sufficiently large.

[failing due to the block gas limit]: https://github.com/ConsenSys/smart-contract-best-practices/blob/8f99aef/docs/known_attacks.md#gas-limit-dos-on-a-contract-via-unbounded-operations

### Pausing

- Mutator: `stop()`
  - Permission required: `PAUSE_ROLE`
- Mutator: `resume()`
  - Permission required: `RESUME_ROLE`
- Accessor: `isStopped() returns (bool)`

When paused, `Lido` doesn't accept user submissions, doesn't allow user withdrawals and oracle
report submissions. No token actions (burning, transferring, approving transfers and changing
allowances) are allowed. The following transactions revert:

- Plain Ether transfers;
- calls to `submit(address)`;
- calls to `depositBufferedEther(uint256)`;
- calls to `withdraw(uint256, bytes32)` (withdrawals are not implemented yet).
- calls to `pushBeacon(uint256, uint256)`;
- calls to `burnShares(address, uint256)`;
- calls to `transfer(address, uint256)`;
- calls to `transferFrom(address, address, uint256)`;
- calls to `approve(address, uint256)`;
- calls to `increaseAllowance(address, uint)`;
- calls to `decreaseAllowance(address, uint)`.

### Execution layer rewards

Lido implements an architecture design which was proposed in the Lido Improvement Proposal [#12](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-12.md) to collect the execution level rewards (starting from the Merge hardfork) and distribute them as part of the Lido Oracle report.

These execution layer rewards are initially accumulated on the dedicated [`LidoExecutionLayerRewardsVault`](/contracts/lido-execution-layer-rewards-vault) contract and consists of priority fees and MEV.

There is an additional limit to prevent drastical token rebase events.
See the following issue: [`#405`](https://github.com/lidofinance/lido-dao/issues/405)

- Mutator: `setELRewardsVault()`
  - Permission required: `SET_EL_REWARDS_VAULT_ROLE`

- Mutator: `setELRewardsWithdrawalLimit()`
  - Permission required: `SET_EL_REWARDS_WITHDRAWAL_LIMIT_ROLE`

- Accessors:
  - `getELRewardsVault()`;
  - `getELRewardsWithdrawalLimit()`.

### Staking rate limiting

Lido features a safeguard mechanism to prevent huge APR losses facing the [post-merge entry queue demand](https://blog.lido.fi/modelling-the-entry-queue-post-merge-an-analysis-of-impacts-on-lidos-socialized-model/).

New staking requests could be rate-limited with a soft moving cap for the stake amount per desired period.

Limit explanation scheme:
```
    * ▲ Stake limit
    * │.....  .....   ........ ...            ....     ... Stake limit = max
    * │      .       .        .   .   .      .    . . .
    * │     .       .              . .  . . .      . .
    * │            .                .  . . .
    * │──────────────────────────────────────────────────> Time
    * │     ^      ^          ^   ^^^  ^ ^ ^     ^^^ ^     Stake events
```

- Mutators: `resumeStaking()`, `setStakingLimit(uint256, uint256)`, `removeStakingLimit()`
  - Permission required: `STAKING_CONTROL_ROLE`

- Mutator: `pauseStaking()`
  - Permission required: `STAKING_PAUSE_ROLE`

- Accessors:
  - `isStakingPaused()`
  - `getCurrentStakeLimit()`
  - `getStakeLimitFullInfo()`

When staking is paused, `Lido` doesn't accept user submissions. The following transactions revert:

- Plain Ether transfers;
- calls to `submit(address)`.

For details, see the Lido Improvement Proposal [#14](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-14.md).

### TODO

- Transfer to vault (`transferToVault()`)

## [NodeOperatorsRegistry](/contracts/node-operators-registry)

### Node Operators list

- Mutator: `addNodeOperator(string _name, address _rewardAddress, uint64 _stakingLimit)`
  - Permission required: `ADD_NODE_OPERATOR_ROLE`
- Mutator: `setNodeOperatorName(uint256 _id, string _name)`
  - Permission required: `SET_NODE_OPERATOR_NAME_ROLE`
- Mutator: `setNodeOperatorRewardAddress(uint256 _id, address _rewardAddress)`
  - Permission required: `SET_NODE_OPERATOR_ADDRESS_ROLE`
- Mutator: `setNodeOperatorStakingLimit(uint256 _id, uint64 _stakingLimit)`
  - Permission required: `SET_NODE_OPERATOR_LIMIT_ROLE`

Node Operators act as validators on the Beacon chain for the benefit of the protocol. Each
node operator submits no more than `_stakingLimit` signing keys that will be used later
by the protocol for registering the corresponding ETH 2.0 validators. As oracle committee
reports rewards on the ETH 2.0 side, the fee is taken on these rewards, and part of that fee
is sent to node operators’ reward addresses (`_rewardAddress`).

### Deactivating a node operator

- Mutator: `setNodeOperatorActive(uint256 _id, bool _active)`
  - Permission required: `SET_NODE_OPERATOR_ACTIVE_ROLE`

Misbehaving node operators can be deactivated by calling this function. The protocol skips
deactivated operators during validator registration; also, deactivated operators don’t
take part in fee distribution.

### Managing node operator’s signing keys

- Mutator: `addSigningKeys(uint256 _operator_id, uint256 _quantity, bytes _pubkeys, bytes _signatures)`
  - Permission required: `MANAGE_SIGNING_KEYS`
- Mutator: `removeSigningKey(uint256 _operator_id, uint256 _index)`
  - Permission required: `MANAGE_SIGNING_KEYS`

Allow to manage signing keys for the given node operator.

> Signing keys can also be managed by the reward address of a signing provider by calling
> the equivalent functions with the `OperatorBH` suffix: `addSigningKeysOperatorBH`, `removeSigningKeyOperatorBH`.

### Reporting new stopped validators

- Mutator: `reportStoppedValidators(uint256 _id, uint64 _stoppedIncrement)`
  - Permission required: `REPORT_STOPPED_VALIDATORS_ROLE`

Allows to report that `_stoppedIncrement` more validators of a node operator have become stopped.

## [LidoOracle](/contracts/lido-oracle)

### Lido

Address of the Lido contract.

- Accessor: `getLido() returns (address)`

### Members list

The list of oracle committee members.

- Mutators: `addOracleMember(address)`, `removeOracleMember(address)`
  - Permission required: `MANAGE_MEMBERS`
- Accessor: `getOracleMembers() returns (address[])`

### The quorum

The number of exactly the same reports needed to finalize the epoch.

- Mutator: `setQuorum(uint256)`
  - Permission required: `MANAGE_QUORUM`
- Accessor: `getQuorum() returns (uint256)`

When the `quorum` number of the same reports is collected for the current epoch,

- the epoch is finalized (no more reports are accepted for it),
- the final report is pushed to the Lido,
- statistics collected and the [sanity check][1] is evaluated,
- [beacon report receiver][2] is called.

### Sanity check

To make oracles less dangerous, we can limit rewards report by 0.1% increase in stake and 15%
decrease in stake, with both values configurable by the governance in case of extremely unusual
circumstances.

- Mutators: `setAllowedBeaconBalanceAnnualRelativeIncrease(uint256)` and
  `setAllowedBeaconBalanceRelativeDecrease(uint256)`
  - Permission required: `SET_REPORT_BOUNDARIES`
- Accessors: `getAllowedBeaconBalanceAnnualRelativeIncrease() returns (uint256)` and
  `getAllowedBeaconBalanceRelativeDecrease() returns (uint256)`

### Beacon report receiver

It is possible to register a contract to be notified of the report push to Lido (when the quorum is
reached). The contract should provide
[IBeaconReportReceiver](https://github.com/lidofinance/lido-dao/blob/develop/contracts/0.4.24/interfaces/IBeaconReportReceiver.sol)
interface.

- Mutator: `setBeaconReportReceiver(address)`
  - Permission required: `SET_BEACON_REPORT_RECEIVER`
- Accessor: `getBeaconReportReceiver() returns (address)`

Note that setting zero address disables this functionality.

The receiver's slot occupied with [`CompositePostRebaseBeaconReceiver`](/contracts/composite-post-rebase-beacon-receiver)
which supports adding multiple contracts to be notified about new reports.
Up to now, the only one [`SelfOwnedStETHBurner`](/contracts/self-owned-steth-burner) contract is set (or wrapped) with
beacon receiver through the composite adapter.

### Current reporting status

For transparency we provide accessors to return status of the oracle daemons reporting for the
current "[expected epoch][3]".

- Accessors:
  - `getCurrentOraclesReportStatus() returns (uint256)` - returns the current reporting bitmap,
    representing oracles who have already pushed their version of report during the [expected][3]
    epoch, every oracle bit corresponds to the index of the oracle in the current members list,
  - `getCurrentReportVariantsSize() returns (uint256)` - returns the current reporting variants
    array size,
  - `getCurrentReportVariant(uint256 _index) returns (uint64 beaconBalance, uint32 beaconValidators, uint16 count)` - returns the current reporting array element with the given
    index.

### Expected epoch

The oracle daemons may provide their reports only for the one epoch in every frame: the first
one. The following accessor can be used to look up the current epoch that this contract expects
reports.

- Accessor: `getExpectedEpochId() returns (uint256)`.

Note that any later epoch, that has already come _and_ is also the first epoch of its frame, is
also eligible for reporting. If some oracle daemon reports it, the contract discards any results of
this epoch and advances to the just reported one.

### Version of the contract

Returns the initialized version of this contract starting from 0.

- Accessor: `getVersion() returns (uint256)`.

### Beacon specification

Sets and queries configurable beacon chain specification.

- Mutator: `setBeaconSpec( uint64 _epochsPerFrame, uint64 _slotsPerEpoch, uint64 _secondsPerSlot, uint64 _genesisTime )`,
  - Permission required: `SET_BEACON_SPEC`,
- Accessor: `getBeaconSpec() returns (uint64 epochsPerFrame, uint64 slotsPerEpoch, uint64 secondsPerSlot, uint64 genesisTime)`.

### Current epoch

Returns the epoch calculated from current timestamp.

- Accessor: `getCurrentEpochId() returns (uint256)`.

### Supplemental epoch information

Returns currently reportable epoch (the first epoch of the current frame) as well as its start and
end times in seconds.

- Accessor: `getCurrentFrame() returns (uint256 frameEpochId, uint256 frameStartTime, uint256 frameEndTime)`.

### Last completed epoch

Return the last epoch that has been pushed to Lido.

- Accessor: `getLastCompletedEpochId() returns (uint256)`.

### Supplemental rewards information

Reports beacon balance and its change during the last frame.

- Accessor: `getLastCompletedReportDelta() returns (uint256 postTotalPooledEther, uint256 preTotalPooledEther, uint256 timeElapsed)`.

[1]: #sanity-check
[2]: #beacon-report-receiver
[3]: #expected-epoch
