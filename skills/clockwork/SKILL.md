---
name: clockwork
description: Set up and run a ClockWorks machine for a pons v2 token on Robinhood Chain. Use when a founder asks to automate their creator fees, buy back and burn, build a treasury, or "run ClockWorks" for their token. Walks the human through the machine wallet, the config, the repository, the secrets, the dry run, pointing fees at the machine, and going live. Never touches a private key.
---

# ClockWorks, with an assistant next to you

ClockWorks runs a token's creator fees on a clock: claims from the pons fee escrow, splits by percentages the
founder sets once, buys the token back and burns it in slices, sends the treasury share to a wallet the software
never sends from, can pay a board of holders, and prints every hash. It runs in the founder's own GitHub repository
on a wallet the founder created.
Docs: https://gmerald.xyz/clockwork/docs/

## Rules you follow, without exception
- **Never ask for, read, print, store, or type a private key or a bot token.** The human pastes them on
  GitHub's secrets page or into a `gh secret set` prompt in their own terminal. If a key appears in the chat,
  tell the human it is burned and to create a fresh wallet.
- **Never send funds and never sign.** You prepare transactions; the human signs them in their own wallet.
- **Never create wallets or accounts for the human.** Tell them what to create and where.
- The treasury wallet must not be the machine wallet. The machine wallet must be fresh, funded with a little
  ETH for gas, and never a main wallet.
- **Never choose a split for the human.** Show the three presets for their pair side by side and let them pick one or
  write their own. The board is never the default.
- Past tense for what the machine did; no promises about what holders will receive.

## What you need from the human before you start
1. The token address (a pons v2 launch on Robinhood Chain, chain id 4663), paired with USDG, WETH or native ETH.
   clockwork-press 0.4.3 serves no other pair: a file with a stock-token pair or any other pair does not load.
2. A fresh machine wallet address, created on their device, with about 0.02 ETH on Robinhood Chain.
3. A treasury wallet address (a cold wallet is best).
4. The split, as whole basis points in `split` that add up to 10000 (22.5% is 2250). Three presets for each kind of
   pair, none selected until the human picks. On a USDG or WETH pair:
   - burn and treasury: `burnBps` 4500, `treasuryBps` 4500, `boardBps` 0, `opsBps` 0, `serviceBps` 1000;
   - board: `burnBps` 2250, `treasuryBps` 2250, `boardBps` 4500, `opsBps` 0, `serviceBps` 1000;
   - keep half: `burnBps` 2250, `treasuryBps` 2250, `boardBps` 0, `opsBps` 4500, `serviceBps` 1000, with
     `wallets.ops` set to the creator's own wallet.
   On a native ETH pair there is no burn leg until 0.5.0, so `burnBps` is 0 and a file with a burn share does not load:
   - treasury: `burnBps` 0, `treasuryBps` 9000, `boardBps` 0, `opsBps` 0, `serviceBps` 1000;
   - board: `burnBps` 0, `treasuryBps` 4500, `boardBps` 4500, `opsBps` 0, `serviceBps` 1000;
   - keep half: `burnBps` 0, `treasuryBps` 4500, `boardBps` 0, `opsBps` 4500, `serviceBps` 1000, with
     `wallets.ops` set to the creator's own wallet.
   Or their own numbers, with `burnBps` 0 on a native pair. `serviceBps` is at least 1000 and pays for the software
   (see the last section).
5. The pace: gentle (a claim over a day), steady (over six hours, the default), or once (one slice).
6. Telegram: their own bot token set as a secret (they create the bot in @BotFather) and the group's chat id from `clockwork tgcheck`, or `telegram.mode` set to `off` for now. The config defaults to `own`; with no token set, the machine logs and posts nothing.
7. The brand: `brand.name`, the words for treasury and burn (`brand.treasuryWord`, `brand.burnWord`, `brand.claimWord`), `brand.avatar` (an image URL the status page and the wallet card use), and `brand.art` (one image or short mp4 URL per Telegram card).
8. How often to claim: `claim.floor` is how much of the pairing asset must be waiting in the escrow before the machine claims (default 5), and `claim.atLeastEveryHours` claims whatever waits at least that often (default 24). A claim costs gas and posts a card; most machines keep the defaults.
9. The board, only if they chose it: `split.boardBps` holds that share of every slice in the machine wallet and pays it at the first tick after 00:00 UTC, in the pairing asset, by balance, to every seat (a wallet holding at least `site.seatThreshold` tokens, default 250,000). `board.rule` says who a seat is:
   - `"at-drop"`, the default when the key is absent: the wallets holding a seat in the holder scan taken at the drop. A seat bought at 23:50 is in that night's drop. On a token with a creator tax this is usually enough.
   - `"lowest-since-last-drop"`: a seat counts its lowest balance in the hourly holder scans since the last drop, and a seat bought after the first of those scans waits one more drop. Suits tokens with no creator tax.
   Wallets that hold a seat's worth but should not be paid (the founder's own, a partner's) go in `board.exclude`. No `boardBps` means no board.
