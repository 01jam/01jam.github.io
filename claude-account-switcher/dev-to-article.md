---
title: Switching between Claude Code accounts without the sign-out dance
published: false
description: A small Tauri app that keeps several Claude Code logins side by side, shows what each one has left, and hands over to the next when a limit is reached. Plus what it took to poll an undocumented endpoint politely.
tags: claude, linux, rust, opensource
cover_image: https://01jam.github.io/claude-account-switcher/img/window-light.webp
canonical_url:
---

> 🤖 **Written by an AI.** I gave Claude an outline of the points I wanted made
> and it wrote the prose. I have read it, checked it against the code, and I
> stand behind every claim in it — but the sentences are not mine, and it seemed
> wrong not to say so, given what the article is about.

I have three Claude accounts. One for client work, one personal, one that
belongs to a project. Claude Code has room for exactly one of them at a time,
and moving between them means `claude logout`, `claude`, a browser tab, a code
to paste, and — because I use the VSCode extension too — a small prayer that
both ends noticed.

I did that maybe six times a day for a week before writing
[Claude Account Switcher](https://01jam.github.io/claude-account-switcher/): a
desktop app for Linux and macOS that keeps every login saved and puts the one
you want back in place, from a window or from the icon in the tray.

![The app window: three saved accounts, each with a five-hour and a weekly usage meter, the active one highlighted](https://01jam.github.io/claude-account-switcher/img/window-light.webp)

The rest of this post is what I learned building it, which turned out to be
more interesting than the app itself.

## What a Claude Code login actually is

Two pieces, and that is the whole reason this is possible:

- **the OAuth tokens.** On Linux, `~/.claude/.credentials.json`. On macOS, an
  entry in the login keychain under the service `Claude Code-credentials`.
- **`~/.claude.json`**, of which only a handful of keys identify the account —
  `oauthAccount`, `userID`, and friends. Everything else in that file is your
  projects, your history, your preferences.

So a "profile" is a copy of the first and a surgical copy of the second.
Switching is putting them back. The app keeps each one under
`~/.config/claude-switch/profiles/<id>/` (`~/Library/Application Support/` on
macOS), writes atomically, and drops a snapshot in `backups/` on every switch.

Two things I got wrong before I got them right:

**The rest of `~/.claude.json` must survive.** The first version wrote the file
whole, from the profile. That is a fine way to lose every project entry you had.
Now only the account keys are merged in.

**Tokens rotate while you work.** Claude Code refreshes its access token as it
runs, so the copy you saved an hour ago is already stale. Right before every
switch, the app copies the *live* credentials back into the profile it is
leaving. Without that step, you save a token that has already moved on and the
account looks broken the next time you come back to it.

Nice side effect of working at this level: one switch covers both the CLI and
the VSCode extension, because they read the same two things.

## Showing what each account has left

Claude Code has a `/usage` command, and it gets its numbers from
`GET https://api.anthropic.com/api/oauth/usage`, called with your OAuth token.
It answers with two rolling windows — the five-hour session and the week — as
utilisation percentages plus reset times.

That is not a public API. It is undocumented, its shape can change without
notice, and it is rate limited in a way nobody has written down. Every field is
therefore optional in the client: if one is missing, the meter shows `—` and
nothing downstream fires.

![One account card: the five-hour session at 72% with a threshold marker at 90%, and the week at 38% with a threshold at 85%](https://01jam.github.io/claude-account-switcher/img/card-light.webp)

Each meter carries a draggable marker: the threshold past which the account
counts as spent. Turn auto-switch on and the app moves to the next account in
list order as soon as either counter crosses its own line. Default is 100%, so
out of the box nothing happens until you genuinely hit the limit.

## The part I actually want to talk about: asking politely

The first version polled every account once a minute. Sixty requests an hour per
account. It earned exactly the `429`s you would predict, and then it did the
worst possible thing with them: it blanked every meter and stalled the
auto-switch, right at the moment the user needed to know which account had room.

Fixing that properly needed a number nobody publishes. I did not want to
rediscover it by brute force against someone else's endpoint, and it turns out I
did not have to — [realiti4/claude-swap](https://github.com/realiti4/claude-swap)
had already probed it deliberately and written down the method, the dates, and
the result in a file called `poll_policy.py`. What that work found:

- roughly **28–30 requests per hour per identity**
- over a **trailing 60-minute window**, not a bucket that refills — so a burst
  saturates for a full hour, and going quiet does not hand the headroom back
  early

The constants in my `pace.rs` are theirs. Credit where it is due, and if you are
building something that touches this endpoint, go and read their notes rather
than measuring it again.

From there each account gets its own schedule:

| when | interval |
|---|---|
| the floor, and while a reading still counts as fresh | 3 min (~20/hour against a cap near 30) |
| the active account, once nothing is moving | decays to 5 min |
| another saved account, idle | decays to 10 min |
| an account already spent | 10 min — slow, never abandoned: a grant can free it early |
| the active account within 15 points of its threshold **and** actually moving | 1 min |
| after a refusal | 6 min floor, backing off ×1.5 toward 30 min while they recur |

A few decisions inside that table that took longer than they look:

**The one-minute case is bounded by construction, not by a timer.** It only
applies when the active account is close to its threshold *and* its numbers
changed since last time. Either the threshold gets crossed and the auto-switch
moves away, or the movement stops and the next plan decays back to the floor.
There is no state to reset and nothing to leak.

**Backoff is multiplicative because the budget is shared and invisible.** Your
other laptop is watching the same account, neither process can see the other, and
the endpoint reports no remaining count. That is the situation TCP is in, and it
is why TCP backs off the way it does. Every interval also carries ±10% of jitter
so two machines drift apart instead of arriving together.

**A refusal is about one account, not all of them.** The endpoint answers for the
token it was handed. A `429` almost always means *that* account is at its limit —
which is precisely the account the user is about to switch away from. Pausing the
others would blank the meters of the accounts they are switching *to*.

**`Retry-After` is a suggestion here.** The endpoint answers refusals with a
stock `retry-after: 3600` and is then perfectly willing twenty minutes later. The
cooldown is capped at 15 minutes, because past that the numbers are too old for
the auto-switch to act on anyway — so 15 minutes is as long as it is worth
sitting blind before spending one request to find out.

**The cache lives on disk, cooldowns included.** Held only in memory, every
restart began blind and re-asked for everything at once — which is a burst, and a
burst is the thing that saturates the hour.

**Old numbers are shown, and labelled.** After a refusal the app keeps displaying
the last readings it has rather than emptying the bars. But once a reading passes
five minutes, the card says how old it is — "numbers from 3h ago" — and so does
the tray menu, where a frozen number is easiest to mistake for a live one. Kept
numbers that look freshly loaded are exactly how a spent account appears to have
room left.

**And nothing acts on a reading older than 15 minutes.** Showing an old number is
fine. Rotating accounts on one is not: a five-hour window that has since reset
still reads as full in the cache, and switching on that is a switch nobody asked
for.

## Two features that fell out of the meters

**Start on the freest account.** At launch, optionally, go to the account with
the most week left. "Most" is a ratio, not a percentage:

```
room per day = (100 − week used) / days until the week resets
```

Twenty points to make six days last (3.3 a day) is a tighter week than seventy
points for one day (70), and the second account wins. Below a day the divisor
stops falling, because a window resetting in ten minutes is not a reason to start
on an account with nothing left in it *right now*.

**Renewing the tokens.** Claude Code owns the login *flow*, but not the renewal:
the refresh token it stores was issued to its own public client, so the same
grant works from the app and produces exactly the credentials the CLI would have
written. This matters because usage for a non-active account is read with its
stored token, and a stored token that has expired means two blank meters on the
account you might want to switch to.

The interesting half is doing it without fighting the CLI. Claude Code takes a
lock — `proper-lockfile` semantics on `~/.claude/.storage-write.lock`, a
directory whose `mkdir` decides who holds it — before writing its login. So the
app takes the same lock, and re-reads the file *inside* it: if a session renewed
first, the app stands down rather than overwriting. A lock whose mtime has not
moved for 15 seconds is treated as abandoned and cleared, which is also the cure
for Claude Code's own

> Failed to refresh OAuth token: another Claude Code process is refreshing it or
> exited mid-refresh

left behind by a session that died mid-write. Renewing a token, unlike switching
accounts, is safe while Claude Code is running.

## What it deliberately does not do

**Claude Desktop.** It is an Electron app that authenticates like a browser: the
session is a `sessionKey` cookie inside a Chromium profile in
`~/.config/Claude/`. Nothing in common with the files above. Covering it would
mean closing Desktop on every switch and chasing its updates — so it stays on
whichever account you signed it into. A choice, not a bug.

**Keyring storage.** The tokens sit in the clear with `0600` permissions, exactly
as Claude Code stores them itself. On macOS the live tokens are in the keychain,
but the app's own copies are still files. This is not a secret manager and it
does not pretend to be one.

**Switching while Claude Code is open.** The credentials go in under the CLI's
lock, but `~/.claude.json` is not covered by it, and a running session can
rewrite the account keys there and quietly undo the switch. Close it first.

## Odds and ends

The window draws its own title bar — no system decorations — which on Wayland
meant discovering that starting a drag on `mousedown` eats the `click` that
should follow, so buttons on the bar stop working. It starts on *movement*
instead: a press that stays put is still a click.

The app also finds its own updates: every six hours it asks GitHub for the newest
release, and a newer tag puts a dot on the gear and an offer at the top of
Settings. Installing a `.deb` or `.rpm` hands it to the system package manager
under `pkexec`, so polkit takes the password and the app never sees it. The first
attempt handed the file to the desktop instead, which fails exactly where it
matters: on Ubuntu the default handler for a `.deb` is the Snap Store, which will
not install a local package at all.

![The Settings dialog, with an available update at the top and four switches below it](https://01jam.github.io/claude-account-switcher/img/settings-light.webp)

## Get it

Linux `.deb`, `.rpm` and AppImage, plus a universal macOS `.dmg`, on the
[releases page](https://github.com/01jam/claude-account-switcher/releases).
Source is MIT: [01jam/claude-account-switcher](https://github.com/01jam/claude-account-switcher).
The macOS builds are unsigned — there is no Apple Developer certificate behind
this project — so the first launch needs a one-time allow under System Settings →
Privacy & Security.

One more disclosure, in the spirit of the one at the top: the app itself is
vibe coded. Not one line of it was typed by hand; it was prompted into existence
with Claude Code and reviewed mostly by using it. It is small and self-contained
enough that building it by hand would not have bought anything. Whether that
bothers you is a fair thing to decide before installing something that handles
your logins — the source is right there.

It is an independent tool, not affiliated with or endorsed by Anthropic, and it
reads an undocumented endpoint that may change or stop working without notice.
