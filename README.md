# Tombola

A picker for when you actually want it to be fair.

## What it does

Two modes, one engine.

**General** - pick from any list of options.
- *Equal* - true random, every option has the same chance.
- *Manual* - set custom weights per option.
- *Fair-rotation* - picks what you've chosen least over time, using saved history.

**Captain** - pick two football captains from a squad.
- Whoever's captained least is most likely to be picked.
- Last week's captains can be excluded from this session.
- Goalkeepers pair with goalkeepers.
- Coin flip decides who picks teams first.

## Why this exists

There are three different problems people call "random":

1. *"Just pick one."* You want a single unbiased choice. The Equal mode.
2. *"Some options should count more."* You want a weighted draw. The Manual mode.
3. *"The same things keep getting picked."* This isn't a problem with randomness, it's a problem with statelessness. A truly random picker has no memory, so by chance it'll repeat itself. Fair-rotation gives the picker memory: it favours whatever you've chosen least over time, so the long-run distribution is balanced.

LLMs are good at conversation but a poor fit for any of these. They aren't random number generators - they pattern-match. They have no persistent memory of past picks. Tombola is a small purpose-built tool for the three specific things you'd otherwise (badly) use an LLM for.

## How to use it

**General mode:**
1. Type or paste your options, one per line.
2. Pick how to weight them - equal, manual, or fair-rotation.
3. Hit pick.

For fair-rotation, save your list with a name first so Tombola can remember picks across sessions. Each pick after that gives you the option to log it (or not - sometimes you pick and then change your mind).

**Captain mode:**
1. Paste today's squad, one name per line. Tag goalkeepers as `name (gk)`.
2. The eligibility panel shows current weights and who's blocked.
3. Hit pick captains. Flip the coin if you want. Log the session.

The session log feeds future weight calculations.

## Privacy & data

If you're not signed in, Tombola works without saving anything - equal and manual picks are pure client-side.

If you sign in (magic link, no password), your saved lists and captain sessions are stored under your account. Nobody else can read your data, including other Tombola users.

## Limitations

- **Names need consistent spelling across sessions.** "Kabir" and "Kabeer" become two separate histories.
- **Captain attendance isn't tracked.** Weights are based on captaincy count across all logged sessions, not appearances. A player who attends less frequently accumulates captaincy slower, making them appear "overdue" when they may not be relative to actual attendance.
- **One device at a time.** If you have Tombola open on two devices, edits on one don't appear on the other until you refresh.
- **Captain log entries can't be edited.** To prevent accidental history corruption, logged sessions are append-only. Contact the maintainer to fix mistakes.

## Credits

Built by Samarth Goradia. Part of the tools section at [samarthgoradia.com](https://samarthgoradia.com).