10. Keep half: the ops share goes to `wallets.ops`. With no `pair.usdFeed` in the file, as on every USDG, WETH and native ETH pair, it goes out on every slice. With a feed (in 0.4.3 only the house's GME pair has one) it waits in the machine wallet while the peg check does not pass, and goes out with the first slice whose check does. What is held still goes to `wallets.ops` if the human later drops the ops share; if they remove `wallets.ops` as well, it goes back to the float and is split with the rest.

## The steps
1. **Write the config.** The no-terminal way: open
   https://gmerald.xyz/clockwork/settings/?t=<token address>. The page reads the launch from the pons factory in
   the browser, fills every knob with the defaults, asks for the machine wallet and the treasury wallet, and
   (after step 2) writes `clockwork.json` into the new repository as a commit, given a fine-grained GitHub
   token for that one repository (Contents read and write). The terminal way, same file: with Node 20 or newer,
   in an empty folder, `npx --yes clockwork-press@0.4.3 init <token address>`.
   It reads the launch from the pons factory and writes `clockwork.json` with every leg of the split at 0, so the
   file does not load until the human picks a split. The page also offers three bundles,
   Patient, Steady and Aggressive, that set the claim rule, pace, dip mode, guardrails and cards together (the
   split stays the founder's); a new machine starts on Steady. Offer the human the three in one line and set the
   one they pick. Fill `wallets.machine`,
   `wallets.treasury`, the `split` they picked, and `slices.rule` (`{ "kind": "pace", "pace": "steady" }`). If they want
   Telegram, set `telegram.mode` to `own` and `telegram.chatId` to their group id. Do not put a key or a token
   in this file; it is public by design.
2. **Create the repository.** Open https://github.com/MithrilDaniel/clockwork-template and click
   "Use this template", public or private, any name. Replace its `clockwork.json` with the one from step 1.
3. **Set the secrets.** Repository Settings, Secrets and variables, Actions, New repository secret:
   `MACHINE_WALLET_KEY` (the machine wallet's private key) and, for Telegram, `TELEGRAM_BOT_TOKEN`.
   Or from their terminal: `gh secret set MACHINE_WALLET_KEY -R <owner>/<repo>` (it prompts and hides the
   value). You never see these values.
4. **Dry run.** Actions tab, the "clockwork" workflow, "Run workflow", job = `dry`. Read the log with the
   human. Expect: the peg line, the holders line, `launch: pool` or `curve`, where fees go, what is unswept, the
   float, and `dry: would swap …` in the pool (`dry: would buy …` on the curve). A native ETH pair has no burn leg, so
   its dry run shows `dry: would stash …` and the other legs, and no swap. The machine refuses to run if the key belongs
   to a different wallet than `wallets.machine`; that is the guard working, not a bug.
5. **Point the fees at the machine.** First claim what is already owed to the current recipient
   (`npx --yes clockwork-press@0.4.3 claimcheck` from the folder with `clockwork.json` prints it), because a
   recipient change does not move credited balances. Then the current recipient signs
   `transferCreatorFeeRecipient(token, machineWallet)` on the pons factory
   `0x7eD598BcEf8bd9Edd8C97A195C6d13f40801EC7e`. Prepare the calldata for them (function selector
   `0x2931861b`, then the token address and the machine address each left-padded to 32 bytes) and tell them
   to send it as a raw transaction from that wallet with value 0, or point them at
   https://gmerald.xyz/clockwork/point/?token=<token>&machine=<machine>, which reads the chain, shows what is
   owed, and prepares the same transaction for their own wallet. The pons UI does not show this function;
   the contract has it and it takes effect immediately.
6. **Go live.** Set `claim.mode` to `auto` in `clockwork.json` if it is not already, commit, and let the
   workflow's schedule run it on the quarter hour. Optional: a cron-job.org job that POSTs to
   `https://api.github.com/repos/<owner>/<repo>/actions/workflows/press.yml/dispatches` with body
   `{"ref":"main"}` and a fine-grained GitHub token scoped to that one repository, for punctual ticks.
7. **The status page** is live at `https://gmerald.xyz/clockwork/m/?r=<owner>/<repo>` after the first tick
   commits. Every number on it is read from the repository the machine writes to.

## Reading a run
- `ran N min ago, the next tick is not due; leaving`: the double-tick guard. Nothing wrong.
- `X gme of fees unswept, and they are not ours to sweep`: fees sit on the pons hook until pons sweeps
  them into the escrow. The machine claims what the escrow holds. Ask pons how often the operator sweeps.
- `fees no longer point at the machine`: the recipient changed. If the founder did it, fine. If not, they
  open pons and check, and they can run `clockwork handback <address>` from a machine that is still the
  recipient to move fees to a wallet of their choice.
- `the key is for 0x…, but clockwork.json names 0x…`: the wrong key was pasted. Set the secret again.
- `holding: the curve is swept and the pool is not open yet` (or `the launch was rescued`): the launch is between
  phases; the machine holds the slice.
- `holding: the burn leg can't run (<reason>), so nothing is split`: the split has a burn share and the burn leg cannot
  run this tick, so the whole slice waits in the machine wallet and no leg is sent. Claims still run. `doctor` prints
  the same line when every slice would hold.
- `clockwork.json: native ETH pairs have no burn leg until 0.5.0: set burnBps to 0 and give that share to the treasury, the board or ops`:
  the file gives a native ETH pair a burn share. Use a native preset above.
- `clockwork.json: clockwork-press 0.4.3 serves pons tokens paired with USDG, WETH or native ETH; stock-token and other pairs come in a later release`:
  the token is paired with a stock token or another asset this release does not serve, so no job loads the file.
- `holding: the gme token is paused by its issuer`: the issuer of the stock token paused it, or paused every stock
  token. No claim, no drop and no slice until the pause lifts; one card when it starts.
- `holding: the treasury wallet is blocked by the gme issuer` (or the machine, service or ops wallet): that wallet is
  on the issuer's blocklist, so the machine holds. The founder names another wallet in `clockwork.json`.
- `holding: the stock guard could not read the chain`: the guard's reads failed this tick, so nothing that moves the
  stock token ran; the next tick reads again.
- `[board] 0x… is blocked by the gme issuer: skipped`: that seat was not paid. The drop's `skipped` list in
  `data/board-ledger.json` names it, and its share stayed held for the next drop.
- `[board] no complete holder scan since the last drop; the drop waits for one`: under `lowest-since-last-drop`,
  no holder scan has finished since the last drop; every tick retries with a fresh scan.
- `[step] N claim(s) counted`: this UTC day's claims toward the step, and the credit waiting for later slices.

## Commands
`init <token>` write the config from the chain · `dry` a tick without signing · `press` the tick ·
`claimcheck` what the escrow holds and what is unswept · `doctor` RPC, wallet, gas, recipient, the price, the board
rule, the stock guard, Telegram · `handback <address>` claim what is credited, then move the fee recipient ·
`holders` the holder count · `boardcheck` the drop the board would pay now, seat by seat, without signing ·
`board` pay what is held now, even if today's drop already ran · `payouts` reads every payout the listed memestock
distributors made to holders into a public book, no key, any folder; `PAYOUTS_DIR` names where.

## Stock pairs
clockwork-press 0.4.3 runs a stock-token pair only for the house, GMERALD, whose machine runs on a GME pair: on a stock
pair the service leg would be sent in the stock token, so any other file with one does not load. Every Robinhood stock
token shares one issuer pause and one blocklist. On the house's pair the machine reads both every tick, with nothing to
configure: it holds while the token is paused or one of its own wallets is blocked, never pays a blocked seat, and
records share counts with the token's multiplier beside the raw amounts (`amountUI`, `paidUI`, `pairMultiplier`).
USDG, WETH and native ETH pairs pass straight through.

## What the machine approves
From its first 0.4.2 burn, no allowance is unlimited; a machine upgraded from an earlier version keeps its old
allowances until that burn. Permit2 may pull up to twice the larger of two amounts: the burn share of what the machine
holds, and the last 24 hours of burns. The router gets exactly one swap's amount, for about ten minutes, signed inside
the swap itself. If the router refuses that signature, the slice sends one exact Permit2 approve first; `doctor` shows
how often that happened, and warns when the machine wallet has code. A native ETH pair has no burn leg, so its machine
approves nothing for one.

## What ClockWorks costs
10% of the creator fees the machine claims, and 5% on the part of a UTC day above a step: 2,000 USDG a day on a USDG
pair, 0.8 ETH on a native ETH pair, 0.8 WETH on a WETH pair, the three pairs clockwork-press 0.4.3 serves. The step counts only what the pons escrow paid the machine itself, read from the escrow's own logs; fees
claimed by hand or sent in are charged 10%. The machine sends it on chain, slice by slice, to the ClockWorks wallet
pinned in the package (`0x6EA62Bd07FE08C7491543d495B42F6dA7ad298D0`). No setup fee. The chain is the invoice.
