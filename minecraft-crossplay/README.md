# Reliable Minecraft crossplay egg

One Paper server that Java and Bedrock players share. It is built for a Java player who connects over
the internet and an Xbox player on the same LAN as the server, with as little to babysit as possible.

## How players reach the server

- **Java (internet):** `mc.nrgserver.me` resolves to the home IP through a DNS-only record that
  ddclient keeps current. The router forwards TCP 25565 to the Docker host.
- **Xbox (LAN):** the console sees the server in its **Friends** tab because MCXboxBroadcast publishes
  it as a joinable session from a bot Xbox account. When the console joins, MCXboxBroadcast relays the
  connection request to Geyser inside the container, and Geyser answers with **Bedrock LAN Address**
  and **Bedrock Port**. Gameplay then goes straight over the LAN, and Floodgate lets the console in
  without a Java account.

Consoles cannot add custom servers, and Bedrock 26.60 (2026-10-27) removed the RakNet transport that
most workarounds relied on. Geyser does not answer the new NetherNet LAN discovery, and Docker's bridge
network would not deliver LAN broadcasts to the game container anyway. BedrockConnect supports NetherNet
since 1.72, but it needs its own server plus DNS overrides on the console's network. MCXboxBroadcast
needs neither, because it reaches the console through Xbox Live from inside the container.

## What every start does

1. Checks the settings and stops with a message that names any setting it cannot use.
2. With **Auto Update** on, refreshes Paper (stable builds of exactly **Minecraft Version**), Geyser,
   Floodgate, ViaVersion and MCXboxBroadcast from their official download APIs. Each jar must match
   the publisher's SHA-256 and pass a zip test before it atomically replaces the current one, and the
   replaced jar is kept in `.crossplay/previous/`. Any failure keeps the current jar. The only fatal
   case is a first start that cannot download Paper.
3. Refuses to start if either **Minecraft Version** or the Paper jar on disk is older than the newest
   version this world has run on, which it records in `.crossplay/minecraft-version`.
4. Starts Paper with a 1 GiB initial heap (capped at the maximum for smaller memory limits) and a
   maximum heap of 80% of the memory limit. Pre-touching is disabled so an idle server does not
   allocate its maximum heap upfront. Geyser uses built-in NetherNet signaling over TCP and gameplay
   over UDP, both on **Bedrock Port**, with clients told to use **Bedrock LAN Address** because the
   container's own address is private.

Calagopus also rewrites these settings before every start, so edits to them do not stick:

| File | Setting | Why |
| --- | --- | --- |
| `server.properties` | `online-mode=true`, `white-list=true`, `enforce-whitelist=true` | The Java port is on the internet. |
| `server.properties` | `enforce-secure-profile=false` | Bedrock players cannot sign chat messages. |
| MCXboxBroadcast `config.yml` | `friend-sync.expiry.enabled: false` | The default unfriends anyone who has not joined for 15 days. |

Leave MCXboxBroadcast's `session.remote-address` and `session.remote-port` on `auto`: the extension
then relays joins to Geyser inside the container, while any other address sends them out through
Docker and back in.

## Before merging

This runbook uses the Calagopus/Wings VM at **192.168.0.11** and the Java hostname
**mc.nrgserver.me**. This README is copied to the public egg mirror when merged.

Already configured:

- The VM has a DHCP reservation at **192.168.0.11**.
- The wildcard DNS record for **\*.nrgserver.me** points to the public home IP. It covers
  **mc.nrgserver.me** without a separate A record.

1. Confirm the wildcard is **DNS only** and ddclient keeps it current. If it is proxied for other
   services, leave it as-is and create a separate DNS-only `mc` A record instead; only that case
   requires adding `mc.nrgserver.me` to ddclient as well.
2. Create a dedicated Microsoft account for MCXboxBroadcast, then sign in at xbox.com once to
   create its Xbox profile and gamertag. The bot does not need to own Minecraft. An alternate account
   is recommended because MCXboxBroadcast's client emulation may risk an account ban.
3. Record the Java player's Minecraft Java username and the Xbox player's gamertag. These are for
   the whitelist; neither player's password or email is needed.

## Merge and publish

