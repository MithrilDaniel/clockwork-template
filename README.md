# a clockwork machine

your fees, on a clock, with receipts. this is the repository a machine runs in: one file you own,
`clockwork.json`, one workflow that runs the machine every fifteen minutes, and `data/`, where the machine
writes its ledger and its numbers. the only secret is your machine wallet's key, set as MACHINE_WALLET_KEY in
this repository's settings. clockwork never holds it.

## set it up
1. click "use this template" and create your copy, public or private.
2. in a terminal with node 20 or newer: `npx --yes clockwork-press@0.2.0 init <your token address>` writes
   your `clockwork.json` from the chain. fill in the machine wallet, the treasury wallet, the split, the pace.
   replace this repository's `clockwork.json` with it.
3. settings, secrets and variables, actions: add `MACHINE_WALLET_KEY` (a fresh wallet with a little eth, never
   a main wallet) and, if you run your own telegram bot, `TELEGRAM_BOT_TOKEN`.
4. actions, the "clockwork" workflow, run workflow, job = `dry`. read the log.
5. point your creator fees at the machine wallet (docs: fees to the machine). the machine claims on its own.
6. your status page: `https://gmerald.xyz/clockwork/m/?r=<owner>/<repo>` after the first tick.

docs: https://gmerald.xyz/clockwork/docs/ · with an assistant: `skills/clockwork/SKILL.md` · source:
https://github.com/MithrilDaniel/clockwork · license: source-available, the clockwork license.
