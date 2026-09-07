---
name: clockwork
description: Set up and run a ClockWorks machine for a pons v2 token on Robinhood Chain. Use when a founder asks to automate their creator fees, buy back and burn, build a treasury, or "run ClockWorks" for their token. Walks the human through the machine wallet, the config, the repository, the secrets, the dry run, pointing fees at the machine, and going live. Never touches a private key.
---

# ClockWorks, with an assistant next to you

ClockWorks runs a token's creator fees on a clock: claims from the pons fee escrow, splits by percentages the
founder sets once, buys the token back and burns it in slices, sends the treasury share to a wallet that only
grows, and prints every hash. It runs in the founder's own GitHub repository on a wallet the founder created.
Docs: https://gmerald.xyz/clockwork/docs/

## Rules you follow, without exception
- **Never ask for, read, print, store, or type a private key or a bot token.** The human pastes them on
  GitHub's secrets page or into a `gh secret set` prompt in their own terminal. If a key appears in the chat,
  tell the human it is burned and to create a fresh wallet.
- **Never send funds and never sign.** You prepare transactions; the human signs them in their own wallet.
- **Never create wallets or accounts for the human.** Tell them what to create and where.
- The treasury wallet must not be the machine wallet. The machine wallet must be fresh, funded with a little
  ETH for gas, and never a main wallet.

## What you need from the human before you start
1. The token address (a pons v2 launch on Robinhood Chain, chain id 4663).
2. A fresh machine wallet address, created on their device, with about 0.02 ETH on Robinhood Chain.
3. A treasury wallet address (a cold wallet is best).
4. The split they want. Default: 45 burn / 45 treasury / 0 ops / 10 ClockWorks. The ClockWorks share is at
   least 10 and is what pays for the software.
5. The pace: gentle (a claim over a day), steady (over six hours, the default), or once (one slice).
6. Telegram: their own bot token set as a secret (they create the bot in @BotFather) and the group's chat id from `clockwork tgcheck`, or `telegram.mode` set to `off` for now. The config defaults to `own`; with no token set, the machine logs and posts nothing.
7. The brand: `brand.name`, the words for treasury and burn (`brand.treasuryWord`, `brand.burnWord`, `brand.claimWord`), `brand.avatar` (an image URL the status page and the wallet card use), and `brand.art` (one image or short mp4 URL per Telegram card).
8. How often to claim: `claim.floor` is how much of the pairing asset must be waiting in the escrow before the machine claims (default 5), and `claim.atLeastEveryHours` claims whatever waits at least that often (default 24). A claim costs gas and posts a card; most machines keep the defaults.

## The steps
1. **Write the config.** The no-terminal way: open
   https://gmerald.xyz/clockwork/settings/?t=<token address>. The page reads the launch from the pons factory in
   the browser, fills every knob with the defaults, asks for the machine wallet and the treasury wallet, and
   (after step 2) writes `clockwork.json` into the new repository as a commit, given a fine-grained GitHub
   token for that one repository (Contents read and write). The terminal way, same file: with Node 20 or newer,
   in an empty folder, `npx --yes clockwork-press@0.2.4 init <token address>`.
   It reads the launch from the pons factory and writes `clockwork.json`. The page also offers three bundles,
   Patient, Steady and Aggressive, that set the claim rule, pace, dip mode, guardrails and cards together (the
   split stays the founder's); a new machine starts on Steady. Offer the human the three in one line and set the
   one they pick. Fill `wallets.machine`,
   `wallets.treasury`, the `split`, and `slices.rule` (`{ "kind": "pace", "pace": "steady" }`). If they want
   Telegram, set `telegram.mode` to `own` and `telegram.chatId` to their group id. Do not put a key or a token
   in this file; it is public by design.
2. **Create the repository.** Open https://github.com/MithrilDaniel/clockwork-template and click
   "Use this template", public or private, any name. Replace its `clockwork.json` with the one from step 1.
3. **Set the secrets.** Repository Settings, Secrets and variables, Actions, New repository secret:
   `MACHINE_WALLET_KEY` (the machine wallet's private key) and, for Telegram, `TELEGRAM_BOT_TOKEN`.
   Or from their terminal: `gh secret set MACHINE_WALLET_KEY -R <owner>/<repo>` (it prompts and hides the
   value). You never see these values.
4. **Dry run.** Actions tab, the "clockwork" workflow, "Run workflow", job = `dry`. Read the log with the
   human. Expect: the peg line, the holders line, `launch: pool` or `curve`, where fees go, what is unswept,
   the float, and `dry: would swap …`. The machine refuses to run if the key belongs to a different wallet
   than `wallets.machine`; that is the guard working, not a bug.
5. **Point the fees at the machine.** First claim what is already owed to the current recipient
   (`npx --yes clockwork-press@0.2.4 claimcheck` from the folder with `clockwork.json` prints it), because a
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
- `holding: …`: the launch is between phases (swept or rescued); the machine holds the slice.

## Commands
`init <token>` write the config from the chain · `dry` a tick without signing · `press` the tick ·
`claimcheck` what the escrow holds and what is unswept · `dividends` (0.2.3) reads every payout the listed memestock distributors made to holders into a public book, no key, any folder, `DIVIDENDS_DIR` names where · `doctor` RPC, wallet, gas, recipient, Telegram ·
`handback <address>` claim what is credited, then move the fee recipient · `holders` the holder count.

## What ClockWorks costs
Ten percent of every claim, sent on chain by the machine, slice by slice, to the ClockWorks wallet pinned in
the package (`0x6EA62Bd07FE08C7491543d495B42F6dA7ad298D0`). No setup fee. The chain is the invoice.
