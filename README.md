# Leave TrendEspresso v4.0 with your own wallet

This repository, https://github.com/TrendEspresso-LLP/v4-exit, lets a TrendEspresso v4.0 client take their money out
using nothing but their own wallet. It does not need the TrendEspresso website, its trading engine or anyone at
TrendEspresso: if all of them were gone tomorrow, these steps would still work, because your money is held by contracts
on the blockchain that answer to your wallet alone.

It holds:

- `TrendEspresso-exit-v4.0.html`: a single page that finds your money and sends your withdrawals from your wallet.
- `abi/`: the v4.0 contracts' ABIs (the description of every function they have), for block explorers and other tools.
- `LICENSE.txt`: the licence of everything here, Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International
  (CC BY-NC-SA 4.0), the licence of the v4.0 contracts themselves.
- `LICENSE-ethers.md`: the licence of ethers, the library the page carries inside it, which keeps its own MIT terms.

The v4.0 Controller, where everything starts, has the same address on both chains:

| Chain | Chain id | Controller | Block explorer |
| --- | --- | --- | --- |
| Avalanche C-Chain | 43114 | `0x6FEa59dA5c8C94336294059fEEf4bA8390c0fFEE` | Snowscan |
| Ethereum | 1 | `0x6FEa59dA5c8C94336294059fEEf4bA8390c0fFEE` | Etherscan |

## Using the page

1. Open the page in a browser that has your wallet extension (for example MetaMask or Rabby), either way:
   - from this repository's GitHub Pages site, which serves it once Pages is switched on, at
     https://trendespresso-llp.github.io/v4-exit/TrendEspresso-exit-v4.0.html; or
   - from your own computer: download `TrendEspresso-exit-v4.0.html` and open the file. It is one file: nothing to
     install, and it works the same if this repository or its Pages site is ever gone.
2. If your wallet does not appear when the file is opened from your computer, your browser is keeping extensions away
   from local files: in Chrome or Edge open the extensions page, choose your wallet's Details and turn on "Allow access
   to file URLs", then reload the page. Or open it from the Pages site, or put the file on any web host and open it
   from there.
3. Choose the chain your money is on and connect your wallet. The page refuses to read or send anything while your
   wallet is on another chain.
4. The page shows your Portfolio and each of its Accounts (one per strategy you joined), with what each holds.
5. Choose what to do. Before each transaction the page tells you in plain words what it will do; then your wallet asks
   you to confirm. You pay the network fee, in AVAX on Avalanche or ETH on Ethereum.

The page talks to your wallet and to nothing else. It makes no request to any server: its security policy forbids it,
and you can check that by reading its source.

## What each action does

- **Withdraw from an Account, and keep it.** Sends all of one token (USDC, the coin, or ŦE) from the Account to your
  wallet, including any USDC the Account has lent on Aave. The Account stays open, so you can come back to it. The coin
  arrives as the coin (BTC.b on Avalanche, WBTC or tBTC on Ethereum): it is not sold for USDC. If a trade is open, the
  profit fee on the part you take out is charged first, as on any withdrawal during a trade.
- **Emergency exit.** Sends everything an Account holds to your wallet in one transaction and **retires that Account for
  good**: it can never trade, take a deposit or be withdrawn from the normal way again. An open position arrives as the
  coin, not as USDC, and the profit fee on any gain is taken once. With a trade open the transaction needs more than
  3.5 million gas; the page sets the gas limit at twice your wallet's estimate, and unused gas is refunded.
- **A short.** An Account in a short has borrowed the coin on Aave. Until that debt is repaid, its USDC collateral comes
  out only as far as keeps its health factor at 1.20 or above; the rest stays. To free all of it, buy at least the
  amount owed of that coin, then use "Repay the borrowed coin and exit": your wallet asks twice, first to let the Account
  take the coin from your wallet, then for the emergency exit, which repays the debt, sends back whatever was not needed,
  and sends everything else to you.
- **Withdraw everything from the Portfolio.** Sends the USDC and ŦE held in the Portfolio itself, outside its Accounts,
  to your wallet. The Portfolio stays yours.
- **Claim Aave rewards.** Asks Aave to send any reward tokens an Account has earned straight to your wallet. It works on
  a retired Account too. The emergency exit does not claim them, so claim them afterwards.

## The same steps by hand, on a block explorer

If the page cannot help you, the block explorer can: Snowscan for Avalanche, Etherscan for Ethereum. Open a contract's
address there, use its "Read Contract" tab to look things up, and connect your wallet in its "Write Contract" tab to send.

1. On the Controller (address above), read `getPortfolioOf` with your wallet's address. That is your Portfolio.
2. On your Portfolio, read `getListOfAllAccounts`: one Account for each strategy you joined.
3. To take money out of an Account and keep it, write `directWithdraw` on the Account. Its one input is `0x`, then the
   token's address without its `0x`, then `ffffffffffffffffffff00`. Do it once for USDC and once for the coin: read
   `cashAddr` and `assetLongAddr` on the Account for their addresses.
4. Or, to empty an Account in one transaction and retire it for good, write `emergencyExit` on the Account with:
   - `account_`: the Account's own address (it confirms you mean this one);
   - `extraToken_`: `0x0000000000000000000000000000000000000000`;
   - `repayFromOwner_`: `0`;
   - `collateralToRelease_`: `115792089237316195423570985008687907853269984665640564039457584007913129639935` (the
     largest number, meaning "as much as is allowed").

   With a trade open, set the transaction's gas limit above 3,500,000.
5. A short: read `assetShortAddr` (the coin borrowed) and `getAssetShortDebt` (how much is owed) on the Account. Buy at
   least that much of the coin, write `approve` on that coin's own contract with the Account's address and the amount,
   then send the emergency exit again with that amount as `repayFromOwner_`.
6. On your Portfolio, write `withdrawAll` for the USDC and ŦE it holds itself.
7. Later, write `aaveClaimRewards` with `true` on each Account for any Aave rewards.

Only the wallet that owns the Portfolio can send any of these. Everything they move goes to that wallet.
