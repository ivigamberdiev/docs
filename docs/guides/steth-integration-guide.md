# stETH/wstETH integration guide

This document is intended for developers looking to integrate Lido's stETH or wstETH as an asset into their dApp, with a focus on money markets, DEXes and blockchain bridges.

## Lido

Lido is a family of liquid staking protocols across multiple blockchains, with headquarters on Ethereum.
Liquid refers to the ability for a user’s stake to become liquid. Upon user's deposit Lido issues stToken, which represents the deposited assets along with all the rewards & penalties accrued through deposit's staking. Unlike the staked funds, this stToken is liquid — it can be freely transferred between parties, making it usable across different DeFi applications while still receiving daily staked rewards. It is paramount to preserve this property when integrating stTokens into any DeFi protocol.

This guide refers to Lido on Ethereum (hereinafter referred to as Lido).
For ether staked in Lido, it gives users stETH that is equal to the amount staked.

Lido's stTokens are widely adopted across Ethereum ecosystem:
- The most important liquidity venues include [stETH/ETH liquidity pool on Curve](https://curve.fi/steth) and [wstETH/ETH MetaStable pool on Balancer v2](https://app.balancer.fi/#/pool/0x32296969ef14eb0c6d29669c550d4a0449130230000200000000000000000080).
- stETH is [listed as collateral asset on AAVE v2 market](https://app.aave.com/reserve-overview/?underlyingAsset=0xae7ab96520de3a18e5e111b5eaab095312d7fe84&marketName=proto_mainnet) on Ethereum mainnet.
- wstETH is [listed as collateral asset on Maker](https://daistats.com/#/collateral).
- steCRV (the Curve stETH/ETH LP token) is [listed as collateral asset on Maker](https://daistats.com/#/collateral).
- there are multiple liquidity strategies built on top of Lido's stTokens, including [yearn](https://yearn.finance/#/vault/0xdCD90C7f6324cfa40d7169ef80b12031770B4325), [Harvest Finance](https://harvest.finance/), and [Babylon Finance](https://www.babylon.finance/garden/0xB5bD20248cfe9480487CC0de0d72D0e19eE0AcB6).

#### Integration utilities: ChainLink price feeds
- There are live ChainLink [stETH/USD](https://app.ens.domains/name/steth-usd.data.eth) and [stETH/ETH](https://etherscan.io/address/0x86392dC19c0b719886221c78AB11eb8Cf5c52812) price feeds on Ethereum.
- There are live ChainLink stETH/USD price feeds on [Arbitrum](https://docs.chain.link/docs/arbitrum-price-feeds/) and [Optimism](https://docs.chain.link/docs/optimism-price-feeds/). These also have ChainLink wstETH-stETH exchange rate data feeds.

## stETH vs. wstETH

There are two versions of Lido's stTokens, namely stETH and wstETH.
Both tokens are ERC-20 tokens, but they reflect the accrued staking rewards in different ways. stETH implements rebasing mechanics which means the stETH balance increases periodically. In contrary, wstETH balance is constant, while the token increases in value eventually (denominated in stETH). At any moment, any amount of stETH can be converted to wstETH via trustless wrapper and vice versa, thus tokens effectively share liquidity.
For instance, undercollateralized wstETH positions on Maker can be liquidated by unwrapping wstETH and swapping it for ether on Curve.

## stETH

### What is stETH

stETH is a ERC20 token that represents ether staked with Lido. Unlike staked ether, it is liquid and can be transferred, traded, or used in DeFi applications. Total supply of stETH reflects amount of ether deposited into protocol combined with staking rewards, minus potential validator penalties. stETH tokens are minted upon ether deposit at 1:1 ratio. When withdrawals from the Beacon chain will be introduced, it will also be possible to redeem ether by burning stETH at the same 1:1 ratio.

Please note, Lido has implemented staking rate limits aimed at reducing the post-Merge staking surge's impact on the staking queue & Lido’s socialized rewards distribution model. Read more about it [here](#staking-rate-limits).

stETH is a rebasable ERC-20 token. Normally, the stETH token balances get recalculated daily when the Lido oracle reports Beacon chain ether balance update. The stETH balance update happens automatically on all the addresses holding stETH at the moment of rebase. The rebase mechanics have been implemented via shares (see [shares](#steth-internals-share-mechanics)).

### The Beacon chain oracle

Normally, stETH rebases happen daily when the Lido oracle reports the Beacon chain ether balance update. The rebase can be positive or negative, depending on the validators' performance. In case Lido's validators get slashed, the stETH balances can decrease according to penalty sizes. However, daily rebases have never been negative by the time of writing.
The Beacon chain oracle has sanity checks on both max APR reported (the APR cannot exceed 10%, which means a daily rebase is limited to `10/365%`) and total staked amount drop (staked ether decrease reported cannot exceed 5%). Currently, the Beacon chain oracle report is based on five oracle daemons hosted by established node operators selected by the DAO. As soon as three out of five oracle daemons report matching Beacon chain data, the oracle reports it to the Lido smart contract, and the rebase occurs. There is a [dedicated oracle dashboard](https://mainnet.lido.fi/#/lido-dao/0x442af784a788a5bd6f42a01ebe9f287a871243fb/) to monitor current Beacon chain reports.

#### Oracle corner cases

- In case oracle daemons do not report Beacon chain balance update or do not reach quorum, the oracle does not submit the daily report, and the daily rebase doesn't occur until the quorum is reached.
- In case the quorum hasn't been reached, the oracle can skip the daily report. The report will happen as soon as the quorum for one of the next periods will be reached, and it will include the balance update for all the period since last oracle report.
- Oracle daemons only report the finalized epochs. In case of no finality on the Beacon chain, the daemons won't submit their reports, and the daily rebase won't occur.
- In case sanity checks on max APR or total staked amount drop fail, the oracle report cannot be finalized, and the rebase cannot happen.
- StETH smart contract includes [a method that allows burning stETH shares](https://github.com/lidofinance/lido-dao/blob/816bf1d0995ba5cfdfc264de4acda34a7fe93eba/contracts/0.4.24/StETH.sol#L391). The method is meant for [in-protocol cover](https://research.lido.fi/t/lip-6-in-protocol-coverage-proposal/1468). When stETH shares get burnt, it triggers an immediate rebase, while the underlying ether adds up to the daily rewards for all stETH holders. This extra rebase doesn't interfere with normal rebase schedule in any way.

### stETH internals: share mechanics

Daily rebases result in stETH token balances changing. This mechanism is implemented via shares.
The `share` is a basic unit representing the stETH holder's share in the total amount of ether controlled by the protocol. When a new deposit happens, the new shares get minted to reflect what share of the protocol controlled ether has been added to the pool. When the Beacon chain oracle report comes in, the price of 1 share in stETH is being recalculated. Shares aren't normalized, so the contract also stores the sum of all shares to calculate each account's token balance.
Shares balance by stETH balance can be calculated by this formula:
```
shares[account] = balanceOf(account) * totalShares / totalPooledEther
```

### 1 wei corner case

stETH balance calculation includes integer division, and there is a common case when the whole stETH balance can't be transferred from the account, while leaving the last 1 wei on the sender's account. Same thing can actually happen at any transfer or deposit transaction.

Example:
1. User A transfers 1 stETH to User B.
2. Under the hood, stETH balance gets converted to shares, integer division happens and rounding down applies.
3. Corresponding amount of shares gets transferred from User A to User B.
4. Shares balance gets converted to stETH balance for User B.
5. In many cases the actually transferred amount is 1 wei less than expected.

### Bookkeeping shares

Although user friendly, stETH rebases add a whole level of complexity to integrating stETH into other dApps and protocols. When integrating stETH as an assets into any dApp, it's highly recommended to store and operate shares rather than stETH public balances directly, because stETH balances change both upon transfers, mints/burns, and rebases, while shares balances can only change upon transfers and mints/burns.

To figure out the shares balance, `getSharesByPooledEth(uint256)` function can be used. It returns the value not affected by future rebases and it can be converted back into stETH by calling `getPooledEthByShares` function.

> See all available stETH methods [here](https://github.com/lidofinance/docs/blob/main/docs/contracts/lido.md#view-methods).

Any operation on stETH can be performed on shares directly, with no difference between share and stETH.

The preferred way of operating stETH should be:
1) get stETH token balance;
2) convert stETH balance into shares balance and use it as primary balance unit in your dapp;
3) when any operation on the balance should be done, do it on shares balance;
4) when users interact with stETH, convert shares balance back to stETH token balance.

Please note that 10% APR on shares balance and 10% APR on stETH token balance will ultimately result in different output values over time, because shares balance is stable, while stETH token balance changes eventually.

If using the rebasable stETH token is not an option for your integration, it is recommended to use wstETH instead of stETH. See how it works [here](#wsteth).

### Fees

Lido collects a percentage of the staking rewards as a protocol fee. The exact fee size is defined by the DAO and can be changed in the future via DAO voting. To collect the fee, the protocol mints new stETH token shares and assigns them to the fee recipients. Currenty, the fee collected by Lido protocol is 10% of staking rewards with half of it going to the node operators and the other half going to the protocol treasury.

Since total amount of Lido pooled ether tends to increase, the combined value of all holders' shares denominated in stETH increases respectively. Thus, the rewards effectively spread between each token holder proportionally to their share in the protocol TVL. So Lido mints new shares to the fee recipient, so that the total cost of the newly-minted shares exactly corresponds to the fee taken (calculated in basis points):

```
shares2mint * newShareCost = (_totalRewards * feeBasis) / 10000
newShareCost = newTotalPooledEther / (prevTotalShares + shares2mint)
```
which follows to:
```
                        _totalRewards * feeBasis * prevTotalShares
shares2mint = --------------------------------------------------------------
                (newTotalPooledEther * 10000) - (feeBasis * _totalRewards)
```

### Do stETH rewards compound?

stETH holders receive staking rewards, but those don't compound. Actual APR diminishes slightly over time for two main reasons:

1. Beacon chain rewards don't compound. To get more rewards one should withdraw funds and re-deposit them, skimming rewards. Until withdrawals are enabled, that isn't technically possible. So, while the Lido's beacon chain balance increases over time, rewards bearing asset amount remains equal to deposited ether amount.
2. stETH holders receive rewards proportionally to their share in the stETH total supply. This share diminishes slightly over time because of the protocol fee on rewards eating up from other holders' shares eventually.

## wstETH

Due to the rebasing nature of stETH, the stETH balance on holder's address is not constant, it changes daily as oracle report comes in.
Although rebasable tokens are becoming a common thing in DeFi recently, many dApps do not support rebasing. For example, Maker, UniSwap, and SushiSwap are not designed for rebasable tokens. Listing stETH on these apps can result in holders not receiving their daily staking rewards which effectively defeats the benefits of liquid staking. To integrate with such dApps, there's another form of Lido stTokens called wstETH (wrapped staked ether).

### What is wstETH

wstETH is an ERC20 token that represents the account's share of the stETH total supply (stETH token wrapper with static balances). For wstETH, 1 wei in [shares](#steth-internals-share-mechanics) equals to 1 wei in balance. The wstETH balance can only be changed upon transfers, minting, and burning. wstETH balance does not rebase, wstETH's price denominated in stETH changes instead.
At any given time, anyone holding wstETH can convert any amount of it to stETH at a fixed rate, and vice versa. The rate is the same for everyone at any given moment. Normally, the rate gets updated once a day, when stETH undergoes a rebase. The current rate can be obtained by calling `wstETH.stEthPerToken()`

#### wstETH shortcut

Note, that WstETH contract includes a shortcut to convert ether to wstETH under the hood, which allows to effectively skip the wrapping step and stake ether for wstETH directly. Keep in mind that when using the shortcut, [the staking rate limits](#staking-rate-limits) still apply.

### Rewards accounting

Since wstETH represents the holder's share in the total amount of Lido-controlled ether, rebases don't affect wstETH balances, but change the wstETH price denominated in stETH.

**Basic example**:

1) User wraps 1 stETH and gets 0.9803 wstETH (1 stETH = 0.9803 wstETH)
2) A rebase happens, the wstETH price goes up by 5%
3) User unwraps 0.9803 wstETH and gets 1.0499 stETH (1 stETH = 0.9337 wstETH)


### Wrap & Unwrap

When wrapping stETH to wstETH, the desired amount of stETH is being locked on the WstETH contract balance, and the wstETH is being minted according to the [shares bookeeping](#bookkeeping-shares) formula.

When unwrapping, wstETH gets burnt and the corresponding amount of stETH gets unlocked.

Thus, amount of stETH unlocked when unwrapping is different from what has been initially wrapped (given a rebase happened between wrapping and unwrapping stETH).

### Goerli wstETH for testing

The most recent testnet version of the Lido protocol lives on Goerli testnet ([see the full list of contracts deployed here](https://docs.lido.fi/deployed-contracts/goerli)). Just like on mainnet, Goerli wstETH for testing purposes can be obtained by approving the desired amount of stETH to the WstETH contract on Goerli, and then calling `wrap` method on it. The corresponding amount of Goerli stETH will be locked on the WstETH contract, and the wstETH tokens will be minted to your account. Goerli Ether can also be converted to wstETH directly using the [wstETH shortcut](#wsteth-shortcut) – just send your Goerli Ether to WstETH contract on Goerli, and the corresponding amount of wstETH will be minted to your account.

### wstETH on L2s

Currently, wstETH token is present on Arbitrum and Optimism with bridging implemented via the canonical bridges. Unlike on Ethereum mainnet, wstETH on L2s is a plain ERC20 token and cannot be unwrapped to unlock stETH on the corresponding L2 network. The token does not implement shares bookkeeping, which means it is not possible to calculate the wstETH/stETH rate and the rewards accrued on-chain. However, there're live Chainlink wstETH/stETH rate feeds for [Arbitrum](https://data.chain.link/arbitrum/mainnet/crypto-eth/wsteth-steth%20exchangerate) and [Optimism](https://data.chain.link/optimism/mainnet/crypto-eth/wsteth-steth%20exchangerate) that can and should be used for this purpose. 

## ERC20Permit

wstETH token implements the ERC20 Permit extension allowing approvals to be made via signatures, as defined in [EIP-2612](https://eips.ethereum.org/EIPS/eip-2612).

The `permit` method allows users to modify the allowance using a signed message, instead of through `msg.sender`.
By not relying on `approve` method, you can build interfaces that will approve and use wstETH in one tx.

NB: stETH token itself does not support the ERC20 Permit extension and it's not possible to approve and wrap stETH in one tx at this time.

## Staking rate limits
In order to handle the staking surge in case of some unforeseen market conditions, the Lido protocol implemented staking rate limits aimed at reducing the surge's impact on the staking queue & Lido’s socialized rewards distribution model.
There is a sliding window limit that is parametrized with `_maxStakingLimit` and `_stakeLimitIncreasePerBlock`. This means it is only possible to submit this much ether to the Lido staking contracts within a 24 hours timeframe. Currently, the daily staking limit is set at 150,000 ether.

You can picture this as a health globe from Diablo 2 with a maximum of `_maxStakingLimit` and regenerating with a constant speed  per block.
When you deposit ether to the protocol, the level of health is reduced by its amount and the current limit becomes smaller and smaller.
When it hits the ground, transaction gets reverted.

To avoid that, you should check if `getCurrentStakeLimit() >= amountToStake`, and if it's not you can go with an alternative route.
The staking rate limits are denominated in ether, thus, it makes no difference if the stake is being deposited for stETH or using [the wstETH shortcut](#wsteth-shortcut), the limits apply in both cases.

#### Alternative routes:

1. Wait for staking limits to regenerate to higher values and retry depositing ether to Lido later.
2. Consider swapping ETH for stETH on DEXes like Curve or Balancer. At specific market conditions stETH may effectively be purchased from there with a discount due to stETH price fluctuations.

## Transfer shares function for stETH

The [LIP-11](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-11.md) introduced the `transferShares` function which allows to transfer stETH in a "rebase-agnostic" manner: transfer in terms of [shares](#steth-internals-share-mechanics) amount.

Normally, we transfer stETH using ERC-20 `transfer` and `transferFrom` functions which accept as input amount of stETH, not the amount of the underlying shares.
Sometimes we'd better operate with shares directly to avoid possible rounding issues. Rounding issues usually could appear after a token rebase.
This feature is aimed to provide an additional level of precision when operating with stETH.
Read more abut the function in the [LIP-11](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-11.md).

## General integration examples

### stETH/wstETH as collateral

stETH/wstETH as DeFi collateral is beneficial for a number of reasons:

- stETH/wstETH is almost as safe as ether, price-wise: barring catastrophic scenarios, its value tends to hold the ETH peg well;
- stETH/wstETH is a productive asset: earning rewards on collateral effectively lowers the cost of borrowing;
- stETH/wstETH is a very liquid asset with billions of liquidity locked in liquidity pools ([Curve](https://curve.fi/steth), [Balancer v2](https://app.balancer.fi/#/pool/0x32296969ef14eb0c6d29669c550d4a0449130230000200000000000000000080))

Lido's staked assets have been listed on major liquidity protocols:

- On Maker, [wstETH collateral (scroll down to Dai from WSTETH-A section)](https://daistats.com/#/collateral) can be used to mint DAI stablecoin. See [Lido's blog post](https://blog.lido.fi/makerdao-integrates-lidos-staked-eth-steth-as-collateral-asset/) for more details.
- On AAVE, multiple assets can be [borrowed against stETH](https://app.aave.com/reserve-overview/?underlyingAsset=0xae7ab96520de3a18e5e111b5eaab095312d7fe84&marketName=proto_mainnet). See [Lido's blog post](https://blog.lido.fi/aave-integrates-lidos-steth-as-collateral/) for more details. Please note: stETH is only supported on AAVE as lending collateral. Borrowing stETH on AAVE is not currently supported. However, any asset can be borrowed on AAVe via a flashloan. Due to a known [1 wei corner case](#1-wei-corner-case) there's a certain situation when a flashloan transaction can revert. Please visit [stETH on AAVE caveats](https://docs.lido.fi/token-guides/steth-on-aave-caveats) article for more details.

Robust price sources are required for listing on most money markets, with ChainLink price feeds being the industry standard. There're live ChainLink [stETH/USD](https://app.ens.domains/name/steth-usd.data.eth) and [stETH/ETH](https://etherscan.io/address/0x86392dC19c0b719886221c78AB11eb8Cf5c52812) price feeds on Ethereum.

### Wallet integrations

Lido's Ethereum staking services have been successfully integrated into most popular DeFi wallets, including Ledger, MyEtherWallet, ImToken and others.
Having stETH integrated can provide wallet users with great user experience of direct staking from the wallet UI itself.

Lido DAO runs a referral program rewarding wallets and other apps for driving liquidity to the Lido staking protocol. At the moment, the referral program is in [whitelist mode](https://research.lido.fi/t/switch-referral-program-to-whitelist-mode/1014). Please contact Lido bizdev team to find out if your wallet might be eligible for referral program participation.

When adding stETH support to a DeFi wallet, it is important to preserve stETH's rebasing nature. Avoid storing cached stETH balance for extended periods of time (over 24 hours), and keep in mind it doesn't necessarily take a transaction to change stETH balance.

### Liquidity mining

stETH liquidity is mostly concentrated in two biggest liquidity pools:
- [stETH/ETH liquidity pool on Curve](https://curve.fi/steth) ([contract code](https://github.com/curvefi/curve-contract/blob/master/contracts/pools/steth/StableSwapSTETH.vy))
- [wstETH/WETH MetaStable pool on Balancer v2](https://app.balancer.fi/#/pool/0x32296969ef14eb0c6d29669c550d4a0449130230000200000000000000000080) ([read more](https://docs.balancer.fi/products/balancer-pools/metastable-pools#the-lido-wsteth-weth-liquidity-pool))

Both pools are incentivised with Lido governance token (LDO) via direct incentives and bribes (veBAL bribes coming soon), and allow the liquidity providers to retain their exposure to earning Lido staking rewards.

- Curve pool allows providing liquidity in the form of any of the pooled assets or in both of them. From that moment on, all the staking rewards accrued by stETH go to the pool and not to the liquidity provider's address. However, when withdrawing the liquidity, the liquidity provider will be able to get more than they have initially deposited.
Please note, when depositing exclusively stETH to Curve, the tokens are split between ether and stETH, with the precise balances fluctuating constantly due to price trading. Thus, the liquidity provider will only be eligible for about a half of rewards accrued by the stETH deposited. To avoid that, provide stETH and ether liquidity in equal parts.
- Unlike Curve, Balancer pool is wstETH-based. wstETH doesn't rebase, it accrues staking rewards by eventually increasing in price instead. Thus, when withdrawing liquidity form the Balancer pool, the liquidity providers get assets valued higher than what they have initially deposited.

### Cross chain bridging

The Lido's stTokens will eventually get bridged to various L2's and sidechains.
Most cross chain token bridges have no mechanics to handle rebases. This means bridging stETH to other chains will prevent stakers from collecting their staking rewards. In the most common case, the rewards will naturally go to the bridge smart contract and never make it to the stakers.
While working on full-blown bridging solutions, the Lido contributors encourage the users to only bridge the non-rebasable representation of staked ether, namely wstETH.

## Risks

1. Smart contract security.
There is an inherent risk that Lido could contain a smart contract vulnerability or bug. The Lido code is open-sourced, audited and covered by an extensive bug bounty program to minimise this risk.
To mitigate smart contract risks, all of the core Lido contracts are audited. Audit reports can be found [here](https://github.com/lidofinance/audits#lido-protocol-audits).
Besides, Lido is covered with a massive [Immunefi bugbounty program](https://immunefi.com/bounty/lido/).
2. Beacon chain - Technical risk.
Lido is built atop experimental technology under active development, and there is no guarantee that Beacon chain has been developed error-free. Any vulnerabilities inherent to Beacon chain brings with it slashing risk, as well as stETH balance fluctuation risk.
3. Beacon chain - Adoption risk.
The value of stETH is built around the staking rewards associated with the Ethereum beacon chain. If Beacon chain fails to reach required levels of adoption we could experience significant fluctuations in the value of ETH and stETH.
4. Slashing risk.
Beacon chain validators risk staking penalties, with up to 100% of staked funds at risk if validators fail. To minimise this risk, Lido stakes across multiple professional and reputable node operators with heterogeneous setups, with additional mitigation in the form of self-coverage.
5. stETH price risk.
Users risk an exchange price of stETH which is lower than inherent value due to withdrawal restrictions on Lido, making arbitrage and risk-free market-making impossible. The Lido DAO is driven to mitigate above risks and eliminate them entirely to the extent possible. Despite this, they may still exist and, as such, it is our duty to communicate them.
6. DAO key management risk.
On early stages of Lido, slightly more than 600k ether became held across multiple accounts backed by a multi-signature threshold scheme to minimize custody risk. If signatories across a certain threshold lose their key shares, get hacked or go rogue, Lido risks these funds (<13% of total stake as of October 2022) becoming locked.