1. Merge the Minecraft crossplay PR into `main`.
2. In GitHub Actions, wait for **Publish Calagopus Eggs** on that merge to succeed.
3. Confirm `minecraft-crossplay/egg-minecraft-crossplay.json` exists in
   [the public mirror](https://github.com/nrgbistro/calagopus-eggs/tree/main/minecraft-crossplay).

Merging publishes the egg; the following steps create and configure the server in Calagopus.

## After merging

### 1. Import the egg

Merging the generated `eggs/minecraft-crossplay/dist/` files to `main` publishes them to `https://github.com/nrgbistro/calagopus-eggs.git`. In
**Admin → Egg Repositories**, add that URL if it is not there yet, click **Sync**, open the repository's
**Eggs** tab, and install **Minecraft Crossplay (Reliable)** into a nest. A later change to this egg
reaches the panel the same way: sync again, then use **Update from Repository** on the installed egg.
You can also import `egg-minecraft-crossplay.json` by hand through **Admin → Nests**.

### 2. Create the server

- **Image:** Java 25 (the only one offered; Paper 26.3 needs it).
- **Memory:** keep the 6144 MiB limit for this server's maximum of three players. It starts with a
  1024 MiB heap and can grow to 4915 MiB as needed, with additional memory used outside the heap.
  The JVM may retain expanded heap memory after players leave. The start refuses limits below
  1024 MiB.
- **Allocations:** the primary allocation is the Java port (25565). Add a second allocation for
  Bedrock (19132). Calagopus publishes every allocation on both TCP and UDP, which NetherNet needs.
- **Minecraft Version:** `26.3`.
- **Bedrock Port:** `19132` (the second allocation's port).
- **Bedrock LAN Address:** `192.168.0.11`.
- **Auto Update:** on (`1`).
- **Xbox Friends Broadcast:** on (`1`).

### 3. First start

1. Accept the Minecraft EULA when the console asks.
2. MCXboxBroadcast prints a code to enter at `https://www.microsoft.com/link`. Sign in with the bot's
   Microsoft account: the project recommends an alternate account because its client emulation may
   break Xbox terms. The sign-in is cached in `plugins/Geyser-Spigot/extensions/mcxboxbroadcast/`.
3. Allow the two players from the console:

   ```text
   whitelist add <JavaName>
   fwhitelist add <XboxGamertag>
   ```

   Replace the placeholders with the actual names; omit a leading `/` in the panel console. Quote
   an Xbox gamertag containing spaces, for example `fwhitelist add "Xbox Gamer Tag"`. Floodgate shows
   Bedrock players with a `.` prefix in game; use the Xbox gamertag in `fwhitelist`.

### 4. Router and DNS for the Java player

1. Add this router port-forward rule:

   | Setting | Value |
   | --- | --- |
   | Protocol | TCP |
   | External port | 25565 |
   | Destination LAN IP | 192.168.0.11 |
   | Internal port | 25565 |

   Do not forward port **19132**: the Xbox reaches it over the LAN. If a VM firewall is enabled,
   allow TCP 25565 for Java and TCP/UDP 19132 from the Xbox's LAN.
2. Use the existing wildcard DNS record. With a DNS-only wildcard already maintained by ddclient,
   no new DNS record, ddclient edit or configuration PR is needed. Confirm the hostname resolves to
   your current public home IPv4 address:

   ```sh
   dig +short mc.nrgserver.me A
   ```

   If it resolves to Cloudflare proxy addresses, the wildcard is proxied: use the separate DNS-only
   `mc` record described above. If it resolves to an old home IP, check the existing ddclient updates.
3. Have the Java player test from outside your home network: in Minecraft Java, add
   **mc.nrgserver.me** as the server address and join. Port 25565 is the default, so no port suffix
   is needed. Confirm they can enter the world.

### 5. The Xbox player

1. On the Xbox, add the bot account's gamertag as a friend. The bot accepts within about a minute.
2. In Minecraft, open **Play → Friends**. The server is listed under the joinable friends.
3. Join. If the console asks whether to trust the server, accept it.
4. Confirm the Xbox and Java players can see each other in the same world. A real Xbox join is the
   end-to-end check for the bot, Geyser and LAN allocations.

Keep `plugins/Geyser-Spigot/nethernet/identity.pem`: it is the server's NetherNet identity, and a new
one makes a console that trusted the old one ask again. The Xbox account's privacy and online-safety
settings must allow joining multiplayer and cross-network games, which child accounts often block.

### 6. Schedules

Add these in **Schedules** so the server keeps itself current and recoverable. Turn on **Ignore
Failure** for every step: by default a failed step ends the schedule, which could leave world saving
switched off.

- **Daily backup** at a quiet hour, such as 04:00 in the panel's timezone, keeping several days:
  1. Send command `save-off`.
  2. Send command `save-all flush`.
  3. Sleep 15 seconds. A console-line wait can miss a save that finishes before the wait starts,
     and its default timeout is 5 seconds.
  4. Create a backup with **Run in Foreground** on. Otherwise the next step runs while the archive
     is still being written.
  5. Send command `save-on`.
- **Daily restart** after the backup has time to finish, such as 05:00 in the panel's timezone:
  send command `say Restarting in 1 minute`, sleep
  60 seconds, send the restart power action. Each start installs the newest verified Geyser and
  MCXboxBroadcast, so the Xbox can join again within a day of both supporting a new Bedrock
  version, without anyone touching the server. Because the first step ignores failure, this also
  starts a server that was stopped, so pause the schedule during planned maintenance.

Run the backup schedule manually once and confirm the backup completes and `save-on` runs afterward.
Then run the restart schedule and confirm the server returns and both players can reconnect. Adjust
its time if the backup would overlap it. Leave crash detection on so Calagopus restarts the server
after a crash.

## Operating it

### After a Bedrock update

Consoles update Minecraft automatically, and each MCXboxBroadcast build only lets in consoles on the
Bedrock version it was built for. After an update, the Xbox player cannot join until Geyser and
MCXboxBroadcast have both published builds for the new version, which the daily restart installs.
The Java player is not affected.

### Upgrading Minecraft

1. Take a backup.
2. Change **Minecraft Version** after Paper marks the new version stable, then restart. Until then,
   the start keeps the current version and logs a warning.

Lowering the version is refused, because a world cannot be downgraded. Restore a backup from the
older version instead. ViaVersion lets an updated Java client join before the server is upgraded.
Never delete `.crossplay/`: the version recorded there is what the downgrade check compares against.

### Rolling back a jar

Turn **Auto Update** off, copy the jar from `.crossplay/previous/` over its managed path, and start.
Turn Auto Update back on once upstream has fixed the problem.

Paper can only be rolled back to a build of the same Minecraft version. Right after an upgrade, the
saved `server.jar` is still the old version's Paper, and the start refuses it rather than open the
upgraded world with it. To go back across versions, restore a backup taken before the upgrade.

| Component | Managed path |
| --- | --- |
| Paper | `server.jar` |
| Geyser | `plugins/Geyser-Spigot.jar` |
| Floodgate | `plugins/floodgate-spigot.jar` |
| ViaVersion | `plugins/ViaVersion.jar` |
| MCXboxBroadcast | `plugins/Geyser-Spigot/extensions/MCXboxBroadcastExtension.jar` |

### Updating the launcher

The installer writes `start-minecraft.sh`, so a changed egg reaches an existing server through
**Update from Repository** on the egg, then **Reinstall** on the server. Leave the option to delete all
files unchecked: the installer only rewrites the launcher and creates missing folders, and the world
and configs stay untouched.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| The server is missing from the Xbox Friends tab | The bot is not a friend, needs a new sign-in, or **Xbox Friends Broadcast** is off | Look for MCXboxBroadcast errors or a new sign-in code in the console, and confirm the friendship in the Xbox app. |
| The Xbox finds the server but cannot join | Wrong **Bedrock LAN Address**, or no allocation for **Bedrock Port** | The console must log `Advertised addresses set from system property` with `192.168.0.11`, and the second allocation must equal Bedrock Port. MCXboxBroadcast logs `Relayed bedrock client` for each join it passes to Geyser, or `Failed to relay offer` with the reason. |
| The Xbox reports an outdated server | The console updated before Geyser or MCXboxBroadcast supported its version | Restart to install the newest builds. If either project has no build for the version yet, wait for one. |
| Every join asks to trust the server again | `identity.pem` was deleted | Restore it from a backup, or accept the prompt once more. |
| Only the Java player cannot connect | Stale DNS record or missing port forward | `mc.nrgserver.me` must resolve to the current home IP, and TCP 25565 must be forwarded to `192.168.0.11:25565`. |
| The start stops with `[crossplay] ERROR` | A setting is invalid, or the first start had no internet access | The message names the setting to fix. |
| ViaVersion warns that it has no compatible versions | The server already runs the newest Minecraft release, so there is nothing newer to translate | Nothing to fix. It starts translating when a newer Java release ships. |

## Developing this egg

In the private nrgserver repository, edit these committed sources:

| File | Purpose |
| --- | --- |
| `eggs/minecraft-crossplay/src/launcher.sh` | Downloads, version checks and Java startup |
| `eggs/minecraft-crossplay/src/install.sh` | Installer template that writes the launcher |
| `eggs/minecraft-crossplay/src/egg-minecraft-crossplay.json` | Egg metadata, variables and panel-managed config |
| `eggs/minecraft-crossplay/README.md` | This runbook |

From the repository root, regenerate and verify:

```sh
python3 scripts/build-eggs.py
python3 scripts/build-eggs.py --check
python3 -m unittest discover -s scripts/tests -p 'test_*egg*.py'
```

Commit the sources and regenerated `dist/` files together. CI rejects a distribution that does not
match its sources. Only `dist/` is published, under the existing `minecraft-crossplay/` path; the
sources and build tools stay in the private repository. Do not edit generated files or the public
mirror directly. Apply launcher changes with the sync, update and reinstall procedure above.

## Upstream references

- [Preparing your Geyser server for NetherNet](https://geysermc.org/blog/preparing-for-nethernet/)
- [Geyser system properties](https://geysermc.org/wiki/geyser/geyser-command-line-arguments-and-system-properties/)
- [Geyser and secure chat](https://geysermc.org/wiki/geyser/secure-chat/)
- [MCXboxBroadcast](https://github.com/MCXboxBroadcast/Broadcaster)
- [Paper downloads service](https://docs.papermc.io/misc/downloads-service/)
- [Calagopus egg repositories](https://calagopus.com/docs/panel/next-steps/egg-repos)
