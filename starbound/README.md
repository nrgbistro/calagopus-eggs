# Reliable Starbound Workshop egg

This custom egg fixes two lifecycle problems in the upstream Starbound egg:

- Steam installs and updates the Linux server in `linux/`, so the egg keeps that layout and
  launches `linux/starbound_server` through a wrapper instead of moving the binaries.
- Steam controls whether subscribed Workshop content is downloaded. The `WORKSHOP` variable in
  this egg controls whether already-downloaded packages are linked into `mods/` and loaded.

The egg remains self-contained and uses the upstream `ghcr.io/ptero-eggs/games:source` runtime
image. It is deliberately named `Starbound (Reliable Workshop)` so updating the official egg
cannot overwrite it.

## Repository sync or manual import

Calagopus can sync a public HTTPS Git repository. In **Admin → Egg Repositories**, create a
repository using its Git URL, click **Sync**, open its **Eggs** tab, and install this egg into the
desired nest. For later changes, sync the repository again and use **Update from Repository** on
the installed egg. Synchronization is explicit rather than automatic on every Git push.

Changes merged to `main` under `stacks/calagopus/eggs/` are published by
`.github/workflows/publish-calagopus-eggs.yml` to the public mirror:

```text
https://github.com/nrgbistro/calagopus-eggs.git
```

This repository is currently private. Calagopus's current repository form accepts only an HTTPS
Git URL and has no separate credential or deploy-key field. Do not embed a GitHub token in the URL:
the complete URL is stored and returned as ordinary repository metadata. The safe choices are:

- Continue importing `egg-starbound-reliable.json` through **Admin → Nests**.
- Publish `stacks/calagopus/eggs/` to a dedicated public, read-only egg repository and register
  that repository with Calagopus.
- Make this entire repository public only after separately reviewing everything it contains.

A private, network-restricted Git mirror that Calagopus can read without URL credentials also
works. A dedicated public egg repository is usually the simplest synchronization option.

## First installation and email Steam Guard

Disabling the Steam mobile authenticator does not necessarily remove Steam Guard. Steam commonly
falls back to emailing a code for a login from an unrecognized machine. A Calagopus installer
cannot receive a variable changed after its container has started, so email authentication is an
expected two-pass installation:

1. Create a server using **Starbound (Reliable Workshop)** and set the admin-only Steam username
   and password. The account must own Starbound and have the desired Workshop subscriptions.
2. Leave **Steam Auth** blank, **Load Downloaded Workshop Content** enabled, and
   **Auto Update Server** disabled. Click **Install**.
3. If Steam recognizes the machine, installation continues normally. Otherwise, the login probe
   triggers the email and installation stops with a `stage 1 is complete` message.
4. Copy the newest five-character email code into **Steam Auth** and immediately click
   **Reinstall**. This second pass is expected, not a sign that the egg lost the first attempt.
5. After installation succeeds, clear **Steam Auth** and start the server.

The installer keeps Steam's machine state under the server root, writes the last attempt to
`steamcmd-install.log`, and leaves `.steam-auth-required` after an authentication failure. It
removes the marker only after Steam reports success and both required Starbound assets exist. If
the code expires, clear **Steam Auth**, run the blank-code pass again to request a new email, then
repeat the second pass. Changing Steam accounts does not remove this new-machine challenge.

On its first start, the launcher prints the number of Workshop items and `.pak` files it linked.
For the mod set used when this egg was created, the expected result is 96 items and 97 packages.
The links are named `workshop_<item-id>_<index>.pak` and point into
`steamapps/workshop/content/211820/`; they do not duplicate the downloaded data.

Files manually placed in `mods/` are preserved. The `workshop_` filename prefix is reserved for
links managed by this egg. If a regular file occupies a managed link name, startup stops instead
of overwriting it. Symlinks with the upstream egg's legacy `<item-id>_<index>.pak` naming are
removed during reconciliation, while ordinary files with those names are preserved.

## Refreshing the game and Workshop subscriptions

Routine starts default to no Steam update, avoiding startup delays and expired Steam Guard codes.
The stored machine authorization will normally make later updates non-interactive. To refresh
deliberately:

1. Stop the server.
2. Leave **Steam Auth** blank and enable **Auto Update Server**.
3. Start the server and confirm SteamCMD succeeds. The launcher then rebuilds all managed links
   from the updated Workshop directory.
4. If Steam emails another new-machine code, stop the server, enter the newest code, and start it
   again immediately with **Auto Update Server** still enabled.
5. After a successful update, stop the server, disable **Auto Update Server**, clear the one-time
   code, and start normally.

Disabling **Load Downloaded Workshop Content** removes only managed links and starts a vanilla
server. It does not delete Workshop downloads because Steam, rather than the egg, owns that cache.

## Verification and troubleshooting

A healthy modded startup includes lines similar to:

```text
[starbound-launcher] linked 97 Workshop packages from 96 items
[Info] Root: Detected asset source named 'FrackinUniverse' at '../mods/workshop_...pak'
```

The client and server asset digests must match. At the time this egg was created, the local client
digest was:

```text
8352b636692d4a6a1b7ab6f4724bce6ce39cb4f2e9681afb80cdcba8c690023c
```

Use the client's current digest after any mod update. The vanilla digest
`531f0b8469832a566d23a0b9870ed717c63291164883670d0ee17086a100d3cd` means no Workshop packages
were loaded.

The egg enforces `allowAssetsMismatch=false` and `checkAssetsDigest=true` when Calagopus prepares
the server configuration. This intentionally overrides the permissive values in the upstream
example config so a partial or stale Workshop set cannot connect silently.

The launcher intentionally refuses to start when Workshop loading is enabled but the content
directory is missing or contains no `.pak` files. Check the SteamCMD output, account subscriptions,
and `steamapps/workshop/content/211820/` rather than enabling Starbound's asset-mismatch override.

## Upstream references

- [Published Starbound egg](https://eggs.pterodactyl.io/egg/games-starbound)
- [Pterodactyl source image entrypoint](https://github.com/pterodactyl/yolks/blob/master/games/source/entrypoint.sh)
- [Calagopus egg repository documentation](https://calagopus.com/docs/panel/next-steps/egg-repos)
- [Steam Support: Steam Guard](https://help.steampowered.com/en/faqs/view/06B0-26E6-2CF8-254C)
