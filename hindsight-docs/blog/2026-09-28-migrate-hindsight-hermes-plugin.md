---
title: "How to Migrate Your Hindsight Memory in Hermes"
authors: [benfrank241]
slug: "2026/09/28/migrate-hindsight-hermes-plugin"
date: 2026-09-28T12:00
tags: [hindsight, hermes, nous-research, plugins, memory-provider, migration, tutorial]
description: "Hermes moved Hindsight out of its core tree and into the plugin catalog. A step by step guide: what happens automatically, how to install it fresh, the three ways to control which version you run, and fixes for every error you might hit."
image: /img/blog/migrate-hindsight-hermes.png
hide_table_of_contents: true
---

![Migrating Hindsight in Hermes: the plugin now installs from the Hermes catalog](/img/blog/migrate-hindsight-hermes.png)

Hindsight used to ship inside [Hermes Agent](https://github.com/NousResearch/hermes-agent) as a bundled memory provider at `plugins/memory/hindsight/`. As of 23 September 2026 it does not. Nous Research is moving every memory provider out of the Hermes core tree, and Hindsight was the first one out.

This is the practical guide: what happens on its own, what you need to type, and what to do when something goes wrong.

**If you already use Hindsight with Hermes, the short answer is that you do not have to do anything.** The rest of this is detail.

<!-- truncate -->

## TL;DR

- **Existing users:** nothing to do. Hermes migrates you on `hermes update` or on first agent start.
- **New users:** `hermes plugins install hindsight`, then `hermes memory setup`.
- **Your data does not move.** The plugin is the client, not the store. Bank, API key and config file are untouched.
- **`hermes update` does not update the plugin.** `hermes plugins update hindsight` does.
- **Three ways to control your version:** follow the catalog pin, freeze a commit with `--ref`, or track our `main` branch.
- Jump to [troubleshooting](#troubleshooting) if you landed here from an error message.

## What changed

Memory providers are moving out of the `hermes-agent` repository into their maintainers' own repositories, published through the Hermes plugin catalog. From Nous' documentation:

> Memory providers are moving out of the Hermes tree into their maintainers' own repositories, published through the plugin catalog — Hindsight is the first.

Practically, that means three things:

1. The bundled `plugins/memory/hindsight/` directory is gone from Hermes.
2. The plugin now lives in [vectorize-io/hindsight](https://github.com/vectorize-io/hindsight/tree/main/hindsight-integrations/hermes) and is maintained by the Hindsight team.
3. Hermes installs it from the catalog, pinned to a commit that Nous has reviewed.

Nothing about the memory itself changed. No data was migrated, because the plugin is the code that talks to Hindsight rather than the place your memories live.

## Why the move is worth having

A migration notice is not usually good news, so it is worth saying what you get out of this one.

When the provider shipped inside Hermes, its release cycle was Hermes' release cycle. A bug in the Hindsight connector had to be fixed by people whose project is an agent, not a memory system, and a fix only reached you when Hermes next shipped. That is a slow path for something that sits in the hot loop of every turn your agent takes.

Now the people who build the memory system maintain the connector to it. Issues go to our repository, we fix them there, and they reach you when the catalog pin moves rather than when an unrelated release happens. You also get a choice you did not have before: stay on the reviewed pin, freeze a specific commit, or track our development branch if you need a fix immediately. Those three options are covered below.

The trade is that updates are no longer automatic in the way they used to be. That is the one real behaviour change, and it is the thing most likely to surprise you a month from now.

## Migrating an existing install

### Step 1: update, or just start an agent

```bash
hermes update
```

You can also do nothing and simply start an agent as usual. Either path triggers the same migration: Hermes sees `memory.provider: hindsight` in your config, finds no provider on disk, resolves the name against the catalog, and installs it.

You will see one line:

```
✓ Memory provider 'hindsight' moved out of core — installed its plugin from the
  catalog (your memory.hindsight settings and data are unchanged).
```

Two things to expect while it runs. The update is **interactive**: when it finds the plugin's Python dependencies it prints `hindsight declares Python dependencies` and waits for a yes, so an unattended run will sit at that prompt rather than finish. And if you have local modifications in the Hermes tree, it stashes them first and asks whether to restore, printing a `git stash apply <sha>` worth keeping.

### Step 2: verify

```bash
hermes plugins list
cat ~/.hermes/plugins/.install-metadata.json
```

The metadata file records exactly what you got: the catalog entry, the source repo, and the resolved commit.

```json
{
  "hindsight": {
    "catalog": { "name": "hindsight", "pin": "176f8c2d…", "tier": "community" },
    "pinned": true,
    "revision": "176f8c2d…",
    "source": "https://github.com/vectorize-io/hindsight#hindsight-integrations/hermes"
  }
}
```

Your `config.yaml` will also have picked up `plugins.enabled: [hindsight]`.

### Step 3: confirm nothing moved

These should all be exactly as you left them:

| Thing | Where |
|---|---|
| Provider setting | `memory.provider: hindsight` in `config.yaml` |
| Plugin config | `~/.hermes/hindsight/config.json` |
| API key | `~/.hermes/.env` |
| Memory bank | Wherever it was: Cloud, a local daemon, or your own instance |
| Tools | `hindsight_retain`, `hindsight_recall`, `hindsight_reflect` |

The quickest real check is to ask your agent something only memory can answer and watch the recall land.

### If automatic installation is disabled

Automatic install depends on `security.allow_lazy_installs`, which is on by default. If you turned it off, Hermes will not install anything for you. It logs a one-line instruction instead, and you run:

```bash
hermes plugins install hindsight
```

## Installing fresh

```bash
hermes plugins install hindsight
hermes memory setup           # select "hindsight"
```

Hindsight is in the catalog, so the name is all you need. The setup wizard installs dependencies into the Hermes venv, walks you through configuration, and offers to seed the bank with a starter memory template.

**Do not skip `hermes memory setup`.** Installing the plugin does not switch it on. Hermes treats memory providers as `kind: exclusive`, and its plugin-enable gate deliberately skips them, so `hermes plugins enable` is not what activates a provider. What activates one is `memory.provider` in `config.yaml`, which setup writes for you.

Note that `plugins install` asks for confirmation before enabling. Pass `--enable` or `--no-enable` to skip the prompt in a script.

## Controlling which version you run

**Nothing updates the plugin on its own.** In particular `hermes update` does *not* move it: it reinstalls the plugin's Python dependencies and leaves the checkout where it is. If you came from the bundled provider, that is the habit to unlearn, because memory improvements used to arrive as a side effect of updating Hermes and no longer do.

### Option 1: follow the official pin (default)

What you get from `hermes plugins install hindsight`: the commit Nous reviewed and pinned.

```bash
hermes plugins update hindsight
```

`hermes plugins list` flags the plugin `update_available` once your installed commit differs from the catalog's. Re-running `plugins install` does **not** update it, and refuses with "already exists" — `plugins update` is the command that re-pins.

Two timing details. Our merges do not reach you until Nous bumps the pin, and Hermes re-fetches the published catalog at most once every six hours, so a fresh release can take that long to even appear as available.

### Option 2: freeze a specific commit

```bash
hermes plugins install vectorize-io/hindsight/hindsight-integrations/hermes \
  --force --ref <40-character-commit-sha>
```

Replace the whole placeholder, angle brackets included. `--ref` takes a full 40-character SHA and rejects tag names, so take the SHA from the [release notes](https://github.com/vectorize-io/hindsight/releases) rather than typing a version number. The copy button beside a commit in [the plugin's history](https://github.com/vectorize-io/hindsight/commits/main/hindsight-integrations/hermes) gives you the full SHA; the abbreviated one shown on screen is too short.

A `--ref` install is marked pinned, and `hermes plugins update hindsight` will deliberately refuse to move it. Install again with a new `--ref` when you want a different version.

### Option 3: track the latest development code

```bash
hermes plugins install vectorize-io/hindsight/hindsight-integrations/hermes
hermes plugins update hindsight
```

Installed from the source path rather than the catalog name, `plugins update` becomes a git pull of our `main` branch. Unreviewed by definition: you get whatever is there when you run it. Useful for picking up a fix before the pin moves, not what we would run in production.

## Troubleshooting

### `Timeout context manager should be used inside a task`

You have `hindsight-embed` 0.10.0, which breaks `local_embedded` mode outright. Its daemon probe cleared the calling thread's event loop, so the next client call failed. This affects anyone who installed between **14 and 21 September 2026**.

```bash
hermes update          # reinstalls dependencies, which is the fix here
```

Cloud and `local_external` users are unaffected.

### The provider reports "not available" in embedded mode

You installed the plugin but skipped `hermes memory setup`. `local_embedded` needs the `hindsight-all` package rather than just the client, and `pyproject.toml` deliberately does not declare it, because that would push the whole local ML stack onto cloud-mode users. The setup wizard installs it.

```bash
hermes memory setup
```

### I installed and enabled the plugin, but memory does nothing

`hermes plugins enable` does not activate a memory provider. Check `memory.provider` in your `config.yaml`, or just run `hermes memory setup`.

### `already exists` when I try to update

`plugins install` will not overwrite an installed plugin. Use `hermes plugins update hindsight`, or add `--force` if you are deliberately reinstalling at a different `--ref`.

### `plugins update` says nothing to do, but I know there is a newer version

Either you are pinned with `--ref`, in which case update refuses by design, or the catalog has not re-fetched yet. Hermes refreshes it at most once every six hours.

### `zsh: parse error near '\n'`

The `--ref` command spans two lines. Either keep the trailing `\` at the end of the first line, or join it into a single line. A `\` followed by anything other than a newline is a shell parse error. Also make sure you replaced the angle brackets, since `<` and `>` are redirection operators.

### `unrecognized arguments: --force`

Same cause: a stray `\` before `--force` got passed through as an escaped space. Put the whole command on one line.

### `hermes update` seems to hang

It is waiting for input. It prompts before preparing a plugin's Python dependencies, and again about restoring stashed local changes if your Hermes tree is dirty.

### `--ref` rejects the version I gave it

It takes a commit SHA, not a tag. `v1.1.0` will not work, and neither will an abbreviated seven or nine character SHA. You need all forty characters.

### I am not sure which version I am on

```bash
hermes plugins list
cat ~/.hermes/plugins/.install-metadata.json
```

The first shows the installed version and flags `update_available`. The second shows the resolved commit and whether you are pinned, which is what actually determines whether `plugins update` will move you.

### Nothing happened when I ran `hermes update`

If you are still on a Hermes build that predates the removal, the bundled provider is still present and wins the lookup, so there is nothing to migrate yet. The migration runs when you update past the point where Hermes drops its bundled copy.

### Recall fails with an HTTP error rather than a plugin error

If the plugin is installed and configured but recall returns an HTTP status such as 401, 402 or 403, that is Hindsight rejecting the request rather than anything to do with the migration. Check your API key in `~/.hermes/.env` and your account status. A quick way to separate the two: a read-only call that succeeds while recall fails points at credentials or billing, not at the plugin.

## Frequently asked questions

**Do I need to migrate manually?**
No. Hermes does it on `hermes update` or on first agent start, provided `security.allow_lazy_installs` is on, which is the default.

**Will I lose any memories?**
No. Nothing about your bank is read or written during the migration. The plugin is the client; your memories live in Hindsight Cloud, a local daemon, or your own instance, and none of those move.

**Does this affect the Hermes desktop app?**
It migrates the same way, on first agent start. Desktop users configure Hindsight entirely in Settings → Memory & Context and never touch the plugin layer. Note that desktop supports Cloud and Local External modes only, with no embedded mode.

**Can I keep using the bundled provider?**
Not once you update past the removal. While both existed the bundled copy won, because provider lookup goes bundled, then `~/.hermes/plugins/`, then project, then entry point, first hit wins.

**Where do I report a bug now?**
The [Hindsight repository](https://github.com/vectorize-io/hindsight/issues). That is the practical upside of the move: issues land with the team that maintains the memory system, and fixes ship on the next catalog pin rather than waiting on an unrelated release.

**Is there anything else that changed at the same time?**
One default worth knowing: `recall_types` now returns observations only, where it previously returned all three fact types. Restore the old behaviour with `"recall_types": "observation,world,experience"` in `~/.hermes/hindsight/config.json`.

## Learn more

- [Hermes Agent integration reference](https://hindsight.vectorize.io/sdks/integrations/hermes) for the full configuration surface
- [Hermes Desktop](https://hindsight.vectorize.io/sdks/integrations/hermes-desktop) for the settings-only path
- [Does Hindsight + Hermes = AGI?](https://hindsight.vectorize.io/blog/2026/09/25/hindsight-hermes-plugin-catalog) for the shorter, more opinionated version of this story
- [Give Every Hermes Bot Its Own Memory](https://hindsight.vectorize.io/blog/2026/08/18/hermes-bot-mode-memory) on per-bot bank isolation
